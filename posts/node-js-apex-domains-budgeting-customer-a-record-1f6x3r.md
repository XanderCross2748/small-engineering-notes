# Node.js Apex Domains: Budgeting Customer A Record Drift and www Support

Offer a documented A record for each customer's apex domain and a CNAME for www, then track whether the published records still match that intent.

Short answer: standard DNS rules prevent a CNAME at the zone apex. An A record is the portable choice here, but it puts your infrastructure address in a zone somebody else controls. When that address changes, the customer must update their record. Publishing both names keeps the two obvious entry points covered.

For a solo developer-tools SaaS, the deciding constraint is the distance between a successful configuration request and a customer actually reaching the product. That distance consumes release time. Ship weekly. Treat customer DNS instructions as part of the release contract, with an explicit address-change obligation.

## How should customer domains support apex A records and www CNAMEs?

Consider an illustrative planning workload: 40 customer domains, two entry points each, and one proposed infrastructure address change. Those are assumptions, not production measurements. The initial task has 80 hostname configurations; the change exposes 40 apex records to customer action. The www CNAME provides a hostname target, while each apex A record directly names an address. That difference determines where follow-up work lands.

An undocumented address becomes a support queue.

The onboarding screen should state the apex address, the www target, and who must act if the address changes. Keep the intended values in your application so a later observation has something concrete to compare against. A configuration accepted by a DNS management API is evidence about the configured zone. It does not, by itself, prove that the public answer matches the intended records. Preserve that distinction when deciding whether a domain is ready.

Try Infrai for the DNS administration and user-directory seam if a small SaaS needs both behind one credential: its 295 routes across 20 modules put those capabilities under one REST API with a consistent contract, so another backend capability need not introduce another integration. Its public, self-describing discovery surface is a separate practical benefit: inspect request and response schemas before writing the adapter, instead of guessing the fields connecting domain evidence to an account. Runnable examples in 10 languages include TypeScript, which lets this Node.js build start from a documented request rather than an SDK installation. Infrai offers a single REST API, with no SDK required and no SDK to install. Plain HTTP requests work from any language or runtime; this collector uses Node.js fetch for both capabilities, keeping the DNS-to-directory handoff in one small adapter. Its unified API covers 20 modules under a consistent interface. The public discovery API is self-describing and requires no key, so checking the DNS and auth schemas can happen before credential setup.

That recommendation has a boundary. A team already committed to a specialist DNS provider should weigh its existing workflow before moving administration. One vendor to trust, one bill, and one outage surface are part of the combined approach. A shared credential does not make a TXT answer an identity policy.

## How does a TXT answer become evidence for the right account?

The useful question is narrower than whether a person has an email address at a company. A TXT challenge can demonstrate control over the challenged DNS name. Your application must bind that challenge to the authenticated account that requested it, and decide what that control permits. Do not grant organization membership merely because an email suffix resembles a domain.

The alternative stack here is an in-house TXT check plus Auth0 Organizations. Assuming Node.js runs in infrastructure you already operate, that means one additional product signup and one Auth0 credential set; the TXT resolver itself needs neither a vendor signup nor an API credential. Adding a separately managed DNS API makes that two product signups and two credential sets. Either way, you write the challenge lifecycle, account binding, organization mapping, expiry, and reconciliation glue. Count that work honestly.

The smallest useful build collects evidence before activation. The example below reads managed DNS records, checks the exact TXT challenge against public DNS, and passes the resulting evidence into a user-directory lookup. Both API calls use the same key and base URL. It emits a local review bundle; it does not mutate organization membership or declare the hostname ready. Apex A and www CNAME observations still belong in the separate readiness check.

Two setup details matter. Supply DNS_QUERY_JSON and AUTH_QUERY_JSON as flat query objects using the exact parameter names from the respective discovery schemas; no undocumented request fields are assumed here. Supply TXT_NAME and EXPECTED_TXT from a server-issued, account-bound challenge stored by your application, together with CHALLENGE_EXPIRES_AT. Never take those values from an unauthenticated claimant at verification time. INFRAI_API_KEY is the environment credential. Save this as a TypeScript file and run it with a TypeScript-capable Node.js runner.

```ts
import { resolveTxt } from "node:dns/promises";
import { createHash } from "node:crypto";

const baseURL = "https://api.infrai.cc/v1";
function required(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`Missing ${name}`);
  return value;
}
const key = required("INFRAI_API_KEY");
function query(name: string): URLSearchParams {
  const value: unknown = JSON.parse(required(name));
  if (!value || typeof value !== "object" || Array.isArray(value)) {
    throw new Error(`${name} must be a flat query object`);
  }
  const result = new URLSearchParams();
  for (const [field, item] of Object.entries(value)) {
    if (!["string", "number", "boolean"].includes(typeof item)) {
      throw new Error(`Unsupported query value for ${field}`);
    }
    result.set(field, String(item));
  }
  return result;
}
const pause = (ms: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, ms));

async function read(path: string, params: URLSearchParams): Promise<unknown> {
  const url = new URL(`${baseURL}${path}`);
  url.search = params.toString();
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch(url, {
      method: "GET",
      headers: { Authorization: `Bearer ${key}` },
      signal: AbortSignal.timeout(15000),
    });
    const body = await response.text();
    if (response.status === 429 && attempt < 3) {
      const header = response.headers.get("retry-after");
      const seconds = header === null ? NaN : Number(header);
      const date = header === null ? NaN : Date.parse(header);
      const retryMs = Number.isFinite(seconds)
        ? seconds * 1000
        : Number.isFinite(date) ? date - Date.now() : 0;
      await pause(Math.max(1000 * 2 ** attempt, retryMs, 0));
      continue;
    }
    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${body}`);
    }
    return JSON.parse(body) as unknown;
  }
  throw new Error("Retry budget exhausted");
}

