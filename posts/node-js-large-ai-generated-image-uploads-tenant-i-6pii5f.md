# Node.js Large AI-Generated Image Uploads: Tenant Isolation in Multipart Object Storage

Short answer: a customer-support app storing large AI-generated images needs private object storage, tenant-scoped keys, short-lived presigned part uploads, and a database-owned multipart state machine; smaller images can stay on a single upload. The deciding constraint is tenant isolation, not whether multipart sounds more production-ready.

I tested the design decision against the failure that is expensive to explain later: a retry for tenant A accidentally reuses an upload record, object key, or download capability belonging to tenant B. A single request is easier to ship, but it makes a failed large transfer start over. Multipart is more work, yet it gives the worker a place to resume and an explicit place to abort. Before copying either path, measure image size, transfer duration, retry rate, and the number of abandoned sessions in your own support workload.

## What governance contract should protect Node.js large AI-generated image uploads?

Start with identity, then bytes. Every generated image should have an application record containing a tenant ID, an image ID, the intended object key, and an upload status. The storage key should be derived from the trusted tenant and image record on the server. A browser-supplied key is input, not authorization.

For example, a key can be shaped like `tenants/{tenantId}/images/{imageId}/original`. The exact naming convention is less important than making the tenant boundary visible in the record and checking it at every control-plane operation. A request to presign a part must verify that the caller can access the image record before it receives any URL. The same check belongs on completion, abort, and retrieval.

Keep the image private by default. A presigned URL grants temporary access to a specific operation; it is not a replacement for application authorization. AWS describes presigned URLs as bearer tokens, which is a useful security rule for any S3-compatible design: do not log them casually, put them in durable job records, or treat possession of a URL as proof of tenant membership.

The small detail that saves a large incident is the upload ID. Store it with the image record, never only in the Node.js process that created it. A worker restart should lose memory, not ownership information.

## How should Node.js implement large AI-generated image parts in object storage?

The protocol has five distinct decisions. Create a multipart upload for one server-derived key. Presign one part at a time, with a part number bound to that upload ID. Send the bytes directly to object storage. Complete only from the set of successfully uploaded parts. Abort when the job is cancelled, expires, or cannot make progress.

The state machine below is intentionally storage-agnostic. The adapter behind each action can speak the object storage API used by the deployment, while the application keeps the important invariants in one place.

```ts
type UploadStatus = "created" | "uploading" | "completing" | "complete" | "aborted";

type PartReceipt = {
  partNumber: number;
  etag: string;
  size: number;
};

type ImageUpload = {
  tenantId: string;
  imageId: string;
  objectKey: string;
  uploadId: string;
  status: UploadStatus;
  parts: PartReceipt[];
};

function recordPart(upload: ImageUpload, receipt: PartReceipt): ImageUpload {
  if (upload.status !== "uploading") {
    throw new Error(`part ${receipt.partNumber} is not valid in ${upload.status}`);
  }

  const alreadyRecorded = upload.parts.some(
    (part) => part.partNumber === receipt.partNumber,
  );
  if (alreadyRecorded) return upload;

  return { ...upload, parts: [...upload.parts, receipt] };
}

function beginCompletion(upload: ImageUpload): ImageUpload {
  if (upload.status !== "uploading" || upload.parts.length === 0) {
    throw new Error("completion requires recorded parts");
  }
  return { ...upload, status: "completing" };
}

function markAborted(upload: ImageUpload): ImageUpload {
  if (upload.status === "complete") return upload;
  return { ...upload, status: "aborted" };
}
```

The receipt is part of the correctness boundary. Save the part number and the storage response before asking for another part or attempting completion. Sort receipts by part number when constructing the completion request, and reject a receipt whose tenant, image ID, upload ID, or object key does not match the database row. Those checks are cheap. Reconstructing which bytes belonged to which tenant after a retry is not.

Do not infer success from the response to the last part. Completion is a separate operation. Only after completion succeeds should the image row become readable and the application create a download URL. If completion fails, retain enough state to retry the completion operation or abort the upload according to the job policy.

Then the process dies.

That ordinary failure is why the state record deserves more design attention than the upload loop. Imagine the worker has sent part 7, received its receipt, and is about to persist it when the runtime exits. On restart, a naive loop sees no receipt and sends the bytes again; a different naive loop sees an upload ID but assumes every earlier part exists. Neither assumption is a tenant policy. A reconciler should reload the row, verify the object key and upload ID against the image and tenant, inspect the storage-side part inventory through the adapter, merge the known receipts idempotently, and decide whether to continue, complete, or abort. The same sequence handles a duplicate queue message, a delayed database write, and a user cancelling an image while the final part is in flight. I don't need a clever retry algorithm here. I need one that cannot turn an ambiguous transfer into a cross-tenant read.

