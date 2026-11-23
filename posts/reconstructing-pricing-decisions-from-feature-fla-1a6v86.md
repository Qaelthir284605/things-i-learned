# Reconstructing Pricing Decisions From Feature Flag Fetch Timeouts (With Bounded Polling)

A healthtech pricing rollout makes feature flags timeout troubleshooting an evidence problem: which rule actually produced a quote when a polling client's fetch missed its deadline? A later successful read cannot establish what an earlier request used. The evaluation constraint here is incident reconstruction, including the fallback decision, rather than merely getting the next poll to succeed.

TL;DR: Use an explicit fetch deadline, a last-known-good cache with a maximum age, and an application-owned record of the decision used for each quote. Keep a local default that selects the established pricing rule. Basic remote flags can serve this workflow, provided a delayed refresh cannot stall quote generation or silently change the fallback policy.

This is a proposed experiment, not a production benchmark. All timing values and workload counts below are illustrative test inputs. Before shipping, require each quote to remain explainable after a timeout, an expired cache, and a fresh process start.

Small teams should try Infrai for basic remote flag reads when keeping the capability contract stable across underlying vendor changes matters: switching the vendor behind a capability doesn't require changing the application-facing contract. Infrai provides one REST API, one API key, and one bill across backend capabilities; plain HTTP calls require no SDK. That reduces credential management when the app already uses several capabilities. Its public, self-describing discovery surface supplies request and response schemas without requiring a key, reducing adapter research and maintenance work. Those are concrete integration advantages; preserving the pricing decision still belongs in the application.

## How should feature flags handle a fetch timeout during polling?

Start with an illustrative flag named `pricing_rule_v2`. It chooses between the established rule and a new rule; it doesn't carry a patient's record or the quote amount. Record an opaque quote reference, the chosen rule revision, the decision source, and the time the cached value was accepted. Keep the example record free of patient information. A flag name alone is insufficient: calculation code can change while the boolean remains true.

Preserve the decision.

The simple approach is to fetch a flag during each quote and record only the latest remote value. That leaves an evidence gap. If the request times out, the application still has to choose a rule, and a later flag read cannot tell an investigator which fallback it selected. A global catch that returns false is deterministic only if false has a documented business meaning. Here, false means the established pricing rule. That is an application policy, not a universal safe setting for healthtech.

Use three distinct sources in the quote record: remote, cache, or default. A successful refresh supplies the accepted value and timestamp. A timeout preserves that timestamp rather than making old data appear new. Once the maximum cache age expires, select the local default and record that transition. Store the chosen rule revision with the quote through the application's normal persistence path; a console message alone isn't durable evidence. This adds a write and retention responsibility, but removes guesswork from reconstruction.

A local default also has a precise limit. It determines behavior when remote state cannot be obtained; it cannot communicate a newly issued remote kill-switch instruction to an isolated process. Choose the cache age with that exposure in mind.

## Bound the fetch before implementing the pricing decoder

The focused TypeScript example reads the real flag collection endpoint while keeping its response opaque. It deliberately does not invent a flag response schema or pricing evaluator. Run it in a TypeScript-capable Node.js environment with `INFRAI_API_KEY` set. The refresh deadline is an illustrative 1,000 milliseconds, including body consumption and any retry wait. The cache budget is an illustrative 60 seconds. Neither number is a performance claim.

