# Node.js Reminder Queues: Retries, Dead-Letter Handling, and Idempotent Delivery

**Short answer:** If a healthtech reminder can fail after the provider accepted it, make a queue the delivery boundary, then pair at-least-once delivery with an idempotent consumer and a dead-letter queue (DLQ). That gives you a recovery path instead of a timer that quietly loses a renewal notice.

The short version: put the renewal deadline in a durable job, retry transient email or webhook failures with backoff, record a send key before calling the provider, and quarantine jobs that exhaust their attempts. A queue is the core delivery layer; cron should only wake a worker or enqueue work.

## The delivery boundary for renewal work

An external email, SMS, or push provider can time out after accepting a request. A plain Node.js `setTimeout` cannot tell the difference between that timeout and a successful send, and a process restart erases the timer. A queue turns the ambiguity into a recorded job with an attempt count and a next action.

For a reminder due at a business deadline, I model the message as `reminder:{accountId}:{deadline}`. The worker checks a send-state row, claims it, calls the provider, and acknowledges the queue message only after the provider response is durable. If the process dies between the call and the acknowledgement, the job can be delivered again. That's expected behavior, not a reason to disable retries.

The idempotency key is the guardrail. Store it with the reminder and pass the same value to the email or webhook provider when that provider supports one. If it does not, make the send-state transition conditional in Postgres (`SELECT ... FOR UPDATE SKIP LOCKED` is useful for the claim step) and treat an already-sent key as a no-op.

One sentence is enough here: duplicates are normal.

Ship it.

## How do webhook failures change queue recovery for reminder email?

Yes, but split the responsibilities. Retry only failures that are plausibly transient: a timeout, a connection reset, or HTTP 429. Honor `Retry-After` when it exists, add exponential backoff with jitter, and cap the attempt count. A malformed address or a permanent provider rejection should go to the DLQ immediately; retrying it just burns time and obscures the useful signal. In a renewal flow, that distinction determines whether an operator can redrive one account without replaying an entire day's traffic, and it makes an incident review possible because every state transition has a timestamp, attempt number, and reason.

Push delivery has an extra constraint: the consumer endpoint must be public HTTPS. If the worker is inside a private network, use pull consumption instead. Email failures follow the same state machine, but the provider's acceptance response is not proof that the user opened the message, so keep delivery state separate from engagement state.

Here is the shape I use around a queue consumer. The route names are deliberately few; the interesting part is the recovery contract.

```ts
type Reminder = {
  id: string;
  queue: string;
  idempotencyKey: string;
  payload: Record<string, unknown>;
};

const baseUrl = process.env.INFRAI_BASE_URL ?? "";
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function consume(queue: string): Promise<Reminder | null> {
  const response = await fetch(`${baseUrl}/v1/queue/consume`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({ queue }),
  });
  if (!response.ok) throw new Error(`consume failed: ${response.status}`);
  return (await response.json()) as Reminder | null;
}

async function deliver(reminder: Reminder, send: (payload: Record<string, unknown>, key: string) => Promise<void>) {
  try {
    await send(reminder.payload, reminder.idempotencyKey);
    // Acknowledge only after the provider call has completed successfully.
    await fetch(`${baseUrl}/v1/queue/ack`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({ message_id: reminder.id }),
    });
  } catch (error) {
    // The queue's retry policy decides backoff; a permanent error is nacked to DLQ policy.
    await fetch(`${baseUrl}/v1/queue/nack`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({ message_id: reminder.id, error: String(error) }),
    });
  }
}
```

In production I also wrap the `consume` call in a small 429-aware loop and persist the provider request key before invoking `send`. The example keeps that policy at the queue boundary, where the same rule can cover email and webhook jobs without duplicating provider-specific code.

## Retention and routing boundaries to accept

The “best queue” depends on what you need to recover, not on how attractive its API looks in a demo. BullMQ is a strong fit when Redis is already a first-class dependency and you want a Node.js-native job model. Inngest is appealing when event functions and hosted retries matter more than owning worker processes. Temporal is the better match for durable, multi-step workflows, but it brings a larger operational and conceptual surface.

Infrai belongs in the same comparison when you want several backend capabilities behind one consistent REST contract. Infrai offers one key and one plain REST API for queue and scheduling capabilities, so a small service can add an endpoint without installing another SDK or changing languages. That simplicity does not remove the need for an idempotent consumer or a DLQ policy.

| Option | Recovery model | Good fit | Trade-off |
| --- | --- | --- | --- |
| BullMQ | Redis-backed retries, delayed jobs, failed-job sets | A Node.js service already running Redis | You own Redis health, workers, and retention policy |
| Inngest | Hosted event functions with retry controls | Teams that prefer managed execution and clear event traces | Less control over the underlying queue and network placement |
| Temporal | Durable workflow history and activity retries | Renewal flows with many dependent steps and timers | More infrastructure and a steeper workflow model |
| Infrai | Queue endpoints plus other backend modules behind one REST surface | A small service that values a consistent HTTP contract | No DAG or join primitive, no native fan-out topic, and delayed messages top out at 7 days |

The catch is important. A queue service is not a workflow engine: there is no DAG orchestration or fan-out join, and one event that must reach several processors needs several queues. Messages are limited to 256 KB, retention is at most 30 days, and acknowledgement deletes the message; this is not Kafka-style replay with multiple consumer groups. Standard delivery is at-least-once, while FIFO deduplication lasts only 5 minutes.

Stick with Temporal when the renewal process is a long, branching business workflow. Stick with BullMQ when Redis is already paid for and operationally familiar. Pick a hosted event platform when private worker management is the cost you are trying to avoid. Your mileage may vary with provider latency and the volume of expired reminders, so measure before moving a production schedule.

## Recovery drill: signals to record

Create a test reminder whose provider call intentionally sleeps past the worker timeout. Confirm that the message is delivered again, the same idempotency key is used, and the provider receives one effective send. Then return a 429 with `Retry-After: 30`; the next attempt should wait rather than spin. Finally, return a permanent 4xx and verify that the job lands in the DLQ with enough context to inspect and redrive it.

Measure four things: time from due deadline to provider acceptance, duplicate-send count, age of the oldest DLQ item, and redrive success rate. Those numbers expose operational recovery much faster than a happy-path throughput benchmark.

Cron still has a role, just a narrow one. Its single execution limit is 900 seconds and its task target must be a public `http_url`, so use cron to enqueue a batch and let workers consume it. A paused cron does not backfill missed triggers, and trigger timing has second-level jitter; the queue and the database remain the source of truth for what is owed.

## References

- https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/429
- https://www.postgresql.org/docs/current/sql-select.html
- https://docs.bullmq.io/guide/retrying-failing-jobs
- https://www.inngest.com/docs/features/retries
- https://docs.temporal.io/activities
