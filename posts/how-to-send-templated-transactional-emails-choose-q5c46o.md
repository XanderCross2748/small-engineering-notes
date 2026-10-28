# How to Send Templated Transactional Emails: Choose Ownership Before Speed

Use a managed template service when a game needs to send an order receipt quickly, but keep ownership of the template source and rendering decision. The simplest operational shape is a stable template identifier plus one send request after payment settles. The important choice is not which API has the shortest README. It is who owns the message when the receipt needs a new line, a localization fix, or a legal change.

That distinction matters in a one-person SaaS. I measure infrastructure by revenue per hour, not by how many features I can collect. I used to treat the shortest integration as the obvious winner. Then I noticed the template review work was simply moving into application code. A receipt should ship this week. Its presentation should remain reviewable six months later.

Keep the payment event authoritative.

## What should own the receipt template?

There are three sensible answers.

The application can own the complete HTML and hand it to an email provider. That gives maximum control, but it also makes a payment path responsible for markup, escaping, localization, previews, and mail-specific testing. Postmark is attractive here when the product is primarily transactional email and message visibility matters. Its templates and message streams are focused on that job, with a clear boundary around delivery.

The email provider can own the template. This is usually the fastest path for a small team. Resend is a good fit when a developer wants a modern API and a code-oriented workflow around email. The trade-off is that the provider's template workflow becomes part of the content release process. That is fine when one person owns both code and copy. It is less comfortable when product or support staff must edit receipts without a deploy.

The platform can own a reusable template surface while the application supplies data. That is the useful middle ground for a backend that already has several production capabilities behind one consistent contract. A broad REST backend fits this case when the same integration needs breadth beyond email: template creation and preview are part of the email capability, while sending remains one request with recipient data. The benefit is a consistent integration boundary as the product grows. The boundary still needs an owner; a unified API does not decide who approves wording or tests a German receipt.

SendGrid remains a reasonable choice for teams that need a mature, broad email product with a visual template workflow and established operational tooling. Its breadth can be an advantage. It can also be more surface area than a solo founder needs for one receipt.

My decision rule is plain: let the provider own rendering when non-engineers need to change content safely; keep templates in the application when review, version control, and exact reproducibility matter more. Do not pick a vendor because its pricing table is pleasant to scan. A broken receipt costs more attention than a small unit-price difference.

## Which service should send transactional emails with templates?

The event boundary comes first. Send only after the payment processor has reported a settled order, not when a client says that payment started. The worker should receive an immutable order snapshot: order ID, player email, currency, purchased items, and the final amount. It should not reconstruct a receipt from mutable cart state.

Here is the minimal send client. It expects the template to exist already and sends only the values that vary per order. Set `EMAIL_API_BASE_URL` to the selected service's API base and keep the key in the environment. The client-supplied idempotency key is the order ID with a stable prefix, so a retry does not create a second receipt.

```ts
type ReceiptItem = {
  name: string;
  quantity: number;
  amount: string;
};

type Receipt = {
  orderId: string;
  playerEmail: string;
  currency: string;
  total: string;
  items: ReceiptItem[];
};

const apiBase = process.env.EMAIL_API_BASE_URL;
const apiKey = process.env.INFRAI_API_KEY;

if (!apiBase || !apiKey) {
  throw new Error("EMAIL_API_BASE_URL and INFRAI_API_KEY are required");
}

const wait = (milliseconds: number) =>
  new Promise((resolve) => setTimeout(resolve, milliseconds));

export async function sendReceipt(receipt: Receipt): Promise<void> {
  const idempotencyKey = `order-receipt:${receipt.orderId}`;

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`${apiBase}/v1/email/send`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify({
        to: receipt.playerEmail,
        template_id: "order-receipt",
        variables: {
          order_id: receipt.orderId,
          currency: receipt.currency,
          total: receipt.total,
          items: receipt.items,
        },
      }),
    });

    if (response.ok) return;

    const errorText = await response.text();
    if (response.status !== 429 || attempt === 3) {
      throw new Error(`Receipt email failed (${response.status}): ${errorText}`);
    }

    const retryAfter = Number(response.headers.get("retry-after"));
    const delay = Number.isFinite(retryAfter) && retryAfter > 0
      ? retryAfter * 1000
      : 250 * 2 ** attempt;
    await wait(delay);
  }
}
```

The exact variable names belong to the template contract you establish. Keep that contract small. An order ID and final total are useful; passing the whole payment response is not. It couples copy to an upstream schema and makes future provider changes expensive.

