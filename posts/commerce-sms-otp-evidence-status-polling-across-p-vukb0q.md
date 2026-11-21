# Commerce SMS OTP Evidence: Status Polling Across Password Reset Expiry

A short-lived password-reset code creates an evidence problem: a delivery update can arrive after the application has stopped accepting the challenge. For a US/EU commerce app, retain the authorization decision separately from the delivery observation. TL;DR: choose managed SMS OTP send and verify, with resend and status polling, then connect the delivery record to application telemetry. Make compliance evidence the acceptance criterion. An HTTP success alone cannot establish that a reset was authorized.

This engineering note proposes an acceptance experiment, not a measured vendor benchmark. The simple candidate is to send a message and save the successful request. Reject that design for this workflow: it leaves support unable to distinguish a message submission from a valid reset. The chosen candidate keeps challenge validity, resend decisions, and delivery observations related but distinct. Its extra bookkeeping buys an explanation of what the application actually permitted.

## How should an SMS OTP API handle resend before reset expiry?

Use a five-minute expiry as an application policy for the experiment, not a claim about any provider's default. Consider a shopper resetting access before checkout. The application creates a challenge, requests an SMS, and accepts a submitted code only through its verification path while that challenge remains eligible. A later delivery observation belongs in the record even when the challenge has expired. It must never reopen authorization.

Expiry wins here.

Delivery evidence cannot renew an expired challenge.

The evidence record should explain the challenge creation time, the application's expiry deadline, each resend decision, the provider message identifier, the time a delivery status was observed, and the verification outcome. Those are proposed application record fields, not a vendor request schema. Keep codes, reset links, phone numbers, and authentication secrets out of general-purpose logs. Choose retention and access rules with the people responsible for the application's compliance obligations; a delivery API doesn't establish US or EU compliance by itself.

There is a subtle trap in the timeline. Polling tells you when your application learned something. It doesn't automatically tell you when the carrier event occurred. Preserve a source timestamp separately when its documented response supplies one, and otherwise leave it unknown. Subtracting the local observation time from challenge creation would measure your observation delay as well as delivery behavior. That is a poor basis for promising a shopper that the next message will arrive before expiry.

## Compare the evidence boundary before choosing the sender

Infrai combines a plain REST API with one key, one wallet, and one bill across SMS and observability, without an SDK to install or maintain. An HTTP-capable backend can hand a delivery result to application logs through the same interface. That fits a small application whose immediate requirement is a managed OTP loop with resend and pull-based status evidence. Polling introduces observation delay, so choose its interval deliberately instead of treating it as a real-time notification stream.

The trade-off is observation lag: polling is less immediate than webhooks. This approach is not suitable for a workflow that requires an immediate pushed delivery notification; evaluate Twilio Verify with Datadog against that requirement instead.

Public discovery makes the API self-describing: request JSON Schema and response schemas are available without an API key. That lets a reviewer check the ingestion contract before provisioning credentials. The same workflow uses one API key for both services, so credential rotation doesn't require coordinating two independently configured clients.

The alternatives deserve an explicit comparison.

The table identifies the integration boundary to evaluate, rather than asserting that any product confers compliance or has already passed this experiment.

| Candidate | Objective difference in the proposed stack | Evidence to require before selection |
| --- | --- | --- |
| Twilio Verify plus Datadog | Verification and operational evidence span two products. | Trace a verification result into the application record and monitoring system without losing correlation. |
| Vonage Verify | A verification-focused alternative to evaluate for the managed-code boundary. | Check documented resend behavior, verification semantics, and delivery evidence for intended destinations. |
| Amazon SNS | A messaging alternative to evaluate separately from the application's challenge engine. | Establish which component owns code creation, expiry, attempt limits, and verification before calling the design complete. |

Twilio plus Datadog would mean two service signups and two sets of service credentials. The application would supply the glue: capture the verification or message identifier, carry its own challenge reference, translate observations into telemetry, and arrange ingestion. That is reasonable if those services already match the team's operating model. A shared interface removes some credential and client integration work, but the application still owns correlation, retention, and the decision to grant access.

For the combined approach, accept one vendor to trust, one bill, and one outage surface. Don't trade away an existing compliance evidence process merely to reduce the number of client libraries. For a solo builder shipping LLM features alongside checkout, the useful savings are engineering attention and bounded request volume; neither model choice nor token spending should enter the password-reset authorization path.

## A focused handoff from delivery status to an evidence record

The TypeScript example performs one status read and hands its result to log ingestion with the same base URL and key. It treats the status response as opaque.

A status label isn't permission to reset a password.

Request fields for log ingestion must come from its current discovery schema. Set `RESET_LOG_BODY_TEMPLATE` to a schema-valid JSON request with a string value of `__DELIVERY_RECORD__` in the documented field that accepts the record. The program replaces that value with serialized evidence. This is explicit configuration, not a proposed vendor field name. Configure that destination as a restricted evidence stream before running it: the example preserves the response, so its contents need a retention and redaction review.

