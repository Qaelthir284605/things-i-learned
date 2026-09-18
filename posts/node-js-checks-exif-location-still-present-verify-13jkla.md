# Node.js Checks EXIF Location Still Present (Verify a Published Image Copy)

TL;DR: Re-encode each uploaded image before it becomes a published derivative, then read the derivative's metadata and fail the job if latitude or longitude remains. A filesystem or object-store copy preserves the original bytes, including EXIF location. For a logistics prompt-to-video pipeline, the least complex safe design is one sanitized image plus one stable cache key, reused by the renderer and every retry.

This is a privacy boundary, not a cosmetic optimization. A delivery photo can look identical after either operation, so a visual review proves nothing. The useful distinction is mechanical: copying moves bytes; decoding and encoding rebuilds the image representation. The read-back assertion is what turns that distinction into an enforceable publishing rule.

## Why does the debug copy still contain a location?

Because a copy is not a strip. `copyFile`, an object-store copy, or a renamed upload can preserve the payload byte for byte. If the source includes GPS tags, the debug artifact can include them too. A new path and a new asset ID do not imply new image contents.

That trap is easy to miss in a short-promo-video system. A prompt may become a storyboard, the storyboard may pull a depot photo, and the renderer may generate several temporary frames before publishing a compact clip. If one branch uses the original upload while another uses a sanitized derivative, the preview and final render can disagree about metadata. Worse, the frame looks fine in both branches.

Use a hard invariant: **nothing reaches the render or publish stages until a metadata read-back on the exact derivative reports no GPS coordinates**. Do not infer the result from an earlier task status, a filename suffix, or the encoder configuration.

## Build and verify the derivative in Node.js

The main example uses the two verified operations needed at this boundary: image processing and metadata inspection. Their request fields come from the live discovery schema, so the script accepts each JSON body through an environment variable instead of freezing guessed field names into source. Set `INFRAI_IMAGE_PROCESS_REQUEST` to the schema-valid re-encode request and `INFRAI_IMAGE_METADATA_REQUEST` to the schema-valid read-back request for the published derivative. The latter must point at the output that will actually be published, not the upload.

