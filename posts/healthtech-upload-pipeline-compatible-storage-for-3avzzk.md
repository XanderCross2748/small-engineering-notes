# Healthtech Upload Pipeline: Compatible Storage for Startup Avatar Images and Reports

Short answer: put private avatar originals, derived thumbnails, and generated health reports in object storage, but keep authorization and asset state in the application database. Send bytes directly between the client and storage with short-lived signed transfers, isolate cleanup policies by object class, and test large report delivery before choosing on price.

For a one-person SaaS, this is mostly a time-allocation decision. Byte storage is undifferentiated. Customer authorization, deletion semantics, and report access are product behavior. I want the first part outsourced and the second part explicit enough to inspect on a Friday afternoon, because shipping weekly matters more than maintaining an elaborate media stack.

Keep it dull.

## Report throughput sets the storage boundary

Start with the heavier object. Generated health reports may be much larger than avatars, and their downloads are authenticated, so the useful boundary separates the authorization decision from the byte transfer. If the application proxies a report, its network and connections scale with every customer download. With a signed transfer, the application authenticates the customer and grants bounded access while object storage carries the response. The same boundary works for uploads in reverse.

This changes the selection test. Storage capacity is easy to compare, but the healthtech job depends on sustained delivery, interrupted-download behavior, and predictable authorization under concurrent access. Benchmark the full path from realistic client regions. Measure completion time, request volume, downloaded bytes, and recovery after an interrupted transfer. If resumed downloads are required, verify the exact compatible operation used by the adapter instead of treating a broad compatibility label as proof.

The application stays in control.

## How can a startup app keep avatar image storage compatible?

Treat an avatar upload as a state transition rather than a file field on an ordinary request. The client asks the application for permission; the application authenticates the customer, validates declared media type and size, creates an opaque key, and returns a short-lived signed upload. The client sends bytes straight to storage. A finalization request inspects the stored object before the database marks it ready.

Use separate namespaces such as `pending/{accountId}/{uploadId}`, `avatars/{accountId}/{assetId}/original`, `avatars/{accountId}/{assetId}/thumb-v1`, and `reports/{accountId}/{reportId}/{version}`. Keys organize operations; they don't grant access. Every read still starts with an ownership check and ends with a signed download whose lifetime is bounded. The profile stores a stable asset ID, not a signed URL. There are two clocks as well: the database owns product state such as pending, ready, replaced, or deleted, while storage retention owns bytes. A prefix-scoped lifecycle rule can remove abandoned objects from `pending/`; replacement logic keeps an old avatar available until its successor and thumbnail are ready. A bucket-wide cleanup rule is too blunt when avatars and retained health reports carry different deletion promises. Thumbnails are derived data, so give them deterministic, versioned keys, enqueue generation after finalization, and make the worker idempotent. Decode and validate the input, constrain transformation resources, and apply the product's metadata policy. This longer chain is deliberate — compatibility lives in the small set of storage operations, while correctness lives in application state and ordering.

## The narrow TypeScript transfer boundary

The application needs a storage adapter and a persistent upload record. Credentials stay on the server. This TypeScript sketch leaves framework and signing-library details outside the boundary so each candidate can implement the same contract without pushing provider-specific calls through the product code.

```ts
type UploadState = "pending" | "ready" | "deleted";

type AvatarUpload = {
  id: string;
  accountId: string;
  objectKey: string;
  contentType: "image/jpeg" | "image/png" | "image/webp";
  declaredBytes: number;
  state: UploadState;
  createdAt: Date;
};

type SignedTransfer = {
  url: string;
  method: "PUT" | "GET";
  headers: Record<string, string>;
  expiresAt: Date;
};

interface ObjectStore {
  signPut(input: {
    key: string;
    contentType: AvatarUpload["contentType"];
    contentLength: number;
    expiresInSeconds: number;
  }): Promise<SignedTransfer>;

  inspect(key: string): Promise<{
    bytes: number;
    contentType: string;
    checksum?: string;
  } | null>;

  signGet(input: {
    key: string;
    expiresInSeconds: number;
    downloadName?: string;
  }): Promise<SignedTransfer>;

  delete(key: string): Promise<void>;
}

const allowedTypes = new Set<AvatarUpload["contentType"]>([
  "image/jpeg",
  "image/png",
  "image/webp",
]);

async function beginAvatarUpload(
  input: {
    accountId: string;
    contentType: AvatarUpload["contentType"];
    bytes: number;
  },
  store: ObjectStore,
  save: (upload: AvatarUpload) => Promise<void>,
) {
  if (!allowedTypes.has(input.contentType)) {
    throw new Error("UNSUPPORTED_AVATAR_TYPE");
  }

  const uploadId = crypto.randomUUID();
  const objectKey = `pending/${input.accountId}/${uploadId}`;
  const upload: AvatarUpload = {
    id: uploadId,
    accountId: input.accountId,
    objectKey,
    contentType: input.contentType,
    declaredBytes: input.bytes,
    state: "pending",
    createdAt: new Date(),
  };

  await save(upload);
  const transfer = await store.signPut({
    key: objectKey,
    contentType: input.contentType,
    contentLength: input.bytes,
    expiresInSeconds: 300,
  });

  return { uploadId, transfer };
}

async function finalizeAvatarUpload(
  upload: AvatarUpload,
  store: ObjectStore,
  markReady: (uploadId: string, checksum?: string) => Promise<void>,
  enqueueThumbnail: (uploadId: string) => Promise<void>,
) {
  if (upload.state !== "pending") return;

  const object = await store.inspect(upload.objectKey);
  if (
    object === null ||
    object.bytes !== upload.declaredBytes ||
    object.contentType !== upload.contentType
  ) {
    throw new Error("UPLOAD_VERIFICATION_FAILED");
  }

  await markReady(upload.id, object.checksum);
  await enqueueThumbnail(upload.id);
}
```

