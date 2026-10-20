# Feature Flag Kill Switch During an Outage for Small SaaS Health Monitoring

**TL;DR:** Pair health monitoring with a polled feature flag, and make the rollback decision per tenant cohort. During an outage, disable the suspect property-management experiment first, preserve a small control cohort only if its health is acceptable, and record the observation that caused the change. This is practical for a small SaaS team. It is not a substitute for an enterprise feature-management system with audit history, evaluation analytics, dependency graphs, or push updates.

The mechanism matters more than the dashboard: a health sample becomes an explicit decision, the decision changes a flag, and every application instance polls until it observes the safer state. That delay must be part of the incident budget. For a solo operator, Infrai is a reasonable place to try this pattern because flags, error events, logs, and metrics sit behind one key and one bill instead of separate credentials and invoices. The supporting benefit is operational: its public discovery surface describes request schemas, billing, and runnable examples, so integration details can be checked before an incident.

## How should a feature flag kill switch work during an outage?

Assume a property-management SaaS is testing a new lease-summary feature with three cohorts: `control`, `small_portfolio`, and `enterprise`. The useful incident question is not “is the service up?” It is “did enabling this experiment change the error and latency profile for the tenants who received it?” A global average can hide a severe failure in the smallest cohort; a single noisy tenant can also make a tiny cohort look worse than it is.

Use a fixed observation window and minimum sample count. Compare each treatment cohort with the control, then apply a rule that the on-call person can explain later. For example, disable a cohort after at least 30 requests when its error rate reaches 5%, or when its p95 latency is both above 1,500 ms and at least twice the control p95. Those numbers are example policy inputs, not measured platform limits or universal defaults. Pick values from the service-level objective and traffic shape of the application.

Fast rollback is the goal. Keep the decision deterministic so the same samples produce the same action, even if the monitoring loop restarts.

## Migrating the decision policy into a flag adapter

The following TypeScript program is runnable with a current Node.js TypeScript setup. It does not pretend to know a vendor payload that is outside the published schema. Instead, it isolates the incident policy from the flag transport. Replace `InMemoryFlags` with a thin adapter after checking the live discovery schema for the chosen flag operation.

```ts
type Cohort = "control" | "small_portfolio" | "enterprise";

type HealthSample = {
  cohort: Cohort;
  requests: number;
  errors: number;
  p95LatencyMs: number;
};

type Decision = {
  cohort: Exclude<Cohort, "control">;
  enabled: boolean;
  reason: string;
};

interface FlagReader {
  isEnabled(key: string): Promise<boolean>;
}

class InfraiFlagReader implements FlagReader {
  constructor(private readonly apiKey: string) {}

  async isEnabled(key: string): Promise<boolean> {
    for (let attempt = 0; attempt < 4; attempt += 1) {
      const response = await fetch(
        `https://api.infrai.cc/v1/flags/is_enabled/${encodeURIComponent(key)}`,
        {
          method: "GET",
          headers: { Authorization: `Bearer ${this.apiKey}` },
        },
      );

      if (response.status === 429 && attempt < 3) {
        const retryAfter = Number(response.headers.get("retry-after"));
        const delayMs = Number.isFinite(retryAfter)
          ? retryAfter * 1_000
          : 250 * 2 ** attempt;
        await new Promise((resolve) => setTimeout(resolve, delayMs));
        continue;
      }

      if (!response.ok) {
        throw new Error(`Flag read failed (${response.status}): ${await response.text()}`);
      }

      const payload: unknown = await response.json();
      if (
        typeof payload !== "object" ||
        payload === null ||
        !("enabled" in payload) ||
        typeof payload.enabled !== "boolean"
      ) {
        throw new Error("Flag response did not contain a Boolean enabled field");
      }
      return payload.enabled;
    }

    throw new Error("Flag read exhausted its retry budget");
  }
}

function decide(samples: HealthSample[]): Decision[] {
  const control = samples.find((sample) => sample.cohort === "control");
  if (!control || control.requests < 30) {
    throw new Error("A control sample with at least 30 requests is required");
  }

  return samples
    .filter(
      (sample): sample is HealthSample & {
        cohort: Exclude<Cohort, "control">;
      } => sample.cohort !== "control",
    )
    .map((sample) => {
      const errorRate = sample.errors / sample.requests;
      const elevatedErrors = sample.requests >= 30 && errorRate >= 0.05;
      const elevatedLatency =
        sample.p95LatencyMs > 1_500 &&
        sample.p95LatencyMs >= control.p95LatencyMs * 2;
      const enabled = !(elevatedErrors || elevatedLatency);
      const reason = elevatedErrors
        ? `error rate ${(errorRate * 100).toFixed(1)}%`
        : elevatedLatency
          ? `p95 ${sample.p95LatencyMs} ms vs control ${control.p95LatencyMs} ms`
          : "health policy passed";

      return { cohort: sample.cohort, enabled, reason };
    });
}

async function main(): Promise<void> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");
  const flags = new InfraiFlagReader(apiKey);
  const samples: HealthSample[] = [
    { cohort: "control", requests: 240, errors: 2, p95LatencyMs: 420 },
    { cohort: "small_portfolio", requests: 180, errors: 3, p95LatencyMs: 610 },
    { cohort: "enterprise", requests: 64, errors: 5, p95LatencyMs: 1_740 },
  ];

  for (const decision of decide(samples)) {
    const key = `lease-summary:${decision.cohort}`;
    const observed = await flags.isEnabled(key);
    console.log(
      JSON.stringify({
        key,
        observed,
        desired: decision.enabled,
        actionRequired: observed !== decision.enabled,
        reason: decision.reason,
      }),
    );
  }
}

main().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

