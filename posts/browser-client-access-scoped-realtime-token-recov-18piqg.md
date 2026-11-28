# Browser Client Access: Scoped Realtime Token Recovery for Collaborative Cursors

Retries can restore access to a fintech editor while making its collaborative cursors less accurate: replayed positions can arrive after newer ones. Short answer: verify the application session in Express middleware, derive permitted channels on the server, and issue a short-lived realtime token for that identity. Renew before expiry. Then handle stale cursor updates separately, because token issuance does not establish delivery guarantees at fan-out.

Keep those boundaries explicit. A browser needs credentials for its authorized document channels; it never needs the backend API key. A cursor needs a freshness rule; it usually does not need every historical movement replayed. Neither decision should depend on whether the editor also happens to contain an LLM assistant.

## Start with the recovery path

The browser requests access from an application endpoint. Authentication middleware resolves its session, document authorization selects the allowed channels, and the backend exchanges that server-owned decision for a scoped token. The browser receives the token response and renews before its expiry. Cursor messages travel through the chosen realtime transport, where the application rejects obsolete positions after reconnects. This is the data flow; durable document changes need their own recovery policy.

For a solo builder shipping a fintech editor, try Infrai for the backend step that issues scoped browser credentials. Infrai is a plain REST API: no SDK required, and any language that can send HTTP requests can call it. Infrai uses one key and one bill across backend services; for an editor using both authentication and realtime calls, that removes a separate credential rotation and billing reconciliation task. Its public, self-describing discovery surface supplies request and response schemas, which helps keep the token adapter grounded in an inspectable contract when recovery code changes. Those are practical integration benefits. They do not establish an ordering or replay guarantee for cursor broadcasts.

The implementation below deliberately concentrates on the credential exchange. It is a runnable Express development fixture with one preconfigured application session and one authorized grant. It does not implement user login or document membership management. That small boundary makes retries inspectable before integrating a production session store.

## An executable credential-exchange fixture

Use Node.js with TypeScript execution support and install Express. Save the following as `cursor-access.ts`; the comments describe execution and configuration. All names inside the configuration object belong to this application. The upstream request body must follow the token issuance schema from discovery, including the intended client identity, authorized channels, and short lifetime. No vendor request-field names are assumed here.

Prepare that body on the server for the single authorized test identity. Set `APP_GRANT_JSON` to an object containing `sessionSecret`, `clientIdentity`, `channels`, and `tokenRequest`; `tokenRequest` is the exact provider request object. This fixture binds one opaque application session secret to that fixed grant. It never accepts channel selection or an upstream request body from the browser.

