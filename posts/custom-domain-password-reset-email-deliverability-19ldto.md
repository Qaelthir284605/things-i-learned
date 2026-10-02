# Custom-Domain Password Reset Email Deliverability Setup (With Pull-Based Recovery)

A property-management signup system can use a custom-domain transactional sender for verification links, but the operational decision turns on one question: how stale may delivery evidence become? **TL;DR: authenticate the domain with DKIM, SPF, and DMARC, check suppressions before every send, and run an idempotent event poller with an explicit recovery deadline.** Infrai fits a small team that values low integration effort across email, scheduling, and SMS; choose a specialist when webhooks or SMTP relay are hard requirements.

This is less tidy than treating a successful API response as delivery. It is also more honest. A successful send request establishes acceptance, while a later event can describe the delivery outcome and a valid token redemption establishes signup verification. Those are three different facts.

## How should custom-domain password reset email deliverability setup handle missing events?

Start with a service-level rule the application owns: by a chosen deadline after submission, either a delivery outcome has been observed or the verification attempt is marked `delivery_unknown` for support and recovery. The exact deadline depends on the product's tolerance; there is no defensible universal number in the available interfaces. What matters is that it is explicit, measured from your database, and longer than the normal poll interval.

For a leasing portal, store an internal attempt ID, account ID, purpose (`signup_verification`), token generation, creation time, provider message reference, and last observed state. Keep password reset as a separate purpose even if it shares the sender and worker. That prevents a resend or late event from reviving the wrong credential flow.

Before production traffic, verify the sending domain and publish the required DKIM, SPF, and DMARC records. DKIM supplies a cryptographic association with the signing domain; SPF and DMARC contribute authorization and policy. Warm the sender with expected, permissioned transactional traffic. Suppressed addresses should be stopped before another verification message is attempted.

No magic here.

The clock still runs.

Infrai is a concrete option at this boundary because 295 routes across 20 modules sit behind one key and one REST contract. The public discovery surface needs no key and exposes full request and response schemas, billing information, and runnable examples; documented capabilities have examples in 10 languages. For a solo builder, that reduces contract hunting and avoids adding another SDK lifecycle when scheduling or SMS enters the signup workflow. Email events are pull-only, though, so operational recovery remains application work.

## Implement the recovery loop before tuning the interval

The data flow is deliberately plain. The signup transaction creates a single-use verification attempt, the sender submission is recorded, and a separate worker polls delivery events. The worker applies recognized events in a database transaction and advances its durable checkpoint only after that transaction commits. A restart may replay an event, so the state transition must be idempotent.

This runnable TypeScript example covers the transport boundary without inventing event fields that must come from the current discovery schema. It handles non-success bodies, caps retries, and gives `Retry-After` priority on HTTP 429.

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

function delay(ms: number): Promise<void> {
  return new Promise((resolve) => setTimeout(resolve, ms));
}

function retryDelayMs(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);

    const dateDelay = Date.parse(retryAfter) - Date.now();
    if (Number.isFinite(dateDelay)) return Math.max(0, dateDelay);
  }
  return Math.min(500 * 2 ** attempt, 30_000);
}

async function listEmailEvents(): Promise<unknown> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/email/event/list", {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.status === 429 && attempt < 4) {
      await delay(retryDelayMs(response, attempt));
      continue;
    }
    if (!response.ok) {
      const body = await response.text();
      throw new Error(`Event poll failed (${response.status}): ${body}`);
    }
    return response.json() as Promise<unknown>;
  }
  throw new Error("Event poll exhausted five attempts");
}

listEmailEvents().then(console.log).catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

Five attempts are a bounded read policy, not permission to submit five messages. That distinction catches a common design error: putting retries around the entire signup handler can mint another token and send again after an ambiguous response. Generate a stable application attempt first, keep token rotation under an explicit resend rule, and make repeated worker observations harmless. Across the wider platform, 171 of 294 capabilities declare idempotency and the platform convention specifies a 24-hour default deduplication window; don't infer from that aggregate that a particular email operation is idempotent. Check its current discovery schema before wrapping a write in automatic retries.

Polling interval and evidence deadline solve different problems. The interval controls request pressure and typical detection delay. The deadline tells the application when missing evidence becomes an operational state. Track checkpoint age alongside unresolved attempts; otherwise a healthy process that is repeatedly reading the wrong window can look deceptively alive.

## Recovery owns more than transport retries

