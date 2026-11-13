# Postgres Recipient Preferences: Owning Templates for Batch Email, SMS and Polling

A property manager's bulk event notification system needs batch email and SMS records that preserve the approved notice. Start with application-owned templates and a Postgres worker that records recipient preferences, suppression decisions, and status polling separately.

TL;DR: For a Node.js bulk event notification system, freeze the notice content before creating batch email and SMS deliveries. Page through a fixed audience, record preference and suppression decisions, and let a worker poll delivery status separately. Owning the template makes the content record easier to explain; it also makes your application responsible for rendering, approval, and revision control.

This is an engineering record. Whether a channel satisfies a legal service requirement needs a separate decision before the worker sends anything.

## How should a bulk event notification system batch email and SMS?

For this implementation, the application owns the approved text and stores a rendered snapshot for each delivery. The transport receives that snapshot rather than selecting a mutable template by name.

Consider an illustrative notice for property P-17, revision 3. Someone edits the template while the worker is halfway through the residents. A template name alone cannot explain which wording each resident received. Pinning the revision and retaining its rendered result removes that ambiguity from the proposed design. Keep email and SMS renderings separately approved, too: an abbreviated SMS should not silently become the authoritative version of a longer notice.

Freeze the wording.

The data flow is straightforward: an approved notice creates a frozen recipient list; a worker pages through it, evaluates current preferences and suppression state, and passes eligible deliveries to a channel adapter; polling then adds status observations to the ledger. Content revision and transport status stay separate because they answer different questions.

If an LLM drafts the notice, put generation before approval. Don't regenerate wording during retries. That decision avoids extra model calls while preserving the content that was actually approved.

## Run a small batch worker

This TypeScript example uses an in-memory transport simulator, without credentials or external packages. Run it in a TypeScript-capable Node.js environment. It models channel decisions, cursor pagination, frozen text, and status polling; the simulated statuses are fixtures rather than evidence of real delivery.

The four residents and page size of 2 are illustrative data. The example sends individual messages within each audience page. A transport offering a bulk submission API would still need a result associated with every delivery.

