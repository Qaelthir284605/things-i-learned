# Customer Support Suppression: How Password Reset Retries Avoid Duplicate Email Sends

A timeout does not mean a password reset email failed. Treat it as an unknown outcome: keep one reset token and one durable send record for the request window, reconcile that record, and retry only when the provider confirms there is no accepted message. Feed hard bounces into the same local recipient-suppression check before any later reset attempt.

TL;DR: exactly-once delivery is not an email-provider promise. The practical target is exactly one accepted send per reset token, enforced in the application. This also gives customer support a useful answer when someone reports multiple links: one token, one delivery ledger entry, and a visible terminal state.

## How should password reset email sends retry after a timeout?

The simple implementation calls `send`, waits, and calls it again after a timeout. It confuses a missing response with a rejected request. The first request may already have crossed the provider boundary, so the retry can create a second message with a second valid link.

Wait.

The safer experiment changes the unit of work. Generate one short-lived token for a request window, derive a stable operation ID from it, and persist the outbound record before touching the network. A retry reuses both values. If a later attempt succeeds, invalidate any older token for that account so that an inbox containing delayed mail does not present several usable credentials.

Do not schedule a delayed reset email that support may need to revoke. Email scheduling has no cancel operation in the capability described here; send the reset promptly or keep the delay in an application queue that you control.

## Build the ledger before the sender

The following TypeScript is deliberately provider-neutral. It runs as-is with an in-memory store and a fake transport, while the interfaces map cleanly to a database transaction and any transactional email API. The important decision is the state machine: `pending` means no call has begun, `unknown` means a call began without a conclusive response, and `accepted` means the provider returned a send ID.

```ts
import { createHash, randomUUID } from "node:crypto";

type State = "pending" | "unknown" | "accepted" | "suppressed";
type Delivery = {
  operationId: string;
  recipient: string;
  token: string;
  state: State;
  providerSendId?: string;
};

interface MailTransport {
  send(input: {
    operationId: string;
    recipient: string;
    resetUrl: string;
  }): Promise<{ sendId: string }>;
  find(operationId: string): Promise<{ sendId: string } | null>;
}

const deliveries = new Map<string, Delivery>();
const suppressed = new Set<string>();

async function lookupSend(sendId: string, attempt = 0): Promise<unknown> {
  const baseUrl = process.env.EMAIL_API_BASE_URL;
  const apiKey = process.env.INFRAI_API_KEY;
  if (!baseUrl || !apiKey) throw new Error("Set EMAIL_API_BASE_URL and INFRAI_API_KEY");

  const response = await fetch(`${baseUrl}/v1/email/get/${encodeURIComponent(sendId)}`, {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });
  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return lookupSend(sendId, attempt + 1);
  }
  if (!response.ok) {
    throw new Error(`Email lookup failed (${response.status}): ${await response.text()}`);
  }
  return response.json();
}

function operationId(recipient: string, token: string): string {
  return createHash("sha256").update(`${recipient}:${token}`).digest("hex");
}

async function sendReset(recipient: string, mail: MailTransport): Promise<Delivery> {
  if (suppressed.has(recipient)) {
    return { operationId: "blocked", recipient, token: "", state: "suppressed" };
  }

  const existing = [...deliveries.values()].find(
    (row) => row.recipient === recipient && row.state !== "suppressed",
  );
  const token = existing?.token ?? randomUUID();
  const id = operationId(recipient, token);
  const row = existing ?? { operationId: id, recipient, token, state: "pending" as const };
  deliveries.set(id, row);

  if (row.state === "accepted") return row;
  if (row.state === "unknown") {
    const found = await mail.find(id);
    if (found) {
      const accepted = { ...row, state: "accepted" as const, providerSendId: found.sendId };
      deliveries.set(id, accepted);
      return accepted;
    }
  }

  deliveries.set(id, { ...row, state: "unknown" });
  const result = await mail.send({
    operationId: id,
    recipient,
    resetUrl: `https://accounts.example/reset?token=${token}`,
  });
  const accepted = { ...row, state: "accepted" as const, providerSendId: result.sendId };
  deliveries.set(id, accepted);
  return accepted;
}