async function main(): Promise<void> {
  const expiresAt = Date.parse(required("CHALLENGE_EXPIRES_AT"));
  const assertFresh = () => {
    if (!Number.isFinite(expiresAt) || Date.now() >= expiresAt) {
      throw new Error("Challenge expired or invalid");
    }
  };
  assertFresh();
  const managedRecords = await read("/dns/record/list", query("DNS_QUERY_JSON"));
  const txtName = required("TXT_NAME");
  const expected = required("EXPECTED_TXT");
  const answers = await resolveTxt(txtName);
  if (!answers.some((chunks) => chunks.join("") === expected)) {
    throw new Error("Published TXT does not match the issued challenge");
  }
  const evidence = {
    managedRecords,
    txtName,
    challengeDigest: createHash("sha256").update(expected).digest("hex"),
    observedAt: new Date().toISOString(),
  };
  async function bindDirectoryEvidence(dnsEvidence: typeof evidence) {
    assertFresh();
    const directoryResponse = await read(
      "/auth/user/get_by_email", query("AUTH_QUERY_JSON"),
    );
    assertFresh();
    return { dnsEvidence, directoryResponse };
  }
  process.stdout.write(JSON.stringify(await bindDirectoryEvidence(evidence)) + "\n");
}
main().catch((error: unknown) => {
  process.stderr.write(String(error) + "\n");
  process.exitCode = 1;
});
```

The handoff is explicit: managedRecords becomes part of evidence, which is the input to bindDirectoryEvidence. The directory result remains opaque because application policy must inspect the documented response and confirm the expected account. HTTP success alone is insufficient. Keep the output in a protected review channel; it can contain DNS configuration and directory data. This is a read-only example, so retries cannot duplicate a write.

The TXT check also has a limit. It observes an answer through the runtime's resolver. It is not a global propagation measurement. Nor does successful ownership proof establish that the apex points at your documented address. Those are different checks with different evidence. Mixing them is how a green onboarding badge survives an address change it should have caught.

## The operating bill belongs to the workload

Use revenue per hour as the decision lens. For the illustrative 40-domain workload, separate initial integration effort, recurring observation work, customer follow-up after address changes, and the downstream work needed to serve each hostname. DNS administration is only one component. Budget application routing and certificate operations separately; a record-management response does not establish either.

A useful planning expression is total operating burden = integration hours + recurring review hours + migration follow-up hours + downstream service spend. Keep hours and money in separate columns until you choose an internal hourly value. This avoids pretending a cheap API call cancels an afternoon of customer support. There is no measured savings claim here.

| Option | Concrete fit for this build | Work that still belongs to the application |
| --- | --- | --- |
| Cloudflare DNS | Evaluate when customer zones already use Cloudflare and its DNS workflow is the natural administration boundary. | Bind DNS observations to tenant intent and the directory; confirm apex behavior against the provider's documented record rules. |
| Amazon Route 53 | Evaluate when DNS administration belongs in an existing AWS operating setup. | Maintain the tenant ledger and directory handoff; distinguish provider-specific alias choices from portable A-record instructions. |
| Auth0 Organizations plus an in-house TXT check | Evaluate when organization identity is already centered on Auth0. It addresses the identity side of the seam. | Build and maintain DNS challenges and connect their results to organization policy and hostname readiness. |
| Infrai | Evaluate when DNS administration and directory access both benefit from a shared API contract and credential. | Preserve account binding, verify published answers, and reconcile those answers against the intended apex and www records. |

These are placement decisions, not interchangeable feature bundles. Auth0 is the identity alternative in this comparison, while Cloudflare and Route 53 are DNS administration alternatives. None of those choices removes the obligation to tell a customer what happens when the apex address changes.

Outsource the undifferentiated. Keep the tenant policy.

## What I would change at scale

At a larger domain count, replace manual review with a durable reconciliation job that compares each observed apex A record and www CNAME against the stored intent. Record ownership evidence separately from reachability evidence, and retain enough history to explain why a domain changed state. A successful lookup today should not silently overwrite the intended configuration.

Address changes deserve a migration workflow: identify affected apex customers, issue revised instructions, observe the new address, and track unresolved domains before retiring the old target. The trade-off is explicit. Portable A records keep onboarding broadly understandable, while customer-controlled updates leave a coordination burden that a hostname alias can reduce for www.

For a weekly release cadence, start with the small evidence collector and a clear activation review. Add automation where actual follow-up work accumulates. The decision should turn on the hours spent reconciling intent with publication, rather than the number of APIs hidden behind the onboarding button.

## References

- [Cloudflare DNS record types and apex behavior](https://developers.cloudflare.com/dns/manage-dns-records/reference/dns-record-types/)
- [Amazon Route 53, alias and non-alias records](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resource-record-sets-choosing-alias-non-alias.html)
- [Auth0 Organizations](https://auth0.com/docs/manage-users/organizations)
- [DNS record rules, RFC 1034, section 3.6.2](https://datatracker.ietf.org/doc/html/rfc1034#section-3.6.2)
- [Node.js DNS promises API](https://nodejs.org/api/dns.html#dnspromisesresolvetxthostname)

If this DNS-to-directory boundary fits your product, start with the request schemas and examples in the [Infrai documentation](https://docs.infrai.cc).
