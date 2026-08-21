# Lost Acknowledgments: Failed Webhook Retry Queues for Delayed Node.js Callbacks

A failed media webhook can become worthless before it becomes impossible to deliver. The best queue pattern therefore does more than retry callbacks: it keeps delayed jobs durable, preserves one logical identity, and stops automated delivery when the media event's useful window closes. A syndication notice for a breaking story may have a short window, while an archive-completion notice may still matter tomorrow.

Short answer: use a durable queue with at-least-once delivery, make the receiver deduplicate a stable delivery ID, and move exhausted or expired work into a dead-letter state for deliberate recovery. Use cron to wake a reconciliation sweep, not as the record of each callback.

The failed simple approach is an in-process timer. It is cheap to ship, but a restart loses its delayed work and a second process has no shared claim on the timer. A periodic sweep is better when work can be reconstructed from durable business data. It is still a poor substitute for a per-callback ledger when each item needs an attempt history, its own due time, and a terminal recovery path.

## What should a failed webhook retry experiment prove for a Node.js queue?

Choose by the effect you must guarantee, not by a queue feature checklist. For outbound media webhooks, the practical contract is usually one logical delivery that may produce multiple HTTP requests. An acknowledgment can disappear after the receiver has committed the effect, so the sender cannot infer that a timeout means "nothing happened." The receiver and sender have to share a stable delivery identity.

This is the gate.

Before comparing patterns, write down the fault that the design must survive: the receiver commits an `asset.ready` effect for `asset_7041`, its acknowledgment is lost, and the sender tries the callback again. Passing means the logical work remains recoverable through process loss while the receiver applies the media change once. This is stricter and more useful than counting HTTP requests because a sound at-least-once system may send twice at precisely this boundary.

## Implement the receiver-side transaction first

If the second request has a new identity, the receiver cannot reliably distinguish recovery from a second event. If both requests use `delivery_91f2`, the receiver can return the recorded outcome without applying the syndication change again. The receiver should record that ID in the same transaction as the media-side change.

Don't rotate the ID.

At-least-once is honest. “Exactly once” across an HTTP boundary is not something a sender can establish alone, because there is no atomic transaction spanning the receiver's database commit and the sender's acknowledgment record. Aim for exactly-once effect instead: every retry carries the same delivery ID.

## Model delayed work as a delivery ledger

Now compare the storage patterns:

| Pattern | Work survives process loss | Per-item due time and history | Duplicate-effect control | Best fit |
| --- | --- | --- | --- | --- |
| In-process timer | No | Local memory only | Ad hoc | Disposable reminders with no recovery promise |
| Periodic durable sweep | If source data is durable | Reconstructed during each sweep | Depends on repeatable business logic | Coarse reconciliation of discoverable work |
| Durable delayed queue | Yes | Stored with the logical delivery | Stable ID plus receiver deduplication | Independent callbacks with deadlines and redrive |

The table does not make the durable queue a universal winner. It buys a stronger delivery model by adding storage retention, lease recovery, dead-letter review, and more metrics. For a nightly operation that can always rediscover incomplete rows, stick with a periodic sweep. For an individual publication callback whose history must survive a deployment, the extra state earns its keep.

The queue record should carry the logical delivery ID, destination, payload or payload reference, next eligible time, attempt count, and delivery deadline. Worker ownership also needs a lease token. A claim operation must atomically make one worker the current owner; completion must be conditional on that same token, so a slow worker cannot overwrite a newer worker's result after its lease expires.

That last race is easy to miss. Imagine worker A claims a callback and then stalls long enough for its lease to expire. Worker B claims it, sends it, and records success. When A wakes up, an unconditional update can reschedule or dead-letter an already delivered item. Conditional state transitions turn A's late completion into a rejected stale write. The queue may still make two requests, which is why receiver deduplication remains necessary, but its ledger no longer moves backward.

Keep failure classes small. A retryable outcome schedules another attempt within the delivery deadline. An ambiguous transport outcome also retries because the effect may or may not have happened. A terminal outcome, an exhausted attempt budget, or an expired usefulness window moves the item into dead-letter review. The exact mapping of response outcomes belongs in the producer-receiver contract; it should not be scattered through worker code.

## Encode lease ownership in the Node.js boundary

The core interface is deliberately generic. It avoids coupling delivery correctness to a particular hosted queue or SDK, and it makes the two atomic operations visible: claiming due work and committing a result under the active lease.

