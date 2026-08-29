# Marketplace Seller Alerts: Invalid Phone, Email Templates, and JSON Schema in Node.js

Short answer: put a JSON Schema boundary in front of rendering, then classify invalid email, invalid phone, and missing template variables before the event can enter a retry loop.

For a marketplace seller waiting for a new-order alert, “no notification” is too vague to debug. The event may be malformed JSON, valid JSON with the wrong shape, a template with an absent variable, or a channel handoff that the application cannot repair. Those failures need different owners and different retry rules.

I build this around integration effort. A one-person SaaS does not have spare hours for a notification abstraction that hides the useful evidence. The first version should make the boundary explicit, keep the event small, and ship weekly. Revenue per hour is the constraint. Outsource the undifferentiated delivery work, but keep the application contract under local control.

## The reliability boundary for a seller alert

It should prove only what the application can know: the event name, the channel, the recipient representation, the template key, and the variables required by that template. It should not claim that an email inbox exists or that a phone can receive a message. Those are channel outcomes.

For a seller alert, I would keep the event close to this shape:

- `event`: `marketplace.order.created`
- `channel`: `email` or `sms`
- `recipient`: one channel-appropriate string
- `template`: `seller-new-order`
- `variables`: named values such as `order_id` and `item_count`

That separation is more important than the particular schema library. A producer owns the order facts. A renderer owns presentation. A channel adapter owns delivery status. If the producer sends an already-rendered body, a template failure becomes difficult to distinguish from a sender-policy failure.

The first guard is JSON parsing. The second is schema validation. Only after both pass should the system resolve template variables. I keep the stages visible in logs with an event ID, contract version, stage, diagnostic code, and a redacted path. Do not put the full message body or raw recipient into a default log line.

Name the stop.

An invalid phone number should be a permanent input result, not a transient delivery error. The same applies to a missing `order_id`. Retrying either one only produces more noise and consumes the time that should go into product work.

## How do I debug a malformed event notification payload for email and SMS?

The following TypeScript example is intentionally boring. It parses once, validates once, and returns field-level diagnostics. The regular expressions are shape checks, not proofs of deliverability. An E.164-shaped value can still be unreachable, and an email-shaped value can still be rejected by a receiving domain.

```ts
type Channel = "email" | "sms";

type Diagnostic = {
  code: "INVALID_JSON" | "INVALID_SCHEMA" | "INVALID_EMAIL" | "INVALID_PHONE" | "MISSING_VARIABLE";
  path: string;
  message: string;
};

const emailShape = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
const e164Shape = /^\+[1-9]\d{7,14}$/;

function isRecord(value: unknown): value is Record<string, unknown> {
  return typeof value === "object" && value !== null && !Array.isArray(value);
}

function parseJson(raw: string): { value?: unknown; error?: Diagnostic } {
  try {
    return { value: JSON.parse(raw) };
  } catch {
    return {
      error: {
        code: "INVALID_JSON",
        path: "$",
        message: "The event body is not valid JSON",
      },
    };
  }
}

function validateOrderNotification(value: unknown): Diagnostic[] {
  if (!isRecord(value)) {
    return [{ code: "INVALID_SCHEMA", path: "$", message: "Expected an object" }];
  }

  const errors: Diagnostic[] = [];
  if (value.event !== "marketplace.order.created") {
    errors.push({ code: "INVALID_SCHEMA", path: "$.event", message: "Unexpected event" });
  }
  if (value.channel !== "email" && value.channel !== "sms") {
    errors.push({ code: "INVALID_SCHEMA", path: "$.channel", message: "Expected email or sms" });
  }

  if (typeof value.recipient !== "string" || value.recipient.length === 0) {
    errors.push({ code: "INVALID_SCHEMA", path: "$.recipient", message: "Recipient is required" });
  } else if (value.channel === "email" && !emailShape.test(value.recipient)) {
    errors.push({ code: "INVALID_EMAIL", path: "$.recipient", message: "Expected an email address shape" });
  } else if (value.channel === "sms" && !e164Shape.test(value.recipient)) {
    errors.push({ code: "INVALID_PHONE", path: "$.recipient", message: "Expected an E.164-shaped phone number" });
  }

  if (value.template !== "seller-new-order") {
    errors.push({ code: "INVALID_SCHEMA", path: "$.template", message: "Unexpected template" });
  }

  if (!isRecord(value.variables)) {
    errors.push({ code: "INVALID_SCHEMA", path: "$.variables", message: "Variables must be an object" });
  } else {
    for (const name of ["order_id", "item_count"]) {
      if (typeof value.variables[name] !== "string" || value.variables[name].length === 0) {
        errors.push({
          code: "MISSING_VARIABLE",
          path: `$.variables.${name}`,
          message: `Missing ${name}`,
        });
      }
    }
  }

  return errors;
}

function inspect(raw: string): Diagnostic[] {
  const parsed = parseJson(raw);
  return parsed.error ? [parsed.error] : validateOrderNotification(parsed.value);
}

const diagnostics = inspect(process.argv[2] ?? "null");
if (diagnostics.length > 0) {
  process.stderr.write(`${JSON.stringify({ diagnostics })}\n`);
  process.exitCode = 1;
} else {
  process.stdout.write("notification accepted\n");
}
```

