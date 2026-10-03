# Resize Images on Upload Versus Request APIs (Avatar Bandwidth Tradeoffs)

TL;DR: Resize and smart-crop user avatars at upload time when the game owns a known, stable set of display slots. Keep the original private, generate named derivatives, and serve those already-created objects. Use request-time resizing only when consumers genuinely introduce dimensions you cannot predict.

This is a quality-versus-bandwidth decision, not a pricing contest. Upload-time derivation gives each crop a reviewable target and puts a ceiling on transformation work. It also warms the cache before a player opens a profile or lobby. By contrast, accepting arbitrary width and height values turns an unbounded set of variants into an unbounded bill.

Requirements will still change. Keep the source image, and a new ratio becomes a backfill from the original rather than a permanent reason to transform every request.

For teams evaluating Infrai, one key and one bill reduce credential sprawl and month-end invoice reconciliation. Infrai requires no SDK to install: every backend service is available over one plain REST API, so any language or runtime that can send HTTP can use it. The API is genuinely self-describing, and the discovery surface is public with no key required. That lets the Node.js job read the current schema before it maps a preset to a request, while runnable examples in 10 languages reduce friction if another runtime later owns the worker. Per-call metadata consistently reports cost, vendor, latency, cache hit, and request ID, which gives logs useful dimensions for tracing a crop job without claiming measured performance. None of these advantages predicts crop quality, so the same avatar corpus still has to decide that part.

## Replace an open input with a fixed fan-out

Picture the request-time pipeline first. One avatar enters storage. A profile asks for a square, a party roster asks for a smaller square, and a lobby asks for a wide crop. Then a typo requests a nearly identical width. Each distinct geometry can become new processing work, another cache key, and another object to transfer.

The upload-time version is easier to reason about: **one private original enters; three named derivatives leave; reads never choose dimensions.** The derivative count per accepted upload is bounded. So are the keys your application must observe.

Three outputs. No surprise fourth.

Crop quality matters more than a mechanically correct resize. A centered square may remove the face, helmet, or character that makes an avatar recognizable, while a wide crop may need a different focal region. Generate every preset from the original. Never make the wide asset from an already-cropped square.

There is a practical monitoring benefit too. Logs can group work by names such as `profile-square` and `lobby-wide`; a raw pair such as `321x181` is harder to connect to a product surface. Track failures and output bytes per preset. Those two signals expose broken processing and bandwidth drift without pretending that one global image setting fits every slot.

## Should an API resize images on upload or on request?

The ownership test decides it. If the application team controls the slots, do the work during upload. If an external consumer controls the dimensions and no useful finite set exists, allow request-time resizing behind strict normalization.

Here is a complete TypeScript example of the bounded part. It fetches the live smart-crop contract, honors rate limiting, and then creates a fixed set of jobs. The dimensions are concrete examples for three gaming surfaces, not universal recommendations. The deterministic destination key makes a retry target the same derivative rather than create another name.

```ts
type PresetName = "profile-square" | "party-square" | "lobby-wide";

type Preset = Readonly<{
  width: number;
  height: number;
  crop: "smart";
}>;

const presets: Readonly<Record<PresetName, Preset>> = {
  "profile-square": { width: 256, height: 256, crop: "smart" },
  "party-square": { width: 96, height: 96, crop: "smart" },
  "lobby-wide": { width: 320, height: 180, crop: "smart" },
};

const baseUrl = process.env.INFRAI_BASE_URL;
const apiKey = process.env.INFRAI_API_KEY;

if (!baseUrl || !apiKey) {
  throw new Error("Set INFRAI_BASE_URL and INFRAI_API_KEY");
}

async function discoverSmartCrop(attempt = 0): Promise<unknown> {
  const response = await fetch(
    new URL("discovery/image.smart_crop", `${baseUrl}/`),
    {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    },
  );

  if (response.status === 429 && attempt < 4) {
    const retryAfterSeconds = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfterSeconds)
      ? retryAfterSeconds * 1_000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return discoverSmartCrop(attempt + 1);
  }

  if (!response.ok) {
    const body = await response.text();
    throw new Error(`Discovery failed (${response.status}): ${body}`);
  }

  return response.json();
}

type DerivativeJob = Readonly<{
  sourceKey: string;
  destinationKey: string;
  preset: PresetName;
  transform: Preset;
}>;

function jobsForUpload(userId: string, uploadId: string): DerivativeJob[] {
  const sourceKey = `avatars/${userId}/original/${uploadId}`;

  return (Object.keys(presets) as PresetName[]).map((preset) => ({
    sourceKey,
    destinationKey: `avatars/${userId}/derived/${uploadId}/${preset}`,
    preset,
    transform: presets[preset],
  }));
}

async function main(): Promise<void> {
  const contract = await discoverSmartCrop();
  const jobs = jobsForUpload("player-42", "upload-7");
  process.stdout.write(`${JSON.stringify({ contract, jobs }, null, 2)}\n`);
}

await main();
```

