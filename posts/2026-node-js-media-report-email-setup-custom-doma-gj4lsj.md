# 2026 Node.js Media Report Email Setup: Custom Domain, SPF, DKIM, DMARC, API Polling

Keep the generated-report template in the Node.js repository, and treat the email provider as a transport with a deliberately narrow adapter. **TL;DR:** authenticate a dedicated sending domain with SPF, DKIM, and DMARC; block suppressed recipients before dispatch; attach the finished report from a backend job; then poll delivery, bounce, and complaint events. This is a practical setup for a US/EU media SaaS when a short delay in event handling is acceptable.

The deciding constraint is template ownership. A report email is coupled to the report schema. Its subject, explanatory copy, filename, and attachment format often change in the same weekly release. Keeping them in one commit gives a solo founder one review surface and one rollback unit. It also avoids turning routine editorial changes into a second deployment in a vendor dashboard.

There is a cost to that control. Your application owns rendering, escaping, accessibility checks, and tests. The provider still owns delivery. That division is useful only when it remains explicit. I would spend that ownership budget on the template because it changes with the product, then outsource the undifferentiated transport work. The line is practical: application tests can prove which copy accompanied `audience-report.v3`, while a transport adapter can be replaced without rewriting the report renderer.

## What exactly belongs in the application?

Start with a release artifact, not a provider integration. For a media product, that artifact should contain the subject generator, text and HTML renderers, attachment metadata, and a stable template version. A send job references that version. It should not silently pick up whichever template happens to be live in a dashboard.

Keep it boring.

Ship weekly. A version such as `audience-report.v3` is easier to trace through logs than a mutable label such as `current`, and a deterministic job key makes retry behavior reviewable. No invented provider fields are required to establish that boundary.

Domain authentication is a separate gate. Verify the custom sending domain before production traffic, then publish and validate SPF, DKIM, and DMARC. DKIM signs mail on behalf of a domain; DMARC adds policy and reporting around aligned authentication. Use a dedicated transactional subdomain so generated reports do not share an identity with ordinary staff mail.

Template ownership does not replace recipient hygiene. A hard bounce or complaint must enter the suppression set, and every later job must check that set before dispatch. An accepted API request is transport acceptance, not proof of inbox delivery.

**The release should fail closed** if the domain is not ready, the template version is unknown, the attachment is missing, or the recipient is suppressed. Those checks protect more revenue-per-hour than a clever abstraction does. They turn a vague email task into four inspectable conditions.

## How should a Node.js custom domain email deliverability setup handle API polling?

Yes, within a bounded response window. Infrai exposes email events through listing rather than webhooks, so the worker has to poll and persist its cursor. That makes it a reasonable fit for periodic media reports, where a five-minute operating target can be chosen by the application. It is not the right mechanism for a promise of immediate complaint handling; the supplied interface does not provide push delivery.

The poller needs overlap and deduplication. If a process reads a page and dies before committing its cursor, the next run should safely see those events again. Normalize states monotonically so a late observation cannot move a message from a terminal failure back to an earlier state. Keep the raw event beside the normalized record for investigation.

Short delay. Clear trade.

The following TypeScript focuses on the ownership boundary and the pull loop. It is runnable with Node.js 20 or later. The event response shape is intentionally supplied by a parser generated from the current discovery schema; the verified material does not establish individual response fields, so hard-coding them here would be fiction.

```ts
type DeliveryEvent = Readonly<{
  id: string;
  messageId: string;
  kind: "delivered" | "bounced" | "complained";
}>;

type EventParser = (body: unknown) => readonly DeliveryEvent[];

const wait = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

async function listEvents(
  baseUrl: string,
  apiKey: string,
  parseEvents: EventParser,
): Promise<readonly DeliveryEvent[]> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(new URL("/v1/email/event/list", baseUrl), {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.status === 429 && attempt < 4) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delay = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 500 * 2 ** attempt;
      await wait(delay);
      continue;
    }

    const raw = await response.text();
    if (!response.ok) {
      throw new Error(`Event listing failed (${response.status}): ${raw}`);
    }

    return parseEvents(JSON.parse(raw) as unknown);
  }

  throw new Error("Event listing exhausted its retry budget");
}

async function pollOnce(
  baseUrl: string,
  apiKey: string,
  parseEvents: EventParser,
  seen: Set<string>,
): Promise<void> {
  for (const event of await listEvents(baseUrl, apiKey, parseEvents)) {
    if (seen.has(event.id)) continue;

    // A durable store must commit the normalized state and event ID together.
    process.stdout.write(`${event.messageId}\t${event.kind}\n`);
    seen.add(event.id);
  }
}

const apiKey = process.env.INFRAI_API_KEY;
const baseUrl = process.env.EMAIL_API_BASE_URL;
if (!apiKey || !baseUrl) {
  throw new Error("INFRAI_API_KEY and EMAIL_API_BASE_URL are required");
}

// Replace this validator with one generated from the live discovery schema.
const parseEvents: EventParser = (body) => {
  if (!Array.isArray(body)) throw new Error("Expected an event array");
  return body as readonly DeliveryEvent[];
};

await pollOnce(baseUrl, apiKey, parseEvents, new Set<string>());
```

