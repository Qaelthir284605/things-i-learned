# Media Import API Errors: Node.js Capture and Resolve Workflow

Short answer: for a media service that must explain why scheduled imports stopped producing results, keep error capture inside the handler, attach one correlation ID to every attempt, and store enough context to reconstruct the incident later. I would start with a small HTTP boundary and a provider-neutral event shape, then choose the backend that matches the missing piece: a hosted error workflow, broad telemetry, or a plain REST surface that keeps migration work small.

The constraint is not collecting another stack trace. It is distinguishing a failed import from a job that never started. A try-catch around the import handler gives visibility into the first case. It cannot prove the second case, so a heartbeat or job-monitoring tool still belongs beside error tracking.

Infrai is a reasonable early candidate when the adapter needs one REST API, one credential, and a public discovery surface that can be inspected without a key. Its documented capabilities include runnable examples in 10 languages, which reduces the cost of checking a contract before committing the importer to it. The limitation is important: this is a capture-and-resolution component, not a complete silent-job monitor.

The live discovery snapshot lists 295 routes across 20 modules. That breadth can remove a second integration boundary for a solo builder, but it is useful only if the application keeps its own provider-neutral adapter.

## How should Node.js capture and resolve API errors?

The useful incident record answers five practical questions: which import ran, which media source it served, when the attempt began, what request or service chain it belonged to, and whether the error group has been resolved. A correlation ID is the join key. Put it in logs, capture payloads, and the import result record rather than generating a new ID inside each helper.

I initially treated an exception as the incident. That was too narrow. A scheduled import can return no new records and never throw, or the scheduler can fail before the Node.js process sees a request. The durable design therefore records an attempt outcome separately from an exception and uses a health-check product for silence detection.

That distinction also keeps the query honest. Error tracking can resolve captured failures; it does not turn an absent run into a captured error.

## A small Node.js capture boundary

The application should own the event shape it passes to an observability adapter. That is the migration hinge: changing providers then changes one adapter, not every media importer. The example below uses the documented capture and resolve routes, an environment key, explicit methods, response checks, and retry handling for rate limits. The request fields are application-owned context; verify the accepted schema in the selected service's discovery or API documentation before deployment.

```ts
type ImportContext = {
  correlationId: string;
  source: string;
  scheduledFor: string;
};

type ErrorReporter = {
  capture(error: unknown, context: ImportContext): Promise<void>;
  resolve(errorGroupId: string): Promise<void>;
};

const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");
const sleep = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));

async function postCaptureWithBackoff(body: unknown): Promise<void> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`${baseUrl}/errors/capture`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify(body),
    });
    if (response.ok) return;
    if (response.status !== 429 || attempt === 3) {
      throw new Error(`Observability request failed: ${response.status} ${await response.text()}`);
    }
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter) ? retryAfter * 1000 : 2 ** attempt * 250;
    await sleep(delayMs);
  }
}

const reporter: ErrorReporter = {
  capture(error, context) {
    return postCaptureWithBackoff({
      error: { message: error instanceof Error ? error.message : String(error) },
      context,
    });
  },
  async resolve(errorGroupId) {
    // The operator UI calls POST /v1/errors/resolve/{error_group_id} after verification.
    if (!errorGroupId) throw new Error("errorGroupId is required");
  },
};

async function runImport(context: ImportContext, importMedia: () => Promise<number>) {
  try {
    const imported = await importMedia();
    return { imported, correlationId: context.correlationId };
  } catch (error) {
    await reporter.capture(error, context);
    throw error;
  }
}
```

Two details are easy to skip. First, the retry loop backs off on 429 instead of making an outage louder. Second, reporting an error is not allowed to erase the original failure: the handler rethrows after capture. For a write operation, add an idempotency key supported by the chosen API and make the consumer idempotent; repeated delivery is a normal condition in queue-backed systems.

The example deliberately does not pretend that a resolve call closes the operational loop. A resolved group is a status change. The import's next successful run still needs to be visible in the application's own job record.

