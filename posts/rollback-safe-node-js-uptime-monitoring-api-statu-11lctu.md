# Rollback-Safe Node.js Uptime Monitoring: API Status, Cron Heartbeats, Logs, and Metrics

Short answer: monitor a Node.js notification service with three independent signals: pull an API status endpoint for process uptime, require a cron job heartbeat after completed delivery work, and use logs plus metrics to explain failures. Keep those signals stable across releases so a rollback restores application code without erasing the evidence that tells operators whether delivery recovered.

One green endpoint isn't enough.

For a B2B SaaS notification service, “the API is up” and “customers received notifications” are different claims. The first is a process observation. The second depends on scheduled selection, queueing, provider handoff, and a completion deadline. Rollback safety adds another constraint: monitoring must distinguish a bad release from old work still draining, without changing meaning when the previous version returns.

## Start with a before-and-after delivery model

Before: one uptime monitor calls `/status`, gets `200`, and paints the service green. A scheduler can stop invoking the delivery sweep while the web process keeps answering. The monitor has measured availability and missed the failed job.

After: draw the system in words as two lanes. The request lane is `external probe -> status endpoint -> liveness result`. The delivery lane is `scheduler -> job run -> notification outcomes -> completion heartbeat -> deadline evaluator`. Structured logs attach detail to both lanes, while metrics show rates and ages over time. Each signal has one job, so an operator can read a failure without guessing what “healthy” was supposed to mean.

This separation also makes a rollback legible. Give every log and metric a low-cardinality `release` value, but keep the heartbeat identity tied to the stable job name, such as `notification_delivery_sweep`, rather than a deployment identifier. During a rollback, old and new releases may briefly emit telemetry together. Their release labels explain the overlap; the stable job identity preserves one continuous deadline. Never treat deployment itself as proof that the job succeeded.

The deadline should come from the actual schedule and the longest acceptable delivery delay. If a sweep runs every five minutes, the team still has to decide how much lateness violates its customer promise. I'm not sure a universal grace period exists, because queue depth, provider latency, and retry policy differ. Resolve that uncertainty with the service objective and observed run-duration distribution, then store the chosen threshold as reviewed configuration.

## How should a Node.js API combine uptime monitoring, cron job heartbeats, logs, and metrics?

Use a cheap status handler for liveness and a separate snapshot for delivery health. Liveness answers whether the event loop can serve a request. Delivery health reports the last completed run and its age, but it should not turn every dependency wobble into a process restart. An external probe supplies the view from outside the process; an in-process timer cannot report that its own process disappeared.

Here is a focused TypeScript example. It deliberately keeps transport details behind small interfaces, so the same semantics survive a monitoring backend change or an application rollback.

```ts
type DeliveryOutcome = "delivered" | "retryable" | "rejected";

type RunSummary = {
  job: "notification_delivery_sweep";
  runId: string;
  release: string;
  startedAt: string;
  completedAt: string;
  outcomes: Record<DeliveryOutcome, number>;
};

interface Telemetry {
  log(event: Record<string, unknown>): void;
  increment(name: string, labels: Record<string, string>, value?: number): void;
  set(name: string, labels: Record<string, string>, value: number): void;
}

interface Heartbeats {
  complete(job: string, completedAt: string, runId: string): Promise<void>;
}

const release = process.env.RELEASE_ID ?? "unknown";
let lastCompletedRun: RunSummary | undefined;

export async function recordCompletedSweep(
  summary: RunSummary,
  telemetry: Telemetry,
  heartbeats: Heartbeats,
): Promise<void> {
  const labels = { job: summary.job, release: summary.release };

  for (const [outcome, count] of Object.entries(summary.outcomes)) {
    telemetry.increment(
      "notification_delivery_attempts_total",
      { ...labels, outcome },
      count,
    );
  }

  const completedAtSeconds = Date.parse(summary.completedAt) / 1_000;
  telemetry.set("notification_delivery_last_completion_timestamp_seconds", labels, completedAtSeconds);
  telemetry.log({ event: "delivery_sweep_completed", ...summary });

  await heartbeats.complete(summary.job, summary.completedAt, summary.runId);
  lastCompletedRun = summary;
}

type StatusResponse = {
  statusCode: number;
  body: {
    live: true;
    release: string;
    uptimeSeconds: number;
    delivery: { lastCompletedAt: string | null; ageSeconds: number | null };
  };
};

export function getStatus(now = Date.now()): StatusResponse {
  const completedAt = lastCompletedRun?.completedAt ?? null;
  const ageSeconds = completedAt === null ? null : (now - Date.parse(completedAt)) / 1_000;

  return {
    statusCode: 200,
    body: {
      live: true,
      release,
      uptimeSeconds: process.uptime(),
      delivery: { lastCompletedAt: completedAt, ageSeconds },
    },
  };
}
```