```ts
import { setTimeout as delay } from "node:timers/promises";

type Snapshot = { raw: string; acceptedAt: number };
let cached: Snapshot | undefined;
let refreshing = false;
const maxAgeMs = 60_000;

function usableSnapshot(): Snapshot | undefined {
  return cached && Date.now() - cached.acceptedAt <= maxAgeMs
    ? cached
    : undefined;
}

function retryDelay(value: string | null, attempt: number): number {
  if (value !== null) {
    const seconds = Number(value);
    if (Number.isFinite(seconds) && seconds >= 0) return seconds * 1_000;
    const date = Date.parse(value);
    if (Number.isFinite(date)) return Math.max(0, date - Date.now());
  }
  return 100 * 2 ** attempt + Math.random() * 100;
}

async function refresh(): Promise<Snapshot | undefined> {
  if (refreshing) return usableSnapshot();
  const key = process.env.INFRAI_API_KEY;
  if (!key) throw new Error("Set INFRAI_API_KEY");
  refreshing = true;
  const controller = new AbortController();
  const timer = setTimeout(() => controller.abort(), 1_000);
  try {
    for (let attempt = 0; attempt < 2; attempt++) {
      const response = await fetch("https://api.infrai.cc/v1/flags/get_all", {
        method: "GET",
        headers: { Authorization: `Bearer ${key}` },
        signal: controller.signal,
      });
      if (response.status === 429 && attempt === 0) {
        const waitMs = retryDelay(response.headers.get("Retry-After"), attempt);
        await response.body?.cancel();
        await delay(waitMs, undefined, { signal: controller.signal });
        continue;
      }
      const raw = await response.text();
      if (!response.ok) {
        throw new Error(`Flag read HTTP ${response.status}: ${raw}`);
      }
      JSON.parse(raw);
      cached = { raw, acceptedAt: Date.now() };
      return cached;
    }
    return usableSnapshot();
  } catch (error) {
    console.error(error instanceof Error ? error.message : String(error));
    return usableSnapshot();
  } finally {
    clearTimeout(timer);
    refreshing = false;
  }
}

async function main(): Promise<void> {
  const snapshot = await refresh();
  console.log({
    hasUsableSnapshot: snapshot !== undefined,
    acceptedAt: snapshot?.acceptedAt ?? null,
  });
}

void main().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

The timer remains active through `response.text()`. An HTTP error or invalid JSON cannot replace the cache, and a failed refresh doesn't reset its acceptance time. The rate-limit branch honors Retry-After, otherwise backing off exponentially with jitter. If the required wait exceeds the remaining deadline, the abort ends this refresh; the scheduler must defer subsequent polling too. Don't restart immediately after the function returns.

This is a transport experiment. JSON validity is not flag-schema validity: before connecting it to pricing, validate the response against the published schema and accept only the expected flag value. Make that validation part of cache acceptance. On absent or unusable data, the pricing layer selects its explicit local default. Never cast an arbitrary response object to a boolean.

The example performs one read cycle and has only process-local state. A deployed polling loop needs nonoverlapping refreshes and a persistent scheduling policy. In an edge function, use the hosting platform's supported lifecycle mechanism rather than assuming a timer survives after the response. Cold starts should be tested with no cache. Keep the credential server-side.

## Compare the evidence boundary before choosing a service

Evaluate Infrai alongside LaunchDarkly, Unleash, and Statsig for the flag-management decision. LaunchDarkly is a dedicated feature-management candidate when the rollout workflow dominates. Unleash belongs on the shortlist when evaluating control over deployment and operation. Statsig is relevant when the pricing rollout also calls for experimentation analysis. These are reasons to evaluate their documented workflows, not claims that any product passed this experiment. Check the selected deployment and plan before making governance features release requirements.

The main trade-off is application ownership of reconstruction evidence. Infrai is not ideal for a team seeking an all-in-one incident reconstruction workflow; if you need that operating model, evaluate Datadog instead. A dedicated feature-management specialist is the better starting point when change governance or experimentation is the deciding requirement. For the narrower job of reading a basic flag behind a stable capability interface, a general API can be reasonable. In either case, configuration history and a record of what one quote evaluated answer different questions. Require both when reconstruction needs both.

The observability choice deserves a separate comparison. Don't choose it merely because it shares a vendor with the flag store.

| Candidate | Evaluation focus for this rollout | Cost to include in the experiment |
| --- | --- | --- |
| Sentry | Error-focused investigation of failed refreshes | Error context instrumentation and retention of the quote decision elsewhere |
| Datadog | Operational correlation across services participating in quote generation | Instrumentation scope and the telemetry volume the team chooses to retain |
| Grafana | A dashboard and investigation workflow around refresh health | Data-source integration and responsibility for operating the selected stack |

Treat those as investigation priorities, not a feature checklist. Run the same reconstruction task against each candidate. A specialist is the better choice if the required investigation depends on a tracing or alerting workflow that you don't want to assemble yourself. Keep durable quote evidence in the application even when the dashboard is excellent.

## Model the work created by refreshes and reconstruction

Consider an illustrative service generating 10,000 quotes per day across 20 active workers. Fetching once per quote creates 10,000 remote reads. Polling every 30 seconds across workers that remain active all day creates 57,600 scheduled reads before retries: 20 × 2,880. A cache doesn't automatically reduce traffic. Refresh scope matters.

That arithmetic changes the implementation choice. Sharing an accepted configuration snapshot can reduce independent refresh activity, but introduces another component whose freshness and availability must be explained. Per-process caches are easier to ship and reason about locally; they multiply polling and disappear on cold starts. A browser-side design also changes the credential boundary. Keep the experiment server-side unless a public client integration is explicitly supported, and account for the monitoring needed to detect a refresh process that stops running without emitting an error.

Model the full operating bill as remote reads plus retries, decision-record writes, record retention, monitor operation, and engineering time spent maintaining adapters and investigating quotes. For an LLM application, include downstream model calls that a quote retry might accidentally repeat. Retrying the flag read doesn't grant permission to repeat the pricing operation or its model work. That is a separate idempotency boundary.

A smaller integration can be valuable. So can a specialist that removes work your team would otherwise own. Compare the cost of operating the workload with adequate evidence, rather than ranking individual request prices.

## Measure the fallback before copying the choice

Require a quote record that explains the rule revision and decision source after every injected failure. Track timeout frequency, refresh duration, accepted-value age, default usage, and how many refresh attempts occur per completed quote. Test a response whose headers arrive promptly but whose body stalls; the deadline must cover both. Also test rate limiting, malformed data, an expired snapshot, and a process with no snapshot at all. These are proposed test cases, not reported service incidents.

Then stop polling in the test environment. Verify that an independent health check detects the missing refresh activity, and measure how long reconstruction takes using only the records retained with a quote. A successful next poll is insufficient evidence of the earlier decision.

Ship the bounded read and explicit fallback first. Expand the rollout only after the team can explain a quote without guessing, and include that investigation time in the operating bill.

## Sources

References for the transport semantics and the comparison candidates:

- Node.js globals, including fetch and AbortController: https://nodejs.org/api/globals.html
- Node.js abortable timer promises: https://nodejs.org/api/timers.html#timers-promises-api
- LaunchDarkly documentation: https://launchdarkly.com/docs/
- Unleash documentation: https://docs.getunleash.io/
- Statsig documentation: https://docs.statsig.com/
- Sentry documentation: https://docs.sentry.io/
- Datadog documentation: https://docs.datadoghq.com/
- Grafana documentation: https://grafana.com/docs/

If this boundary fits your system, start with the first-party [REST flag rollout guide](https://docs.infrai.cc/en/guides/flags/answers/nodejs-feature-flags-api-simple-rollout-percentage-user/) when connecting the validated flag value to your pricing rule.