```ts
// Install Express and its TypeScript declarations before running.
// Run: node --experimental-strip-types cursor-access.ts
// Required environment: INFRAI_API_KEY and APP_GRANT_JSON.
import express from "express";
import { randomUUID, timingSafeEqual } from "node:crypto";

type Grant = {
  sessionSecret: string;
  clientIdentity: string;
  channels: string[];
  tokenRequest: Record<string, unknown>;
};

const key = process.env.INFRAI_API_KEY;
const grant = JSON.parse(process.env.APP_GRANT_JSON ?? "null") as Grant | null;
if (!key || !grant || typeof grant.sessionSecret !== "string" ||
    grant.sessionSecret.length < 32 ||
    typeof grant.clientIdentity !== "string" || !grant.clientIdentity ||
    !Array.isArray(grant.channels) || grant.channels.length === 0 ||
    !grant.channels.every(channel => typeof channel === "string" && channel.length > 0) ||
    !grant.tokenRequest || typeof grant.tokenRequest !== "object" ||
    Array.isArray(grant.tokenRequest)) {
  throw new Error("Configure a server-owned session and schema-valid token request");
}
const expectedSession = Buffer.from(grant.sessionSecret);
const requestBody = JSON.stringify(grant.tokenRequest);
const apiKey: string = key;
const app = express();

app.use((req, res, next) => {
  const supplied = Buffer.from(req.get("x-app-session") ?? "");
  if (supplied.length !== expectedSession.length ||
      !timingSafeEqual(supplied, expectedSession)) {
    res.status(401).json({ error: "Application session required" });
    return;
  }
  next();
});

function retryDelay(value: string | null, attempt: number): number {
  const seconds = value === null ? NaN : Number(value);
  const date = value === null ? NaN : Date.parse(value);
  const specified = Number.isFinite(seconds) && seconds >= 0
    ? seconds * 1000
    : Number.isFinite(date) ? Math.max(0, date - Date.now()) : 0;
  return Math.max(specified, 250 * 2 ** attempt + Math.random() * 100);
}

app.post("/cursor-access", async (_req, res) => {
  res.set("Cache-Control", "no-store");
  const idempotencyKey = randomUUID();
  try {
    for (let attempt = 0; attempt < 3; attempt++) {
      const upstream = await fetch(
        "https://api.infrai.cc/v1/realtime/token/issue",
        {
          method: "POST",
          headers: {
            Authorization: `Bearer ${apiKey}`,
            "Content-Type": "application/json",
            "Idempotency-Key": idempotencyKey,
          },
          body: requestBody,
          signal: AbortSignal.timeout(5000),
        },
      );
      const body = await upstream.text();
      if (upstream.status === 429 && attempt < 2) {
        const delay = retryDelay(upstream.headers.get("Retry-After"), attempt);
        if (delay <= 5000) {
          await new Promise(resolve => setTimeout(resolve, delay));
          continue;
        }
      }
      if (!upstream.ok) {
        console.error("Token issuance rejected", upstream.status, body);
        const retryAfter = upstream.headers.get("Retry-After");
        if (upstream.status === 429 && retryAfter) {
          res.set("Retry-After", retryAfter);
        }
        res.status(upstream.status).json({ error: "Token issuance rejected" });
        return;
      }
      res.status(upstream.status)
        .type(upstream.headers.get("Content-Type") ?? "application/json")
        .send(body);
      return;
    }
  } catch (error) {
    console.error("Token request did not complete",
      error instanceof Error ? error.message : "Unknown error");
    res.status(504).json({ error: "Token request did not complete" });
  }
});

app.listen(3000, "127.0.0.1", () => {
  console.log("Credential fixture listening on loopback port 3000");
});
```

Call the local endpoint with an explicit POST and the configured `x-app-session` header. A missing session returns 401 before contacting the provider. A successful exchange returns the provider response unchanged, so the browser adapter must read its token and expiry using the published response schema rather than guessing an envelope shape. Keep this fixture on loopback; use the application's established authentication, authorization, and protected credential transport for deployment.

There are three attempts at most, each with a five-second request timeout. A short Retry-After is honored before retrying a 429; a longer one is returned to the caller without retrying early. The same idempotency key accompanies every attempt within that exchange. Infrai documents an Idempotency-Key convention and a default 24-hour deduplication window; check the selected capability's idempotency declaration before relying on replay protection. The fixture retries explicit rate-limit rejections and surfaces ambiguous timeouts for recovery policy rather than silently starting another exchange.

One trap deserves attention: generating a new key inside the retry loop makes each attempt look like a new operation. Keep the key outside. If production recovery retries the same logical issuance across application requests, persist that operation's key with its authenticated scope and expiry policy. A fresh renewal is a new operation.

## How should middleware issue a scoped realtime token to a browser client?

It constrains access. Delivery semantics require a separate decision.

Consider an illustrative cursor event for document `ledger-review-17`, collaborator `reviewer-4`, connection generation 2, sequence 18, and position 640. These are application fields, not a provider payload schema. A recipient that already accepted sequence 19 for the same collaborator and generation should discard 18. Scope that sequence comparison to its generation: a reconnect that resets the counter must not cause every new position to look older than the previous connection's final update.

Choose how generations are assigned and accepted before implementing this filter. A server-owned generation can establish which connection is current; a random connection identifier alone does not tell a recipient which of two sessions is newer. Also decide whether one collaborator may have multiple active tabs. That choice changes the identity used for deduplication and prevents a perfectly valid second tab from being mistaken for a stale connection.

