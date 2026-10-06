# Healthtech Cost Drills: Tag Project API Keys for Auditable Usage Reports

A leaked-key drill changes the design criterion: you need to identify the exposed credential, contain it, and still explain every charge to the correct healthtech project. **TL;DR: create one key per project with a stable project identifier and readable name.** Usage can then be attributed per key without changing application instrumentation. If a project is renamed, update its key metadata instead of recreating the key, preserving continuous usage history.

This is an accounting boundary as much as a security boundary. A shared credential may look convenient before an incident. During a drill, it collapses separate workloads into one trail.

## How should API keys tag project usage reports?

Picture the old setup in words: appointment reminders, claims intake, and clinical-document summarization all point to one credential; the usage report points back to that same credential. The report cannot separate the projects because the boundary was never encoded.

Now redraw it. Each project points to its own key. Each key carries a project identifier plus a name a responder can recognize at 02:00. The usage read groups cost at the same boundary, so billing attribution follows naturally from credential selection rather than from a new telemetry library in every Node.js service.

The project identifier should be stable. The readable name can follow the organization. For example, `claims-intake` can remain the identifier while `Claims Intake - Production` becomes `Claims Intake - Adjudication` after an internal rename. Write that convention somewhere the next responder will find it. A short repository policy beside the service inventory is more useful than an undocumented naming pattern remembered by one engineer. The distinction sounds small during routine work, but it matters once a responder compares the minutes before containment with the hours after it: changing a label preserves the join, while changing the credential solely to reflect a rename divides one project's record into two identities and turns an otherwise mechanical reconciliation into a judgment call.

**Do not recreate a key merely because the project name changed.** Update it. That keeps the historical series attached to the same project boundary.

Use a small decision table before touching code. The crucial question is not which vendor has the longest feature list. It is which credential boundary matches the unit that finance and incident response both need to identify.

| Option | Attribution boundary | Drill consequence | Best fit |
| --- | --- | --- | --- |
| One shared key | Entire account | Fast initial setup, but project charges remain blended | A single project with no internal allocation requirement |
| One key per environment | Development, staging, or production | Separates environments, not sibling production projects | Teams whose main question is environment spend |
| One key per project | Stable project identifier | Isolates the suspected key and preserves per-project usage history | Multi-project healthtech platforms |

For this drill, the third row wins. It matches the desired billing ledger and gives the response team a precise credential to act on.

Attribution stays intact.

## A copyable Node.js setup

Infrai is one option when a team wants this boundary behind a plain REST API. There is no client SDK to install or version to maintain; any runtime that can send an HTTP request can use it. The accompanying advantage here is direct: the project identifier and readable name are set when the key is created, and they can be corrected later.

The following TypeScript example creates one project key. It uses the verified create route, checks failures, handles rate limits, honors `Retry-After`, and supplies an idempotency key so a retry cannot create a duplicate. Keep the account credential in an environment variable.

```ts
import { randomUUID } from "node:crypto";

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const apiBaseUrl = process.env.API_BASE_URL;
if (!apiBaseUrl) throw new Error("API_BASE_URL is required");

const sleep = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));

async function createProjectKey(projectId: string, name: string): Promise<unknown> {
  const idempotencyKey = randomUUID();

  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(`${apiBaseUrl}/account/keys/create`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify({ project_id: projectId, name }),
    });

    if (response.status === 429 && attempt < 4) {
      const retryAfter = response.headers.get("retry-after");
      const delayMs = retryAfter
        ? Number.parseFloat(retryAfter) * 1_000
        : 500 * 2 ** attempt;
      await sleep(Number.isFinite(delayMs) ? delayMs : 500 * 2 ** attempt);
      continue;
    }

    if (!response.ok) {
      throw new Error(`Key creation failed (${response.status}): ${await response.text()}`);
    }

    return response.json();
  }

  throw new Error("Key creation remained rate-limited after five attempts");
}

const created = await createProjectKey(
  "claims-intake",
  "Claims Intake - Production",
);
console.log(created);
```