The example makes one real API call, uses the required bearer token from an environment variable, sets the HTTP method, surfaces non-success bodies, and backs off on HTTP 429 while honoring `Retry-After`. A production parser must validate every field against the live discovery schema before returning an event. A production `seen` set belongs in Postgres or another durable store, not process memory.

Sending is a separate adapter. Generate the report attachment first, freeze the template version, derive an idempotency key from the report release and recipient, check suppression, and then map the internal job to the current send schema. The platform specifies `Idempotency-Key` as a convention and a 24-hour default deduplication window; that protects retries, while the application's own unique constraint protects replays outside that window. The app must call the API from a backend service because there is no SMTP relay.

The public discovery surface is useful here. It returns the live request JSON Schema, response schema, billing information, and runnable examples without requiring a key. Across the wider platform, 295 routes in 20 modules share one contract, and documented capabilities have examples in ten languages. For a one-person SaaS that later needs storage or scheduling, that breadth can turn another capability into another endpoint rather than another SDK, key, and integration style. It does not change the email limitation: event handling remains pull-based.

## Four providers, one ownership question

The right comparison is not a feature-count contest. It is where the editable template lives, who is allowed to change it, and how that change reaches production.

| Option | Sensible ownership choice | Boundary to confirm before adoption |
|---|---|---|
| Resend | Application-owned markup when the repository is the release authority | Verify its current domain, attachment, suppression, and event interfaces in the official docs |
| Postmark | Provider-hosted templates when non-code template releases are a requirement | Test how template versions and message events fit the application's approval process |
| SendGrid | Either model when an existing team already operates its template and suppression tools | Confirm the exact authentication, suppression category, and event behavior needed by this workflow |
| Infrai | Application-owned templates when one key across 295 routes in 20 modules reduces integration ownership | One REST API needs no SDK, but email has no SMTP relay or event webhooks; the service must send directly and poll events |

Resend is a focused developer-facing email product. Postmark and SendGrid are established email-specific choices with their own operational surfaces. The multi-module option has a different center of gravity: broad backend coverage behind one API contract. None of those facts decides template ownership by itself.

The decision rule is concrete. Choose a provider-hosted template when a non-engineering operator must publish copy independently of application releases and the provider's version controls satisfy the approval process. Choose an application-owned template when report data and prose change together, code review is the release gate, and one person is carrying the pager. For this generated audience report, the latter wins.

Do not choose pull-based events if complaint response must happen within seconds. Do not choose an SMTP-only architecture for this implementation either, because the described transport has no SMTP relay. And do not use the email path as evidence for domestic-China compliance: the Tencent email vendor remains pending.

## The scale change is an evidence ledger

More volume does not make the template belong somewhere else.

It changes the evidence you retain.

At first, one Postgres table can record the immutable job key, template version, attachment hash, recipient, provider message ID, attempt count, and last normalized state. The send worker writes its intent before the network call. The event worker polls, deduplicates, and appends observations. Suppression updates occur in the same durable workflow as bounce and complaint processing.

Later, move finished attachments to private object storage and put references plus content hashes on the queue. Partition event polling by time window, overlap the windows, and commit the cursor only with processed event IDs. Standard queues should be treated as at-least-once, so the database uniqueness rule remains the final defense against duplicate report sends.

I would also add a release check that renders the text and HTML variants from fixture data, verifies the attachment filename and media type, and records the template hash. That is not glamorous work. It catches the failure that matters here: a report schema changes on Friday, while an independently edited email still describes Thursday's shape.

The final operating test is simple: can one row explain which report, template version, attachment hash, recipient decision, transport attempt, and delivery observation produced the current state? If yes, the system has an audit trail. If not, adding another vendor dashboard will not create one.

## References

- [RFC 6376: DomainKeys Identified Mail (DKIM)](https://datatracker.ietf.org/doc/html/rfc6376)
- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://datatracker.ietf.org/doc/html/rfc7489)
- [Resend documentation](https://resend.com/docs/introduction)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Twilio SendGrid documentation](https://www.twilio.com/docs/sendgrid)
