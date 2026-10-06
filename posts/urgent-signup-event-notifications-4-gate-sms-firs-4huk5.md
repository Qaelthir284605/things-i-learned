# Urgent Signup Event Notifications — 4-Gate SMS-First Email Fallback Logic

The main integration trade-off is speed versus certainty: send the property-management signup link by SMS first, but treat gateway acceptance as an intermediate event, not proof that the applicant received it. **Use one durable notification record and four explicit gates: accepted, delivered, expired, and escalated.** Poll only while the link remains useful, retry only failures classified as temporary, and send email once when the SMS path cannot establish delivery before the deadline. This keeps US and EU policy outside provider code and makes every extra network call explainable.

Short answer: the link, expiry, region, consent evidence, and channel attempts belong to one state machine. Transport adapters should only send and query status. They should never decide when a signup deserves another message.

## How should urgent event notifications handle SMS-first fallback?

An SMS API can accept a request before the downstream carrier outcome is known. Twilio's SMS documentation, for example, distinguishes message resource states and supports status callbacks; that is evidence for the general rule, not a reason to couple the workflow to one product. A synchronous success answers "did the gateway accept my request?" It does not answer "did the prospective tenant receive a usable verification link?"

Email has the same semantic trap at a different boundary. SMTP defines replies between mail systems, while RFC 5321 explains that successful transfer does not guarantee retrieval or reading by the intended recipient. The state machine should record transport evidence precisely and avoid inventing a universal "seen" state.

The flow is plain. The signup transaction creates a single-use verification token and an outbox record in the same durable operation. A worker claims that record, formats a region-appropriate message, and submits SMS through an adapter. Authenticated callbacks update known outcomes; a bounded poller fills gaps. The scheduler escalates to email if SMS reaches a terminal failure or the delivery deadline passes. Verification consumes the token atomically, so a late SMS cannot validate an already consumed or expired token.

## Build the smallest useful state machine

This TypeScript example makes the control flow concrete. The repository and transports are interfaces because durability and provider integration are deployment choices. Four polls are an example policy, not a carrier promise.

```ts
type Region = "US" | "EU";
type Channel = "sms" | "email";
type Delivery = "pending" | "delivered" | "temporary-failure" | "permanent-failure";

type Attempt = {
  channel: Channel;
  providerId: string;
  status: Delivery;
  checks: number;
};

type Notice = {
  id: string;
  region: Region;
  phone: string;
  email: string;
  verificationUrl: string;
  expiresAt: number;
  escalated: boolean;
  attempts: Attempt[];
};

interface Transport {
  send(to: string, body: string, idempotencyKey: string): Promise<string>;
  status(providerId: string): Promise<Delivery>;
}

interface NoticeStore {
  get(id: string): Promise<Notice>;
  saveIfVersionMatches(notice: Notice): Promise<boolean>;
}

const pause = (ms: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, ms));

async function deliverVerification(
  noticeId: string,
  store: NoticeStore,
  sms: Transport,
  email: Transport,
  now: () => number = Date.now,
): Promise<void> {
  let notice = await store.get(noticeId);
  if (now() >= notice.expiresAt) return;

  let attempt = notice.attempts.find((item) => item.channel === "sms");
  if (!attempt) {
    const body = `Verify your rental account: ${notice.verificationUrl}`;
    const providerId = await sms.send(notice.phone, body, `${notice.id}:sms:1`);
    attempt = { channel: "sms", providerId, status: "pending", checks: 0 };
    notice.attempts.push(attempt);
    if (!(await store.saveIfVersionMatches(notice))) return;
  }

  for (const delayMs of [1_000, 2_000, 4_000, 8_000]) {
    if (now() + delayMs >= notice.expiresAt) break;
    await pause(delayMs);
    attempt.status = await sms.status(attempt.providerId);
    attempt.checks += 1;
    await store.saveIfVersionMatches(notice);
    if (attempt.status === "delivered") return;
    if (attempt.status === "permanent-failure") break;
  }

  notice = await store.get(noticeId);
  if (notice.escalated || now() >= notice.expiresAt) return;

  const body = `Use this single-use link before it expires: ${notice.verificationUrl}`;
  const providerId = await email.send(notice.email, body, `${notice.id}:email:1`);
  notice.attempts.push({
    channel: "email",
    providerId,
    status: "pending",
    checks: 0,
  });
  notice.escalated = true;
  await store.saveIfVersionMatches(notice);
}
```

Those short delays expose the sequence; they are not production guidance. A durable scheduler should persist `nextCheckAt` and release its worker instead of waiting in `setTimeout`. Restarts then become ordinary queue recovery.