Run with a TypeScript-capable Node.js runtime. Required environment variables are `RESET_API_BASE_URL` (the provider's versioned API base), `INFRAI_API_KEY`, `RESET_MESSAGE_ID`, `RESET_CHALLENGE_REF` (an opaque application reference), `RESET_EVIDENCE_ID` (stable for this observation), and `RESET_LOG_BODY_TEMPLATE`. Reuse the evidence ID when retrying the same observation; use a new one for a later poll. No credential belongs in the template.

```ts
import { randomInt } from "node:crypto";

function required(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`Missing ${name}`);
  return value;
}

const base = required("RESET_API_BASE_URL").replace(/\/$/, "");
if (new URL(base).protocol !== "https:") {
  throw new Error("Use an HTTPS API base");
}
const key = required("INFRAI_API_KEY");
const messageId = required("RESET_MESSAGE_ID");
const challengeRef = required("RESET_CHALLENGE_REF");
const evidenceId = required("RESET_EVIDENCE_ID");
const template = JSON.parse(required("RESET_LOG_BODY_TEMPLATE"));
const marker = "__DELIVERY_RECORD__";
if (!JSON.stringify(template).includes(JSON.stringify(marker))) {
  throw new Error("Template must contain a delivery record placeholder");
}

async function request(
  path: string,
  method: "GET" | "POST",
  body?: string,
  idempotencyKey?: string,
): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt++) {
    const headers: Record<string, string> = {
      Authorization: `Bearer ${key}`,
    };
    if (body !== undefined) headers["Content-Type"] = "application/json";
    if (idempotencyKey) headers["Idempotency-Key"] = idempotencyKey;
    const response = await fetch(`${base}${path}`, {
      method,
      headers,
      body,
      redirect: "error",
      signal: AbortSignal.timeout(10_000),
    });
    const raw = await response.text();
    if (response.status === 429 && attempt < 3) {
      const retryAfter = response.headers.get("Retry-After");
      let serverDelay = 0;
      if (retryAfter !== null) {
        const seconds = Number(retryAfter);
        serverDelay = Number.isFinite(seconds)
          ? seconds * 1000
          : Date.parse(retryAfter) - Date.now();
        if (!Number.isFinite(serverDelay)) serverDelay = 0;
      }
      const delay = Math.max(
        0, serverDelay, 500 * 2 ** attempt + randomInt(0, 250),
      );
      if (delay > 60_000) {
        throw new Error(`Retry later; server requested ${delay}ms`);
      }
      await new Promise<void>((resolve) => setTimeout(resolve, delay));
      continue;
    }
    if (!response.ok) {
      throw new Error(`${method} ${path}: ${response.status} ${raw}`);
    }
    return raw.length ? JSON.parse(raw) : null;
  }
  throw new Error("Retry budget exhausted");
}

async function main(): Promise<void> {
  const delivery = await request(
    `/sms/status/${encodeURIComponent(messageId)}`, "GET",
  );
  const record = JSON.stringify({
    evidenceId,
    challengeRef,
    messageId,
    observedAt: new Date().toISOString(),
    delivery,
  });
  const body = JSON.stringify(template, (_name, value) =>
    value === marker ? record : value,
  );
  await request("/logs/ingest", "POST", body, evidenceId);
  console.log("Delivery evidence recorded");
}

main().catch((error: unknown) => {
  console.error(error instanceof Error ? error.message : String(error));
  process.exitCode = 1;
});
```

The application record keys inside `record` belong to this example. They make no assertion about the provider's delivery response fields. The ingestion envelope stays configurable because replacing an undocumented schema with a plausible-looking object would produce a misleading integration example. The HTTP errors retain the response reason; route those diagnostics to restricted operational output.

This is one observation, not a polling daemon or a full reset server. Schedule another read only while the application still needs evidence, and persist a pending observation before arranging durable retries. A bounded retry budget keeps rate limiting from consuming the worker indefinitely. The idempotency key protects a repeated write under the platform's documented deduplication convention; it doesn't replace an application evidence ledger. Keep the same serialized observation for a durable retry.

## Resend is a policy decision, not a second reset

Use the managed resend operation to recover from delayed messages, and keep its relationship to the application challenge visible. Require app-side cooldowns, attempt caps, IP/device throttling, and country allowlists before opening the reset form to public traffic. These controls limit abuse and request volume. They also produce evidence of why the application allowed or refused another send.

Don't silently extend the application's reset deadline because someone pressed resend. Decide that behavior explicitly and test it against the provider's documented challenge semantics. If email recovery is part of the product, specify and test its challenge lifecycle separately; an SMS verification result doesn't establish the validity of an email challenge. Additional recovery methods need their own security and evidence review.

## What to measure before copying this choice

Use three acceptance cases: a code submitted before expiry, a resend requested near expiry, and a delivery observation received after expiry. These are proposed tests, not reported outcomes. For each, check that the reset decision is explainable from the application record without reconstructing a timeline manually across dashboards. In the near-expiry case, retain the original deadline and the resend decision, then check that a subsequent delivery observation changes only the evidence record. Ask support to reconstruct why the reset was accepted or rejected using that record alone. If the explanation depends on a guess about what a delivery label means, the evidence boundary needs more work.

Measure observation lag separately from any source-reported delivery timing. Count poll requests per challenge, resend requests per successful recovery, rejected attempts, and telemetry writes that still need reconciliation. A faster polling interval buys fresher support evidence while adding requests; stop polling according to an explicit evidence policy, rather than letting an expired shopper session drive an endless background loop.

Finally, ask the compliance owner to identify the required evidence, approved processing locations, retention period, and access controls. None of those follows from an API key or an SMS receipt. Choose the shared SMS-and-observability interface if its documented contract satisfies that review and the experiment can reconstruct every reset decision. Choose a separate verification and monitoring stack when its evidence process is a better fit. Ship only after that boundary is clear.

## References

These primary documents support the comparison and messaging-policy review; they aren't certifications of the proposed application.

- Twilio Verify documentation: https://www.twilio.com/docs/verify
- Vonage Verify documentation: https://developer.vonage.com/en/verify/overview
- Amazon SNS SMS documentation: https://docs.aws.amazon.com/sns/latest/dg/sns-mobile-phone-number-as-subscriber.html
- Datadog HTTP log intake documentation: https://docs.datadoghq.com/logs/log_collection/?tab=http

## Sources

- https://www.ctia.org/the-wireless-industry/industry-commitments/messaging-interoperability-sms-mms
- https://support.google.com/a/answer/81126