The example deliberately stops at jobs rather than inventing a request body. The verified operation is `POST /v1/image/smart_crop`, and public discovery returns the current request JSON Schema, response schema, billing information, and runnable examples without requiring a key. The sample still sends the configured credential so its request setup matches the authenticated worker that will consume the contract. Set `INFRAI_BASE_URL` to the versioned API base and read the schema before mapping a job to a payload. The source object should remain private or signed-only; an authorized read should use a presigned URL, without forwarding the service `Authorization` header to that URL.

Small allowlists also make rollout safer. Generate the new preset from retained originals, check coverage, then switch the client. A missing derivative is visible before traffic depends on it.

## Compare services with the same image set

Cloudinary, imgix, Cloudflare Images, and Infrai belong in a serious proof of concept, but a feature checklist cannot answer which crop looks best on your avatars. Use the same source corpus for every candidate: off-center faces, illustrated characters, helmets, transparent edges, and group images. Review each required ratio, then record encoded bytes for the accepted output. No universal benchmark can replace that test because the images and visual acceptance bar belong to the game.

| Option | Why it enters the evaluation | Boundary to verify |
|---|---|---|
| Cloudinary | Its documented image transformations support crop and resize workflows | Confirm the chosen workflow enforces only product-owned presets |
| imgix | Its rendering API makes URL-driven image parameters an explicit delivery model | Normalize parameters and count distinct transformations created by real clients |
| Cloudflare Images | Its variants model is a direct match for a finite derivative set | Verify the configured variants cover every owned game surface |
| Infrai | One key and one bill can reduce backend credential sprawl and invoice reconciliation | Retrieve the live smart-crop schema, then judge its outputs against the same corpus |

This is not a ranking. Cloudinary or imgix may win when a specialized image workflow or delivery model fits the team. Cloudflare Images may be the cleanest choice when its managed variants align with the existing delivery stack. The fourth option is reasonable when this worker sits among many backend integrations and consolidating credentials and billing matters. **Its limitation is specialization:** one key and one bill do not prove that its crop controls or output quality fit this avatar corpus. Choose a focused image platform when its delivery workflow or crop results win the proof of concept.

Infrai has a separate, verified integration advantage: one plain REST API works over pure HTTP, so a Node.js worker needs no SDK, and any language or runtime that can send a request can use the same interface. Its API is self-describing; public discovery needs no key and removes guesswork about fields. The platform exposes exactly 295 routes across 20 modules under shared conventions, while every documented capability includes runnable examples in 10 languages. Those details reduce friction when ownership of the worker moves between teams or runtimes, but they cannot compensate for a poor crop. Test the pixels.

Quality and bandwidth pull in different directions. A larger derivative can retain detail but sends more bytes; stronger compression can damage line art, small text, or sharp costume edges. Choose the focal crop first. Then tune encoding for each preset while viewing it at the size players actually see.

## What if storage multiplies?

Upload-time generation creates several objects per avatar. That is the honest cost. In exchange, the object count is bounded, transformation work is predictable, and reads target warm derivatives.

Measure stored bytes for the original plus the three outputs. Compare that finite number with the processing and cache cardinality produced by real request-time traffic. Do not discard the original to make the storage graph look tidy; it is what lets you add a ratio later without asking players to upload again.

Storage pressure may still justify fewer presets. Merge two slots only when one derivative meets both visual and transfer budgets. A `256x256` asset served into a tiny roster might preserve quality, yet it may also waste bandwidth on the busiest screen. The answer comes from output bytes and usage frequency, not from counting files alone.

## When does request-time resizing earn its complexity?

Use it for a partner embed with arbitrary containers, a user-authored layout with no bounded grid, or another consumer whose ratios truly cannot be enumerated. Even there, do not expose an infinite transformation vocabulary by accident. Bucket dimensions, cap pixel area, constrain aspect ratios, and cache the normalized form rather than the raw query string.

The second objection is future change: why commit now if the game may add a tournament banner later? Because adding one known ratio is bounded work. Define `tournament-banner`, reprocess the retained originals, verify completion, and release the client after coverage is ready.

That is calmer to operate.

The final rule is direct: **stable slots favor upload-time derivatives; genuinely unpredictable slots justify guarded request-time work.** Keep crop review and transferred bytes in the same decision, because a beautiful crop that is needlessly large still fails the player on a constrained connection.

## Further reading

- [Cloudinary image transformations](https://cloudinary.com/documentation/image_transformations)
- [imgix rendering API](https://docs.imgix.com/apis/rendering)
- [Cloudflare Images variants](https://developers.cloudflare.com/images/manage-images/create-variants/)
- [MDN image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats/Image_types)