There is a race worth designing for: a callback may mark SMS delivered while the poller prepares email. The compare-and-set save and `escalated` flag must be enforced by the repository, ideally in one transaction. Otherwise two workers can both observe `false` and send duplicate fallback messages. Annoying. It also spends message budget without helping the tenant.

## Keep regional policy out of adapters

Do not infer US or EU policy from a phone prefix alone. Store an explicit region derived from the property's operating context and validated during signup. Regional classification, consent, retention, templates, and quiet-hour decisions are policy inputs. The adapter's narrower job is to submit a message, return an identifier, and normalize status.

For EU processing, GDPR Article 5 establishes purpose limitation, data minimization, storage limitation, and integrity and confidentiality principles. Those principles support storing only the delivery evidence needed for signup and assigning retention deliberately; they do not establish one universal retention duration. For US messaging, applicable law and FCC guidance should inform consent and content. Legal review must settle the actual policy for the service and message category.

Keep marketing out of this path. Requested account verification is operational, and promotional copy muddies consent and deliverability. RFC 8058 defines one-click unsubscribe for list email through specific headers and an HTTPS POST. It matters when mail is subscription traffic, but it is not a generic header to attach to transactional verification. Classify the message first.

Make the business controls visible rather than burying them in an SDK callback:

```ts
type DeliveryPolicy = {
  region: Region;
  smsDeadlineMs: number;
  maxStatusChecks: number;
  retainAttemptMetadataDays: number;
  requireRecordedConsent: boolean;
};

function validatePolicy(policy: DeliveryPolicy): void {
  if (policy.smsDeadlineMs <= 0) throw new Error("SMS deadline must be positive");
  if (policy.maxStatusChecks < 1) throw new Error("A status check is required");
  if (policy.retainAttemptMetadataDays < 1) {
    throw new Error("Retention must be explicit");
  }
}
```

The exact numbers require product and legal decisions. Their ownership does not.

## Retry evidence, not hope

Retries need a taxonomy. A timeout, throttling response, or temporarily unavailable dependency may justify a delayed retry. An invalid destination, rejected consent state, expired token, or malformed request should stop. The adapter maps provider outcomes into a small internal vocabulary; the orchestrator applies policy.

Idempotency has two layers. Each logical channel attempt gets a stable key such as `notice-id:sms:1`, so a worker restart does not create a new intent. The local repository also reserves that attempt before, or atomically with, dispatch. Remote idempotency can help, but the application still needs its own record because transport implementations and retention behavior differ.

Polling is a fallback evidence path, not the default source of truth. Prefer authenticated callbacks when the transport supports them, reject stale or unverifiable callback requests, and reconcile pending records on a schedule. Add jitter, cap concurrency, and stop querying when the state is terminal or the token expires. **Spend status calls only while their result can change the next action.**

Observability should follow the state machine. Record the notice ID, channel, normalized outcome, attempt number, region, transition time, and policy version. Exclude the token and message body from logs. Track accepted submissions, terminal outcomes, escalations, expirations, rejected duplicate transitions, and time from signup to verification. Alert on stuck pending records and a rising escalation ratio, then investigate by normalized failure class before blaming a carrier or mail system.

## Operate the workflow before trusting it

Test the transition table and the integration edges. A fake clock should prove that expiry stops polling and fallback. Script callbacks arriving before a poll, during fallback reservation, and after email submission. Force a temporary failure followed by delivery, then a permanent failure, then a worker crash immediately after remote acceptance. The invariant is stronger than "the function returned": one active token, no more than one logical attempt per channel and generation, and a stored reason for every transition.

Deploy callback ingestion and reconciliation in shadow mode first, recording status without enabling fallback. Compare callback and poll results, inspect unknown states, then enable escalation for a controlled cohort. A kill switch should stop new fallback sends without blocking token verification or evidence updates.

Before release, read the flow as a prospective tenant. Check that SMS and email use the same single-use token, expiry text agrees with server enforcement, a repeat request creates a defined new generation, and consuming either link invalidates the others. Then read it as the operator: consent evidence is present, region and policy version are explicit, callbacks are authenticated, logs omit secrets, pending records recover after restarts, and dashboards distinguish acceptance from delivery. These checks interact; checking "retry enabled" without expiry and idempotency is how duplicates escape.

The practical finish line is modest: a new transport can implement two methods without rewriting signup policy, every escalation has stored evidence, and an expired link stops the machine. That is enough abstraction to avoid lock-in without building a notification platform before the property-management product has users.

## Sources

References used for the standards and delivery semantics in this note:

- https://datatracker.ietf.org/doc/html/rfc8058
- https://datatracker.ietf.org/doc/html/rfc5321
- https://www.twilio.com/docs/sms
- https://eur-lex.europa.eu/eli/reg/2016/679/oj
- https://www.fcc.gov/consumers/guides/stop-unwanted-robocalls-and-texts
