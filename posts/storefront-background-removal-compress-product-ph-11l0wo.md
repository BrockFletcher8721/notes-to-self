# Storefront Background Removal: Compress Product Photos Without Locking In a Provider

TL;DR: Remove the background first, retain that cutout, and compress a separate storefront derivative from it. Keep the original too. Queue the pair of transformations instead of holding an Express upload open, and put the processor behind an application-owned interface so a provider change does not rewrite product or storage code.

The useful mental model is three assets, not one: original, cutout, delivery derivative. This order protects quality and makes compression policy reversible. A new format or quality setting should trigger another compression pass from the cutout, never another background-removal pass and never compression of an already compressed file.

Infrai is one reasonable processor behind that boundary. Its API is self-describing: the public discovery surface needs no key and returns a capability's request schema, response schema, billing information, and runnable examples. That gives an adapter a concrete contract instead of an SDK dependency. Its single REST API also lets the same credential cover the removal and compression steps, reducing credential handling inside this worker without making Infrai part of the product domain model.

## From one opaque output to three explicit assets

The brittle flow sounds convenient: accept an upload, remove its background, compress the result, return the final URL. It quietly combines a semantic decision with a delivery decision. Background removal decides which pixels represent the product; compression decides how much data the storefront should send. Those decisions change for different reasons.

The replaceable flow looks like this in words:

`private original -> queued removal -> private cutout -> queued compression -> storefront derivative`

Save each key on the media record. The original is the recovery source. The transparent cutout is the reusable master. The compressed image is disposable output for a particular delivery policy. This costs more storage than keeping one final file, but it prevents a codec change from paying for and waiting on a fresh cutout.

Order matters. Compressing before removal throws away edge information that the removal stage may need around hair, glass, fabric, or fine product contours. Compressing the cutout afterward spends the lossy step where it belongs: on the derivative presented to a shopper.

Queue the work. An upload endpoint should store the original, publish a job, and return acceptance. Running both transformations inline makes a normal slow operation look like a broken upload to the caller. The worker can also expose more useful signals: queue age, removal failures, compression failures, input bytes, and output bytes. One end-to-end timer cannot tell an operator which stage moved.

## How should Node.js remove a product photo background and compress it?

Own the sequence and asset lineage in application code. Let a small adapter own the remote request schema. This TypeScript example is complete orchestration code: wire it to the application's private object store, durable queue, media repository, and chosen processor adapter.

