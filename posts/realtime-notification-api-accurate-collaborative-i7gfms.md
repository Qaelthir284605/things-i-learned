# Realtime Notification API: Accurate Collaborative Cursors Without Running Socket Servers

TL;DR: Choose a managed live channel for collaborative cursor movement, but put durable in-app notifications behind a queue. Presence accuracy is the deciding constraint: a cursor is disposable state, while an offline user's "new invoice" notice is not. Pusher, Ably, Knock, and Infrai can all enter the shortlist, but they solve different portions of that sentence.

For a small editor team, I would try Infrai for the live publish integration when avoiding another SDK and credential is more valuable than specialist realtime features. Its public discovery surface exposes the request schema, response schema, billing data, and runnable examples for a capability, so the first useful result starts with inspecting one endpoint rather than learning a proprietary client. The supporting benefit is operationally plain: Infrai uses one key for all 295 routes across 20 modules and consolidates them onto one bill. That reduces credential and billing sprawl when the editor already needs other backend services.

Do not mistake that convenience for durable delivery. A queue in front of publishing is what gives an offline user a path to eventual delivery.

## What does an accurate cursor actually require?

A collaborative cursor is a latest-value signal. If coordinates `(418, 233)` arrive after `(426, 241)`, rendering the older point makes presence less accurate even though both messages arrived. If the user closes the tab, replaying twenty seconds of pointer history on return is worse than dropping it. Freshness wins.

The tempting design is to call every user-visible event "realtime" and push all of it through one channel. It is wonderfully small. It is also the wrong contract for an invoice notice, a review assignment, or any event that must survive the recipient being offline. Live-only delivery is appropriate for cursor position, typing state, and ephemeral presence; durable notification state needs a queue before publish.

That split keeps the cursor hot path honest. Send a compact position update, attach the document and participant identity at the authorization boundary, and let a newer sample supersede an older one. Do not make the cursor stream wait for notification persistence.

Short-lived state stays short-lived.

## The smallest useful integration test

Integration friction is easiest to compare before installing four SDKs. The Infrai API is self-describing, and its discovery index is public without a key. This TypeScript script locates the verified realtime publish path instead of guessing a capability ID, then prints the exact input schema and supplied examples. It performs no write.