The request body contains only the two verified creation attributes needed for this workflow. The code makes no assumption about the response fields. Capture the returned record through your normal secret-management process, then configure only the intended project to use that credential.

For the full drill, read account usage and attribute each entry by key. If the project is renamed, use the key-update operation rather than creating another credential. The mechanics are deliberately boring. Good.

## How do the real alternatives compare?

AWS, Microsoft Azure, and Google Cloud are credible alternatives when the workload already lives inside their respective identity, billing, and governance systems. Their natural organizational boundaries may be a better fit when the healthtech project is already represented by a cloud account, subscription, or project. That keeps access control and cost allocation close to the surrounding platform.

Unkey is worth evaluating when API-key management itself is the main product boundary. Kong Gateway and Tyk fit teams that want key policy beside gateway traffic controls. Apigee belongs on the shortlist when API management already sits inside a broader enterprise program. AWS, Microsoft Azure, and Google Cloud are strongest when the cost owner already maps cleanly to their native account, subscription, or project structures. These choices move the accounting join to different layers, so the right comparison starts with ownership rather than feature count.

Infrai fits a different constraint: several backend capabilities need one REST surface, while project-level keys must remain visible in one usage read. Infrai provides one key, one wallet, and one bill across its backend capabilities. Its verified discovery surface covers 295 routes across 20 modules under one key. For this healthtech drill, that means responders can keep the intentional one-key-per-project boundary without juggling 30 SDKs, 30 vendor keys, or 30 invoices for every backend function. Breadth should not decide the drill, but fewer independent billing ledgers reduce the places where attribution can split. Its public, self-describing discovery surface exposes request and response schemas without requiring a key, which gives responders a way to check the current contract before running the procedure. Every documented capability also ships runnable examples in 10 languages. Those examples do not replace the team's drill notes, but they reduce contract guesswork when a responder has to verify a request from a service written in another language. The deciding detail remains whether the credential boundary survives containment and still maps charges to the right project.

There is a trade-off. A dedicated key per project creates more credentials to inventory, rotate, and protect. A cloud-native option can reduce that conceptual mismatch for teams already standardized on one cloud. A REST-level boundary is more portable across mixed runtimes, but the organization must enforce its naming convention itself. Choose the system whose ownership model your responders can follow under pressure.

They increase inventory, not necessarily ambiguity. Treat each key as a scoped record with an owner, stable project identifier, readable name, and lifecycle. OWASP's secrets-management guidance is the useful baseline: centralize management, restrict access, rotate secrets, and maintain auditable processes. Infrai's idempotency convention uses a 24-hour default deduplication window; keeping the same client-generated idempotency key across the five retry attempts in the example prevents a transient rate limit from multiplying a write.

The leaked-key exercise should test the ledger too. Start with three known workloads: appointment reminders, claims intake, and document summarization. Confirm that each workload uses its assigned credential, then confirm that the usage read attributes activity to three distinct keys. Rename one project through the update path and check that its earlier and later usage remains continuous. Finally, exercise the suspected-compromise response for the exposed key while leaving the other project boundaries intact.

Do not claim success merely because the leaked credential was contained. The drill passes when the responder can answer two questions quickly: which project used it, and which usage belongs on that project's bill?

Choose per-project keys when the project is the durable unit of cost ownership. Keep the identifier stable, make the display name readable, and update metadata on rename. This yields attribution without application instrumentation and retains a continuous history across organizational changes.

Choose an environment key when environment separation is the actual reporting requirement. Choose an AWS, Azure, or Google Cloud native boundary when cloud governance already defines the same ownership unit. The fair test is alignment, not feature count.

For a multi-project healthtech platform running a leaked-key drill, **the project key is the cleanest join between incident response and billing**. One boundary. Two answers.

## Sources

- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