Call `recordCompletedSweep` only after the run reaches the completion boundary your team defined. For an at-least-once delivery pipeline, “completed” may mean every selected notification was either handed off or recorded for a retry; it should not mean “the function started.” The `runId` lets a receiver deduplicate repeated heartbeat submissions after ambiguous client-side failures. Don't put a tenant ID, notification ID, or run ID into metric labels: those unbounded values belong in logs, where an operator can search them without multiplying time series.

The metric names follow a useful convention: counters end in `_total`, timestamps name their base unit in `_seconds`, and each name describes a single quantity. Prometheus documents those naming principles even when the eventual metrics transport is different. The log carries the high-detail run summary. The metric carries bounded dimensions for alerting. The heartbeat carries deadline evidence. Crisp boundaries help.

There is one subtle ordering choice. In this example, the code records telemetry, acknowledges the heartbeat, and only then updates the local status snapshot. That means the status response never claims a completion that the deadline system has not accepted. A production implementation should also persist the last completion outside process memory if operators expect it to survive restarts; the in-memory variable here demonstrates the contract, not durable storage.

## Make the alert survive deployment and rollback

A rollback-safe alert evaluates service behavior, not the mere presence of a new release. Define two independent conditions: the public status probe has failed its availability window, or the stable delivery heartbeat is older than its deadline. Add an outcome-rate alert only when the denominator is large enough to make a ratio meaningful. A tiny customer batch can turn one rejection into a dramatic percentage with little operational value.

Deploy monitoring changes in two phases. First, emit the new metric or field while the current alert still runs. Confirm that both release versions produce compatible data. Then switch the evaluator. Remove the old signal in a later release, after the rollback window closes. This is the same expand-and-contract pattern used for database changes — readers first tolerate both shapes, writers later stop producing the old one.

Keep the alert's job key and deadline configuration outside the release artifact that is being rolled back. Otherwise a rollback can quietly restore yesterday's code and yesterday's threshold at the same time, leaving the team unsure which change cleared the page. Version the configuration, record who changed it in the deployment system, and make the alert annotation show the active threshold and last completion time.

Test the failure paths before production. A useful pre-release check pauses scheduling while leaving the API process reachable: the status probe should remain green, then the heartbeat deadline should fire. A second check stops the API while a worker continues completing jobs: uptime should fail, while the heartbeat remains current. Finally, stage a concrete rollback drill. At 10:00, let release A finish a sweep and record run `run-1000`. At 10:05, deploy release B beside A and let B record `run-1005`; the evaluator should see a later completion for the same `notification_delivery_sweep` identity, while the logs show a different release. At 10:07, direct traffic back to A without clearing monitoring state. When A completes `run-1010`, the job timeline should advance again rather than split into a second monitor or revert to the 10:00 timestamp. Now pause the scheduler and wait past the reviewed deadline. The missed-heartbeat alert should contain the last completion time and stable job name, while the API probe remains healthy. This drill catches the dangerous coupling: if a release-specific heartbeat key, an in-memory evaluator, or deployment automation silently resets the deadline, the test won't produce that final alert. Fix the contract before shipping. The evaluator should preserve one job timeline through the entire exercise, and logs should retain enough release context to explain which version handled each run.

Sampling needs care here. A sampled trace can help reconstruct a slow notification, but the absence of a sampled span is not evidence that the cron job failed. OpenTelemetry distinguishes head sampling, decided near trace creation, from tail sampling, decided after more trace data is available. Either policy may discard a trace. Keep heartbeat and service-level counters unsampled; use traces as diagnostic context rather than the source of truth for a missed deadline.

## What are the limits of this health-checking pattern?

The catch is ownership. A status route plus telemetry contracts is suitable when the team already has an external probe, a metrics evaluator, and a notification path. It is not suitable when nobody owns deadline evaluation or paging. In that case, choose a dedicated managed heartbeat and synthetic-check service, or operate a self-hosted monitoring stack with explicit rule evaluation. The right alternative is the one that has an on-call owner and can test its own alert path.

It also can't prove end-to-end delivery. A provider accepting a request does not prove that an end user saw an email or mobile notification. For workflows where that distinction matters, model provider callbacks or customer-visible acknowledgements as later states, with their own objectives. Keep the cron completion heartbeat focused on the boundary the service actually controls.

Logs alone are a poor missed-event detector because “no matching log” can mean no job, delayed ingestion, a query error, or the wrong time range. Metrics alone lose per-run detail. A heartbeat alone says little about why a run was late. Use all three where the operational consequence warrants them, but don't add a telemetry pipeline to a low-impact batch job without weighing storage, query, maintenance, and on-call cost.

The final decision rule is straightforward: page on an externally observed breach, diagnose with bounded metrics and structured logs, and keep release identity separate from job identity. Then a rollback changes the code under observation without changing what “notification delivery is healthy” means.

## References

- Prometheus, “Metric and label naming”: https://prometheus.io/docs/practices/naming/
- OpenTelemetry, “Sampling”: https://opentelemetry.io/docs/concepts/sampling/