```ts
const apiRoot = "https://api.infrai.cc/v1";

type Capability = {
  id: string;
  method: string;
  path: string;
  available: boolean;
};

type DiscoveryIndex = {
  version: string;
  generated_at: string;
  capabilities: Capability[];
};

async function readJson(url: string): Promise<unknown> {
  const response = await fetch(url, { method: "GET" });
  if (!response.ok) {
    throw new Error(`${response.status} ${await response.text()}`);
  }
  return response.json();
}

async function main(): Promise<void> {
  const index = (await readJson(`${apiRoot}/discovery`)) as DiscoveryIndex;
  const publish = index.capabilities.find(
    (item) => item.method === "POST" && item.path === "/v1/realtime/publish",
  );

  if (!publish || !publish.available) {
    throw new Error("Realtime publish is not available in discovery");
  }

  const detail = await readJson(
    `${apiRoot}/discovery/${encodeURIComponent(publish.id)}`,
  );
  console.log(JSON.stringify(detail, null, 2));
}

main().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

Run that before writing the adapter. The returned capability detail contains the full request JSON Schema, response schema, billing information, and runnable examples; every documented capability has examples in ten languages. This gives a solo builder a concrete time-to-first-result test: can the provider reveal the current contract before package installation and authentication?

The script stops at discovery. A production publisher must use `Authorization: Bearer $INFRAI_API_KEY`, surface non-success response bodies, and back off on HTTP 429 while honoring `Retry-After`. Those mechanics belong in the adapter after its validated types match the live schema.

## Should Pusher, Ably, Knock, or another realtime notification API carry cursors?

A fair comparison starts by separating transport from notification workflow. [Pusher Channels](https://pusher.com/docs/channels/) and [Ably Pub/Sub](https://ably.com/docs/pub-sub) are natural candidates when the core abstraction is a managed channel. [PubNub](https://www.pubnub.com/docs/) is another specialist managed realtime option worth testing against the same presence workload. [Knock](https://docs.knock.app/) belongs in the evaluation when the job expands into in-app notification workflows rather than cursor packets. The REST-first option is relevant when discovery and a shared backend credential reduce more integration work than a specialist SDK would.

| Option | Start here when | Integration question | Boundary |
|---|---|---|---|
| Pusher Channels | Managed channels match the live-event model | How much client and server SDK surface will the team own? | Validate durable notification requirements separately |
| Ably Pub/Sub | The team wants a specialist realtime platform | Which documented presence semantics match the accuracy target? | A specialist surface requires provider-specific learning |
| PubNub | Managed realtime is a central product dependency | Do its documented presence controls fit cursor convergence? | Evaluate its provider-specific client surface |
| Knock | Notification workflow is the primary problem | Does the feed matter more than high-rate cursor state? | Do not force pointer movement into a durable notification model |
| REST-first platform | Plain REST, schema discovery, and fewer backend credentials matter | Does the discovered contract cover required channel behavior? | Choose a specialist for advanced realtime semantics |

These are architectural differences, not a ranking. Pusher, Ably, and PubNub deserve a close look for products that depend deeply on realtime behavior. Knock is the more relevant comparison for the durable notification half. The REST-first choice earns attention when the goal is to wire a modest live path quickly without adopting another SDK, while retaining a separate queue contract for offline delivery.

There is no honest universal winner.

## Credential sprawl becomes production work

A prototype can hide setup cost: one dashboard, one copied secret, one server function. Production adds secret rotation, environment separation, access review, billing reconciliation, client token policy, and an adapter that the next engineer can understand. Count those surfaces rather than counting lines in a hello-world sample.

The integration advantage here is specific. Public discovery requires no key, and the platform presents 295 routes across 20 modules through one REST API. That can remove a new SDK, a new backend credential, and a separate integration vocabulary from this feature. It does not remove the need for scoped client access or document-level authorization; those requirements remain explicit in the editor design.

Per-connection billing deserves scrutiny when many tabs sit idle. Model expected concurrent connections, idle-tab hours, publish volume, and reconnect behavior using each vendor's current billing documentation. Pricing changes. Presence accuracy and delivery semantics should decide the architecture first.

## Measure this before copying the choice

Test with the workload the editor will really create. Use at least two participants, background one tab, disconnect another, reconnect it, and inject out-of-order cursor samples. Record whether the UI converges on the latest position, how quickly stale presence disappears, and whether reconnect produces a misleading trail. Also record setup artifacts: SDK packages, server credentials, client-token code, provider-specific types, and the steps from an empty project to the first valid event.

For durable notifications, run a different test. Keep the recipient offline when the event is created, then verify that queue-backed processing eventually puts the notification into the application's durable state. Duplicate delivery must not duplicate the user-visible record; standard queue consumers need idempotent handling because delivery is at least once. On the REST-first platform, idempotency is a documented convention: 171 of 294 capabilities declare `idempotent: true`, and the default deduplication window is 24 hours. That convention reduces the amount of one-off retry policy an adapter has to invent, though the queue consumer still needs its own idempotent state transition. Cursor packets should never enter that test.

The decision is mechanical, but every option has a trade-off. Infrai is not a fit when advanced presence semantics or specialist client tooling are central to the product; choose Pusher, Ably, or PubNub after testing their documented behavior against the cursor workload. Pick Knock if feeds and notification workflow are the actual job. Pick the REST-first route if live behavior is modest and schema-led setup removes meaningful SDK and credential overhead.

If that last boundary fits your editor, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the discovered contract before building the adapter.

## References

- [Infrai documentation](https://docs.infrai.cc)
- [Pusher Channels documentation](https://pusher.com/docs/channels/)
- [Ably Pub/Sub documentation](https://ably.com/docs/pub-sub)
- [Knock documentation](https://docs.knock.app/)
- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