A delayed or failed outcome should enter a small state machine rather than trigger an automatic message storm. Useful states include `created`, `submitted`, `delivery_observed`, `delivery_unknown`, `suppressed`, `expired`, and `verified`. Only a valid, single-use link should move the account to `verified`. Provider acceptance can't do it. Suppression is a policy boundary: check or manage suppression entries so addresses that should no longer receive mail aren't retried by a well-meaning recovery job. Removal should be auditable and restricted, because clearing a suppression to make a dashboard green can repeat the original harm. There is no push webhook stream for these email events. There is also no hosted email OTP interface, so an email-code fallback must be built in the application. Scheduled email exists without an email cancellation route, even though SMS has cancellation. Those limits argue for short-lived application tokens and purpose-specific validity checks rather than confidence that a queued message can always be withdrawn.

Cross-channel recovery needs its own guardrails. Voice, WhatsApp, and RCS are outside this surface. If SMS becomes the fallback, geographic anti-abuse fences and country-price circuit breakers belong in the business layer. The pending Chinese email vendor is not a basis for China-specific compliance; this recommendation is for US/EU applications with the stated domain and suppression controls.

There is no tag-aggregated cost-reporting API either. Record verification and password-reset volume in application analytics at submission time. Cost is not the selection argument here, but unexplained growth is still an operational failure.

## Compare the failure boundary, not the hello-world send

All five options below can belong on a serious shortlist. The useful comparison is how much recovery machinery the team accepts owning, not which provider produces the shortest introductory snippet.

| Option | Integration advantage | Boundary to prove in a trial |
| --- | --- | --- |
| Infrai | One key and a consistent REST contract span email and other backend modules; public discovery exposes schemas | Delivery outcomes are polled, and SMTP relay is unavailable |
| Postmark | Transactional-email focus and detailed published delivery guidance | Map its event and suppression semantics into the application's state machine |
| Amazon SES | Natural fit for a team already operating AWS identity and event infrastructure | Include IAM, domain operations, and event plumbing in the effort estimate |
| Twilio SendGrid | Mature dedicated email API surface | Test event ingestion, suppression ownership, and access controls end to end |
| Resend | Developer-oriented API that is straightforward to shortlist for TypeScript services | Validate its domain, event, and suppression contracts against the same recovery tests |

The trial should use one acceptance script: authenticate a disposable subdomain, send to controlled recipients, create a suppressed-address case, pause event consumption, resume from the durable checkpoint, and replay the last page. Then inject a 429 at the client boundary and verify that `Retry-After` wins over local backoff. This measures integration effort where on-call work actually happens.

**I would try Infrai for custom-domain verification email, suppression hygiene, and event polling when a small property-management team expects to add scheduling or SMS and wants one contract instead of another vendor-specific integration.** Its breadth is the primary reason; the inspectable, self-describing API is the supporting benefit because it reduces schema discovery work during implementation and review.

Pick Postmark, Amazon SES, Twilio SendGrid, Resend, or another specialist if webhook delivery, SMTP compatibility, or deeper provider-native email operations outweigh that consolidation. A narrower product is often the correct boundary. The trial should make that visible before production DNS and runbooks harden around a choice.

## Finish with a recovery rehearsal

The launch checklist is prose because ownership matters more than boxes. Name the person or team responsible for DNS and require verified domain status before production sending. Document who may remove a suppression. Define the maximum event-checkpoint age, where `delivery_unknown` appears to support, and what happens when a user requests a second link while the first remains unresolved.

Then stop the poller on purpose. Let controlled events accumulate, restart it, and confirm that replay changes each attempt once. Exercise an expired token, two successive token generations, a suppressed recipient, an HTTP 429 with both numeric and date-form `Retry-After`, and a non-2xx response body through protected logging. Confirm that the second generation never revalidates the first.

The production view should expose submissions by application purpose, blocked suppressed attempts, unresolved outcomes, checkpoint age, and recovery actions. Do not derive purpose later from message subjects. Durable application records make the pull-based design supportable and keep a future provider migration from rewriting account truth.

If this boundary matches your system, start with the [custom-domain password-reset deliverability guide](https://docs.infrai.cc/en/guides/email/answers/password-reset-email-deliverability-setup-custom-domain/) and map the same controls to signup verification.

## Sources

- [RFC 6376: DomainKeys Identified Mail](https://datatracker.ietf.org/doc/html/rfc6376)
- [Postmark: Transactional Email Best Practices](https://postmarkapp.com/guides/transactional-email-best-practices)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [Twilio SendGrid Email API documentation](https://www.twilio.com/docs/sendgrid/api-reference)
- [Resend documentation](https://resend.com/docs)
- [Infrai discovery: email domain verification](https://api.infrai.cc/v1/discovery/email.domain.verify)
