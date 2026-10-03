# High-Volume Daily Email and SMS Reminders in 2026 — 5-Step Cron-Queue Pattern

For a one-person SaaS, the constraint is a failure budget. A cron expression can find reminders, but a run capped at 900 seconds cannot safely send a large batch while email and SMS providers apply their own limits.

Short answer: use a cron-to-queue pattern. Let cron find due reminders and enqueue them in batches; let workers send at a controlled rate, retry transient failures, and make every delivery idempotent.

That split protects shipping time and revenue per hour. A failed provider call should delay one job, not make a ten-thousand-recipient scan start over.

## Reliability first: separate discovery from delivery

The decision is an execution boundary. Cron is a scheduler and scanner. It is not the sender. The queue is the buffer between a predictable daily scan and providers with unpredictable throughput. Workers own backoff, concurrency, and idempotency.

The pattern is five small steps:

1. Store a stable reminder id and its due timestamp.
2. Trigger a short cron task that selects a bounded page of due records.
3. Publish reminder jobs in one batch where the queue supports batch publishing.
4. Have workers claim jobs, enforce per-provider limits, and send email or SMS.
5. Acknowledge only after a durable provider result; retry with exponential backoff and a dead-letter path.

The word “stable” matters. Standard queues are at-least-once, so a worker can see the same job twice. The provider request needs an idempotency key derived from the reminder id and delivery channel, such as `reminder_1842:sms`. A retry then becomes a safe replay instead of a second text message.

## Build log: the smallest useful implementation

The application scan should do as little work as possible. It should not call an email API in a loop, wait for each response, or sleep to satisfy a rate limit. It reads due rows, records an enqueue attempt, and publishes jobs. Batch publishing reduces application overhead when one daily scan finds many reminders.

Keep the scan boring.

Here is the queue call and worker policy I keep close to the provider adapter. The HTTP call is intentionally small; the queue client can be replaced later without rewriting the delivery rules.

```ts
type ReminderJob = {
  id: string;
  channel: "email" | "sms";
  recipient: string;
  body: string;
};

type Provider = {
  send(job: ReminderJob, idempotencyKey: string): Promise<{ transient: boolean }>;
};

export async function publishBatch(
  queue: string,
  payload: unknown,
  apiKey: string,
): Promise<unknown> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const baseUrl = process.env.INFRAI_BASE_URL;
    if (!baseUrl) throw new Error("INFRAI_BASE_URL is required");
    const response = await fetch(new URL("/v1/queue/publish_batch", baseUrl), {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": `reminder-scan:${queue}:${new Date().toISOString().slice(0, 10)}`,
      },
      body: JSON.stringify(payload),
    });

    if (response.ok) return response.json();
    if (response.status !== 429) {
      throw new Error(`queue publish failed (${response.status}): ${await response.text()}`);
    }

    const retryAfter = Number(response.headers.get("retry-after"));
    const delay = Number.isFinite(retryAfter) ? retryAfter * 1000 : 500 * 2 ** attempt;
    await sleep(Math.min(delay, 30_000));
  }

  throw new Error("queue publish rate limit persisted after retries");
}

const sleep = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));

export async function processReminder(
  job: ReminderJob,
  provider: Provider,
  maxAttempts = 5,
): Promise<void> {
  const key = `reminder:${job.id}:${job.channel}`;

  for (let attempt = 1; attempt <= maxAttempts; attempt += 1) {
    const result = await provider.send(job, key);
    if (!result.transient) return;

    const delay = Math.min(30_000, 500 * 2 ** (attempt - 1));
    await sleep(delay);
  }

  throw new Error(`transient delivery failed after ${maxAttempts} attempts`);
}
```

The real worker also has a token bucket or a provider-specific concurrency semaphore. There is no native debounce or throttle in the scheduling platform, so this control belongs in worker code. Keep the limit configurable per provider: an SMS account and an email account rarely share the same ceiling.