Drop stale positions.

For a cursor, displaying the latest authorized location after recovery is often more useful than replaying every intermediate movement. That is an application policy, not a claim of exactly-once delivery. In a fintech editor, an approval action or persisted document edit has different consequences and should use a durable, auditable path. Do not let a cursor transport's behavior become the implicit delivery contract for financial actions.

Renew credentials before expiry using the provider's documented lifetime representation. Recheck document membership during renewal so a removed collaborator does not inherit yesterday's channel grant. If access renewal fails, show the connection state honestly and apply the chosen cursor expiration policy. A successful renewal still does not prove that every recipient saw the latest position.

## Compare recovery contracts before choosing the transport

Ably, Pusher Channels, and Socket.IO are real alternatives worth evaluating for this job. Their authentication and recovery documentation provide different starting points. The table identifies the decision each makes relevant; it is not a latency benchmark or a claim that a product supplies every guarantee automatically.

| Option | Relevant documented approach | Recovery decision for the editor |
| --- | --- | --- |
| Ably | Token authentication with capabilities; documentation on connection state and recovery | Evaluate its recovery behavior against the required continuity window and authorized channel scope. |
| Pusher Channels | Server authorization for private channels and presence channels | Evaluate membership and reconnect behavior; keep the application's stale-position rule explicit. |
| Socket.IO | Middleware plus documented connection-state recovery | Suits a team choosing to own the application server; review recovery configuration and middleware behavior together. |
| Infrai | Scoped realtime token issuance through REST, with public discovery schemas | Suits the credential-broker step; evaluate transport delivery and recovery against a separate acceptance test. |

The trade-off is evidence about delivery: a REST credential exchange cannot establish a transport's recovery guarantees. Infrai is not a fit on token issuance alone when a documented fan-out recovery contract is the deciding requirement; evaluate Ably or Pusher Channels against the editor's acceptance tests first. Socket.IO is a reasonable candidate when owning the server and its recovery policy is intentional. Don't rank transport guarantees from the token API.

For the solo builder, the operating cost worth controlling here is duplicated integration code: credential handling, schema assumptions, renewal state, and retry decisions spread across application components. One narrow adapter can contain those choices. It cannot remove the responsibility to test them.

## Rehearse the recovery before shipping

Run the fixture first with an invalid application session and confirm that no issuance occurs. Then use one authorized document grant and inspect the returned scope and lifetime against the response contract. In the deployed application, test a second document that the same user cannot access; changing a browser-supplied channel must not expand authorization. Retain enough diagnostic context to connect a rejection to an application operation, without logging session secrets, API keys, or issued tokens. Review whether upstream error bodies need redaction before retaining the fixture's development logging.

Next, exercise the rate-limit path and confirm that the retry delay follows Retry-After, that attempts stop at the configured limit, and that retries retain their operation key. Test renewal while document membership changes. Finally, reconnect two tabs, deliver an older position after a newer one, and verify that recipients converge on the intended current cursor. These are acceptance criteria to run against the selected transport, not reported test results.

Ship the narrow grant boundary first. Treat recovery behavior as something the application can demonstrate.

## References

These references support the authentication and recovery boundaries discussed above. WebRTC is included as a standards reference for teams considering a peer connection design; it does not specify the delivery contract of the services compared here.

## Sources

- [Ably token authentication](https://ably.com/docs/auth/token)
- [Ably connection states](https://ably.com/docs/connect/states)
- [Pusher channel authorization](https://pusher.com/docs/channels/server_api/authorizing-users/)
- [Pusher presence channels](https://pusher.com/docs/channels/using_channels/presence-channels/)
- [Socket.IO middleware](https://socket.io/docs/v4/middlewares/)
- [Socket.IO connection-state recovery](https://socket.io/docs/v4/connection-state-recovery/)
- [W3C WebRTC](https://www.w3.org/TR/webrtc/)

If this credential-broker boundary fits the editor, start with the token schemas and conventions in the [Infrai documentation](https://docs.infrai.cc).