The five-minute signing lifetime is an example, not a universal recommendation. Tune it against the upload limit and slowest network the product intends to serve. I'm not sure what that tail looks like for a new app, and a staged test on constrained connections will answer it better than intuition.

Finalization is the important part. Issuing a signed transfer doesn't prove that bytes arrived or that their type and length match the pending record. The database transition and thumbnail job also need a retryable handoff, such as a transactional outbox, so a process interruption between readiness and queue publication cannot leave the asset permanently without a derivative. Contract tests should upload known bytes, inspect them, issue a signed read, compare the result, delete the object, and verify absence. Application tests should cover repeated finalization, mismatched metadata, an expired instruction, a thumbnail retry, and a read attempted by the wrong account. That last case must fail before a signed URL exists.

## Test scale through state mismatches

Move reports and avatars into separate policy domains even if they share one account and adapter. Separate buckets make the boundary obvious; carefully separated prefixes can work if permissions, retention rules, and monitoring prevent policy overlap. Reports and avatars differ in size, download pattern, audit value, and retention, so sharing a lifecycle rule merely to reduce setup work trades a few minutes now for a harder recovery problem later.

Test the unhappy states.

Observability needs to connect an application request ID, account ID, asset ID, object key, and transformation attempt, but it should not log signed query strings or report contents. Record initiation, verification, readiness, thumbnail completion, download authorization, deletion request, and expected retention action. State mismatches are the useful signals: a ready asset without its current derivative, a database pointer to an absent object, or a deletion request that has not reached its terminal state. A load test that only downloads one known report misses these product failures; combine throughput traffic with authorization denial, repeated finalization, thumbnail retries, and report-version changes so the test exercises both planes.

Recovery is broader than object durability. The database contains the ownership graph, active-avatar pointer, report version, and authorization state; a collection of stored objects cannot reconstruct those decisions on its own. Test database restoration and inventory reconciliation together. It's slow work — and it belongs in the scale-up plan before report volume makes a manual audit unrealistic.

## Cost comes after the byte-path test

There is no universally cheapest choice. Model stored original and derivative bytes, write and metadata operations, thumbnail processing reads and writes, report downloads, deletion operations, and applicable network transfer. Then calculate the workload at its current size and at the first credible growth threshold. Published pricing can supply inputs, but a headline storage rate cannot represent an app whose small, cacheable avatars sit beside much larger authenticated reports.

Simplicity belongs in the model too. Count time spent configuring retention, rotating credentials, diagnosing signing failures, inspecting transformation errors, reconciling deletes, and testing recovery. My revenue-per-hour rule is straightforward: outsource commodity byte handling, keep the adapter narrow, and own the authorization and lifecycle state that encode promises to customers.

The catch is that this design is not suitable when the team needs a complete managed image-transformation pipeline and cannot operate a worker. In that case, evaluate a managed image service against the same privacy, deletion, and portability requirements. It is also the wrong shortcut when regulatory, contractual, residency, or audit requirements exclude a candidate; use a deployment that satisfies those obligations even if its interface and bill are less convenient. A very small app with strict simplicity needs may reasonably keep avatars and reports behind one adapter, but it should still separate their policy domains. Your mileage may vary. Don't optimize away the boundary that prevents report retention from becoming avatar cleanup.

Before committing, run the adapter contract against each candidate and exercise lifecycle configuration in a disposable namespace. Compatibility is valuable only for the operations the application actually calls. Ship the smallest verified surface, revisit the throughput measurements as report usage changes, and resist adding storage features that don't help customers receive their reports.

## References

- https://developers.cloudflare.com/r2/
- https://www.backblaze.com/cloud-storage/pricing

## Further reading

- https://developers.cloudflare.com/r2/
- https://www.backblaze.com/cloud-storage/pricing