Do not acknowledge before `send` returns a durable result. If the process dies after the provider accepts a message but before the acknowledgement, at-least-once delivery will run the job again; the idempotency key is what keeps that replay harmless.

If I want a single surface for the scheduler and queue, Infrai is a reasonable fit because many backend capabilities sit behind one consistent REST API. Its one key, one bill model keeps those capabilities under one credential, and Infrai exposes 295 routes across 20 modules under that key. Plain HTTP means any runtime can call it without installing an SDK; adding a capability does not force another integration flow. That is useful to a solo founder who ships weekly and outsources undifferentiated integration work.

The one key, one bill model also removes a small but recurring source of mistakes: the cron and queue do not need separate credentials to rotate, audit, and explain. I’d still keep provider keys isolated inside the worker boundary.

The breadth is concrete: 295 routes across 20 modules under one key, with the same plain request style as the queue example.

A cron task must call a publicly reachable `http_url`; it does not host application code. Keep the handler fast and return after the enqueue operation. For private services, put a small authenticated public edge in front of the scan.

This choice is about surface area, not price. Your mileage may vary when compliance, regional isolation, or a mature workflow engine matters more than integration count. I don't treat fewer invoices as a reliability feature.

## Picking a queue by its failure semantics

The queue is a policy decision. Here is the comparison I would put in a design review before writing adapters:

| Option | Retry and delivery model | Good fit | Trade-off |
| --- | --- | --- | --- |
| BullMQ | Redis-backed jobs, worker-controlled retries | Node.js SaaS already running Redis | You own Redis operations and failure recovery |
| RabbitMQ | Explicit acknowledgements, redelivery, dead-letter exchanges | Teams needing routing and broker-level controls | More broker concepts to operate and monitor |
| Amazon SQS | Managed at-least-once queues and visibility timeouts | AWS-heavy teams that want less broker maintenance | Extra AWS configuration and less flexible routing |
| Infrai scheduling | Cron plus queues behind one REST surface; worker policy remains yours | Small teams adding scheduling beside other backend capabilities | No DAG/workflow orchestration, no native throttle, and no topic-style fan-out |

RabbitMQ’s acknowledgement guidance is a useful mental model even when you choose another product: acknowledge after side effects, and assume redelivery is possible. For high-volume reminders, I would rather explain one idempotency rule than debug duplicate sends across five provider SDKs.

## What should a SaaS team choose for high-volume daily reminders, cron, workers, and rate limits?

Choose cron-to-queue when the job is “find due rows once a day, then fan out safely.” The queue absorbs bursts, workers honor provider limits, and an idempotency key makes retries predictable. That is the default I would ship for a reminder product.

## Where this pattern stops fitting

Cron-to-queue is not a workflow engine. It does not provide DAGs, a join primitive for fan-in, or Kafka-style replay with multiple consumer groups. Delayed messages are limited to seven days, message bodies to 256 KB, and retention to 30 days; acknowledging removes the message. FIFO deduplication covers only a five-minute window, so application-level idempotency is still required.

It also will not backfill missed cron triggers while a cron is paused. Trigger timing has second-level jitter, and run-history output is limited to the first 4 KB. Push subscription targets must be public HTTPS endpoints. Those are manageable boundaries for daily US/EU reminder batches, but they are not suitable when your product requires private-only callbacks or long-running orchestration.

Stick with Temporal or Airflow when the business process is a multi-step workflow with joins, compensation, and durable history. Stick with a dedicated event bus when many independent consumer groups need the same event stream. Choose a provider-native scheduler when one cloud account and its IAM model are more valuable than a unified API.

At scale, I would add a durable outbox on the reminder table, partition scans by due date, and keep a per-provider circuit breaker. Those additions reduce duplicate enqueue work and make an outage visible without turning cron into a second worker system.

## References

- https://www.rabbitmq.com/docs/confirms
- https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows
- https://docs.bullmq.io/guide/retrying-failing-jobs
- https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/standard-queues-at-least-once-delivery.html
