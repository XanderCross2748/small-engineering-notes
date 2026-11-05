# Node.js Billing Evidence — Attribute Usage Snapshots and Invoices for Access Review

Metered billing needs historical ownership of usage; an access review needs current permissions. Store both views, linked by stable identifiers, so an approved charge survives later access changes.

| Choice | Evidence retained | Decision |
| --- | --- | --- |
| Accepted usage events, frozen billing snapshots, separate invoice records | Customer attribution at acceptance and the exact inputs approved for a charge | Default for this design |
| Period totals with a frozen membership export | Aggregate charges and an independently retained account mapping | Runner-up when individual work does not need investigation |
| Current credential ownership joined to historical totals | A present-day explanation of past work | Reject when ownership can change |

TL;DR: For a Node.js developer-tools SaaS, meter the contractual unit of completed work, bind each event to its customer when accepted, and store it in a durable billing ledger. Store approved usage snapshots separately from mutable projections, and link each invoice record to the exact snapshot revision it used. Give the access reviewer that chain alongside current permissions.

The recommendation favors attribution accuracy over a smaller data model. For a solo founder, the revenue-per-hour question is how much work a disputed charge will take to explain. Ship weekly, outsource the undifferentiated, and keep the rules that assign customer charges explicit enough to review.

## What usage should metered billing count and store for review?

Define the approval narrowly: this customer was charged for these accepted units, attributed through these identities, under this billing rule. Separately, these identities retain these permissions today. A reviewer should be able to approve one statement and reject the other.

Consider a hypothetical developer tool that bills for completed analysis jobs. Customer Cedar has a service identity with billing scope `scope_17`; its credential is rotated during the period. The illustrative workload contains 7 completed jobs and 2 failed attempts. Under an example contract that charges once per completed job, the billable quantity is 7. Neither HTTP attempts nor credential creations are the billing unit. A retried delivery of an existing completion contributes zero additional jobs.

Count completed work.

A completion that happens after credential rotation raises two separate questions: was the job authorized when accepted, and which customer accepted responsibility for it? In this design, the job carries its accepted billing scope through execution. The worker does not resolve the credential's current owner to decide who pays. Otherwise, an ordinary access change could rewrite the explanation of historical usage without changing a single completed job.

The review packet should identify the period, customer, service identity, accepted billing scope, snapshot revision, and invoice reference. Include the membership or authorization revision used at acceptance, plus a current access export for the reviewer’s permission decision. Keep credential identifiers opaque; the packet has no reason to contain a secret. The OWASP secrets-management guidance is the reference for managing the credentials themselves, including access controls, auditing, and rotation. The billing evidence design here is a proposed application boundary.

## Criterion one: attribution must survive access changes

Make a customer identifier part of the trusted acceptance record. Derive it from server-side authorization state, then persist that binding before scheduling work. Do not accept a request body's customer field as proof of ownership. For a shared service identity, require an explicit authorized billing scope instead of guessing from the identity’s display name.

The awkward case is reassignment. Suppose the same service identity is later allowed to act for customer Birch. Preserve the Cedar binding on already accepted jobs and record the new authorization revision for future jobs. Removing access should stop new acceptance according to the authorization policy; how already accepted work is canceled is a separate policy that must also define whether a completed unit remains billable. An access review needs those decisions stated, because a credential’s present permissions cannot answer them.

A concrete pitfall is using a credential fingerprint as the customer key. Rotation then creates a new apparent customer, or a later lookup silently attaches earlier jobs to a new account. Use the fingerprint only as an audit reference if needed. Keep customer identity, service identity, and credential identity separate.

That costs a few more fields and a historical mapping. It buys a review that doesn't require reconstructing last month’s permissions from memory.

## Criterion two: an approved quantity must be reproducible

Use a transactional store for accepted usage events and for the constraints that prevent duplicate billing. Keep a mutable period-total projection there if it helps the dashboard, but treat that projection as a convenience. The invoice should reference an approved snapshot revision with an explicit inclusion boundary, billing-rule revision, quantities, and links to its supporting events or a retained event manifest.

Storage has three jobs. The event ledger explains individual units. The snapshot preserves the approved aggregation. The invoice record preserves the issued document and its relationship to that aggregation. A relational database is a reasonable starting point for the records; object storage can hold larger manifests and rendered documents. These are architectural choices, not requirements to deploy three different services. For a one-person business, start with the fewest components that can preserve those distinctions.

Freeze approval.

Specify both the billing period and an ingestion cutoff. Otherwise, a delayed completion could change a recomputed total after approval. Define whether a late event belongs in a later revision or an adjustment, and retain that decision with the evidence. Correct attribution with a linked adjustment record; avoid silently editing the event that justified an issued charge. Retention periods should follow the business’s documented obligations and investigation needs, rather than an invented universal number.

The reviewer needs more than a total. A useful packet can explain why an event was excluded, whether it arrived after the cutoff, and which rule handled the correction. That extra context is where attribution accuracy earns its keep: it turns a disagreement into a specific decision someone can inspect.

## An acceptance binding carried into a completion event

This TypeScript example illustrates the boundary between a trusted job record and the billing ledger. All identifiers are illustrative. The `AcceptedJob` must come from durable server-side acceptance state; this function does not authenticate a request or verify a worker message. The ledger adapter must atomically enforce uniqueness for the event ID and the billable job identity within the customer and metric.

```ts
interface AcceptedJob {
  customerId: string;
  serviceIdentityId: string;
  billingScopeId: string;
  authorizationRevision: string;
  jobId: string;
}

interface UsageEvent extends AcceptedJob {
  eventId: string;
  metric: "completed_analysis_job";
  ruleRevision: string;
  quantity: 1;
  completedAt: string;
}

interface UsageLedger {
  appendOnce(event: UsageEvent): Promise<"inserted" | "duplicate">;
}

async function recordCompletedJob(
  ledger: UsageLedger,
  accepted: AcceptedJob,
  completion: { eventId: string; completedAt: string },
): Promise<"inserted" | "duplicate"> {
  if (!Number.isFinite(Date.parse(completion.completedAt))) {
    throw new Error("A valid completion timestamp is required");
  }

  return ledger.appendOnce({
    ...accepted,
    eventId: completion.eventId,
    completedAt: completion.completedAt,
    metric: "completed_analysis_job",
    ruleRevision: "completed-job-v1",
    quantity: 1,
  });
}
```

The important omission is a lookup of current credential ownership. Attribution already lives on the accepted job. The durable adapter should assign its own received timestamp for cutoff decisions and reject conflicting duplicates rather than treating a changed payload as an innocent retry. Timestamp parsing here is a basic validity check; the worker boundary still needs a defined timestamp format and authenticated completion evidence.

Exercise this boundary before shipping. Deliver the same completion twice; rotate the credential between acceptance and completion; change a scope’s membership; deliver an event after approval. Assert the expected customer and snapshot membership for each case. Test the uniqueness constraint with concurrent inserts against the real storage adapter, since an in-memory test cannot establish that property.

For deployment, version the event shape and billing rule together, and make old accepted jobs readable during the transition. Reconcile completed jobs against ledger events and snapshot quantities before issuing invoices. Track conflicting duplicates, unattributed completions, and late arrivals as exceptions someone must resolve. A failure to persist usage should leave a durable retry obligation; acknowledging it and hoping to reconstruct it later weakens the evidence chain.

Keep the weekly operating loop small: review exceptions, approve the frozen snapshot, issue the linked invoice, and retain the access decision. Automate the repeatable assembly of that packet. Spend founder attention on the exceptions that can change a customer’s charge.

## When the smaller model is the better choice

The runner-up is period totals plus a frozen membership export. Choose it when the contract bills only an aggregate quantity, customer ownership stays fixed throughout the period, and the reviewer explicitly accepts evidence without individual job attribution. Document that boundary before discarding event detail.

The trade-off is investigation depth. A limitation of period totals is that a disputed job will be harder to trace, even though the model saves implementation and retention work. If job-level investigation is part of the review requirement, the smaller model is not suitable.

A signable access review connects accepted responsibility, counted work, approved quantities, and issued charges while showing current permissions separately. Preserve that chain at the level the reviewer needs. That is the decision rule for what to meter and where to store the evidence.

## Sources

References: OWASP Secrets Management Cheat Sheet — credential access controls, auditing, and lifecycle management. The billing schemas, illustrative quantities, and review workflow above are proposed design choices.

- https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
