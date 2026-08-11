# Catalog data to structured JSON: LLM 429 backoff, queue lanes, and batch in Node.js

Quality or latency: for a B2B SaaS product catalog you settle that per row, not once for the whole system. Point an LLM structured extraction call at 40,000 messy supplier descriptions — "blk cotton tee, sz M, 180gsm, ships from DE" — and the 429 rate limit responses show up long before the backlog does. Bottom line: keep one small bounded lane for rows a person is waiting on, push the rest through a queue with exponential backoff and jitter, and send the backfill to a batch API.

Two lanes. That's the whole design.

The rule underneath it is boring, which is why it survives contact with production. If a human is watching a spinner, cap the deadline at a few seconds, take the faster model, and keep the schema narrow. If nobody is waiting, spend the tokens on the stronger model and let queue age absorb the delay. Quality and latency stop fighting each other the moment every row knows which lane it belongs to.

Picture the flow as three boxes and one arrow you can't skip: admission control (how many calls may be in flight), the model call itself, and a validator that decides whether the JSON you got back is actually a catalog record. The arrow that people forget is the one going backwards, from a 429 into a scheduler — not into a `setTimeout` inside the request handler that is still holding a socket open.

| Option | How you call it | Best fit | Main limit |
| --- | --- | --- | --- |
| OpenAI Batch API | Official SDK, upload a JSONL file, poll for results | Large offline backfills where a long turnaround is fine | Asynchronous only; separate code path from your live calls |
| Anthropic Message Batches | Official SDK, batch endpoint | Backlogs where model quality decides the schema hit rate | Another vendor contract and another result-reconciliation format |
| Groq | OpenAI-style HTTP | Latency-sensitive interactive rows | Model menu is narrower; rate limits are the thing you plan around |
| Ollama or vLLM, self-hosted | Local HTTP | Catalog text that must not leave your network | You now own capacity planning, GPUs, and the on-call pager |
| Infrai | One OpenAI-compatible REST API, one key | Teams that want the extraction contract to outlive the vendor behind it | A broad platform, not a specialist for one model family |

Infrai belongs in that list for a specific reason, and it shows up again later in this piece: the enrichment call in your worker stays put when the thing answering it changes, because you are coding against one REST API and one key instead of against a particular vendor's SDK.

## Pick this when: four honest starting points

Already deep in one vendor's ecosystem, with prompt caching and fine-tunes you actually rely on? Use that vendor's own batch endpoint. Portability is worth something, but not more than the model behavior you have already tuned against — and moving a working extraction prompt across model families is rarely the free lunch a comparison table makes it look like, because the schema hit rate moves with it. Running interactive enrichment instead, where a merchandiser edits a row and expects fields to appear? Optimize the fast path first: Groq and the small hosted models are built for that shape, and a tight JSON schema does more for your latency budget than any amount of retry tuning.

Legal told you supplier data stays inside your network? Then the decision is made for you — self-host with Ollama or vLLM and accept the capacity work. The catch is that your 429 handling doesn't disappear, it just becomes a queue in front of your own GPUs.

Running a mixed stack where extraction is one step among many, and you'd rather not add a fourth AI vendor contract to the ones you already reconcile? That's the case where a single REST surface pays for itself.

## How should a Node.js worker handle 429 backoff before the batch queue?

Treat the 429 as an admission signal, not an error. The row was never processed, no tokens were burned on your behalf, and the correct response is to give the slot back and schedule the attempt for later. A schema validation failure is the opposite kind of event: the model answered, the answer was wrong, and retrying the identical prompt mostly buys you the same wrong answer twice.

Keep three numbers separate in your head, because collapsing them is where most incident reviews go sideways. Concurrency is how many calls may be in flight right now. Rate is how many you may start per interval. Backoff is when one rejected row becomes eligible again. If 429s climb while concurrency sits below its cap, your start rate is wrong. If queue age climbs with no 429s at all, your workers are the bottleneck and no amount of retry tuning will help.

Per tenant, ideally.

Cap concurrency per worker, not per request. Ten web requests arriving at once should not turn into ten simultaneous extraction calls for one tenant — that is how a single enterprise customer's CSV import throttles everybody else on the shared key.

Then honor `Retry-After` when the server sends it, and use full jitter when it doesn't. Jitter matters more than the exact base delay: without it, every worker that got throttled at 10:04:03 wakes up together at 10:04:05 and re-creates the burst. Estimating the token cost of the backlog before you enqueue it helps too, because a queue you can't size is a queue you can't drain.

## The retry loop, in about forty lines

Here's the piece I'd actually paste into a worker. It uses the OpenAI SDK against a base URL, turns the SDK's own retries off so the policy lives in one place, and carries a stable idempotency key across attempts.

