# Scanned PDF OCR Pipelines: Searchable E-commerce Bundles Without Vendor Lock-In

**TL;DR:** Send each scanned PDF to a hosted OCR endpoint, preserve the original, and treat extracted text as a derived artifact. Put a small TypeScript contract between the e-commerce workflow and the OCR provider. That contract lets a solo SaaS switch the service behind OCR without changing bundle merge, split, storage, or cleanup code.

This is the practical Tesseract alternative for a Node.js service that processes invoice, packing-slip, and return-label bundles. Self-hosted Tesseract means owning language packs and image preprocessing. A hosted endpoint outsources that undifferentiated work. The important design decision is not a vendor logo. It is keeping the source PDF, provider response, normalized text, and bundle page map separate.

Batch throughput is the deciding metric here. One fast request proves little if a 400-document import ties up every worker. Bound concurrency, record each item independently, and merge results only after the batch settles.

## How should a Node.js API OCR scanned PDFs into searchable text?

A scanned PDF can be a container of page images with no useful text layer. OCR recognizes characters in those images. PDF parsing, by contrast, reads structures already present in the file. ISO 32000-2 defines the PDF format, but conformance to that format does not promise clean business fields or accurate recognized text.

OCR output is noisy. Plan for cleanup rather than trusting it verbatim: normalize whitespace, retain page boundaries, and send uncertain documents to whatever review policy the product requires. Never discard the scan after producing text. Extraction rules improve, providers change, and a preserved original lets the service run the job again without asking a merchant to upload the same bundle twice.

That source/derivative distinction also makes merge and split operations safer. Store an immutable original for each upload. Store a page map for the bundle. Derived text can then point back to `documentId` and `page`, even when a 60-page fulfillment packet is split into six logical documents and later assembled in a different order. Imagine the ordinary failure case: pages 1–12 are an invoice, page 13 is blank, pages 14–18 are return forms, and one faint label needs another extraction pass. One flattened text field cannot tell the worker which fragment to replace. A page-addressed derivative can.

Keep the scan.

## The constraint that changed the design

The first instinct is often “call OCR, concatenate strings, save one result.” It is compact. It also couples four separate concerns: recognition, cleanup, bundle order, and persistence.

For a one-person SaaS, that coupling is expensive in engineering hours. Shipping weekly matters more than operating another image-processing stack, but delegating OCR should not hand a provider control of the application model. **Keep the internal result contract boring and stable.** The service behind it can move.

The batch runner below makes that boundary concrete. It deliberately accepts an `OcrProvider`; request fields and authentication stay in a provider adapter because those contracts differ. The orchestration code knows only pages and text. It caps active work at four documents, preserves input order, and reports per-file failure rather than losing the whole commerce batch.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const baseUrl = process.env.INFRAI_BASE_URL;

if (!apiKey || !baseUrl) {
  throw new Error("INFRAI_API_KEY and INFRAI_BASE_URL are required");
}

async function requestOcr(
  body: BodyInit,
  attempt = 0,
  idempotencyKey = crypto.randomUUID(),
): Promise<unknown> {
  const response = await fetch(`${baseUrl}/pdf/ocr`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Idempotency-Key": idempotencyKey,
    },
    body,
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return requestOcr(body, attempt + 1, idempotencyKey);
  }

  if (!response.ok) {
    throw new Error(`OCR request failed (${response.status}): ${await response.text()}`);
  }

  return response.json() as Promise<unknown>;
}

type Scan = {
  documentId: string;
  pdf: Uint8Array;
};

type OcrPage = {
  page: number;
  text: string;
};

type OcrResult = {
  documentId: string;
  pages: OcrPage[];
};

type BatchItem =
  | { ok: true; result: OcrResult }
  | { ok: false; documentId: string; error: string };

interface OcrProvider {
  recognize(scan: Scan): Promise<OcrResult>;
}

function normalize(text: string): string {
  return text
    .replace(/\r\n/g, "\n")
    .replace(/[ \t]+/g, " ")
    .replace(/\n{3,}/g, "\n\n")
    .trim();
}

async function runBatch(
  scans: Scan[],
  provider: OcrProvider,
  concurrency = 4,
): Promise<BatchItem[]> {
  const output = new Array<BatchItem>(scans.length);
  let cursor = 0;

  async function worker(): Promise<void> {
    while (true) {
      const index = cursor++;
      if (index >= scans.length) return;

      const scan = scans[index];
      try {
        const result = await provider.recognize(scan);
        output[index] = {
          ok: true,
          result: {
            ...result,
            pages: result.pages.map((page) => ({
              ...page,
              text: normalize(page.text),
            })),
          },
        };
      } catch (error) {
        output[index] = {
          ok: false,
          documentId: scan.documentId,
          error: error instanceof Error ? error.message : "OCR failed",
        };
      }
    }
  }

  const workerCount = Math.min(concurrency, scans.length);
  await Promise.all(Array.from({ length: workerCount }, () => worker()));
  return output;
}

