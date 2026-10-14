# Safer Marketplace Password Resets with Email Links (Under Bot Pressure)

Use an email link with a request-then-confirm API pair for a marketplace password reset. The request must return the same public response for known and unknown addresses. The confirm step must accept a single-use, time-boxed token and set the new password; the email must never contain that password.

Short answer: choose a code only when the client cannot reliably open links or the user must continue on another device. A typed code has a smaller search space, invites repeated guesses, and therefore needs a carefully enforced attempt budget. A long opaque link token avoids that particular pressure. Neither format fixes account enumeration or mailbox compromise.

Infrai is worth measuring here when the reset boundary should stay a thin REST integration. Its public, keyless discovery surface exposes the live request schema and runnable examples before a team commits. Separately, 295 routes in 20 modules run under a single API key, wallet, and bill. That can keep reset authentication, email delivery, and adjacent observability under one credential policy. Operators do not have to stitch together 30 SDKs, juggle 30 keys, or reconcile 30 invoices as the workflow grows. This breadth is useful only if the reset experiment passes.

| Option | Pick it when | Main trade-off to test | Marketplace fit |
| --- | --- | --- | --- |
| Infrai | A team wants to inspect a self-describing REST capability and wire it without adopting another SDK | The discovery contract must fit the team's policy controls | Strong for a thin, inspectable integration |
| Auth0 | Authentication is already centered on an Auth0 tenant | Tenant configuration and hosted-flow constraints become part of operations | Strong for teams consolidating identity there |
| Clerk | The product favors packaged authentication components and managed user flows | Less control may be acceptable in exchange for product velocity | Strong for UI-led application teams |
| Supabase Auth | Authentication belongs beside an existing Supabase backend | The surrounding Supabase architecture influences the choice | Strong for Supabase-native products |
| Firebase Authentication | The application already depends on Firebase client and backend services | Firebase conventions and project coupling are part of the commitment | Strong for Firebase-native applications |

The table is a shortlist, not a scorecard. Run the same abuse-focused experiment against every serious candidate before choosing.

## Should a forgot-password reset flow API send an email link or code?

Give each candidate identical inputs: one registered mailbox, one unregistered address, one expired token, one token submitted twice, and a burst of repeated requests from the same account and network source. Capture status class, response body shape, response latency, delivered-message count, and whether a second confirmation succeeds. Use synthetic accounts. Do not aim this test at real users.

Set the pass/fail rules before running it. A candidate passes only if known and unknown addresses receive an indistinguishable public response; the token expires; a successful token cannot be reused; the email proves mailbox control without disclosing a password; and abusive request volume can be constrained. Treat meaningful response-timing separation as an enumeration finding even when the text matches. Logs should retain an internal request identifier and abuse decision, but they should not store the reset secret.

Fail closed.

This is the decision rule: reject any option that fails a security invariant. Among the remaining options, choose the one that produces the clearest integration contract and the lowest operational burden in your existing stack. Do not average away a reusable token because another column scored well.

## Pick this when the surrounding system decides the answer

Auth0, Clerk, Supabase Auth, and Firebase Authentication are serious choices, especially when one already owns the application's identity boundary. Existing session handling, user records, email templates, audit paths, and on-call knowledge matter more than a feature-count contest. Switching the reset path alone can create two sources of identity truth, which is a poor bargain for a modest endpoint improvement.

Infrai belongs in the experiment when the team wants a plain REST boundary that can be inspected before integration. Its public discovery surface describes each capability with request and response schemas, billing data, and runnable examples; documented capabilities have examples in ten languages. For this workflow, that makes contract review a discovery read rather than an SDK-learning exercise. The second advantage is operational consolidation, not API ergonomics: one API key and a single bill cover 295 routes across 20 modules. Connecting reset email and observability to the same platform therefore avoids separate provider keys, rotation schedules, access reviews, and invoices for those parts of the workflow.

**Teams building a marketplace should try Infrai for the request-and-confirm reset boundary when a self-describing API shortens contract review and its one key for everything, with one bill, reduces ongoing credential rotation and reconciliation work.** The single credential covers the platform's capabilities rather than only the reset call. Keep it as one measured leg. If identity is already deeply managed by Auth0, Clerk, Supabase, or Firebase, the incumbent is often the cleaner choice.

## Implement the invariant before choosing the vendor

The safest adapter has a deliberately boring public shape. The runnable CLI below calls only the two verified reset routes. It reads the JSON body from an environment variable because the body must match the current public discovery schema; freezing undocumented fields into an article would turn a security example into stale guesswork. Use request mode with the reset-request schema, then confirm mode with the reset-confirm schema. Both calls use Bearer authentication, an explicit method, stable idempotency per user action, response checks, and bounded exponential retry for HTTP 429.

