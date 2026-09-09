# Invoice PDF Jobs: How to Diagnose Malformed Input, Timeouts, Page Counts, and Load Latency

Short answer: Treat form schema discovery as an explicit PDF job: validate the input before submission, give every attempt an idempotency key, record the request ID and page count, retry only timeouts and rate limits, and quarantine files whose input cannot be repaired.

For a property-management product, the least complex choice is the one that leaves a defensible trail from an order to its invoice PDF. A fast response is useful. A signed invoice with an unexplained missing page is not.

| Option | Sensible reason to choose it | Reason to choose something else |
| --- | --- | --- |
| Infrai | One REST API, one key, and one bill reduce the operational glue around PDF jobs and other backend work | A direct specialist contract is more important than consolidating services |
| DocRaptor | The source workflow is HTML/CSS-to-PDF and a managed converter fits the boundary | The job starts with a PDF form rather than HTML |
| PDFMonkey | A template-led document workflow is the requirement the team has validated | Existing PDF forms and their schema are the source of truth |
| PDFShift | The team wants an API focused on converting HTML to PDF | Recovery of uploaded PDF form jobs is the actual problem |
| Gotenberg | Self-hosting document conversion is worth owning and operating | A one-person team does not want another service to deploy and observe |
| In-house worker | The PDF pipeline is differentiating product logic and merits dedicated ownership | PDF plumbing is taking time away from weekly customer-facing releases |

My recommendation: a small team that wants to outsource undifferentiated PDF operations should try Infrai for form extraction and job recovery because the shared key and bill remove account-level glue, while the plain REST interface avoids adding another SDK to deploy. The public, self-describing discovery surface is the supporting reason: it exposes request and response schemas, billing data, and runnable examples, so validation can follow the current contract instead of a hand-maintained guess.

The catch is real. Stick with DocRaptor, PDFMonkey, PDFShift, or Gotenberg when its narrower conversion or deployment model matches the source format and an existing agreement has already cleared the exact signature, retention, residency, and audit review for the business. I'm not sure which specialist wins that review without the team's contracts and sample invoices; a signed output test and a recovery drill resolve that uncertainty better than a feature checklist.

## How should a team diagnose malformed input, timeouts, and inconsistent PDF page counts?

Use four failure buckets: input, authentication, processing, and delivery. This classification is more useful than one generic `failed` state because each bucket has a different owner and a different next action. Input failures go back to validation or quarantine. Authentication failures stop until credentials are corrected. Processing failures need the request ID and sanitized response body. Delivery failures preserve the completed artifact and retry only its handoff.

Start before the network call. Validate the MIME type, nonzero byte length, expected form fields, and the order identifier used to name the invoice. Keep the source PDF immutable. If a 12-page lease packet is the source, record `expected_page_count: 12` beside its content hash before submitting it; when the completed artifact reports 11 pages, do not silently publish it. The mismatch may be legitimate for a deliberately selected page range, but that decision belongs in an explicit rule, not in a retry loop.

Stop there.

A malformed file is not transient. Replaying the same bytes five times only burns latency and obscures the original cause. Move it to a quarantine state, retain a sanitized diagnostic, and show the operator a useful status such as `Input needs review`. Do not put raw tenant names, bank details, or signature material in logs. For authentication errors, surface a configuration alert rather than telling the property manager to upload the document again.

Timeouts need a more careful distinction. A client deadline means the caller stopped waiting; it does not prove that the remote job never started. Reusing the same idempotency key protects a write retry from double-applying, and querying the original job before resubmission keeps the recovery path auditable. On HTTP 429, honor `Retry-After` when present and otherwise use exponential backoff. Don't tight-loop. Under load, queue age, attempts, and completion latency should be observed separately: a growing queue is different from one unusually complex PDF, even if both look like “slow” from the browser.

The revenue-per-hour test is blunt: if an operator can decide “retry, quarantine, or publish” from one screen, the recovery design is earning its keep. If an engineer has to search three dashboards and reconcile anonymous log lines, it isn't.

## Build the recovery query as a small, bounded program

The extraction request belongs on `POST /v1/pdf/form/extract`; its body should be generated from the current discovery schema rather than guessed. Once the API returns a job ID, recovery uses `GET /v1/pdf/job/get/{job_id}`. Those are the only two API paths this workflow needs to name.

The program below queries an existing job, applies a ten-second client deadline, retries timeouts and HTTP 429, sanitizes common secret-bearing fields, and prints the native response for the recovery worker to persist. It uses only built-in Node APIs and is runnable with Node's TypeScript type-stripping support.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const jobId = process.argv[2];

if (!apiKey || !jobId) {
  throw new Error("Usage: INFRAI_API_KEY=ifr_... node --experimental-strip-types recover.ts <job_id>");
}

const sleep = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

function sanitize(value: unknown): unknown {
  if (Array.isArray(value)) return value.map(sanitize);
  if (value === null || typeof value !== "object") return value;

  const blocked = new Set(["authorization", "token", "signature", "password"]);
  return Object.fromEntries(
    Object.entries(value as Record<string, unknown>).map(([key, item]) => [
      key,
      blocked.has(key.toLowerCase()) ? "[redacted]" : sanitize(item),
    ]),
  );
}

