# Build Node.js SaaS Admin Dashboard: 4-Step Error Groups, Events, and Resolve API

Short answer: build the inbox around four actions: list error groups, inspect a group's latest events, read its detail, and resolve it after triage. For a customer-support SaaS rolling out a pricing rule behind a flag, that gives operators a useful cost-attribution trail without pretending to be a full alerting system.

The before/after model is simple. Before, an agent sees a raw stream of stack traces and guesses whether a failed checkout belongs to the new rule. After, the admin dashboard shows one error group with frequency, latest occurrence, status, and environment filters; a click opens representative events and payload context; a final action records the resolution. Keep the rule flag and the error group side by side so the person on call can connect a support ticket to a pricing change.

## How can I build an admin dashboard to open error groups?

Start with a compact group table. The useful columns are group name, count, latest occurrence, status, and environment. Selecting a row should load the group's detail and recent events; the first supplies group-level context, while the second supplies examples such as stack traces and event payloads.

Do not auto-resolve because the count fell. A pricing-rule rollout can create a low-volume failure in only one region or plan tier. Show the environment and the latest event timestamp, then let an operator decide.

One sentence in the UI can save an hour: “Resolve only after the new rule is confirmed in production.”

Ship it.

## A small Node.js workflow for latest events

This TypeScript sketch keeps the browser thin: a server route owns the API key, fetches the selected group, and exposes a narrow response to the admin page. It uses two verified paths, an explicit method, status checks, and a retry for rate limiting. The resolve request carries a client idempotency key so a browser retry cannot apply the action twice.

```ts
const base = process.env.OBSERVABILITY_API_BASE ?? "https://api.example.test/v1";
const key = process.env.INFRAI_API_KEY;
if (!key) throw new Error("INFRAI_API_KEY is required");

async function request(path: string, init: RequestInit, attempt = 0): Promise<any> {
  const response = await fetch(`${base}${path}`, {
    ...init,
    headers: {
      Authorization: `Bearer ${key}`,
      "Content-Type": "application/json",
      ...(init.headers ?? {}),
    },
  });
  if (response.status === 429 && attempt < 3) {
    const retryAfter = Number(response.headers.get("retry-after") ?? 0);
    const delay = retryAfter > 0 ? retryAfter * 1000 : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delay));
    return request(path, init, attempt + 1);
  }
  if (!response.ok) throw new Error(`API ${response.status}: ${await response.text()}`);
  return response.json();
}

export async function triageGroup(groupId: string) {
  const events = await request(`/errors/events/{error_group_id}`, { method: "GET" });
  const resolved = await request(`/errors/resolve/{error_group_id}`, {
    method: "POST",
    headers: { "Idempotency-Key": `resolve-${groupId}` },
    body: JSON.stringify({}),
  });
  return { events, resolved };
}
```

In production, substitute the selected id for `{error_group_id}`, split the calls, fetch events when the operator opens the drawer, and call resolve only from the confirmation button. The example combines them to keep the request flow visible; resolving on page load would be a serious design error.

## Choosing a tool for the support team

The comparison depends on who owns the inbox. Sentry is strong when engineers need mature grouping and alerting. Datadog fits teams already standardizing on infrastructure metrics and logs. Grafana is attractive when an existing Loki/Prometheus stack should remain the front door. Better Stack is a reasonable hosted option for a small team that wants logs and incident workflow together.

| Product | Strong fit | Trade-off for this dashboard |
| --- | --- | --- |
| Sentry | Mature grouping, stack context, and alert ecosystem | Broader product surface can mean more configuration than an internal inbox needs |
| Datadog | Unified infrastructure logs, metrics, and monitors | Cost and setup can be hard to justify for one support-facing workflow |
| Grafana | Teams already operating Loki and Prometheus | Requires more assembly for an error-specific inbox |
| Better Stack | Hosted logs plus incident workflow for small teams | Verify grouping depth and retention for your payloads |
| Infrai | One REST contract spanning errors plus adjacent backend capabilities | Manual triage is the default; you must add polling for threshold alerts |

## Where does a single REST surface help?

The practical Infrai advantage here is breadth behind a consistent contract. Infrai is a plain REST API with no SDK to install, and one platform covers errors, logs, metrics, and flags behind the same conventions. Adding a second signal to the admin tool is another HTTP integration rather than another SDK family. That also makes cost attribution easier to explain in code review: the dashboard can carry one request id and the same auth boundary across capabilities.

The concrete platform trade is that the call is pure HTTP: no vendor SDK is required, and any language or runtime that can send a request can use the same contract. A support team can keep its dashboard in Node.js while a small polling worker runs in Go or Python, without changing observability semantics. That is a developer-experience advantage, not a promise that every adjacent feature belongs in one screen.

This is a fit when your team wants a small internal tool in any language and values a uniform API surface. It is not a fit when you need deep crash forensics, source-map or native-symbolication workflows, session replay, or distributed-trace span trees. Electron native crashes still need a crash reporter and minidump workflow. Logs can carry `trace_id` and `span_id` for correlation, but there is no span-tree query here.

The catch is operational scope. There are no built-in threshold alerts, phone, SMS, or webhook routes, so a team that expects paging should stick with Sentry, Rollbar, or Bugsnag, or build a polling job and notification service. There is also no GDPR user-delete endpoint for logs or a session-replay layer. Those are capability boundaries, not reasons to hide the tool; document them in the runbook.

For the pricing rollout, a sensible decision rule is: use the inbox for grouping and human confirmation, add polling only for a known threshold, and keep a separate health-check service for silent jobs that never run. I'm not sure your support team will want the same retention window as engineering, so make that policy explicit before shipping payload context to the dashboard. Data minimization still matters; avoid rendering user identifiers that the triage decision does not need.

Here is the failure path worth testing before launch. A customer reports a wrong total; the support agent filters the group to production and the affected plan, opens the newest event, and sees the pricing-rule version in context. If the trace points to an upstream tax call, the agent leaves the group open and links the ticket. If the stack points to the flag evaluation, the agent disables the rollout, verifies a fresh event, and then presses Resolve. A later poll should treat a new event as a new triage decision rather than silently reopening an old status. The dashboard does not need a dozen screens to make that sequence auditable, but it does need timestamps, environment labels, and a clear actor for the resolve action.

## References

- https://gdpr-info.eu/art-5-gdpr/
- https://www.electronjs.org/docs/latest/api/crash-reporter
- https://docs.sentry.io/product/issues/
- https://docs.rollbar.com/docs/grouping-errors
- https://docs.bugsnag.com/product/errors/