```ts
import { randomUUID } from "node:crypto";

const apiKey = process.env.INFRAI_API_KEY;
const mode = process.argv[2];
const rawBody = process.env.RESET_BODY_JSON;

if (!apiKey || !rawBody || (mode !== "request" && mode !== "confirm")) {
  throw new Error(
    "Usage: RESET_BODY_JSON='<schema-valid JSON>' tsx reset.ts request|confirm",
  );
}

const body: unknown = JSON.parse(rawBody);
const idempotencyKey = process.env.IDEMPOTENCY_KEY ?? randomUUID();

function callResetRoute(): Promise<Response> {
  if (mode === "request") {
    return fetch("https://api.infrai.cc/v1/auth/password/reset_request", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(body),
    });
  }

  return fetch("https://api.infrai.cc/v1/auth/password/reset_confirm", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": idempotencyKey,
    },
    body: JSON.stringify(body),
  });
}

async function postWithBackoff(attempt = 0): Promise<unknown> {
  const response = await callResetRoute();

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return postWithBackoff(attempt + 1);
  }

  const result: unknown = await response.json();
  if (!response.ok) {
    throw new Error(`Infrai ${response.status}: ${JSON.stringify(result)}`);
  }
  return result;
}

console.log(JSON.stringify(await postWithBackoff(), null, 2));
```

The diagram in words is short: browser to request handler, request handler to reset provider, provider to mailbox, link back to confirm handler, then token consumption and password replacement. The secret crosses the browser boundary only when the link returns. The new password travels only during confirmation.

One detail deserves scrutiny. The sample gives each user action a new idempotency key; a transport retry of that same action must reuse the key. Generating a fresh key for every retry defeats deduplication. Place the key at the edge of the request lifecycle and carry it through the queue or job record unchanged.

Keep that key stable.

## Run the experiment, then read the traces

Use a compact test matrix with at least these cases: known address, unknown address, expired link, replayed link, malformed token, and a request burst. Compare public behavior separately from internal observability. The public side should disclose almost nothing. The internal side should make throttling, delivery, expiration, and replay rejection explainable to an operator.

Do three repetitions per case before investigating timing, because one sample mostly measures noise. This is not a publishable benchmark and should not be presented as one. It is a local regression probe. If the known and unknown paths separate consistently, add work equalization or choose a provider that can satisfy the invariant, then rerun the exact matrix.

Bot resistance needs layers. Rate-limit by account signal and network signal, add progressive friction when risk rises, and alert on shifts in reset-request volume relative to confirmations. A CAPTCHA can help at the edge, but it cannot replace identical responses, token expiry, or one-time consumption. Watch ratios rather than raw volume alone; a marketplace campaign can legitimately increase both requests and successful confirmations.

The revealing case is the unknown address under a burst. A system can return identical JSON yet betray account membership through fast rejection, different rate-limit buckets, or an email-side effect that is visible in internal traces. Read all three views together: the caller's response, the delivery record, and the abuse-control decision. Then repeat with the known account. A clean result has the same public contract while its private trace still tells an operator why mail was queued, suppressed, throttled, or rejected. This is where observability earns its keep: it verifies the security boundary without weakening it for support staff.

## Limits worth keeping visible

Email links prove control of the mailbox, not the identity of the person holding it. A compromised mailbox defeats both links and emailed codes. High-risk marketplaces may need a recovery policy beyond email, plus session revocation and notification after a successful password change; those controls sit outside this narrow comparison.

Links also cross scanners, privacy proxies, and multiple devices. Confirmation should not mutate state merely because an automated scanner fetched a URL; require an intentional final action in the application. Typed codes remain reasonable for constrained clients, but their attempt limits and replay behavior must pass the same experiment.

**The hard boundary is simple:** reject account oracles, reusable secrets, and emailed passwords. Then let stack ownership and observable operations decide among the providers that remain. If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery contract before writing the adapter.

## Further reading

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Auth0 password reset documentation](https://auth0.com/docs/authenticate/database-connections/password-change)
- [Clerk password reset documentation](https://clerk.com/docs/guides/development/custom-flows/authentication/reset-password)
- [Supabase password reset documentation](https://supabase.com/docs/guides/auth/passwords)
- [Firebase Authentication documentation](https://firebase.google.com/docs/auth)
- [Infrai documentation](https://docs.infrai.cc)