const fake: MailTransport = {
  async send(input) {
    return { sendId: `msg_${input.operationId.slice(0, 12)}` };
  },
  async find(id) {
    const row = deliveries.get(id);
    return row?.providerSendId ? { sendId: row.providerSendId } : null;
  },
};

console.log(await sendReset("reader@example.com", fake));
```

In production, lock the ledger row while deciding whether to send. A process-local map cannot stop two workers. Also give the provider the stable operation ID through its idempotency mechanism when one exists; that protects the narrow gap between the network call and the database update. Set `EMAIL_API_BASE_URL` to the documented API base before using `lookupSend`; keeping the base configurable also preserves the transport boundary. Infrai fits this boundary because one key covers 295 routes across 20 modules through one REST contract, and changing the vendor behind a capability does not require application call-site changes. Its `Idempotency-Key` convention has a 24-hour default deduplication window.

## Reconcile first, then decide

After an ambiguous response, query the stored provider send ID and recent message history before issuing another send. Polling is a real constraint here because email events are not pushed by webhook. That adds reconciliation latency, so the reset screen should say that mail may take a moment rather than encouraging repeated clicks.

Keep the decision small: accepted means return the existing outcome; confirmed absent means retry with the same operation ID and token; still unknown means wait and reconcile again. Never mint a fresh token merely because an HTTP client timed out. My decision rule is conservative because a slower reset is easier to explain than two valid credentials in one inbox. The trade-off is explicit: polling adds delay, while blind retries add ambiguity and support work.

Bounce handling belongs just ahead of that decision. Record a hard bounce against the normalized recipient, suppress subsequent sends, and let support remove or correct the address through an audited account flow. A transient delivery delay should not become permanent suppression. The evidence required to distinguish those outcomes comes from provider status or event history, not guesswork in the request handler.

Stop there.

Measure the duration of the `unknown` state, duplicate accepted-send IDs per operation ID, reset requests blocked by suppression, and support cases involving stale links. Keep provider latency separate from the time spent polling; combining them hides the integration penalty that matters during an incident. Also test token expiry against the longest reconciliation interval. Short-lived tokens reduce exposure, but a token that expires before the application can resolve an uncertain send creates a different support failure. The right lifetime comes from your security policy and observed delivery path, not a copied constant. The final acceptance test is blunt: force the transport to accept a message and drop its response, restart the worker, and invoke the same reset request again. The system should discover the prior send, retain one token, and avoid another email. Then inject a hard bounce and verify that the next attempt stops before transport invocation.

## Compare the integration boundary

Amazon SES, SendGrid, Postmark, and the multi-vendor option above can all sit behind the `MailTransport` boundary, but the integration work differs. A direct adapter is sensible when provider-specific event semantics or operational controls matter enough to become application features.

| Option | Integration surface | Best fit | Boundary to inspect |
| --- | --- | --- | --- |
| Amazon SES | AWS API | Teams already operating in AWS | Suppression and event integration |
| SendGrid | Web API | Teams using its mail and suppression model | Suppression-group semantics |
| Postmark | Web API | Transactional-mail workflows | Bounce reconciliation |
| Multi-vendor REST layer | One stable HTTP contract | Small teams minimizing adapter churn | Polling latency and provider readiness |

The trade-off is material for this workflow: delivery events in the multi-vendor layer use polling, there is no SMTP relay, and domestic email vendor readiness is pending, so it should not be used as evidence of China-specific compliance. Those boundaries can outweigh adapter simplicity.

I would choose by counting integration surfaces, not logo features: send, lookup, history reconciliation, suppression, and operational authentication. Then I would run timeout injection against the adapter. Ten concurrent calls with the same operation ID should produce one accepted ledger outcome, while a hard-bounced address should produce zero new transport calls. Those are test assertions, not claimed provider benchmarks.

## Further reading

- [Amazon SES suppression list documentation](https://docs.aws.amazon.com/ses/latest/dg/sending-email-suppression-list.html)
- [SendGrid suppression management](https://www.twilio.com/docs/sendgrid/api-reference/suppressions-global-suppressions)
- [Postmark bounce API documentation](https://postmarkapp.com/developer/api/bounce-api)
- [RFC 6376: DomainKeys Identified Mail](https://datatracker.ietf.org/doc/html/rfc6376)
- [OWASP Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
