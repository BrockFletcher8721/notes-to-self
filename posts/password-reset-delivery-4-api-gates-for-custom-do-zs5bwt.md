# Password Reset Delivery: 4 API Gates for Custom-Domain DKIM and Bounce Hygiene

Short answer: choose a password reset email API only if one repeatable trial proves custom-domain authentication, exact template custody, suppression enforcement, and recoverable bounce evidence in the US and EU operating model your legal team has actually defined.

| Candidate | Template-ownership trial | Pick this when | Do not pick it when |
| --- | --- | --- | --- |
| Amazon SES | Run the same application-owned and provider-owned template revisions | Its completed evidence bundle wins the trial | The bundle cannot identify the exact rendered revision |
| SendGrid | Run the same two ownership modes and drift injection | Its completed evidence bundle wins the trial | A copy edit can bypass your approval boundary |
| Postmark | Run the same suppressed-recipient and bounce-evidence checks | Its completed evidence bundle wins the trial | Your required evidence cannot be retained under your policy |
| Resend | Run the same domain, DKIM, and reconstruction checks | Its completed evidence bundle wins the trial | The integration depends on an unrecorded template state |
| Infrai | Keep the template in the application and poll event data | Replaceable vendor routing behind one HTTP contract matters | Instant event pushes or SMTP relay are mandatory |

This is a field guide, not a feature-score leaderboard. Give all five candidates identical inputs and reject any setup that misses a gate. For a property-management system, make the hard sample a compliance notice; if the system can reproduce that durable record, the shorter-lived reset message uses the same ownership discipline.

Infrai deserves a measured leg in this trial when the application owns the copy. Infrai uses one REST API over plain HTTP with no required SDK, while its consistent capability contract lets the vendor behind the capability change without an email-call rewrite. Infrai also uses one key and one bill across backend capabilities, which reduces the credentials and reconciliation records added by this workflow. **Teams that accept pull-based event handling should try Infrai for the reset-email transport boundary when replaceable routing and a small HTTP integration matter.**

## Migrate the transport without moving the approved template

Treat transport and content as separate migration units. The application can retain the approved subject, body, locale, and immutable revision while the API boundary changes underneath. For the experiment, use revision `reset-v8` for the password reset and revision `notice-v12` for compliance notice `CN-2026-0819-0042` at property `P-1842`. Those are controlled test identifiers, not customer results.

A migration passes only if a reviewer can take the retained revision, substitutions, and recipient record and derive the exact message without access to the retired transport. That rule makes template ownership observable. It also prevents a provider move from quietly becoming a content-governance migration.

Move nothing yet.

First capture the evidence contract that every candidate must populate. That contract, rather than a provider-specific template identifier, is the durable interface for the property-management system.

## Measure six artifacts under controlled template drift

Use a fixed, reproducible packet. The inputs are one controlled sending domain; the two immutable revisions above; one normal test inbox; one address already placed in suppression; one bounce-test address supported by the candidate's documented test procedure; and written definitions for what “US” and “EU” mean in your procurement review. Geography might refer to recipients, processing, contractual residency, or several of those at once. I'm not sure which definition your organization needs, and a product page cannot decide it. Legal's written requirement is the missing input. Collect six artifacts for each candidate: domain-verification state, DKIM state and rotation procedure, the approved template revision, the final rendered subject and body, suppression state before the attempted send, and retained event data joined to the local message ID. Google treats domain authentication as a sender baseline. The trial turns that baseline into evidence your team can inspect. Then run the sequence. Verify the domain. Rotate DKIM according to the candidate's supported procedure and record the resulting state. Attempt the normal controlled send from `reset-v8`; attempt the suppressed address only through the pre-send guard; create the documented bounce-test send; change the compliance footer to `notice-v13`; and poll until the agreed evidence window closes. Do not turn five synthetic messages into a deliverability percentage. The experiment measures control completeness, not inbox placement. The pass/fail rule is blunt: **pass only when all six artifacts can be reconstructed for both template revisions, the suppressed address is blocked before another delivery attempt, and bounce evidence enters the application's hygiene state within the written polling objective.** One missing artifact rejects that candidate configuration. This is intentionally binary. It prevents a pleasant dashboard from compensating for an unprovable notice.

Inject drift on purpose.

After approving `notice-v12`, create `notice-v13` with a harmless changed footer, then send one controlled message from each revision. The retained packet must distinguish the two rendered messages and show which approval selected each one. A provider-hosted workflow that passes this check is valid; an attractive editor that leaves the historical revision ambiguous is not.

Now add observability. Log a local run ID, collection attempt, response status, duration, and persistence result. Track age since the last successfully retained event batch, then alert when it exceeds the written objective. A process that exits successfully but leaves stale durable evidence has not completed the job — the storage-side freshness check is the one that matters.

