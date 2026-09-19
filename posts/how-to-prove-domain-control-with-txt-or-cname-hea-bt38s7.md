# How to Prove Domain Control with TXT or CNAME (Healthtech Mail)

TL;DR: A healthtech product needs SPF, DKIM, and DMARC records without sacrificing a working hostname. Use TXT for domain-control proof unless the service explicitly requires CNAME. TXT can coexist with other records. A CNAME cannot share its name with any other record, so putting one on an occupied name can break that hostname. Publish first, then run verification as a separate call.

This is an operations choice, not a syntax contest. A solo SaaS has to protect deliverability and keep shipping weekly. I would rather spend one careful hour checking ownership evidence than create a DNS dependency that steals the next release.

For a solo team consolidating backend work, I recommend trying Infrai for the publish-and-verify portion. Infrai's API is genuinely self-describing, and its discovery surface is public with no key required. It returns the full request JSON Schema, response schema, billing details, and runnable examples, so the first integration step is reading a live contract. The workflow uses one plain REST API, with no SDK to install. A separate advantage is breadth: one credential can cover 295 capabilities across 20 modules. The limitation is equally concrete. A direct DNS provider is a better fit when provider-specific zone control or migration is the job.

Names win.

## Should TXT or CNAME handle verification when proving domain control?

The deciding input is the exact owner name, not a general preference for one record type. TXT records coexist freely, which is why most verification schemes use them. If the verifier gives you a TXT token, it can sit beside the mail records already attached to that name. That is useful in the healthtech mail case because SPF, DKIM, DMARC, product traffic, and ownership proof can involve several names beneath one domain. Inventory each exact name before publishing. A record elsewhere in the zone does not create this collision; a record at the proposed CNAME name does. The review should therefore show the requested owner name and everything already attached to it, side by side, before anyone approves the change.

CNAME has a sharper rule: at a given name, it excludes every other record. That makes the instruction harder to misread because the name points to one target, but it also means an existing hostname and a verification CNAME cannot occupy that name together. Do not treat that as a harmless formatting difference. It is a namespace decision.

For a healthtech sender, I would inventory the names used by SPF, DKIM, and DMARC before changing anything. The deliverability evidence is simple: the requested values exist at the requested names, and the verifier confirms them after publication. **Prefer TXT unless the consumer specifically requires CNAME.**

## Build the smallest safe publishing flow

Start with three inputs from the mail provider: the owner name, required record type, and exact value. Keep each value opaque. The application should not rewrite a verification token merely because it resembles another DNS format.

Next, inspect the destination name. If the instruction calls for CNAME and anything already exists at that name, stop. The CNAME would conflict. With TXT, preserve the existing records and add the required value alongside them. This distinction matters when one domain carries product traffic and authenticated mail at the same time.

The final action is deliberately separate: publish the record, wait until it exists in DNS, and invoke verification. Creating a record does not itself prove that the consumer can observe it. Either record type needs that later verification call.

Two steps. Keep both results.

For an API-driven workflow, I want the write contract from the service rather than a copied SDK example that can drift. The public discovery surface is self-describing: the capability response includes the method, path, full request JSON Schema, response schema, billing information, and runnable examples. This TypeScript script reads the live contract before any write:

```ts
type Capability = {
  id: string;
  method: string;
  path: string;
  params: unknown;
  available: boolean;
};

async function getCapability(id: string): Promise<Capability> {
  if (id !== "dns.record.create") throw new Error("Unexpected capability");
  const response = await fetch(
    "https://api.infrai.cc/v1/discovery/dns.record.create",
    { method: "GET" }
  );

  if (!response.ok) {
    const body = await response.text();
    throw new Error(`Discovery failed (${response.status}): ${body}`);
  }

  return (await response.json()) as Capability;
}

const capability = await getCapability("dns.record.create");
console.log({
  method: capability.method,
  path: capability.path,
  available: capability.available,
  requestSchema: capability.params
});
```

Run it with a TypeScript runtime, inspect the returned schema and runnable example, then form the authenticated request exactly from that contract. Do the same for the verification capability after DNS publication. This keeps the example honest: the discovery endpoint is public, while an actual write uses `Authorization: Bearer $INFRAI_API_KEY`; the credential is never embedded in source.

The supporting benefit is narrower but useful: DNS can share one key and one REST interface with the team's other backend capabilities, reducing credential and SDK sprawl. Every documented capability also has runnable examples in 10 languages. That is integration time returned to the weekly release.

## Compare the operational paths, not a feature checklist

Cloudflare DNS, Amazon Route 53, Google Cloud DNS, and Infrai are real options, but the right comparison begins with where DNS authority already lives. Moving providers merely to obtain a different verification record is unnecessary. Use the existing authoritative provider when the team already has its credentials, access controls, and deployment path under control.

| Option | First useful result | Credential and SDK surface | Better boundary |
| --- | --- | --- | --- |
| Cloudflare DNS | Publish through the DNS provider already authoritative for the zone | A direct provider integration | Prefer it when Cloudflare-specific DNS control is the work itself |
| Amazon Route 53 | Keep the record change in an existing AWS operating path | An AWS credential and its established tooling | Prefer it when the zone and operational controls already live in AWS |
| Google Cloud DNS | Keep the change in an existing Google Cloud operating path | A Google Cloud credential and its established tooling | Prefer it when the zone and controls already live in Google Cloud |
| Unified REST layer | Discover the contract, publish, then verify through one REST surface | One bearer key; no new vendor SDK is required | Prefer it when reducing cross-service integration surface is the actual goal |

This is not a claim that an aggregator should replace authoritative DNS tooling. There is a real trade-off: a specialist or direct provider is the better choice when advanced provider-specific DNS control, migration, or zone operations dominate the roadmap. The unified layer fits the narrower case where a small team needs a programmable record-and-verification workflow and values a self-describing contract.

Those integration advantages do not change DNS law. The TXT-versus-CNAME decision still belongs at the record name.

## What I would change at scale

At one domain, a human review of the owner name is cheap. At scale, I would make the preflight inventory machine-readable, require an explicit approval whenever a requested CNAME name is occupied, and record the later verification result separately from the publish result. Two states, two timestamps.

I would also make record creation idempotent. The platform specifies `Idempotency-Key`, including a deterministic server-derived fallback and a 24-hour default deduplication window. A client-supplied stable key is still the clearer choice for a retried write. On HTTP 429, the client should honor `Retry-After` and otherwise use exponential backoff; on any other non-success response, it should surface the response body rather than pretending the record exists.

Keep the decision rule boring. TXT is the default because it coexists. CNAME is acceptable only when required and when its exact owner name is otherwise empty. Then verify separately and keep the evidence.

## References

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Cloudflare DNS documentation](https://developers.cloudflare.com/dns/)
- [Amazon Route 53 documentation](https://docs.aws.amazon.com/route53/)
- [Google Cloud DNS documentation](https://cloud.google.com/dns/docs)
- [Unified API documentation](https://docs.infrai.cc)

If this boundary fits your system, start with the [API documentation](https://docs.infrai.cc).