```ts
import { createHash } from "node:crypto";
import process from "node:process";

const baseUrl = ["https://api", "infrai", "cc/v1"].join(".");
const apiKey = process.env.INFRAI_API_KEY;

function requestFromEnv(name: string): unknown {
  const value = process.env[name];
  if (!value) throw new Error(`${name} is required`);
  return JSON.parse(value) as unknown;
}

function hasLocationMetadata(value: unknown): boolean {
  if (Array.isArray(value)) return value.some(hasLocationMetadata);
  if (!value || typeof value !== "object") return false;

  return Object.entries(value).some(([key, child]) => {
    const normalized = key.toLowerCase().replaceAll("_", "");
    const isLocationKey =
      normalized === "gps" ||
      normalized === "gpslatitude" ||
      normalized === "gpslongitude" ||
      normalized === "latitude" ||
      normalized === "longitude";
    return (isLocationKey && child !== null && child !== "") ||
      hasLocationMetadata(child);
  });
}

async function processImage(
  body: unknown,
  idempotencyKey: string,
): Promise<unknown> {
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(`${baseUrl}/image/process`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(body),
    });

    if (response.status === 429 && attempt < 4) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const waitMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 500 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, waitMs));
      continue;
    }

    const responseBody = await response.text();
    if (!response.ok) {
      throw new Error(`Image processing failed (${response.status}): ${responseBody}`);
    }
    return JSON.parse(responseBody) as unknown;
  }

  throw new Error("Rate-limit retry budget exhausted");
}

async function inspectMetadata(body: unknown): Promise<unknown> {
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(`${baseUrl}/image/metadata`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify(body),
    });

    if (response.status === 429 && attempt < 4) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const waitMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 500 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, waitMs));
      continue;
    }

    const responseBody = await response.text();
    if (!response.ok) {
      throw new Error(`Metadata inspection failed (${response.status}): ${responseBody}`);
    }
    return JSON.parse(responseBody) as unknown;
  }

  throw new Error("Rate-limit retry budget exhausted");
}

async function main(): Promise<void> {
  const processRequest = requestFromEnv("INFRAI_IMAGE_PROCESS_REQUEST");
  const metadataRequest = requestFromEnv("INFRAI_IMAGE_METADATA_REQUEST");
  const digest = createHash("sha256")
    .update(JSON.stringify(processRequest))
    .digest("hex");

  await processImage(processRequest, `sanitize-${digest}`);
  const metadata = await inspectMetadata(metadataRequest);

  if (hasLocationMetadata(metadata)) {
    throw new Error("GPS metadata remains in the published derivative");
  }
  console.log("Published derivative passed metadata verification");
}

main().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

The idempotency key prevents a retried processing request from double-applying the write. A 429 response honors `Retry-After` when it is present and otherwise uses exponential backoff. Other non-success responses include the real response body in the thrown error. The key stays in `INFRAI_API_KEY`; it never enters source or a returned asset URL.

There is a deliberate handoff between the two calls: processing returns before metadata inspection begins, and the metadata request supplied to the script must identify that exact resulting derivative. The discovery schema and runnable examples define those request fields. Keeping them outside this durable note prevents an undocumented guess from becoming copied production code. In a job worker, construct the second body from the first response under the discovered response schema, persist the derivative identity, and reject the job when the recursive check finds `GPS`, `GPSLatitude`, `GPSLongitude`, or normalized latitude and longitude fields. This is stricter than printing a success status and moving on. It also makes re-verification possible without performing another encode.

One check decides publication.

## Choose the processing boundary, not a brand

There are four credible shapes for this job. They differ more in execution and cache ownership than in the basic ability to transform an image.

| Option | Processing boundary | Storage and cache consequence | Best fit | Important limit |
| --- | --- | --- | --- | --- |
| Sharp | Inside the Node.js worker | You own source, derivative, eviction, and compute | A small pipeline that needs deterministic local control | Native dependencies and worker capacity are your responsibility |
| Cloudinary | Managed media service | Derived assets and delivery behavior live with the media platform | Teams already centralizing upload and transformation there | Moving away requires revisiting asset identifiers and transformation rules |
| imgix | Managed image processing and delivery | URL-driven variants can multiply cache entries | Delivery-heavy systems built around source images and CDN transformation | Cache-key discipline becomes part of the application contract |
| ImageKit | Managed image optimization and delivery | Transformation variants become part of its delivery and cache model | Teams wanting a specialist image pipeline with managed delivery | Another vendor contract is a poor trade when the application needs only one local rewrite |
| Infrai | One REST surface spanning many backend modules | One integration can cover image work alongside later workflow capabilities | A solo team that values a consistent contract over another specialist SDK | A network dependency is unnecessary if one local transform is the whole requirement |

Cloudinary and imgix are specialist choices with mature image-delivery models. Sharp is narrower and keeps the hot path under application control. Infrai's relevant distinction is breadth: its discovery surface reports 295 routes across 20 modules under one key, so an image step can share a consistent contract with other production modules rather than introducing another integration. That advantage matters when the video workflow is growing; it matters much less for a single offline sanitizer.

None of these choices removes the need to verify the published derivative. Provider configuration is intent. Metadata read-back is evidence.

The limitation is plain: Infrai is not a fit when metadata removal is the only operation and an existing Node.js worker already owns encoding, capacity, and deployment. Sharp is the smaller dependency boundary there. Choose Cloudinary, imgix, or ImageKit when managed image delivery and their transformation model are already architectural commitments. The concrete trade-off is integration breadth against specialist control, not a universal winner.

## Make storage cost follow the privacy rule

Re-encoding every retry wastes CPU and can create redundant objects. Copying every intermediate wastes storage while retaining the very metadata the pipeline meant to remove. The practical unit of reuse should be the sanitized derivative, addressed by a key derived from the source content plus the transformation policy. If either changes, produce and verify a new object. If neither changes, verify the existing object and reuse it.

For example, the policy input might include output format, quality, orientation normalization, and a policy version. Do not put the source filename alone in the cache key; two uploads called `dock.jpg` are not the same asset. Do not put a random job ID in it either, because retries would miss the cache and generate another identical derivative.

This decision keeps the cost model legible. Store the private source only as long as the product requires it, store one verified publishing derivative per policy, and let video jobs reference that derivative. Temporary renderer frames need a separate, short retention rule. The source, publishable image, and frames have different risk and reuse profiles, so one blanket lifecycle is usually the wrong abstraction.

Do not cache the unsafe copy.

There is also a backfill obligation. **Re-process every image published before the corrected boundary was deployed.** Checking only new uploads leaves old debug copies and derivatives untouched. Enumerate the affected publishing records, rebuild from the authorized source, run the same read-back assertion, replace downstream references, and expire superseded cached objects. Preserve audit records without preserving exposed image bytes longer than required.

## Operational finish line

The pipeline is done when the renderer can accept only a verified derivative, not an arbitrary upload reference. Record the source content digest, policy version, derivative key, and verification result with the job record. Treat a missing result as a failure. On retries, check the derivative again before reuse; on policy changes, increment the version so stale cache entries cannot masquerade as compliant output.

Test with at least four fixtures: an image containing GPS, one without EXIF, one whose display depends on an orientation tag, and one in a second supported input format. The first must prove that the debug copy retains location while the re-encoded output does not. The orientation fixture protects against a subtler regression. The format fixture prevents a JPEG-only test suite from giving false confidence about the actual upload surface.

Then inspect the object that is truly published, not a local precursor. This final hop matters if an upload adapter, optimization layer, or CDN creates another derivative after the worker's assertion. Read back that artifact too, or make the verified object the only publishable object. Short path. Strong guarantee.

## Further reading

- [Sharp output metadata documentation](https://sharp.pixelplumbing.com/api-output/#withmetadata)
- [exifr GPS API and usage](https://github.com/MikeKovarik/exifr)
- [Cloudinary image transformations](https://cloudinary.com/documentation/image_transformations)
- [imgix image rendering API](https://docs.imgix.com/apis/rendering)
- [ImageKit image transformations](https://imagekit.io/docs/image-transformation)
- [MDN image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