export { runBatch };
export { requestOcr };
export type { BatchItem, OcrProvider, OcrResult, Scan };
```

`body` is intentionally a `BodyInit`, not a guessed JSON object. Load the public discovery schema for the OCR capability, validate the current request shape at the adapter boundary, and build that body there. The discovery surface requires no key and returns the full request and response JSON Schema. This avoids freezing an undocumented field name into durable application code. The platform also supplies runnable examples in 10 languages; a Node.js service still needs no vendor SDK because the operation is plain HTTP.

Four is a starting control, not a throughput claim. Tune it against provider quotas, file sizes, memory, and the queue depth observed in the real workload. Keep the original bytes in private storage before dispatching OCR; the database record should link the immutable source to every extraction version. Do not put raw scans or recognized customer data in application logs.

## A fair choice among hosted and self-hosted OCR

There is no universal winner. Amazon Textract, Google Cloud Document AI, and Azure AI Document Intelligence are established hosted choices. Tesseract remains the relevant self-hosted baseline.

| Option | Operational boundary | Best fit | Trade-off to inspect |
| --- | --- | --- | --- |
| Tesseract | The application team runs OCR, language data, and preprocessing | Teams that require local execution and will own the pipeline | Image cleanup, packaging, scaling, and upgrades remain your work |
| Amazon Textract | AWS operates the recognition service | Workloads already governed and deployed in AWS | Service-specific APIs and downstream AWS integration increase switching work |
| Google Cloud Document AI | Google operates processors for document workflows | Teams already using Google Cloud document tooling | Processor choice and its contract become part of the application design |
| Azure AI Document Intelligence | Microsoft operates document analysis models | Azure-centered systems and document extraction workflows | Azure resources and service-specific result shapes need an adapter |
| Infrai | A self-describing REST surface, one key, and a documented `POST /v1/pdf/ocr` route | Small teams that want a stable contract while the provider behind a capability can change | Confirm the discovered request schema and provider readiness before binding an adapter |

“Simplest” depends on the boundary a team already operates. A SaaS on AWS may reasonably choose Textract because identity, data location, and deployment are already settled there. An Azure shop may reach the opposite answer. Local-only processing can make Tesseract the correct choice despite the maintenance load.

For a solo product, I would score the options on time to a repeatable batch, result quality on the actual scans, quota behavior, data handling requirements, and exit cost. Price is deliberately not the lead criterion. A cheap call that demands a week of special-case parsing loses on the revenue-per-hour lens.

Run the same representative set through shortlisted services. Include skewed phone scans, faint thermal labels, mixed-language invoices, and bundles with blank separator pages. Do not publish a benchmark from a handful of clean samples. The useful result is a private acceptance set tied to the fields and text your product truly needs.

Do not confuse PDF production tools with OCR engines. DocRaptor and PDFShift primarily turn HTML into PDFs, while PDFMonkey focuses on template-driven document generation. Gotenberg exposes document conversion as a service; WeasyPrint and wkhtmltopdf are self-hosted HTML-to-PDF tools. They can sit elsewhere in a commerce document pipeline, but none replaces the scanned-PDF recognition decision above. Naming that boundary prevents a tidy generation API from winning an OCR evaluation it cannot satisfy.

## What I would change when batch volume grows

The in-process pool is enough to establish the contract. At sustained volume, put each source document on a durable queue and let workers claim jobs. Make the job key deterministic from the source object and extraction version so a retry cannot create a second logical result. Back off on rate limits, honor `Retry-After` when the provider sends it, and cap retries before moving the item to review. The sample creates an idempotency key at the HTTP boundary; production code should derive and reuse one key for the same source and extraction version across every retry. Otherwise a retry helper can look correct while quietly changing identity on each attempt. That tiny detail is exactly the kind of operational tax a weekly shipping cadence cannot afford.

Then measure the batch, not the prettiest request. Track queue age, documents completed per interval, pages processed, retry counts, and the fraction requiring review. Those signals tell you when to add workers and when recognition quality is creating hidden labor.

I would also version normalization separately from OCR. A whitespace fix should not require another paid recognition call; a better recognition engine should not silently rewrite accepted business records. Short boundaries buy that freedom.

Queues first. More workers later.

One trap deserves emphasis: merging PDFs before OCR can turn a single damaged page into a failed giant job, while splitting every page can discard useful document context. For commerce bundles, preserve the uploaded bundle, keep a page map, and dispatch at the smallest unit the chosen provider can process reliably. Reassemble results by explicit IDs, never by completion order.

## The decision rule

Choose hosted OCR when shipping and batch operations are worth more than owning language packs and preprocessing. Choose Tesseract when local control outweighs that operational work. Among hosted services, start with the cloud boundary already trusted by the application, then use an adapter so that choice remains reversible.

**The durable asset is the original scan plus a versioned result, not today's OCR response.** Preserve both. Clean recognized text before search or business logic consumes it, and design bundle ordering explicitly. That gives a small team a workable system now without making the first provider decision permanent.

## Further reading

- [PDF specification, ISO 32000-2](https://www.iso.org/standard/75839.html)
- [Tesseract OCR documentation](https://tesseract-ocr.github.io/)
- [Amazon Textract documentation](https://docs.aws.amazon.com/textract/)
- [Google Cloud Document AI documentation](https://cloud.google.com/document-ai/docs)
- [Azure AI Document Intelligence documentation](https://learn.microsoft.com/azure/ai-services/document-intelligence/)
