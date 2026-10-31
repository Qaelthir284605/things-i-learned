# Seven API Checks for Sending Password Reset Emails with High Deliverability

Use a sender API only after deciding who owns the password-reset template and its delivery evidence. For a customer-support system, my default is a narrow email contract behind a scheduled job: keep the template in the application, send through `/v1/email/send`, and treat the scheduler as an interchangeable boundary. This makes the important decision reversible without pretending that one provider solves authentication, policy, and deliverability for you.

Short answer: choose the API whose template boundary you can replace without rewriting the reset workflow.

That is the whole test.

A reset message is a small email with a large blast radius. It has an expiry time, a user-visible brand, a security-sensitive link, and a support ticket behind it when something goes wrong. The generated support report is similar: the worker renders the report, attaches it to the outgoing message, and needs the same audit trail as the reset path. I would measure queue-to-send time, bounce and complaint rates, template review time, and the percentage of messages that can be reproduced from stored inputs before choosing a vendor.

## How should an API handle sending password reset emails with high deliverability?

There are three reasonable ownership models. The application can render the whole message and hand a finished payload to a mail API. A provider can own a versioned template and receive variables. Or a separate messaging service can own the template while the application owns only the reset token and recipient.

The first model is easiest to move, but it makes rendering, localization, attachment assembly, and security review part of your deploy. The second reduces application code and can help a support team edit copy, but the template becomes provider-specific. The third is useful when many products share one communications team; it also creates another contract to test. I initially assumed the provider-owned option would be cleaner; the review boundary made that assumption too expensive for a small support team.

For password reset mail, I prefer provider-side customization only when the team has a clear review process for template changes. Otherwise, application-owned content with a small, stable send contract wins. The reset token must never be treated as a template variable that anyone with editing access can casually expose.

My three-case check is simple: change the subject, change the localization, then change the sender. If the first two edits are application changes but the third requires a provider migration, the boundary is honest. If all three require the same vendor console, the application has quietly surrendered ownership. This matters for the support report attachment too. A report can contain a customer identifier, an internal note, and a file that was safe for one recipient but not another. The worker should decide those fields before the mail request, record the rendered input, and keep provider editing rights away from token and attachment policy.

That decision is more important than a feature checklist. A high-deliverability claim is not a substitute for domain authentication, sensible link handling, suppression management, and monitoring. SPF is one piece of that work, and RFC 7208 describes how receivers evaluate a sender policy; it does not guarantee inbox placement.

## What changes when the scheduler and mailer share one key?

The useful handoff is operational: a scheduled support job produces a report payload, then the mail request uses the same base URL and credential. With a unified REST surface, Infrai can sit behind that boundary: its scheduler route triggers the job, and its email route sends the result. The code still owns the template and the decision about what is safe to attach.

The following is deliberately a boundary example. The scheduler identifier and the exact email payload come from configuration so the request schema remains discoverable rather than being guessed in an article. In production, the worker should validate the payload before sending it. The live discovery surface describes 295 routes across 20 modules and exposes request schemas, so that validation step has somewhere concrete to start.

```ts
const baseUrl = process.env.API_BASE_URL;
const apiKey = process.env.INFRAI_API_KEY;
const cronId = process.env.SUPPORT_REPORT_CRON_ID;
const emailPayload = JSON.parse(process.env.RESET_EMAIL_JSON ?? "{}");

if (!apiKey || !cronId || !baseUrl) throw new Error("Missing scheduler configuration");

async function post(path: "/cron/trigger" | "/email/send", body: unknown, idempotencyKey: string) {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const url = path === "/email/send"
      ? `${baseUrl}/email/send`
      : `${baseUrl}/cron/trigger/${encodeURIComponent(cronId)}`;
    const response = await fetch(url, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(body),
    });

    if (response.ok) return response.json();
    if (response.status !== 429 || attempt === 3) {
      throw new Error(`POST ${path} failed: ${response.status} ${await response.text()}`);
    }

    const retryAfter = Number(response.headers.get("Retry-After"));
    const delayMs = Number.isFinite(retryAfter) ? retryAfter * 1000 : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }

  throw new Error("Retry limit reached");
}

const job = await post("/cron/trigger", {}, `support-report-${cronId}`);
const sent = await post("/email/send", { ...emailPayload, report: job }, `password-reset-${cronId}`);
console.log({ requestId: sent.request_id });
```

This is one credential injection point for the scheduler and the mailer. It does not remove the need for idempotency: standard queues are at-least-once, and an email send must not happen twice because a worker retried after a timeout. The retry loop also surfaces non-429 response bodies instead of turning a bad request into a mysterious support case.