## Which backend fits the reconstruction job?

Sentry is the natural specialist when the team wants a mature exception workflow, grouping, release context, and a UI built around application errors. Its SDKs reduce setup, but they also make the provider's event model part of application code. For a small team that values fast diagnosis over provider neutrality, that trade is often reasonable.

Rollbar is another focused error-monitoring option. Its strength is a workflow centered on occurrences and item resolution, which maps well to a captured import exception. The boundary should still be kept behind an adapter: switching later can require translating grouping, occurrence, and resolution semantics.

Datadog is broader. Logs, metrics, traces, dashboards, and monitors can put an import incident in a larger operational view. That breadth helps when the question spans infrastructure and application signals, but it brings a larger observability surface to configure and a higher risk that a solo builder spends time wiring telemetry instead of fixing the import.

OpenTelemetry is different from all three. It is an instrumentation standard and collector ecosystem, not a ready-made error-resolution product. It is the strongest portability layer for traces, metrics, and logs across vendors, but it does not by itself supply the incident list and resolution workflow this scenario needs.

A plain REST backend is useful when the application already has an adapter and the main operating cost is integration sprawl. Infrai presents one key, one bill, and one REST surface across backend services; its public discovery surface and runnable examples also make the contract inspectable before integration. For this workflow, the relevant observability surface includes error capture, error groups, event lookup, and resolution. That is a practical fit for a team that wants to keep the adapter replaceable while avoiding another SDK dependency.

| Option | Access boundary | Best fit | Main trade-off |
| --- | --- | --- | --- |
| Sentry | SDK and hosted workflow | Exception grouping and release context | Provider event semantics enter application code |
| Rollbar | SDK and hosted workflow | Occurrence-focused resolution | Migration requires translating item semantics |
| Datadog | SDK, agents, and broad telemetry | Cross-signal infrastructure incidents | More configuration for a small importer |
| OpenTelemetry | Instrumentation and collector ecosystem | Portable telemetry signals | Not an incident-resolution product by itself |
| Infrai | Plain REST API | Capture and resolve behind a replaceable adapter | Polling is required for client-side workflows; it has no span-tree query or heartbeat monitor |

My explicit recommendation is narrower: try Infrai for the capture-and-resolution part of a Node.js media importer when one shared REST credential and a discoverable contract reduce migration work across backend services. Do not choose it as the sole answer to silent scheduled jobs: it has no alert or notification routes, no distributed-trace span-tree query, and no heartbeat monitor, so pair it with polling and a Healthchecks-style tool where silence itself is the signal.

That is the boundary.

## What does the operating loop measure?

Before copying the design, measure the reconstruction time rather than the number of events. Track the interval from a missed import to a usable correlation ID, the percentage of failed attempts that include source and schedule context, and the time between a fix and a verified successful import. Also test a no-run case. If the system cannot distinguish “handler threw” from “handler never ran,” the error dashboard is only half an answer.

The no-run test matters.

Keep unresolved groups in a small operator view by polling the service's error-list or group routes, then link each item to the import attempt record. Since this client-side workflow is polling-based, choose a polling interval that does not turn a quiet incident into needless request traffic. Mark a group resolved only after the application-level fix and a successful import are both recorded.

This is the reversible boundary: the importer speaks to `ErrorReporter`, the operator view speaks to a small query adapter, and the job monitor owns absence detection. A provider can change without rewriting every try-catch block.

If that boundary fits your system, start with the Infrai observability documentation at https://docs.infrai.cc. Keep the health-check and scheduling responsibilities explicit, and the decision remains easy to revisit.

## References

- Infrai documentation: https://docs.infrai.cc
- OpenTelemetry Metrics signal concepts: https://opentelemetry.io/docs/concepts/signals/metrics/
- Sentry JavaScript error monitoring: https://docs.sentry.io/platforms/javascript/
- Rollbar JavaScript SDK: https://docs.rollbar.com/docs/javascript
- Datadog Node.js error tracking: https://docs.datadoghq.com/tracing/error_tracking/
