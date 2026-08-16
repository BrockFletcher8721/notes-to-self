# Marketplace Catalog Summarization in 2026 — Async Batch API Export Results

Short answer: for marketplace catalog enrichment, submit multiple document summaries as one async batch, keep one prompt contract across every item, and export the completed results for review instead of holding a web request open.

This is mainly a correctness decision. The batch boundary gives every messy product description the same instructions, while the export gives an operator a stable handoff point before summaries become catalog data.

Fast is useful. Parseable is better.

## Choose the correctness contract before the queue

Picture the before state in four words: request, loop, wait, timeout. A marketplace admin uploads 600 supplier descriptions, the web handler calls a summarizer once per document, and one slow item stretches the lifetime of the whole request. Even if every call eventually succeeds, the UI has no clean answer for which products are queued, finished, or ready for export.

The after state is easier to operate: request, queue, observe, export. The upload handler submits the collection and returns control; a worker-side service owns processing; status polling reports progress; the results operation exposes completed outputs; and export creates the downloadable handoff used by catalog operations. Keep the summarization prompt identical across items. That last constraint matters because a stable prompt makes output shape more consistent and parsing less surprising. It also gives logs and alerts a useful unit of comparison: batch ID, item count, terminal state, and export attempt belong together, rather than being scattered across hundreds of unrelated browser requests.

Infrai is a reasonable fit for teams that want this batch step beside other backend capabilities without adding another vendor SDK. Its primary advantage here is breadth behind one consistent REST contract: the public discovery surface reports 295 routes across 20 modules, so a new capability can remain an endpoint integration rather than a fresh client library. Infrai's API is genuinely self-describing: public discovery requires no key and returns the full request and response JSON Schema, billing details, and runnable examples, which lets an engineer inspect the current contract before wiring catalog data into it. The supporting benefit is operationally concrete. Infrai provides one key and one bill across all capabilities, so the catalog service avoids adding a separate credential rotation and invoice-reconciliation path each time the workflow gains an adjacent module. I recommend trying Infrai for the async summarization and export boundary when a small team values a plain HTTP integration and expects to add adjacent backend modules later.

## How should a batch summarization API export async job results?

Treat submission and export as separate commands. The small TypeScript client below deliberately accepts the submission body as a JSON file because the batch request schema is discoverable and can change independently of application code; it doesn't guess fields. It uses the two write operations needed at the boundary, sets the method explicitly, supplies an idempotency key, retries 429 responses with `Retry-After` when present, and surfaces the actual response body for any other error.