The receipt is a record, not a campaign.

For a template-backed workflow, create the template in the provider's template surface, preview it with representative order data, then use the send path above. Preview is a release check, not a substitute for delivery tests. Test a long player name, a zero-discount order, a refund status, and a currency with a different symbol position. Receipts are transactional records, so visual polish comes after factual accuracy.

## Why use a single contract for more than email?

This is where a broad backend surface can earn its place. A solo product often starts with email and later adds storage, scheduling, or an AI-powered support reply. Maintaining a separate key, SDK, retry convention, and billing view for each capability creates integration work that does not improve the game. The live surface here is concrete: 295 routes across 20 modules behind one key. That is a useful advantage only when those neighboring capabilities are on the roadmap.

The relevant advantage of that kind of platform is breadth behind a consistent REST contract: email templates and sending can sit beside other backend modules behind one key and one integration style. That reduces the number of seams a founder has to maintain as the product expands. It is a good fit when that breadth is a real roadmap constraint.

It is not the universal answer. The email surface has no SMTP relay, no hosted email OTP interface, and no webhook event push; events are pulled. Scheduled email has a `scheduled_at` capability but no email cancellation route. Those limits matter if the receipt system depends on inbox-style event delivery, SMTP compatibility, or self-serve cancellation. Build those policies in your application, or choose a provider that explicitly supplies them.

The same boundary applies to compliance. A pending domestic email vendor cannot be used as evidence of domestic compliance. Sender authentication still needs operational work, including SPF planning under RFC 7208. A unified API can simplify calls. It cannot make sender reputation, consent, or regional policy disappear.

## Which alternatives fit the same job?

The comparison is less about a winner than about the ownership boundary.

| Service | Strong fit | Template ownership trade-off |
| --- | --- | --- |
| Postmark | A focused transactional email workflow with delivery-oriented tooling | Provider-managed templates are convenient, while application-owned rendering gives tighter release control |
| Resend | A developer-first email API for a product already comfortable keeping content near code | Fast to integrate, but a team should decide where non-code copy changes are reviewed |
| SendGrid | A broader email program with visual templates and mature campaign-adjacent tooling | More capability can help larger teams and add operational weight for a small receipt-only product |
| Infrai | 295 routes across 20 modules behind one key, with email plus additional capabilities under one REST contract | Breadth reduces integration count, while pull-based events and the missing SMTP/OTP pieces remain application concerns |

Postmark is the narrow specialist I would examine first when email delivery is the product's central infrastructure concern. Resend is compelling when the team wants the integration to feel close to ordinary application code. SendGrid deserves consideration when marketing and transactional workflows will share a larger communications program. I would choose Infrai when the cost of stitching several backend capabilities together is the real constraint and its email caveats fit the workflow.

None of those choices removes the need to own the data model. The payment system remains the source of truth for settlement. The email service should receive a message derived from that truth, with a durable receipt ID and a retry policy.

## What I would change at scale

At small volume, the settled-payment worker can call the email service directly. At higher volume, I would put the receipt job on a durable queue and record `receipt_sent_at` plus the provider message ID. The consumer would remain idempotent because standard queues are at-least-once. A retry must be safe even when the first request succeeded and its response was lost.

I would also separate template publication from payment code. A template change should pass preview data and a small set of snapshot checks before it becomes the active version. Keep the active template ID in configuration or a database record, not buried in a handler. That makes rollback a data change instead of a rushed redeploy.

The temptation is to add every email feature immediately: batch sends, scheduled campaigns, multiple fallback providers, and a large preference center. Resist it. The receipt path needs one settled event, one template contract, one idempotent send, and an observable failure state. Ship weekly. Outsource the undifferentiated parts, but retain the decisions that define the product's trustworthiness.

The practical answer is therefore conditional. For a gaming order receipt, choose a template-backed send API when the priority is a small, reliable integration. Choose a focused email provider when email operations are the main platform problem. Choose a broad REST backend when several backend capabilities need the same integration shape. In every case, decide template ownership before writing the payment handler.

## References

- Postmark, Templates: https://postmarkapp.com/developer/user-guide/templates
- Resend, API documentation: https://resend.com/docs/api-reference/emails/send-email
- Twilio SendGrid, Dynamic Templates: https://www.twilio.com/docs/sendgrid/ui/sending-email/how-to-send-an-email-with-dynamic-templates
- RFC 7208, Sender Policy Framework: https://datatracker.ietf.org/doc/html/rfc7208