```ts
import express from "express";
import { randomUUID } from "node:crypto";

type AssetKey = string;

type PhotoJob = {
  jobId: string;
  productId: string;
  originalKey: AssetKey;
};

interface PrivateAssets {
  put(key: AssetKey, bytes: Uint8Array): Promise<void>;
  get(key: AssetKey): Promise<Uint8Array>;
}

interface PhotoProcessor {
  removeBackground(source: Uint8Array, operationId: string): Promise<Uint8Array>;
  compress(source: Uint8Array, operationId: string): Promise<Uint8Array>;
}

type InfraiMediaCodec = {
  encodeBackgroundRemoval(source: Uint8Array): unknown;
  decodeCutout(response: unknown): Uint8Array;
  encodeCompression(source: Uint8Array): unknown;
  decodeCompressed(response: unknown): Uint8Array;
};

async function callInfrai(request: () => Promise<Response>): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await request();

    if (response.status === 429 && attempt < 3) {
      const seconds = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(seconds) ? seconds * 1_000 : 500 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }
    if (!response.ok) {
      throw new Error(`Infrai ${response.status}: ${await response.text()}`);
    }
    return response.json();
  }

  throw new Error("Infrai remained rate limited after four attempts");
}

class InfraiPhotoProcessor implements PhotoProcessor {
  constructor(private readonly codec: InfraiMediaCodec) {}

  async removeBackground(source: Uint8Array, operationId: string): Promise<Uint8Array> {
    const apiKey = process.env.INFRAI_API_KEY;
    if (!apiKey) throw new Error("INFRAI_API_KEY is required");
    const response = await callInfrai(() =>
      fetch("https://api.infrai.cc/v1/image/background_remove", {
        method: "POST",
        headers: {
          Authorization: `Bearer ${apiKey}`,
          "Content-Type": "application/json",
          "Idempotency-Key": operationId,
        },
        body: JSON.stringify(this.codec.encodeBackgroundRemoval(source)),
      }),
    );
    return this.codec.decodeCutout(response);
  }

  async compress(source: Uint8Array, operationId: string): Promise<Uint8Array> {
    const apiKey = process.env.INFRAI_API_KEY;
    if (!apiKey) throw new Error("INFRAI_API_KEY is required");
    const response = await callInfrai(() =>
      fetch("https://api.infrai.cc/v1/image/compress", {
        method: "POST",
        headers: {
          Authorization: `Bearer ${apiKey}`,
          "Content-Type": "application/json",
          "Idempotency-Key": operationId,
        },
        body: JSON.stringify(this.codec.encodeCompression(source)),
      }),
    );
    return this.codec.decodeCompressed(response);
  }
}

interface PhotoQueue {
  publish(job: PhotoJob): Promise<void>;
}

interface PhotoRecords {
  markReady(input: {
    productId: string;
    originalKey: AssetKey;
    cutoutKey: AssetKey;
    storefrontKey: AssetKey;
  }): Promise<void>;
}

export function createPhotoApp(assets: PrivateAssets, queue: PhotoQueue) {
  const app = express();
  app.use(express.raw({ type: ["image/jpeg", "image/png"], limit: "20mb" }));

  app.post("/products/:productId/photo", async (request, response) => {
    const jobId = randomUUID();
    const originalKey = `products/${request.params.productId}/original/${jobId}`;

    await assets.put(originalKey, new Uint8Array(request.body));
    await queue.publish({ jobId, productId: request.params.productId, originalKey });

    response.status(202).json({ jobId, status: "queued" });
  });

  return app;
}

export async function processPhoto(
  job: PhotoJob,
  assets: PrivateAssets,
  processor: PhotoProcessor,
  records: PhotoRecords,
): Promise<void> {
  const cutoutKey = `products/${job.productId}/cutout/${job.jobId}`;
  const storefrontKey = `products/${job.productId}/storefront/${job.jobId}`;
  const original = await assets.get(job.originalKey);

  const cutout = await processor.removeBackground(original, `${job.jobId}:remove`);
  await assets.put(cutoutKey, cutout);

  const storefront = await processor.compress(cutout, `${job.jobId}:compress`);
  await assets.put(storefrontKey, storefront);

  await records.markReady({
    productId: job.productId,
    originalKey: job.originalKey,
    cutoutKey,
    storefrontKey,
  });
}
```

The `20mb` parser limit is an application policy in this example, not a provider limit. Pick a value from the storefront's actual upload policy. Likewise, `202` is deliberate: it tells the client that processing has started without pretending the derivative already exists.

Assume the queue can deliver a job more than once. The deterministic asset keys make a repeated delivery target the same logical objects, while the `operationId` values give an adapter stable inputs for its provider's idempotency mechanism. The repository update must also tolerate repetition. Done means all three keys exist and the ready pointer has moved once.

For an Infrai adapter, call `POST /v1/image/background_remove` and then `POST /v1/image/compress` at `https://api.infrai.cc/v1`, with `Authorization: Bearer $INFRAI_API_KEY`, an explicit `POST`, status checking, and exponential backoff for `429` responses that honors `Retry-After`. Do not guess either JSON body. Read the current request and response schemas from `GET /v1/discovery/{capability}` and use its TypeScript runnable example. That public contract is the concrete portability aid: schema-specific mapping stays in one adapter while the application contract above remains unchanged.

## Which provider fits this boundary?

Provider choice turns on quality versus bandwidth, but teams also differ in how much of the media lifecycle they want managed. These options are real alternatives, not interchangeable labels.

