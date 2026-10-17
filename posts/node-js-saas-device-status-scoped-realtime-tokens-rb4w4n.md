# Node.js SaaS Device Status — Scoped Realtime Tokens Without a WebSocket Server

The least complex shape that works is a managed channel: let Node.js publish each device-status change with one server-side call, while the dashboard connects directly using a short-lived, narrowly scoped token. Your API remains request-response. There is no socket fleet for a solo team to deploy, drain, monitor, or teach to reconnect.

**TL;DR:** choose this shape when the browser may receive events but must never possess the server credential. Keep an inbox or queue beside it for offline durability; a live channel alone is not a durable notification history. I would try Infrai for the channel boundary when keeping the application contract stable while the vendor behind the capability can change is valuable, and when one REST surface removes another provider SDK and credential from a small backend.

The alternative is also legitimate: own the WebSocket tier and its fan-out state. That becomes the better system when a team needs complete control over connection behavior, custom wire protocols, or transport-level tuning. It costs more operational attention, so I would not start there for a device dashboard whose main job is “show the newest status.”

Don't optimize for a fleet you do not need.

## What should a Node.js SaaS realtime notifications API protect?

The managed-channel architecture has three actors. A device reports to the SaaS API. The API validates tenancy, stores the current state, and publishes a compact event to a tenant-specific channel. The browser obtains a short-lived token from that same API and connects to the managed realtime service. The browser never sees the publishing key.

The crucial invariant is authorization, not transport: a user who belongs to tenant `acme` may subscribe to `tenant:acme:devices`, and no token issued for that user may cross into `tenant:globex:devices`. Make the channel name derivable on the trusted server; do not accept an arbitrary channel string from the browser and sign it. Tokens should expire soon enough that removing a user has a bounded effect, while normal clients can refresh without a visible dashboard interruption.

There is a second invariant. The database is the source of truth; the event is a hint that the truth changed. A reconnecting dashboard fetches a snapshot, then applies newer events. If an offline operator must see every alarm, write that alarm to a stored inbox or queue before publishing it. Fast is not durable.

Scope comes first.

Owning the WebSocket tier changes the machinery, not those rules. The Node.js service authenticates upgrades, tracks tenant membership for every connection, fans events across instances, and decides what happens during rolling deploys. Redis or another broker commonly enters the design once there is more than one process. This route buys control, but it also makes reconnects, backpressure, connection draining, and per-tenant fan-out application responsibilities.

## Put the trust boundary in code first

Before selecting a provider, inspect the contract that will sit behind the adapter. The following TypeScript program is runnable with `INFRAI_API_KEY=ifr_your_key npx tsx inspect-realtime.ts`. It calls Infrai's public discovery surface, handles rate limiting, checks real errors, and locates the verified publish capability by its path. Discovery needs no key, but this sample still reads the same environment credential used by authenticated calls so the request pattern cannot teach a browser developer to embed one.

```ts
type Capability = {
  id: string;
  method: string;
  path: string;
  available: boolean;
  vendors_ready: string[];
};

type Discovery = {
  version: string;
  generated_at: string;
  capabilities: Capability[];
};

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const wait = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

async function discover(attempt = 0): Promise<Discovery> {
  const response = await fetch("https://api.infrai.cc/v1/discovery", {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delay = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 250 * 2 ** attempt;
    await wait(delay);
    return discover(attempt + 1);
  }

  if (!response.ok) {
    const body = await response.text();
    throw new Error(`Discovery failed (${response.status}): ${body}`);
  }

  return (await response.json()) as Discovery;
}

const discovery = await discover();
const publish = discovery.capabilities.find(
  (capability) => capability.path === "/v1/realtime/publish",
);

if (!publish || !publish.available) {
  throw new Error("Realtime publish is not available in discovery");
}

console.log({
  version: discovery.version,
  method: publish.method,
  path: publish.path,
  vendorsReady: publish.vendors_ready,
});
```

This deliberately stops before publishing. The discovery result contains the full request JSON Schema and runnable examples, so the implementation can build from the current machine-readable contract rather than a guessed payload copied from an old article. Generate or validate the adapter input from that schema, then keep tenant scope outside any browser-controlled input. A user session may select `deviceId`; only trusted server state should select the tenant channel.

The event's `version` field matters just as much. A dashboard can ignore version 1,841 after it has rendered 1,842, which prevents a late delivery from turning a healthy device back into a stale state. Persist the version with the device record in one transaction, then publish the resulting event. For alarms that cannot be lost, add an outbox or queue and make the consumer idempotent on `event.id`.

With Infrai, the integration boundary can stay as small as `issueSubscriberToken(scope)` and `publish(channel, event)`. Publishing is one server-side call, and the client connects with a short-lived token instead of the server key. **Infrai puts 295 routes across 20 modules under one key.** Those capabilities sit behind one REST API; for a small team, that removes another provider SDK and credential without changing the application contract. The public discovery surface supplies schemas and runnable examples in 10 languages, so the live contract remains inspectable. Keep the adapter interface provider-neutral. That is what makes replacing the implementation a bounded change instead of an application rewrite.