The useful output is not “validation failed.” It is `$.variables.order_id` with `MISSING_VARIABLE`, or `$.recipient` with `INVALID_PHONE`. A support ticket can then point to the producer, the template release, or the recipient input without reading a worker's stack trace.

There is a small trap here. If `channel` is invalid, the validator should not pretend it knows whether the recipient is an email or phone number. The diagnostic should stay at `$.channel` until the producer fixes the discriminator. Error paths are part of the interface, so keep their names stable and test them.

Consider the seller who reports that an order notification disappeared. The raw body first fails JSON parsing if a producer cut off the final brace; that is a producer or transport record, and no retry can repair the bytes. If parsing succeeds but `variables.order_id` is absent, the schema stage records a permanent contract error and the queue stops. If both checks pass but the template asks for `{{seller_name}}`, the render stage rejects the release before any email or SMS adapter sees it. If all three stages pass and the handoff is later rejected, the diagnostic belongs to delivery policy. I want those four records to remain distinguishable because each one changes the next action: fix the producer, fix the event contract, fix the template, or inspect channel rules. Without that sequence, a worker can retry a bad payload for hours while the seller keeps waiting for an alert that was never deliverable.

## Treat template variables as a release artifact

JSON Schema can establish that `variables` is an object. It cannot, by itself, guarantee that a template uses only variables the event supplies. That is a separate contract between the template manifest and the renderer.

Keep the required variable names beside each template. For the seller's new-order email, the manifest might require `order_id` and `item_count`; the SMS version can use the same names while applying its own length and formatting rules. The event-to-template mapping should be reviewable in source control or in the application database, rather than hidden in a conditional inside the worker.

Test three things separately:

1. A valid JSON fixture passes the schema.
2. A render fixture supplies every required variable and produces readable email and SMS output.
3. Invalid fixtures cover malformed JSON, an invalid email, an invalid phone, an unknown template, and a missing variable.

This catches a quiet failure: the JSON is valid, the schema passes, but the stored template contains `{{seller_name}}` while the event only supplies order data. A renderer that leaves that placeholder in the final message has turned a detectable contract error into a seller-facing defect. Fail the render instead.

Do not confuse preview with deliverability. A preview can prove that the variables resolve and that the message is readable. It cannot prove sender authentication, domain policy, mailbox existence, carrier reachability, or acceptance by a downstream system. Email sender requirements belong to the delivery layer; Google's sender guidance is a useful reference for that boundary.

## Keep permanent failures out of the retry queue

I use two broad failure classes. Permanent failures include `INVALID_JSON`, `INVALID_SCHEMA`, `INVALID_EMAIL`, `INVALID_PHONE`, and `MISSING_VARIABLE`. Transient failures include a delivery attempt whose outcome may change without changing the event or template. The exact transient codes belong to the channel adapter and its documented contract.

This distinction protects the queue. A malformed order event should be retained in a reviewable dead-letter store with its event ID, contract version, redacted payload, and diagnostics. It should not be retried ten times while the same invalid `recipient` produces ten identical records. A transient result can follow an explicit backoff policy and idempotency key.

The idempotency key should be derived from stable event identity and channel. A worker restart must not send a second new-order alert simply because the first attempt completed before the worker recorded its local state. Store the decision and delivery result in a way that makes duplicate handling explicit. Retention is a product and compliance decision; document it rather than burying it in a queue library.

Observability should preserve the same stages as the code: parsed, schema-valid, rendered, handed off, and delivered or rejected. Count by event name, channel, and diagnostic code. Avoid raw recipient values as metric labels. When those stages collapse into one `notification_error`, debugging becomes archaeology.

## Compare integrations by what they make you own

Once the local contract is stable, compare integrations by the concepts they force into application code: channel-specific status models, template identifiers, sender configuration, retry semantics, and delivery callbacks. A shared interface can reduce adapter code. A channel-focused integration can expose controls the shared interface cannot represent. The engineering cost is the tests, dashboards, and release notes that follow each leaked concept.

For the first release, I would keep the boundary narrow: one marketplace order event, one seller-facing template per channel, field-level diagnostics, redacted logs, idempotency, and separate permanent and transient paths. That is enough to ship weekly and measure whether the alert helps sellers respond to orders. More channels can wait until the failure records justify them.

The catch is that this design is not suitable when the product needs a channel-specific compliance workflow, provider callbacks as part of business orchestration, or a delivery feature the local adapter cannot express. Choose an integration whose documented capabilities fit that requirement, even if its adapter is more involved. Preserve the domain event and schema so the choice remains local to delivery.

I'm not sure a universal notification interface stays valuable once every channel has different policy and observability needs. Your mileage may vary. The decision should follow the failure records and product requirements, not the elegance of a diagram.

My practical rule is short: reject malformed data before rendering, reject unresolved variables before delivery, and retry only outcomes that can change on their own. That is the smallest useful system for a solo SaaS, and it gives a marketplace seller a much better answer than “the alert failed.”

## References

- [Google: Email sender guidelines](https://support.google.com/a/answer/81126)
- [NIST SP 800-63B: Digital Identity Guidelines](https://pages.nist.gov/800-63-3/sp800-63b.html)