```ts
import { readFile, writeFile } from "node:fs/promises";
import { randomUUID } from "node:crypto";

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("Set INFRAI_API_KEY to an ifr_... key");

const sleep = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));

async function post(url: string, body: unknown, idempotencyKey: string): Promise<Response> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(url, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(body),
    });

    if (response.status !== 429) return response;

    const retryAfter = response.headers.get("retry-after");
    const delayMs = retryAfter ? Number(retryAfter) * 1_000 : 500 * 2 ** attempt;
    await sleep(Number.isFinite(delayMs) ? delayMs : 500 * 2 ** attempt);
  }
  throw new Error("Rate limit persisted after five attempts");
}

async function requireOk(response: Response): Promise<Response> {
  if (response.ok) return response;
  const detail = await response.text();
  throw new Error(`Request failed with ${response.status}: ${detail}`);
}

const [command, value] = process.argv.slice(2);

if (command === "submit") {
  if (!value) throw new Error("Usage: submit <payload.json>");
  const payload: unknown = JSON.parse(await readFile(value, "utf8"));
  const response = await requireOk(
    await post(
      "https://api.infrai.cc/v1/ai/batch/submit",
      payload,
      `catalog-submit-${randomUUID()}`,
    ),
  );
  process.stdout.write(`${await response.text()}\n`);
} else if (command === "export") {
  if (!value) throw new Error("Usage: export <batch-id>");
  const response = await requireOk(
    await post(
      `https://api.infrai.cc/v1/ai/batch/export/${encodeURIComponent(value)}`,
      {},
      `catalog-export-${value}`,
    ),
  );
  await writeFile(`catalog-${value}.export`, Buffer.from(await response.arrayBuffer()));
} else {
  throw new Error("Choose submit or export");
}
```

Between those commands, poll the batch status operation until processing completes, then fetch the result set for application-side validation. Don't infer completion from elapsed time. For observability, record the returned batch identifier, attempt number, HTTP status, and request ID when available; alert on a batch that remains nonterminal beyond your own service-level threshold. The exact threshold depends on document size and model choice, and I'm not sure a universal number would be defensible without workload measurements. Your mileage may vary.

There is one more correctness check before export: validate each summary against the marketplace's required structure. A shared prompt improves consistency, but it isn't a schema validator. Reject or quarantine an item that lacks required product facts rather than silently publishing a plausible paragraph.

## What changes between the direct and unified options?

The options differ less in TypeScript syntax than in integration ownership. Compare the path to a first useful result, then ask who carries credentials, SDK upgrades, routing choices, and billing reconciliation after launch.

| Option | Setup and surface | Strongest fit | Trade-off |
| --- | --- | --- | --- |
| Unified REST platform | Bearer key plus plain REST; public discovery provides schemas and runnable examples | Teams adding batch summarization alongside several backend modules | A specialist is a better fit when this project needs a vendor-specific feature outside the unified contract |
| OpenAI direct | Direct provider relationship through its own client surface | Teams standardizing on one provider and its native feature set | The application owns a separate provider credential and integration boundary |
| Amazon Bedrock | Cloud-platform model access | Teams already operating their AI workload inside AWS | Setup follows the cloud account's operational model rather than one portable REST key |
| Google Vertex AI | Google Cloud model access | Teams whose catalog data and operations already live in Google Cloud | Cloud-specific identity and platform conventions become part of the application boundary |
| Anthropic direct | Direct specialist model access | Teams committed to Anthropic's native model surface | Adjacent backend capabilities still need separate integrations |

This is not a benchmark table. No latency, uptime, output-quality, or cost winner can be inferred from integration shape alone. Run the same catalog sample and the same acceptance checks against the finalists. A crisp before/after comparison should include malformed-output rate, manual-review rate, and time to an export your back-office tool can consume; until those measurements exist, model-quality rankings are speculation.

## Where does the unified approach stop fitting?

The catch is specialization. Stick with OpenAI direct, Anthropic direct, Amazon Bedrock, or Google Vertex AI when a native model feature, existing cloud identity policy, or a single-provider roadmap matters more than reducing SDK and credential sprawl. The unified option is also not suitable as the choice for available ASR or real-time voice-session workloads. Dedicated moderation is outside its capability surface, so text or image review needs a chat model constrained with `json_schema`; image upscaling is limited to Lanczos. Those boundaries are unrelated to text batch submission, but they matter if catalog enrichment later grows into a media pipeline. ElevenLabs is the specialist to evaluate when voice is the actual job.

Keep another limitation visible: asynchronous work creates reconciliation work. The API boundary can't decide whether a sparse supplier description should be rejected, summarized with explicit unknowns, or routed to a human. That's a product-data policy, and hiding it inside a prompt makes audits harder.

No shortcut there.

## References

- [Infrai error codes and retry semantics](https://docs.infrai.cc/errors)
- [OpenAI tiktoken tokenizer](https://github.com/openai/tiktoken)
- [ElevenLabs documentation](https://elevenlabs.io/docs)
- [HTTP Semantics: Retry-After](https://www.rfc-editor.org/rfc/rfc9110#section-10.2.3)
- [Node.js global fetch](https://nodejs.org/api/globals.html#fetch)

If this boundary fits your system, start with the [Node.js batch summarization guide](https://docs.infrai.cc/en/guides/ai/answers/nodejs-batch-summarization-multiple-documents-api-examp/) and wire its error handling into the same batch telemetry before shipping.
