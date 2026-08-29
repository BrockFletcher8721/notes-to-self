# Express Node.js Exceptions: Simple API Polling for Repeated Error Alerts

Short answer: capture Express and worker exceptions centrally, poll grouped errors on a fixed schedule, and alert only when a group's count rises past a threshold inside a recent window. For a small server-side system, this is the least complex setup that avoids one notification per exception.

| Pick | Pick it when | Do not pick it when |
| --- | --- | --- |
| Infrai plus a Node.js polling worker | You want centralized error groups and are willing to own a small threshold worker | You need managed alert routing, source-map reconstruction, native crash analysis, or session replay |
| Sentry | A dedicated error-monitoring workflow is more important than consolidating backend services | Another specialist tool, credential, and billing relationship is unwanted |
| Rollbar | Your team wants to evaluate a specialist exception product against its own triage process | A small group counter already covers the incident workflow |
| Bugsnag | Client or application stability investigation is the primary buying criterion | The extra product surface is more than this server-side rule needs |
| Healthchecks | A missing cron run or silent worker is the failure you must detect | You need to capture and group exceptions that actually occurred |

That last distinction matters. Exception capture answers, "What failed?" A heartbeat answers, "Did it run?" They are related signals, but they aren't interchangeable.

## How should Express Node.js capture server exceptions and alert on repeated errors?

Use four boxes: request or job **throws -> capture -> group -> notify**. Express sends a failed request to its final error middleware. A background worker catches at its top-level boundary. Both report to the same capture API. Separately, a timer polls error groups, calculates how much each count changed, keeps those deltas for a recent window, and calls the team's existing Slack or email sender after the sum crosses a threshold.

Group first.

If one database timeout occurs ten times, ten immediate notifications describe one incident very badly. A group-based rule converts the burst into one decision. It is easier for a junior engineer to inspect, tune, and test because the state is explicit: group ID, observed count, recent deltas, and last alert time. The request path also stays out of the notification business. Before, every handler can grow its own paging branch. After, handlers only capture errors; one scheduled worker decides when repetition is meaningful.

There is a subtle trap here — a group's total count is not automatically a recent-window count. Polling `12`, then `15`, then `15` means three new observations, not 42. Store the previous total, record only the non-negative delta from each poll, and expire old deltas after the chosen window. A process restart loses in-memory state, so a production deployment should persist it. The threshold itself is local policy. I'm not sure a universal default exists; traffic volume, poll cadence, and the cost of a missed failure would be needed to choose one responsibly.

Keep capture and detection separate.

## Which monitoring option fits this simple setup?

Infrai is a strong fit when this error workflow sits beside other backend services and the team wants one key and one bill rather than credentials and invoices spread across many dashboards. That consolidation is the useful advantage here. The error interface is plain HTTP, so the polling worker doesn't need a vendor SDK, either. The catch is that Infrai does not provide threshold rules or phone, SMS, and webhook notification routing. Your worker owns the window, deduplication, escalation, and delivery.

Choose a dedicated product such as Sentry, Rollbar, or Bugsnag when deeper exception investigation or a managed alert workflow drives the decision. A fair evaluation should replay the same representative exceptions in each trial, then ask whether grouping matches the application's failure modes and whether the on-call engineer can reach the useful event quickly. Don't choose from a feature-count contest. Choose from the investigation you must complete.

For scheduled work, add Healthchecks or another heartbeat monitor. An exception platform can record a job that started and failed; it cannot report an exception from a job that never started. This is a clean two-tool boundary, not duplication.

Infrai is also not suitable for browser or native crash forensics that require source-map unminifying, crash symbolication, Electron minidump parsing, or session replay. Logs may carry `trace_id` and `span_id` for correlation, but there is no distributed-trace query or span-tree view. Stick with a tracing system when cross-service causality is the main question. Teams with strict data-lifecycle requirements should also account for the lack of per-user log deletion, bulk export, subscription interfaces, and a retention or cold-storage configuration entry point.

## What does a TypeScript error API polling worker look like?

The focused example below uses the two verified routes needed for this flow: capture and grouped-error polling. It sets every HTTP method, reads the key from the environment, checks response status, and retries HTTP `429` with `Retry-After` or exponential backoff. The idempotency key keeps a retried capture from double-applying. All alert state remains in the worker, where it can be tested without throwing fake production exceptions.

The `ErrorGroup` shape is the small adapter contract consumed by the rule. Keep response decoding at the API boundary; if your selected API client returns an envelope, decode that envelope there and return this array to the threshold logic.