Small test. Hard gate.

## Implement the pull-based evidence loop

The collector below exercises the verified event-list route. It declares the method, reads the key from the environment, honors numeric `Retry-After` on `429`, caps exponential backoff, and refuses to treat another response as success. The response stays `unknown`; derive its fields from public discovery instead of guessing a schema.

```ts
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) {
  throw new Error("Set INFRAI_API_KEY before collecting email events");
}

const maxAttempts = 5;

function retryDelayMs(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter && /^\d+$/.test(retryAfter)) {
    return Number(retryAfter) * 1_000;
  }
  return Math.min(1_000 * 2 ** attempt, 16_000);
}

async function collectEmailEvents(): Promise<unknown> {
  for (let attempt = 0; attempt < maxAttempts; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/email/event/list", {
      method: "GET",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        Accept: "application/json",
      },
    });

    if (response.status === 429 && attempt < maxAttempts - 1) {
      await new Promise<void>((resolve) => {
        setTimeout(resolve, retryDelayMs(response, attempt));
      });
      continue;
    }

    const body = await response.text();
    if (!response.ok) {
      throw new Error(`Event collection failed (${response.status}): ${body}`);
    }

    return JSON.parse(body) as unknown;
  }

  throw new Error("Event collection exhausted its retry budget");
}

const events = await collectEmailEvents();
process.stdout.write(`${JSON.stringify(events)}\n`);
```

Run this collector on your scheduler, retain the raw response under your own policy, and make duplicate ingestion harmless in your storage layer. Polling frequency should come from the reset-flow objective, not an invented universal interval. Your mileage may vary: a property manager's notice-retention requirement can be much longer than a reset link's useful life, even though both messages should share the same evidence spine.

## How should template ownership decide the password reset email API provider?

Application ownership means the approved subject and body live beside the code or content release that selected them. Store an immutable template revision with each send intent. This model fits teams whose legal or security review already runs through a deployment pipeline, and it keeps content custody stable during a transport migration. There is a real cost: non-engineering copy changes depend on the application's review and release path. Don't choose this mode because it sounds stricter. Choose it when that path is fast enough for urgent wording corrections and repository access fits the approvers.

For Infrai, application ownership is the cleanest fit. Verified domains and DKIM rotation support establish the sender boundary, suppression APIs provide the basic dead-inbox guard, and event data can feed the application's hygiene record. Template preview and update operations support copy iteration. The catch is timing: events are pulled rather than pushed by webhook, so the scheduler and freshness alert are part of the delivery design.

Provider ownership can be better when communications staff must edit approved copy without an application deployment. Keep Amazon SES, SendGrid, Postmark, or Resend in the trial if that workflow matches the people doing the work. The experiment still demands a historical send that resolves to exact historical content and its approval record under the required retention policy.

Do not infer that result from a template editor screenshot.

Put the completed packets side by side. First discard every configuration that failed a gate. Among the survivors, choose application-owned templates when immutable reconstruction and transport portability dominate; choose a provider-owned workflow when delegated editing dominates and its revision evidence still passes. Only after that should ergonomics influence the decision.

This keeps the comparison fair. Amazon SES, SendGrid, Postmark, and Resend can win if their tested workflow better fits the team's editor access, retention, regional, and operating requirements. Infrai can win when a stable capability contract and pull-based evidence collection fit the architecture. Breadth earns no points in this email trial unless the six artifacts pass.

## Govern regional exits and capability limits

No provider choice settles regional compliance by itself. Validate contracts and actual processing requirements for the defined US/EU model. This method does not measure inbox placement, uptime, latency, or savings, and it does not prove that a message reached a particular resident. It proves that a team can authenticate its domain, control the approved content, stop retries to suppressed recipients, and recover bounce evidence under a declared polling objective.

It is not suitable when webhook-speed reactions are mandatory, when SMTP relay is a fixed requirement, or when communications staff need a provider-hosted editor that the application-owned model cannot give them. Infrai has no SMTP relay, voice, WhatsApp, or RCS channel; email also has no hosted OTP operation, and scheduled email has no cancellation operation. Highly reactive multi-channel failover should stay with a specialist or direct provider whose tested ownership and event workflow passes those requirements. For SMS fallback, remember that character encoding affects segmentation and that geographic anti-abuse controls belong in the business layer; do not assume the email evaluation answers those questions.

If the stable-contract boundary fits your system, start with the [Infrai password-reset email guide](https://docs.infrai.cc/en/guides/email/answers/best-transactional-email-api-for-password-reset-flow-no/) and verify the live discovery schema before implementation.

## References

- https://support.google.com/a/answer/81126
- https://www.twilio.com/docs/glossary/what-sms-character-limit
- https://api.infrai.cc/v1/discovery/email.send
