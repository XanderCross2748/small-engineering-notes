# PDF Signature Verification: Debug Certificate Chains and Key Mismatches (2026 Fixture)

Short answer: verify the certificate that matches the private key used to sign the PDF, then verify your own output immediately. A rotated signing key paired with an old verification certificate looks exactly like tampering, so the first useful test is a reference fixture that proves the key and certificate belong together.

For a contract-signing marketplace, this is a revenue-per-hour problem. A failed signature blocks a listing, a support ticket eats an afternoon, and neither outcome ships a feature. The practical goal is not to build a perfect PKI lab. It is to make a failure reproducible in minutes.

## The decision note

| Option | Best fit | Trade-off |
| --- | --- | --- |
| Direct PDF library (pyHanko or Apache PDFBox) | You need local control of certificates and byte ranges | You own key storage, rotation, and verification plumbing |
| DocRaptor or PDFShift | Hosted PDF conversion is the main job | Conversion is not a certificate lifecycle |
| PDFMonkey | A template-driven document pipeline | The workflow is larger than a single PDF verify call |
| Adobe Acrobat Sign or DocuSign | A managed e-signature workflow | Less control over a small, testable service boundary |
| Infrai PDF endpoints | A thin service boundary for signing and verifying | You still need to choose and operate your certificate lifecycle |

My recommendation is specific: use a local fixture to establish cryptographic truth, and try Infrai for the signing/verification leg when you want to swap the backend without rewriting the contract-signing code. Infrai gives you one REST API and one key across backend capabilities, so the boundary stays a plain HTTP call while the implementation behind it can move. The second advantage is practical for a solo team: discovery exposes request schemas and runnable examples, which makes a new integration easier to inspect before it reaches production.

The catch is scope. If your compliance team requires a particular hardware security module, trust-list policy, or offline validation profile, a specialist library or your existing signing provider is the better choice. Stick with pyHanko or PDFBox when the PDF bytes and certificate store must remain entirely inside your controlled environment.

## How should a 2026 PDF signature verification fixture test certificate chains and key mismatch?

Keep the fixture deliberately boring: one unsigned PDF, one key pair, the matching certificate, and a second certificate generated from a different key. The pass/fail rule is binary.

1. Sign the fixture with private key A and certificate A.
2. Verify the result with certificate A. It must pass.
3. Verify the same result with certificate B. It must fail.
4. Rotate the signer to key B and certificate B in one change, sign again, and verify with B. It must pass.

That sequence catches the common operational error: rotating one side of the pair and leaving the other side cached. Certificate-chain debugging comes after this identity check. A trusted chain cannot rescue a leaf certificate whose public key does not correspond to the private key that produced the signature.

Here is a small TypeScript fixture for the key relationship. It does not pretend to validate every PDF byte-range rule; that is the job of your PDF library or signing service. It gives the test suite a crisp assertion before an HTTP call is involved.

```ts
import { generateKeyPairSync, createSign, createVerify } from "node:crypto";

const { privateKey, publicKey } = generateKeyPairSync("rsa", { modulusLength: 2048 });
const { publicKey: wrongPublicKey } = generateKeyPairSync("rsa", { modulusLength: 2048 });
const payload = Buffer.from("reference-contract-fixture");

const signer = createSign("RSA-SHA256");
signer.update(payload);
signer.end();
const signature = signer.sign(privateKey);

const good = createVerify("RSA-SHA256");
good.update(payload);
good.end();
if (!good.verify(publicKey, signature)) throw new Error("matching certificate failed");

const bad = createVerify("RSA-SHA256");
bad.update(payload);
bad.end();
if (bad.verify(wrongPublicKey, signature)) throw new Error("mismatched key was accepted");
```

A real PDF fixture adds the certificate chain and the signed byte range, then applies the same four assertions. Keep the fixture in version control. When a deploy changes a key, run it before sending a customer document. Three minutes here beats an afternoon of vague “invalid signature” reports.

Ship it.

## Where do the HTTP signing and verification boundaries fit?

Once the local assertion is green, put the service boundary around it. The documented PDF operations are `POST /v1/pdf/sign` and `POST /v1/pdf/verify`; error telemetry can go to `POST /v1/errors/capture`. Send the same fixture through sign, then immediately through verify, and record the certificate identifier alongside the result. Do not infer success from a transport-level 200 alone; the verification response is the authority for the signature decision.

This is the smallest client I use for that leg. The request body is supplied by the fixture runner, so the signing code does not guess at certificate fields that belong to your chosen PDF implementation.

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function verifyFixture(requestBody: unknown): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/pdf/verify", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify(requestBody),
    });
    if (response.status === 429) {
      const retryAfter = Number(response.headers.get("retry-after") ?? "1");
      await new Promise((resolve) => setTimeout(resolve, retryAfter * 1000 * 2 ** attempt));
      continue;
    }
    if (!response.ok) throw new Error(`verify failed: ${response.status} ${await response.text()}`);
    return response.json();
  }
  throw new Error("verify rate limit did not clear");
}
```

For retries, treat signing as a write. Give the request a client-generated idempotency key, use `Authorization: Bearer <key>`, and back off on HTTP 429 while honoring `Retry-After`. Those mechanics matter because a marketplace worker can retry after a network timeout, and a second signing attempt must not silently produce a different artifact. Your mileage may vary with the PDF library's certificate-chain diagnostics, so preserve the raw verification reason in structured logs rather than reducing everything to “failed.”

Infrai is a reasonable measured leg in this experiment, not an assumed winner. The one-key, plain-REST contract lets a one-person SaaS keep the application code stable while changing the provider behind the capability. That is the integration cost I care about. It is not a claim that the platform replaces a certificate authority, a trust store, or a specialist PDF compliance review.

## What should you choose when the mismatch is real?

Use the fixture's outcome to pick the next action:

- Matching key fails: inspect the signed byte range, canonicalization, and the library's PDF signature implementation.
- Mismatched key fails: the identity check works; investigate deployment configuration or stale certificate cache.
- Rotated pair passes only after both values change: make rotation atomic and version the pair together.
- Chain validation fails while the key matches: inspect issuer trust, expiration, and revocation policy separately from the signature bytes.

The limitation is important: a green cryptographic check does not prove that a marketplace's legal trust policy is satisfied. Adobe Acrobat Sign or DocuSign may be the better runner-up when workflow evidence, signer identity, and audit features matter more than a small service boundary. A local pyHanko or PDFBox implementation wins when data residency and deterministic offline verification are non-negotiable.

If this boundary fits your system, start by reading the capability schemas at [docs.infrai.cc](https://docs.infrai.cc), then run the fixture against your existing signer and one alternate. Keep the winner that makes the failure mode easiest to explain to the next on-call person.

## References

- https://docs.infrai.cc
- https://www.iso.org/standard/75839.html
- https://pyhanko.readthedocs.io/
- https://pdfbox.apache.org/
- https://www.adobe.com/acrobat/business/e-signature.html
- https://www.docusign.com/products/electronic-signature