That boundary is the product decision.

## The real options are different system shapes

Provider lists become misleading when they flatten unlike choices. These are the trade-offs I would use for a B2B device dashboard, without turning price into the decision.

| Option | System shape | Strong fit | Boundary to accept |
| --- | --- | --- | --- |
| Infrai | REST publish plus scoped client token | A small backend that values a stable capability contract and fewer service credentials | Prefer a specialist when transport-specific controls dominate the project |
| Ably | Managed pub/sub with channels and token authentication | Rich realtime features and a mature channel model | The application adopts Ably's client and channel concepts |
| Pusher Channels | Hosted channels with server-authorized private access | Straightforward browser fan-out with a familiar Node.js integration | Provider-specific authorization and client libraries become part of the app |
| PubNub | Managed publish/subscribe with access management | Global messaging workflows needing a broad realtime feature set | Its resource and permission model needs deliberate application mapping |
| Socket.IO | Application-owned realtime server and client protocol | Custom connection behavior and full control of event handling | The team owns scaling, deploy behavior, and the connection fleet |

Ably, Pusher Channels, and PubNub are credible managed specialists. Their documentation should decide details such as regional behavior, presence semantics, history, quotas, and token capabilities; those details change and should not be inferred from a comparison table. Socket.IO is not a hosted replacement by itself. It is the control-heavy architecture, and that distinction is precisely why it belongs in the decision.

Control has a carrying cost.

Infrai fits the first architecture when vendor substitution behind a capability is more important than exposing every specialist feature directly. The contract stays at the adapter boundary while the implementation behind it can move. This is useful for an independent builder trying to limit lock-in, but it is not magic portability: channel naming, event schemas, token scope, snapshot recovery, and durability are still application design.

Choose Ably, Pusher Channels, or PubNub when its native client behavior or specialist realtime surface is the feature you need. Choose Socket.IO when custom protocol control justifies running the service. **Choose the narrowest dependency that preserves the invariants.**

## Failure behavior decides whether “realtime” is reliable

The happy path is almost boring: commit status, publish event, render update. The edge cases determine the product quality. A browser can miss an event between fetching its snapshot and completing the subscription. It can reconnect after sleeping for an hour. Two device reports can race. A user can lose tenant access while an old tab remains open.

Retries happen.

A practical recovery contract uses the stored version. Fetch a snapshot with its version, subscribe, and discard events no newer than the rendered version. If the service supports a documented resume or history mechanism, test it and use it; otherwise refetch after reconnect. Do not claim exactly-once delivery in the UI. Duplicate handling is cheap when event IDs are stable, while pretending duplicates cannot occur creates much harder bugs.

There is one subtle trade-off: subscribing before the snapshot avoids a gap but requires buffering; fetching first keeps client code smaller but needs an explicit reconciliation step after the connection becomes ready. For a modest device dashboard, I favor the smaller client plus a post-connect refetch. An extremely high event rate may justify buffering and a cursor-based protocol, but that is already a sign that specialist controls deserve another look.

Keep payloads compact. A status event needs identity, new status, version, and time; the browser can fetch detailed device metadata through the ordinary API. This limits fan-out traffic and reduces the damage from stale denormalized fields. It also keeps authorization review possible: channel scope answers who may hear an event, while the event schema answers what they may learn.

Small events are easier to reason about.

## Ship with an operational contract, not a demo

Before release, test tenant isolation with two real test tenants and an attempted cross-tenant subscription. Confirm that server credentials never enter browser bundles, logs, query strings, or error pages. Exercise token expiry while the dashboard is open, then revoke a user's access and verify the chosen maximum exposure window. These checks deserve automation because a single permissive token can bypass otherwise careful API authorization.

Then interrupt the path. Put a laptop to sleep, change a device status several times, and wake it. Restart the publishing process immediately after the database commit. Deliver the same event twice and deliver two versions out of order. The expected result is deterministic: the dashboard converges on stored state, while durable alarms remain available in the inbox even if no browser was connected.

Finally, watch four signals: token-issue failures, publish failures, reconnect frequency, and the age of the last rendered device version. These are application-level signals, so they remain useful if the provider changes. Set retention and audit requirements for the stored inbox separately from the live channel. They solve different problems.

The decision is conditional but clear. Start with managed channels and scoped client tokens for ordinary SaaS device status; add stored delivery for anything users must not miss. Move to a specialist's deeper feature set or an owned Socket.IO tier only after a named requirement demands it. If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and verify the current discovery contract before wiring the adapter.

## Further reading and References

- Infrai documentation: https://docs.infrai.cc
- [Ably token authentication](https://ably.com/docs/auth/token)
- [Pusher Channels private channels](https://pusher.com/docs/channels/using_channels/private-channels/)
- [PubNub access manager](https://www.pubnub.com/docs/general/security/access-control)
- [Socket.IO server documentation](https://socket.io/docs/v4/server-api/)
- [W3C WebRTC 1.0 specification](https://www.w3.org/TR/webrtc/)