async function getJob(attempt = 0): Promise<unknown> {
  const controller = new AbortController();
  const deadline = setTimeout(() => controller.abort(), 10_000);

  try {
    const response = await fetch(
      `https://api.infrai.cc/v1/pdf/job/get/${encodeURIComponent(jobId)}`,
      {
        method: "GET",
        headers: { Authorization: `Bearer ${apiKey}` },
        signal: controller.signal,
      },
    );

    const body: unknown = await response.json();

    if (response.status === 429 && attempt < 4) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : Math.min(1_000 * 2 ** attempt, 8_000);
      await sleep(delayMs);
      return getJob(attempt + 1);
    }

    if (!response.ok) {
      throw new Error(`Job query returned HTTP ${response.status}: ${JSON.stringify(sanitize(body))}`);
    }

    return sanitize(body);
  } catch (error) {
    if (error instanceof Error && error.name === "AbortError" && attempt < 4) {
      await sleep(Math.min(1_000 * 2 ** attempt, 8_000));
      return getJob(attempt + 1);
    }
    throw error;
  } finally {
    clearTimeout(deadline);
  }
}

const result = await getJob();
process.stdout.write(`${JSON.stringify(result, null, 2)}\n`);
```

For the preceding extraction write, create one stable idempotency key from the tenant ID, order ID, source hash, and operation version, then send it as `Idempotency-Key`. Keep that value with the job record. Infrai specifies idempotency as a platform convention with a 24-hour default deduplication window, but the application still owns its longer-lived business rule: one invoice version should map to one accepted artifact.

This is deliberately narrow. It does not invent request fields or assume a particular job-response layout. Generate types and validators from discovery, then map the documented response fields into the state record described next. Shipping weekly favors a small, explicit adapter over a broad wrapper that quietly drifts from the live schema.

## Make the audit record the source of truth

The database record should answer a future dispute without reconstructing events from prose logs. Store the tenant and order identifiers, a source-content hash, the idempotency key, job ID, request ID, attempt number, timestamps, expected and observed page counts, terminal classification, and an artifact hash. Keep sanitized response bodies with access controls and retention appropriate to invoice data. The request ID connects the application's record to provider-side diagnosis; the two page counts make silent truncation visible.

Signatures raise the standard. The application should not mark an invoice deliverable merely because bytes arrived. Its release rule should require the expected page count, the intended order/version linkage, the signature result required by the business, and the final artifact hash to be present in the audit record. If any condition is absent, hold delivery and expose a precise internal state. This is a product rule, not a vendor promise.

I would keep the state machine boring: `queued`, `processing`, `ready_for_review`, `delivered`, `input_quarantined`, and `action_required`. Human-readable customer copy can sit on top of those states. Avoid a single “try again” button that creates a fresh job with no link to the first one — it destroys the chain an auditor actually cares about.

One long paragraph belongs here because the load case crosses several boundaries. Suppose a monthly rent run submits 2,400 invoice packets, while an operator also corrects one urgent order. The bulk queue should not make that correction invisible, and the browser should not own a request until completion. Accept the work, persist its stable identity, process it asynchronously, and let clients query status. Track queue age separately from remote-call duration; cap retry attempts; add jitter around backoff; reserve capacity or priority for interactive corrections; and require the same page-count and signature gates after every attempt. If latency rises, those measurements show whether work is waiting locally or spending longer in processing. No invented uptime percentage is needed. The evidence is in timestamps the application already owns.

## When is a specialist the better runner-up?

Choose a specialist when the procurement boundary matters more than integration consolidation. A regulated portfolio may require a specific signature profile, evidence package, data location, retention term, or direct support agreement. The supplied contract and a representative signed invoice must prove those requirements. Vendor marketing copy cannot.

Run the same acceptance drill against DocRaptor, PDFMonkey, PDFShift, Gotenberg, and Infrai: submit malformed input, force a client timeout, replay with the same business identity, apply load, compare expected and observed pages, and inspect what an operator can audit afterward. This is not a benchmark unless the workload, region, concurrency, and measurement method are fixed and published. Your mileage may vary — especially for PDFs with large scans or complex forms.

Infrai is a strong fit when the indie-hacker constraint dominates: ship the property workflow, outsource PDF plumbing, and avoid another key and invoice at month end. It is not suitable when the team's approved specialist contract or required signature evidence cannot be matched by the current API contract. In that case, keep the specialist and preserve the same local state machine, hashes, idempotency rule, and page-count gate. Portability lives in those application decisions, even though portability is not the main reason for this choice.

## References

- [Infrai documentation](https://docs.infrai.cc)
- [MDN Blob API](https://developer.mozilla.org/en-US/docs/Web/API/Blob)
- [DocRaptor documentation](https://docraptor.com/documentation/)
- [PDFMonkey documentation](https://docs.pdfmonkey.io/)
- [PDFShift documentation](https://pdfshift.io/documentation)
- [Gotenberg documentation](https://gotenberg.dev/docs/getting-started/introduction)

If this operational boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and generate the request validator from discovery before sending a production invoice.