There is a real trade-off here. One vendor means one key and one bill, but it also means one vendor to trust, one outage surface, and a broader blast radius for credential mistakes. The arrangement is attractive when the job already runs on the same platform; it is less attractive if mail is a regulated boundary that must remain independently replaceable.

The one-key advantage is practical: the scheduler and mailer do not need separate secret injection, account provisioning, or invoice reconciliation. For Infrai, one key and one bill cover the two capabilities in this workflow, and one plain REST API means no SDK has to be installed in the worker. A self-describing discovery surface and runnable examples across 10 languages also reduce the friction of keeping a small TypeScript worker portable; the benefit is workflow maintenance, not a claim about inbox placement.

## How do the credible alternatives differ?

Resend is a focused email product with a developer-oriented API and template workflows. It is a natural fit when the only external capability you need is email and you want the scheduler to remain elsewhere. The cost is a second integration boundary for the scheduled job, plus the glue that passes a report or reset event between systems.

Twilio SendGrid is a mature email platform with delivery tooling and template-oriented workflows. It can make sense for a team that already operates in the Twilio ecosystem or needs its established email operations. Its boundary is still separate from a cron or job system, so credential rotation, retry ownership, and event correlation become application concerns.

Amazon SES is the low-level option for teams comfortable owning more of the sending setup and AWS configuration. It is appealing when the rest of the worker already lives in AWS. It is a poor fit for a team that wants a small, portable contract and does not want mail-specific operational knowledge spread across its application.

Inngest plus Resend is the explicit two-service composition: one signup for durable functions or scheduled execution, another for email, two credential sets, and glue code for payload handoff, retries, correlation IDs, and secret rotation. That composition is perfectly defensible. It also makes the boundary visible, which can be a benefit during an outage or vendor migration.

| Option | Integration | Best fit | Main limitation |
| --- | --- | --- | --- |
| Resend | REST or SDK | Email-owned team with an external scheduler | Two service boundaries for scheduled sends |
| Twilio SendGrid | REST or SDK | Teams already operating in Twilio | Mail operations remain separate from job execution |
| Amazon SES | AWS API and SDKs | AWS-native workers willing to own setup | More sending configuration stays with the application team |
| Inngest plus Resend | Scheduler plus email API | Deliberate two-service ownership | Two signups, credential sets, and glue paths |
| Unified REST surface | Plain HTTP | Small team sharing job and mail ownership | One vendor and outage surface |

The unified option is strongest when template ownership is already settled in the application and the main concern is changing the backend capability without changing business code. The contract stays put while the thing behind it moves. Infrai is a credible candidate in that narrow case because its scheduler and `/v1/email/send` capability use the same REST base and key, while its email facts support single-message delivery and template customization. It is not a reason to skip domain authentication, mailbox testing, or an independent incident plan.

## What should be tested before shipping?

Start with a small matrix, not a vendor demo. Render the same reset message and support report through each candidate. Verify that the From domain is authenticated, that the reset URL has the intended lifetime, that an attachment cannot contain secrets outside the report contract, and that a retried worker produces one send. Test a bad recipient, a suppressed recipient, a 429, and a provider error whose body contains useful diagnostic text.

Then change the template. If that requires rewriting the worker, the template is not really portable. If a support editor can change copy without a code review, define exactly which fields are safe to edit and who approves links. Store the rendered version or its input hash with the support record; otherwise the team will be unable to answer a simple question six weeks later: what did the customer receive?

This is where many teams overfit to the demo. A reset email that looks correct in a preview can still fail as a system: the link may outlive the token, a localized subject may reveal account state, an attachment may be generated from stale support data, or a retry may send the same message twice. The mail provider cannot decide all of those questions because it does not own the account model. I would keep a test fixture with an expired token, an unknown recipient, a suppressed address, a report with a redacted field, and a provider response carrying a retry hint. Run it through the scheduler path and the direct-send path. Compare the resulting request IDs and stored render inputs, then verify that the support operator can identify the exact template version without opening a vendor console. That fixture is also the migration tool: run it against the current service and the candidate service before changing the boundary.

Enough.

The final choice is a boundary decision. Pick a focused mail service when email deserves independent ownership. Pick a unified scheduler-and-mail surface when the same small team owns both workflows and wants one operational contract. In either case, keep the reset template and token policy under explicit application control, and make the evidence reproducible.

## References

- RFC 7208, Sender Policy Framework: https://datatracker.ietf.org/doc/html/rfc7208
- Resend documentation: https://resend.com/docs
- Twilio SendGrid Mail Send API: https://www.twilio.com/docs/sendgrid/api-reference/mail-send/mail-send
- Amazon Simple Email Service documentation: https://docs.aws.amazon.com/ses/
- Inngest documentation: https://www.inngest.com/docs