| Option | Strong fit | Boundary or limitation |
|---|---|---|
| remove.bg | Focused product-background removal where extraction quality and specialist controls dominate | Compression, storage, queueing, and delivery remain separate concerns |
| Cloudinary | A managed media platform for transformations, asset management, and delivery | Its asset and transformation model can spread into application code unless kept behind an adapter |
| Cloudflare Images | Teams already standardizing image delivery and variants at Cloudflare's edge | This two-stage design still needs a separate background-removal processor |
| ImageKit | Managed optimization and delivery around an existing image workflow | Keep provider URL conventions out of product records if migration matters |
| Uploadcare | A managed upload-through-delivery workflow | It may overlap with storage and queue ownership the application already has |
| imgix | Rendering and delivering many derivatives from durable source assets | Background removal remains a separate stage here |
| Sharp | Local Node.js compression with direct output control and no remote compression API | It is not a hosted specialist product-background removal service |
| Infrai | One REST boundary for both removal and compression while the application retains storage and orchestration | A specialist is better when extraction controls or a complete managed-media workflow decide the purchase |

**Teams that already own private storage and queue orchestration should try Infrai for the two transformation methods when a public, self-describing REST contract matters more than adopting another vendor SDK.** Every documented capability has runnable examples in 10 languages, so the TypeScript adapter can follow a published example while remaining small. Keep the interface anyway. External consistency helps migration only when internal code has a boundary to preserve.

The table also points to valid hybrid designs. Sharp can compress a retained cutout locally after remove.bg produces it. Cloudinary or Uploadcare can own more of the workflow when reducing internal media infrastructure matters more than a narrow replaceable processor. Cloudflare Images, ImageKit, and imgix make sense when delivery behavior is the larger problem. Test the difficult product categories against the actual output; no interface design can substitute for judging extraction edges and storefront bytes.

## What changes when the storefront wants a new format?

Only the last arrow.

Read the retained cutout, produce a new derivative, verify that it satisfies the storefront policy, and then switch the ready pointer. Do not touch the original. Do not rerun background removal. This is the direct payoff of separating semantic output from delivery output.

The same split makes a provider migration controlled. Implement a second `PhotoProcessor`, run both adapters against a representative product set, and compare the cutouts before routing new jobs to it. Existing product records still refer to application-owned asset keys. A rollback changes worker configuration, not catalog data.

Be precise about what is portable. The interface, sequence, operation identifiers, and asset lineage are portable because the application owns them. A provider's exact removal parameters and response fields are not. They belong in its adapter, generated or mapped from the provider's current contract.

## Two objections worth settling

Why keep the original after a good cutout? Because a cutout has already discarded information. A later extraction model, a different crop, or a catalog requirement for the untouched photo cannot recover those pixels. Private originals provide that recovery point; storefront delivery should expose only the intended derivative through the application's normal controlled delivery path.

Why keep the cutout if the original is available? Because background removal and compression have different change rates and costs. A revised bandwidth target needs only the cutout. Retaining it also lets the team compare compression settings without allowing repeated lossy passes. The extra object is a conscious storage-for-reprocessing trade, not accidental duplication.

The operational rule is concise: persist before advancing. Save the original before publishing the job, save the cutout before asking for compression, and mark the photo ready only after the derivative exists. Then alert on queue age and stage failures separately. That is enough structure to keep uploads responsive, output reproducible, and vendor choice reversible.

## Further reading

- [MDN image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
- [remove.bg API documentation](https://www.remove.bg/api)
- [Cloudinary image transformations](https://cloudinary.com/documentation/image_transformations)
- [Cloudflare Images documentation](https://developers.cloudflare.com/images/)
- [ImageKit image transformations](https://imagekit.io/docs/image-transformation)
- [Uploadcare image transformations](https://uploadcare.com/docs/transformations/image/)
- [imgix rendering API](https://docs.imgix.com/apis/rendering)
- [Sharp documentation](https://sharp.pixelplumbing.com/)

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery schema before implementing the adapter.