## How should reliability work govern an orphaned upload?

Multipart creates storage state before it creates a finished object. That state needs a deadline. A cancellation endpoint should mark the application job as aborting, call the storage abort operation, and then record the terminal result. A scheduled worker should find rows stuck in `created`, `uploading`, or `completing` beyond the product's allowed age and reconcile them.

The order matters during retries. A worker can crash after storage accepts a part but before the database records its receipt. On retry, list or otherwise reconcile the upload through the storage adapter, then merge receipts idempotently. A second completion attempt should use the same upload ID and the complete, sorted part set. An abort request should be safe to repeat at the application layer, with the terminal row preventing a later worker from reviving the image.

Here is the operational checklist I would put beside the queue consumer:

| Event | Application record | Storage action |
| --- | --- | --- |
| Create | Save tenant, image, key, upload ID, and `uploading` status | Create multipart upload |
| Part accepted | Save part number, receipt, and size idempotently | Upload the numbered part |
| All parts present | Change status to `completing` | Complete with the sorted receipts |
| Completed | Mark the image readable | Issue a bounded presigned download |
| Cancelled or expired | Mark `aborted` and stop retries | Abort the multipart upload |

One sentence is enough for the on-call rule: an active upload without a progressing job is a cleanup candidate.

## How can an evaluation of tenant boundaries go beyond an object-storage API?

“S3-compatible” describes an interoperability target, not a complete tenancy model. Evaluate the controls around the object API as well as the upload verbs. The minimum review should cover bucket or namespace separation, server-side key construction, credentials, URL lifetime, encryption policy, deletion behavior, audit events, and recovery procedures.

There are two sensible layouts. A shared bucket with tenant-prefixed keys is operationally compact, but every read, write, list, and delete path must enforce the prefix. Separate buckets or namespaces can make administrative boundaries clearer, at the cost of more provisioning and policy management. Neither layout is automatically isolated because of its name.

The presigning service should accept an image ID, not an arbitrary object key. It resolves the image under the authenticated tenant, confirms the expected status, and then signs the exact part number or download action. Never let a client choose a different bucket or upload ID by editing a request body. In TypeScript, that means the control-plane function should take a validated domain object rather than pass untrusted storage parameters through to an adapter.

No shared upload IDs.

The catch is that a shared-prefix design is not suitable when your compliance boundary requires independently administered storage accounts or separate retention policies. Choose separate namespaces or a provider with those controls when that requirement is real. Conversely, do not choose separate buckets just to feel safer if your team cannot automate policy creation, deletion, and audit coverage; an isolation boundary that drifts is a paper boundary.

For regulated support data, map the required controls to the applicable program and contract. FedRAMP is a federal authorization program, not a generic stamp that proves a particular storage configuration is appropriate. Your threat model still needs to state who can create URLs, how long they live, and what happens to abandoned data.

## Which cost and reliability signals decide single upload or multipart?

Measure the distribution, not the largest image someone can generate. Record object size, time to first byte, total transfer time, retry count, bytes resent, completion latency, and active-upload age by tenant. Track authorization denials and key-mismatch attempts separately; those are isolation signals, not just HTTP errors.

For a small image, a single presigned upload may have the better failure surface: one status transition and no multipart cleanup. For a large image crossing an unreliable connection, multipart can reduce the amount resent after a transient interruption, but it adds parts, receipts, retries, reconciliation, and abort work. Your mileage may vary because the threshold depends on network path and worker placement; I wouldn't hard-code a size threshold without those measurements.

Cost belongs in the same review, after correctness. Count stored bytes, request volume, retry bytes, and abandoned multipart state. Token spend from image generation is already a concern for a solo builder, so the storage worker should avoid re-generating an image merely because its transfer bookkeeping was lost. A durable state record is usually the cheaper engineering decision than an elegant but memory-only loop, even before a bill is involved.

The result of this experiment is a decision rule, not a universal recipe: keep ordinary output on the simplest private upload path; move genuinely large or failure-prone output to multipart; make tenant identity and upload ownership explicit; and treat completion and abort as first-class state transitions. Test tenant cross-talk, worker restart, duplicate part receipts, expired URLs, cancelled jobs, and cleanup reconciliation before calling the image pipeline ready.

## References

- https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html
- https://www.fedramp.gov/