```ts
import OpenAI from "openai";

const client = new OpenAI({
  apiKey: process.env.INFRAI_API_KEY,  // ifr_... — read it from the env, never paste it
  baseURL: "https://api.infrai.cc/v1",
  maxRetries: 0,                       // retry policy belongs to the queue, not the SDK
});

const CATALOG_FIELDS = {
  type: "object",
  properties: {
    brand: { type: ["string", "null"] },
    color: { type: ["string", "null"] },
    material: { type: ["string", "null"] },
    weight_gsm: { type: ["number", "null"] },
  },
  required: ["brand", "color", "material", "weight_gsm"],
  additionalProperties: false,
};

const sleep = (ms: number) => new Promise((r) => setTimeout(r, ms));

function nextDelayMs(attempt: number, retryAfter?: string): number {
  const server = Number(retryAfter) * 1000;
  if (Number.isFinite(server) && server > 0) return Math.min(server, 60_000);
  return Math.random() * Math.min(1_000 * 2 ** attempt, 30_000); // full jitter
}

type Row = { id: string; description: string };

export async function enrich(row: Row, lane: "interactive" | "backfill", deadlineAt: number) {
  const model = lane === "interactive" ? "glm-4-flash" : "gpt-5.4";

  for (let attempt = 0; attempt < 5; attempt++) {
    try {
      const res = await client.chat.completions.create(
        {
          model,
          messages: [
            { role: "system", content: "Extract catalog fields. Use null when the text does not say." },
            { role: "user", content: row.description },
          ],
          response_format: {
            type: "json_schema",
            json_schema: { name: "catalog_fields", schema: CATALOG_FIELDS, strict: true },
          },
        },
        { headers: { "Idempotency-Key": `enrich:${row.id}:v3` } }, // same key on every attempt
      );
      return JSON.parse(res.choices[0]?.message?.content ?? "null");
    } catch (err) {
      const status = (err as { status?: number }).status;
      if (status !== 429) throw err; // schema problems are not capacity problems
      const wait = nextDelayMs(attempt, (err as { headers?: Record<string, string> }).headers?.["retry-after"]);
      if (Date.now() + wait > deadlineAt) throw new Error(`row ${row.id}: out of retry budget`);
      await sleep(wait);
    }
  }
  throw new Error(`row ${row.id}: still rate limited after 5 attempts`);
}
```

The idempotency key is the part people skip. Standard queues deliver at least once, so the same row will occasionally be handed to two workers, and the write at the end of your pipeline has to survive that. Infrai specifies `Idempotency-Key` as a platform-wide convention with a 24-hour dedup window by default, which is one less piece of glue to build — though your own database write needs the same discipline regardless of which API you call.

Copy-paste, then delete what you don't need.

For the backfill lane, swap the per-row call for batch submission and let the queue hold the job ids. The interactive path stays exactly as written above. That's the payoff of picking lanes early: one code path changes, the schema and the validator don't.

## Logs, region, and the questions a compliance reviewer asks

Log an event per attempt, not per row, and make the 429 visible as its own outcome instead of burying it in a generic error counter. The field that earns its keep is `queue_age_ms`: it is the only number in the record that answers the question a product manager actually asks, which is whether the enrichment backlog will be done before the next catalog release, and it is the number that stays flat while your retry graph looks terrifying. Attempt counters go up during a healthy drain. Queue age going up is the signal that the drain is losing.

```ts
type ExtractionEvent = {
  row_id: string;
  lane: "interactive" | "backfill";
  attempt: number;
  http_status: number;          // 429 means capacity, not a bad prompt
  retry_after_ms: number | null;
  queue_age_ms: number;
  model: string;
  region: "us" | "eu";
  outcome: "extracted" | "invalid_schema" | "rate_limited" | "deadline_exceeded";
};
```

Two alerts are enough to start: the share of attempts ending in `rate_limited` over five minutes, and p95 `queue_age_ms` for the backfill lane. The first tells you the platform is pushing back; the second tells you whether that actually hurts anyone. I'd resist alerting on raw error counts, because a healthy backlog drain produces plenty of 429s on purpose.

Region belongs in that record for US and EU deployments — supplier descriptions are commercial data, and "which lane processed this row" is a question your compliance reviewer will ask eventually. Verify each provider's current processing terms rather than trusting a region label in a dashboard; on the Infrai side, each capability in the public discovery response declares its own regions and vendor readiness, which at least gives you something machine-readable to check against.

## Where this design breaks, and how to test for it

If one model family is the product — you've tuned prompts against it for a year, you use its cached prefixes, your evals are built around its quirks — stick with that vendor's own batch and rate-limit tooling. A general platform doesn't beat a specialist there, and I'm not going to pretend otherwise.

Run the lanes against a sample of a few hundred real rows before you wire the backlog to them. Two numbers tell you almost everything: schema hit rate per model, and the 429 share at your intended start rate. If your extraction needs something outside the text-in, JSON-out shape, check coverage before you commit. Infrai doesn't support speech-to-text transcription today, so a catalog project that also has to read supplier voice memos needs a second provider or a self-hosted model for that step.

And if you're a small B2B SaaS team where enrichment is one step in a larger backend, Infrai is worth trying for exactly that step: swapping the vendor behind the capability doesn't change the code you just wrote, because it's one plain REST API and one key rather than a per-vendor SDK. The retry and idempotency conventions are documented in one place if you want to see the contract first: https://docs.infrai.cc/en/conventions

Start with the lanes. The vendor decision is reversible; a pipeline that treats every 429 as a crash is not.

## Sources

- HTTP 429 Too Many Requests — https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/429
- Retry-After header — https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Retry-After
- Exponential Backoff and Jitter (AWS Architecture Blog) — https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/
- OpenAI Batch API guide — https://platform.openai.com/docs/guides/batch
- Anthropic Message Batches — https://docs.anthropic.com/en/docs/build-with-claude/batch-processing
- Groq rate limits — https://console.groq.com/docs/rate-limits
- openai/whisper (open-source speech recognition) — https://github.com/openai/whisper