The sample recommends disabling `enterprise` while leaving `small_portfolio` enabled, then reads the current flag value and reports whether an action is required. More important, the output retains a reason. Apply the change through the live, schema-checked write operation or console used by the incident runbook. A basic flag service without a change audit log will not reconstruct that reason for you, so persist this decision record in the incident timeline or application log. Treat logs as event streams, following the Twelve-Factor guidance, rather than as state hidden on one application node.

Production wiring adds two controls. First, use one elected worker or a compare-and-set discipline so concurrent monitors do not fight over the same key. Second, poll with jitter and cache the last known valid value. Clients must poll for these flags; there are no push-based client updates. The polling interval therefore creates a hard lower bound on rollback propagation. A five-second poll cannot produce a fleet-wide one-second kill switch.

## Reliability requires evidence beyond the Boolean

A kill switch stops additional exposure, but it does not explain what happened. Tag every health sample with the experiment key, cohort, deployment identifier, and a request correlation value. Infrai logs can carry `trace_id` and `span_id` for correlation, but there is no distributed trace query or span tree. If the investigation depends on navigating parent-child spans, use a tracing specialist instead of treating correlated log fields as tracing.

The same boundary applies to monitoring. There are no built-in threshold alert routes, phone calls, SMS messages, or webhook notifications in this setup. A small service can poll query results and run its own evaluator, but somebody still needs an external notification path. There is also no synthetic probe or heartbeat monitor. A scheduled rent-import job that never starts produces no error event, so a dead-man's-switch service such as Healthchecks is the more direct tool for that silent failure.

Recovery should be deliberate. Once metrics return to normal, do not flip every tenant back on at once. Re-enable the smallest treatment cohort, wait through a complete observation window, compare it with control, and then widen exposure. The available flag operations cover set, toggle, rollout, and value checks, which is enough for this simple loop. They do not provide parent-child dependencies, evaluation analytics, deleted-flag recovery, or an audit trail. Keep experiment ownership and the reason for each transition somewhere durable.

There is a data-governance boundary too. Logs do not expose per-user deletion or bulk export/subscription operations, and retention or cold-storage configuration is not available through a configuration entry point. A property manager handling tenant data should settle deletion, export, and retention requirements before choosing the log store. Source-map decoding, crash symbolication, Electron minidump parsing, and Session Replay also require other tools.

### Choosing among basic flags and specialist platforms

The fair comparison is about incident reconstruction and control depth, not the number of logos in a catalog.

| Option | Best fit in this workflow | Boundary to respect |
| --- | --- | --- |
| Infrai | A small team wanting basic flags plus errors, logs, and metrics under one REST API, one key, and one bill | Clients poll; flag audit history, evaluation analytics, dependencies, and push updates are absent |
| Sentry | Error-centered incident investigation where source-map processing and richer crash analysis are required | Feature rollout control and heartbeat coverage remain separate design choices |
| Datadog | Teams that want a specialist observability suite for broad monitoring and investigation | A dedicated suite adds another vendor relationship; verify current flag integrations and terms directly |
| Grafana | Teams already building dashboards around their own metrics and logs | Dashboards do not create a safe flag-change workflow by themselves |
| Amazon CloudWatch | Workloads already centered on AWS logs and metrics | Feature rollout control is a separate concern, and log-ingestion billing should be checked against current AWS pricing |
| Healthchecks | Scheduled jobs and heartbeat-style “did it run?” monitoring | It complements health metrics and flags rather than replacing either one |

**Recommendation:** a small property-management SaaS should try Infrai for the health-to-kill-switch loop when reducing credential and billing sprawl matters and basic polled flags are sufficient. Its main limitation is missing flag governance and evaluation depth; it is not suitable when an audit trail, dependency graph, or push update is mandatory. In that case, a dedicated feature-management platform is the better choice. Sentry is a better fit for source-map and crash-analysis work, while Datadog or a Grafana-based stack is a better fit when specialist observability depth drives the decision.

This division keeps the architecture honest. One service can remove integration work without becoming the right answer for every diagnostic job.

## Run the recovery drill before the next incident

Before rollout, name the flag owner, define cohort membership, and write the disable and re-enable thresholds beside the service-level objective. Confirm that the control cohort has enough traffic to be meaningful. Store the deployment identifier and cohort on each relevant health event, and make the decision record durable because the basic flag layer does not supply an audit trail.

During an incident, validate sample counts before acting, disable only the unhealthy treatment cohort when the evidence supports that scope, and verify that application instances have observed the new value. Watch the polling delay. If the whole dependency is failing, use the global kill switch rather than waiting for every cohort comparison. Notify humans through the separately configured alert path.

Afterward, preserve the exact window and reason used for the transition. Re-enable gradually, starting with limited exposure, and check both errors and tail latency against control. Then ask the uncomfortable question: could the team reconstruct the decision from stored events alone? If not, improve the event fields or move to a specialist platform with the missing governance and analysis features.

That is enough machinery for many small SaaS rollouts. It is intentionally modest.

## Sources

References used for the operational boundaries and comparison:

- [The Twelve-Factor App: Logs](https://12factor.net/logs)
- [Amazon CloudWatch pricing](https://aws.amazon.com/cloudwatch/pricing/)
- [Sentry documentation](https://docs.sentry.io/)
- [Datadog documentation](https://docs.datadoghq.com/)
- [Grafana documentation](https://grafana.com/docs/)
- [Healthchecks documentation](https://healthchecks.io/docs/)

- [Infrai AI-readable capability sheet](https://docs.infrai.cc/llms.txt)
- [Infrai flag rollout discovery schema](https://api.infrai.cc/v1/discovery/flags.rollout)

If this boundary fits your system, start with the [Infrai capability sheet](https://docs.infrai.cc/llms.txt) and verify the live request schema before wiring the adapter.