```ts
type Channel = "email" | "sms";
type Status = "accepted" | "delivered" | "failed";
type Delivery = {
  key: string;
  residentId: string;
  channel: Channel;
  revision: number;
  body: string;
  decision: "eligible" | "disabled" | "suppressed";
  receipt?: string;
};

const notice = Object.freeze({
  id: "inspection-P-17",
  revision: 3,
  template: "Property P-17: inspection notice for {{resident}}.",
});
const residents: { id: string; channels: Channel[] }[] = [
  { id: "R-101", channels: ["email", "sms"] },
  { id: "R-102", channels: ["sms"] },
  { id: "R-103", channels: ["email", "sms"] },
  { id: "R-104", channels: [] },
];
const channels: Channel[] = ["email", "sms"];
const suppressed = new Set(["R-103:sms"]);
const ledger = new Map<string, Delivery>();
const observations: {
  key: string; status: Status; observedAt: string;
}[] = [];

// Simulator only: this map does not survive a process restart.
const receipts = new Map<string, string>();
const transport = {
  async submit(delivery: Delivery): Promise<string> {
    const previous = receipts.get(delivery.key);
    if (previous) return previous;
    const receipt = "demo-" + (receipts.size + 1);
    receipts.set(delivery.key, receipt);
    return receipt;
  },
  async poll(_receipt: string): Promise<Status> {
    return "delivered";
  },
};

async function main() {
  let after = "";
  for (;;) {
    const page = residents
      .filter((resident) => resident.id > after)
      .sort((a, b) => a.id < b.id ? -1 : a.id > b.id ? 1 : 0)
      .slice(0, 2);
    if (page.length === 0) break;

    for (const resident of page) {
      for (const channel of channels) {
        const key = JSON.stringify([
          notice.id, notice.revision, resident.id, channel,
        ]);
        if (ledger.has(key)) continue;
        const decision: Delivery["decision"] =
          suppressed.has(resident.id + ":" + channel)
            ? "suppressed"
            : resident.channels.includes(channel)
              ? "eligible" : "disabled";
        const delivery: Delivery = {
          key, residentId: resident.id, channel,
          revision: notice.revision,
          body: notice.template.replace("{{resident}}", resident.id),
          decision,
        };
        ledger.set(key, delivery);
        if (decision === "eligible") {
          delivery.receipt = await transport.submit(delivery);
          observations.push({
            key, status: "accepted",
            observedAt: new Date().toISOString(),
          });
        }
      }
    }
    after = page[page.length - 1].id;
  }

  for (const delivery of ledger.values()) {
    if (!delivery.receipt) continue;
    observations.push({
      key: delivery.key,
      status: await transport.poll(delivery.receipt),
      observedAt: new Date().toISOString(),
    });
  }
  console.log(JSON.stringify({
    deliveries: [...ledger.values()], observations,
  }, null, 2));
}

main().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

The fixture produces 8 channel decisions and 4 eligible submissions. R-102 has disabled email, R-103 is suppressed for SMS, and R-104 has no enabled channels. Those decisions leave records. A resident excluded by policy should not look like someone the worker forgot.

The simulator deliberately polls once. Production polling needs a schedule, an expiry policy, and explicit handling of uncertain submissions.

## Move the boundary into a database worker

Before sending real notices, replace the maps with durable storage. A proposed Postgres schema separates notice revisions, audience membership, deliveries, submission attempts, and status observations. Put a unique constraint on notice ID, revision, resident ID, and channel. PostgreSQL documents the constraint and locking mechanisms; this particular schema is an application design.

Use a fixed audience for cursor pagination. Otherwise, a new recipient inserted behind the cursor can disappear from the job's view. Select audience members after the last processed ID using a deterministic ordering. If the audience changes, record an explicit expansion rather than quietly redefining an existing notice.

Preferences have a different lifetime. Evaluate current eligibility close to submission, and store the result and observation time. Define what happens if a preference changes between that check and the network call: a database lookup does not make the two systems atomic. For required notices, an unavailable channel should create an exception for review under the approved delivery policy. An email failure alone is not permission to send SMS.

The awkward failure is a crash after the transport accepts a request but before the worker saves its receipt. The demo map returns the same receipt for repeated submissions, but that behavior establishes nothing about an external transport. Persist an attempt before submission. Reuse a stable idempotency key only where the adapter's documented contract supports it; otherwise retain an uncertain state and reconcile before retrying. A local unique constraint prevents duplicate local rows. It cannot prove the network side effect happened once.

Keep transactions short. Workers can claim pending rows with row locks and `SKIP LOCKED`, commit the claim, and then make the network call. PostgreSQL describes this mechanism as useful for multiple consumers of a queue-like table, while warning that it provides an inconsistent view of the data. Use it to claim work, not to establish audit completeness. Give claims a lease expiry and a recovery policy.

That machinery earns its place at the first real send.

## Keep observations separate from conclusions

Store the adapter's original status alongside your normalized state. Acceptance should mean acceptance under that adapter's contract. It should not become a claim that a resident read the notice or that a service obligation was satisfied.

Polling also needs a budget. Check pending deliveries on a schedule, increase intervals within a configured bound, and stop under a documented expiry policy. Keep “no new observation” separate from “delivery failed.” If callbacks are also supported, retain event identifiers and timestamps so duplicates and late updates do not silently rewrite the record.

The trade-off is explicit: application-owned templates require application-side review and rendering work. This approach is not suitable for a workflow whose approved editing process must live entirely outside the application. In that case, capture an immutable content export from the external approval process before dispatch. Here, preserving approved wording justifies that work. Avoid coupling template identifiers to a transport's account-specific naming scheme, and keep the rendering boundary explicit enough to test with a simulator.

For a solo builder, the useful cost controls are bounded pages, capped concurrency, restrained polling, and no model calls inside the delivery loop. Measure submissions, pending age, and review exceptions per notice. Keep resident addresses and message bodies in access-controlled records; operational logs can use identifiers without duplicating all the sensitive content.

Before deployment, replay the fixture and verify its 8 decisions and 4 submissions. Exercise a crash after acceptance, an eligibility change, and a late status update. Check that a changed template creates a deliberate new revision, that a crashed worker's lease can be recovered, and that the audit export reconstructs both approved text and subsequent observations. Ship when those records tell a coherent story.

## References

The sources below document database constraints, ordered selection, work claiming, and native TypeScript execution. The architecture and fixture counts are illustrative design choices rather than measured delivery results.

## Further reading

- PostgreSQL constraints: https://www.postgresql.org/docs/current/ddl-constraints.html
- PostgreSQL SELECT, ordering, and locking clauses: https://www.postgresql.org/docs/current/sql-select.html
- Node.js TypeScript support: https://nodejs.org/api/typescript.html