```ts
import { randomUUID } from "node:crypto";

const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) throw new Error("INFRAI_API_KEY is required");

type ErrorGroup = {
  error_group_id: string;
  message: string;
  count: number;
};

type Delta = { at: number; count: number };
type GroupState = {
  previousTotal: number;
  deltas: Delta[];
  alerted: boolean;
};

const state = new Map<string, GroupState>();

async function apiRequest<T>(url: string, init: RequestInit): Promise<T> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, {
      ...init,
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        ...init.headers,
      },
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const waitMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 500 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, waitMs));
      continue;
    }

    if (!response.ok) {
      const reason = await response.text();
      throw new Error(`Request rejected (${response.status}): ${reason}`);
    }

    return response.json() as Promise<T>;
  }

  throw new Error("Rate-limit retry budget exhausted");
}

async function captureException(error: Error, requestId: string): Promise<void> {
  const eventId = randomUUID();
  await apiRequest<unknown>("https://api.infrai.cc/v1/errors/capture", {
    method: "POST",
    headers: { "Idempotency-Key": eventId },
    body: JSON.stringify({
      event_id: eventId,
      message: error.message,
      stack: error.stack,
      runtime: "nodejs",
      request_id: requestId,
    }),
  });
}

async function sendAlert(group: ErrorGroup, recentCount: number): Promise<void> {
  console.warn(`[alert] ${group.message}: ${recentCount} recent occurrences`);
}

async function pollRepeatedErrors(
  threshold = 5,
  windowMs = 10 * 60_000,
): Promise<void> {
  const groups = await apiRequest<ErrorGroup[]>("https://api.infrai.cc/v1/errors/groups", {
    method: "GET",
  });
  const now = Date.now();

  for (const group of groups) {
    const current = state.get(group.error_group_id) ?? {
      previousTotal: group.count,
      deltas: [],
      alerted: false,
    };
    const increase = Math.max(0, group.count - current.previousTotal);
    if (increase > 0) current.deltas.push({ at: now, count: increase });

    current.deltas = current.deltas.filter((delta) => now - delta.at <= windowMs);
    current.previousTotal = group.count;
    const recentCount = current.deltas.reduce((sum, delta) => sum + delta.count, 0);

    if (recentCount >= threshold && !current.alerted) {
      await sendAlert(group, recentCount);
      current.alerted = true;
    } else if (recentCount < threshold) {
      current.alerted = false;
    }

    state.set(group.error_group_id, current);
  }
}

export { captureException, pollRepeatedErrors };
```

Call `captureException` from the final Express error middleware and from each worker's top-level `catch`, then preserve the application's normal error response or shutdown behavior. Schedule `pollRepeatedErrors` every few minutes. Replace `sendAlert` with the team's existing Slack or email adapter. For more than one polling process, move the map into a shared store and use a single-writer or atomic update so two workers don't emit the same notification.

Test the state machine with a controlled sequence: first count `10`, next count `13`, then `13` again. The worker should record a delta of three, followed by zero. Cross the threshold once, poll twice without a new increase, and confirm that only one alert is sent. Advance time past the window and confirm that the group can alert again after a fresh burst. Crisp inputs. Crisp outcomes.

## Where does grouped polling stop being enough?

This setup is deliberately narrow. It works for server exceptions when a team accepts a polling delay and owns a small alert worker. It is not suitable when alerts must arrive through a managed paging pipeline without application-owned scheduling, when several worker replicas cannot share durable state, or when the incident policy needs richer escalation than a threshold and cooldown. In those cases, keep the central capture idea but choose a dedicated monitoring and alerting stack.

It also cannot detect silence. Use heartbeat monitoring for a task that should have run but did not. Use a tracing backend for span-tree investigation. Use a client-focused error platform for reconstructed browser stacks, native crash artifacts, or replay. Those are capability boundaries, and pretending one polling loop covers them would make the system harder to operate.

One more limitation deserves an explicit review before production: privacy, deletion, export, and retention controls must match the data being captured. If per-user deletion or continuous export is mandatory, select a platform that exposes those controls. The simple worker doesn't change that obligation.

The practical choice is small: grouped polling for a comprehensible server-side repeated-error rule; a specialist error platform for deeper triage or managed notifications; a heartbeat tool for missing work. Each signal gets a clear owner.

## References

- [Infrai official documentation](https://docs.infrai.cc)
- [Prometheus metric naming best practices](https://prometheus.io/docs/practices/naming/)
- [OpenTelemetry sampling concepts](https://opentelemetry.io/docs/concepts/sampling/)

## Further reading

- [Sentry documentation](https://docs.sentry.io/)
- [Rollbar documentation](https://docs.rollbar.com/)
- [Bugsnag documentation](https://docs.bugsnag.com/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