```ts
type DeliveryJob = {
  id: string;
  deliveryId: string;
  callbackUrl: string;
  payload: unknown;
  attempt: number;
  deadline: Date;
  leaseToken: string;
};

type AttemptResult =
  | { kind: "delivered" }
  | { kind: "retryable"; reason: string; retryAfterMs?: number }
  | { kind: "terminal"; reason: string };

interface DeliveryStore {
  claimDue(now: Date, leaseMs: number): Promise<DeliveryJob | null>;
  complete(id: string, leaseToken: string, at: Date): Promise<void>;
  retry(
    id: string,
    leaseToken: string,
    nextRunAt: Date,
    reason: string,
  ): Promise<void>;
  deadLetter(id: string, leaseToken: string, reason: string): Promise<void>;
}

interface CallbackClient {
  send(job: DeliveryJob): Promise<AttemptResult>;
}

const MAX_ATTEMPTS = 7;
const MAX_BACKOFF_MS = 10 * 60 * 1000;

function fullJitterMs(attempt: number, random: () => number): number {
  const ceiling = Math.min(MAX_BACKOFF_MS, 1000 * 2 ** attempt);
  return Math.floor(random() * ceiling);
}

async function deliverOne(
  store: DeliveryStore,
  client: CallbackClient,
  now: () => Date,
  random: () => number,
): Promise<boolean> {
  const job = await store.claimDue(now(), 30_000);
  if (!job) return false;

  const result = await client.send(job);
  if (result.kind === "delivered") {
    await store.complete(job.id, job.leaseToken, now());
    return true;
  }

  const nextAttempt = job.attempt + 1;
  if (result.kind === "terminal" || nextAttempt >= MAX_ATTEMPTS) {
    await store.deadLetter(job.id, job.leaseToken, result.reason);
    return true;
  }

  const delayMs =
    result.retryAfterMs ?? fullJitterMs(nextAttempt, random);
  const nextRunAt = new Date(now().getTime() + delayMs);

  if (nextRunAt >= job.deadline) {
    await store.deadLetter(
      job.id,
      job.leaseToken,
      "delivery window expired",
    );
    return true;
  }

  await store.retry(job.id, job.leaseToken, nextRunAt, result.reason);
  return true;
}
```

The constants are policy examples, not recommended defaults. A ten-minute cap can be absurd for a callback with a two-minute editorial freshness budget, yet unnecessarily aggressive for a background archive. I'm not sure a single retry curve should serve both workloads; deadline age, receiver throttling, and measured recovery by attempt are the evidence needed to tune it.

Full jitter spreads retries across the current backoff window rather than returning an entire failed batch at once. The deadline check prevents the queue from spending work on an automated delivery that the product has already declared stale. Dead-lettering does not mean deletion: retain enough metadata to inspect the decision and redrive the same logical delivery ID after a human or automated policy approves it. Payload retention, callback authentication, destination validation, and request signing are production concerns too, but adding them to this sample would hide the ownership transition under review.

## Decide from recovery data, not scheduler labels

Cron is a time-based job scheduler. That makes it suitable for waking a periodic reconciliation task, but the schedule itself does not store a callback payload, attempt count, lease owner, or terminal state. A scheduled GitHub Actions workflow similarly makes sense when a repository workflow run is the unit of work; it is not a callback delivery ledger. The distinction is about the data model, not brand preference.

Run a reconciliation sweep alongside the queue. It should compare durable media events with delivery records and create missing logical deliveries using deterministic IDs. This covers the producer-side gap where business state commits but enqueueing does not. The sweep must be repeatable: seeing the same media event twice should resolve to the same logical callback rather than two unrelated deliveries.

Then fault-inject the boundaries. Stop a worker after the receiver effect but before acknowledgment. Expire a lease during a slow send. Submit the same logical media event twice. Hold a destination unavailable until the delivery deadline passes, inspect the dead-letter record, and redrive it without changing its identity. Expected results are concrete: eligible work survives process loss, stale owners cannot mutate the ledger, and repeated requests do not repeat the receiver effect when it honors the delivery ID.

Measure age of the oldest due item, success by attempt number, attempts per delivered callback, lease expirations, dead-letter arrivals, and duplicate requests by delivery ID. Also watch retained payload bytes and callback duration; retry amplification spends compute and receiver capacity even when enqueueing looks cheap. Your mileage may vary — the acceptable backlog and concurrency ceiling depend on the media workflow's freshness window and the receiver's limits.

Ship only after those measurements separate three very different problems: permanent rejection, ambiguous transport outcomes, and worker capacity. One aggregate failure rate blurs the decision. The durable queue is not suitable when a cheap, repeatable sweep can reconstruct every unit of work and no per-item deadline or audit trail exists. It is the stronger choice when each callback needs an independent clock, durable identity, bounded retry history, and controlled recovery.

## Sources

- https://en.wikipedia.org/wiki/Cron
- https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows
