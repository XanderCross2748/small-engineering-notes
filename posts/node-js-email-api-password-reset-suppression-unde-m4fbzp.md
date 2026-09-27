# Node.js Email API: Password Reset Suppression Under Low Marketplace Volume

Short answer: choose a simple email API by running one seller-order notification through controlled failures, then buy the option whose evidence lets the application decide what to do next. For a low-volume SaaS, a cheap accepted request is useless if the operator cannot distinguish queued, rejected, suppressed, and delivered mail. The same test protects password reset email, where a blind retry can create confusing overlapping messages.

The concrete constraint is a marketplace order that commits before its seller alert is sent. Revenue depends on the order record, not the email, so notification transport must never become the system of record. I would outsource transport, keep a very small Node.js attempt journal, and ship that boundary weekly with the application. It is undifferentiated infrastructure, but the correlation data is mine to operate.

## What Should a Low-Volume Email API Prove for Password Reset Emails?

A feature grid hides the moment that matters: the order exists, the worker stops, and nobody yet knows whether an external system accepted the alert. Starting with that gap turns a broad vendor search into an observable contract. Can an attempt be correlated? Can a repeated event be absorbed? Can a permanently bad address stop later sends? Can support see uncertainty without opening a transport dashboard?

Run the exercise with synthetic recipients and non-production orders. Create one order event and stop the worker before transport submission. On restart, the pending record should still be claimable; nothing outside the application has seen it, so there is no reason to invent a successful send. The second failure window is harder. Stop the worker after submission but before the receipt is stored. On restart, the application knows that an attempt began but cannot honestly say whether the transport accepted it. The adapter's documented idempotency behavior now matters, as does the journal's ability to retain uncertainty instead of converting it into success. Finally, replay one verified delivery event twice, then deliver an older event after a newer terminal one. The desired result is not “one HTTP request.” It is one business event, an explainable sequence of attempts, and no delivery callback that changes the order itself. These 2 failure windows tell me more than a long feature matrix because they expose who owns recovery at the exact boundaries where a solo operator loses context.

Break it on purpose.

This also exposes an important password-reset distinction. A reset secret should not be an order identifier, delivery key, log field, or template version. Delivery evidence may need to survive long enough for support and operations; a credential should follow the application's much narrower security lifetime. Keep those data classes apart.

Use 3 states for the first decision: pending, submitted, and terminal. Preserve the external event separately rather than pretending every transport uses the same vocabulary. “Submitted” means the transport accepted responsibility for processing. It does not mean a person saw the message.

That distinction pays rent.

## The smallest implementation I would ship

The first implementation is an outbox-shaped record written with the order transaction, plus a worker that claims due records. That closes the most expensive ambiguity: an order can no longer commit with no durable notification intent. No second queueing platform is required at this volume. Fewer moving parts leave more revenue-producing hours for product work.

```ts
type NoticeKind = "seller.new-order" | "account.password-reset";
type AttemptState = "pending" | "submitted" | "terminal";

type Notice = {
  eventId: string;
  kind: NoticeKind;
  recipient: string;
  templateVersion: string;
  fields: Record<string, string>;
};

type Attempt = {
  attemptId: string;
  eventId: string;
  state: AttemptState;
  externalId?: string;
};

interface NoticeStore {
  claimDue(): Promise<{ notice: Notice; attempt: Attempt } | null>;
  isSuppressed(recipient: string): Promise<boolean>;
  markSuppressed(attemptId: string): Promise<void>;
  markSubmitted(attemptId: string, externalId: string): Promise<void>;
  release(attemptId: string): Promise<void>;
}

interface EmailTransport {
  submit(notice: Notice, attemptId: string): Promise<{ externalId: string }>;
}

export async function deliverNext(
  store: NoticeStore,
  transport: EmailTransport,
): Promise<void> {
  const job = await store.claimDue();
  if (!job) return;

  if (await store.isSuppressed(job.notice.recipient)) {
    await store.markSuppressed(job.attempt.attemptId);
    return;
  }

  try {
    const receipt = await transport.submit(job.notice, job.attempt.attemptId);
    await store.markSubmitted(job.attempt.attemptId, receipt.externalId);
  } catch {
    await store.release(job.attempt.attemptId);
  }
}
```

The adapter supplies authentication and an idempotency mechanism only if its documented contract supports one. The store must still tolerate another attempt because process failure can land between `submit` and `markSubmitted`. That is the uncomfortable line to test. Hiding it behind a generic `sendEmail()` function does not remove it.

This approach has a real trade-off. It is unsuitable when nobody can own callback verification, suppression review, retention, and deletion. In that case, choose a managed workflow that exposes an auditable history and accept the tighter coupling. An application-owned template pipeline is also the wrong choice when a non-engineering operator must publish urgent copy without a deployment; use a hosted, versioned workflow and record that version on every attempt.

Callbacks take a separate path. Verify authenticity before updating the journal, retain the original external event for diagnosis, correlate by the returned identifier, and make duplicate processing harmless. A callback may enrich notification evidence. It must not create, cancel, or fulfill an order.

For password resets, render the short-lived link at the last responsible point and exclude it from stored transport metadata. Return the same user-facing response whether the account exists or not. The notification journal records that a reset notice was attempted; it does not become an account-discovery surface.

## What passes the selection exercise?

The winning candidate is the one that completes the failure drill with the least application-specific ambiguity. I use a one-page scorecard, but no weighted total. A failed gate stays failed; a pleasant template editor cannot compensate for unverifiable events.

| Gate | Evidence to retain | Reject when |
| --- | --- | --- |
| Submission correlation | Internal attempt ID and external message ID | Support cannot connect a request to later events |
| Event authenticity | Documented verification method and a rejected invalid sample | Unsigned or unverified input can change delivery state |
| Suppression control | A permanent-failure test and a reviewed release path | The worker repeatedly sends to a known bad address |
| Template reproducibility | Rendered text and HTML tied to a version | A past send cannot be reconstructed from its variables |
| Data boundary | Written flow for recipients, variables, logs, and events | “EU support” is the only answer to where data moves |
| Operational access | Exportable evidence available to the solo operator | Diagnosis requires an opaque dashboard-only trail |

For US and EU users, draw the data flow before signing up: application region, transport processing, event storage, log storage, retention, and deletion path. Do not infer that an endpoint location settles every one of those questions. Check the current contract and documentation for the exact service under review.

Templates deserve the same concrete treatment. I would keep source and fixtures beside the Node.js code when one person owns releases. A pull request can then show the subject, text body, HTML body, variable schema, and template version together. Hosted editing is a reasonable trade when someone outside engineering must publish transactional copy independently, but each attempt still needs a version that can be identified later.

Price comes last. At low volume, compare the current commercial terms only after every reliability gate passes; do not bake a volatile unit price into architecture. The revenue-per-hour calculation favors fewer support mysteries and fewer consoles over shaving a small, changing transport charge.

Cheap is contextual.

## What changes when the marketplace grows

The first scale change is priority isolation. Password resets and seller order alerts should not sit behind bulk announcements. Separate worker capacity and retry policy when observed queue age shows contention, and alert on the oldest pending notice plus submitted attempts that never receive a correlated terminal event.

Only then consider a dedicated queue or a second transport. A second transport requires synchronized suppression policy, authenticated event handling, comparable template output, domain configuration, and deliberate duplicate behavior. It adds failure modes on day one. Adopt it when measured risk justifies that operating surface, not because a comparison table has a “failover” row.

SMS is also a separate decision. A seller-alert fallback adds phone-number handling and messaging compliance; the CTIA material below provides a US industry reference for messaging practices. Do not smuggle that scope into an email project. Add the channel only after the escalation policy, consent handling, and operational owner are explicit.

The final rule is narrow: commit notification intent with the business event, force each candidate through partial failure, and select only after the journal tells a coherent story. Ship the small adapter. Revisit machinery when actual queue age, support work, or delivery uncertainty demands it.

## References

- Amazon Simple Email Service documentation: https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- CTIA messaging interoperability and compliance best practices: https://www.ctia.org/the-wireless-industry/industry-commitments/messaging-interoperability-sms-mms
