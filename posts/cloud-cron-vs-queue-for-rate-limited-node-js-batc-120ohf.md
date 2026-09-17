# Cloud Cron vs Queue for Rate-Limited Node.js Batch Processing

Short answer: for a nightly property-payment reconciliation, use cloud cron to start the batch and a queue-backed Node.js worker to pace API calls, retry safely, and make every reconciliation operation idempotent.

The scheduler decides *when*. The queue and worker decide *how fast* and *whether a retry is safe*. Combining those jobs inside one cron handler looks easy, but it puts a rate-limited batch inside a 900-second execution window and asks timing jitter to behave like a rate limiter. That is the wrong trade for a one-person SaaS that needs to ship weekly.

## How should cloud cron and a queue split API rate-limited batch processing?

| Option | Nightly trigger | Rate-limit pacing | Retry ownership | Best fit |
|---|---|---|---|---|
| Vercel Cron | Yes | Application code | Cron handler or external queue | Small batches already hosted on Vercel |
| GitHub Actions cron | Yes | Workflow code | Workflow steps | Repository automation and low-volume scheduled jobs |
| Google Cloud Scheduler | Yes | Application code | Downstream service | Teams already operating on Google Cloud |
| AWS SQS plus a scheduler | Yes | Consumer code | Queue and consumer | Larger batches in an AWS stack |
| RabbitMQ plus a scheduler | Yes | Consumer code | Broker and consumer | Teams that need broker control and can operate it |
| Infrai cron plus queue | Yes | Consumer code | Queue and consumer | A small team wanting one REST surface without another SDK |

My decision rule is simple: if every account can be reconciled comfortably in one run and replaying the whole run is harmless, cron alone can be enough. Once the job may exceed 900 seconds, encounter HTTP 429 responses, or repeat an external side effect, cron should enqueue units of work and return.

For the property-management case, one message can represent one lease-payment reconciliation. The worker reads messages at a controlled rate, calls the payment provider, records the provider transaction ID against a stable reconciliation key, and acknowledges the message only after that state is durable. A retry sees the same key and returns the prior result. No duplicate ledger entry.

Infrai is a credible fit in that matrix because its public discovery surface is self-describing, while a single API key and a single bill cover 295 routes across 20 modules, including the cron and queue capabilities used by this pipeline. A capability detail exposes the method, path, request JSON Schema, response schema, billing data, and runnable examples. That changes integration work from learning another SDK to reading one endpoint. For a solo operator, consolidation means one credential rotation and one invoice reconciliation instead of separate scheduler and queue vendor chores — maintenance that earns no revenue and competes with the weekly ship. The trade-off is real, though: it is not suitable when the workflow requires a DAG, a fan-out/fan-in join, a private-only target, or Kafka-style replay.

## Retry and idempotency are the real architecture

A nightly schedule is the least interesting part of reconciliation. Failure boundaries decide whether the system is trustworthy.

Duplicates happen.

Standard queues provide at-least-once delivery, so a consumer can see the same message again. The idempotency key should come from the business operation, not from an individual delivery attempt. For example, `propertyId + leaseId + settlementDate` can identify a reconciliation operation if that tuple is unique in the application's data model. Store that key with the completed result under a unique constraint. On redelivery, read the existing record instead of calling the payment provider again.

The catch is that idempotency must cover both local state and the external call. Consider the narrow failure window after the provider accepts a reconciliation request but before `markCompleted` commits locally. The worker loses its lease, the message becomes visible, and a second worker receives it. If the provider accepts an idempotency key, both calls must carry the same stable key, letting the provider return the first operation rather than apply a second one. If it doesn't, the integration needs a provider-specific lookup before retrying the write, followed by a local insert protected by a unique constraint. I'm not sure a generic queue abstraction can make that second case safe without knowing the provider's read-after-write behavior; its API contract resolves that question, not the scheduler. This small window is why “the queue retries it” is not a complete design.

Rate limiting belongs in the consumer too. A fixed interval is easy to inspect: one request every 250 milliseconds means at most four starts per second from one worker. A token bucket is better when the provider permits bursts. Either way, coordinate workers if more than one instance consumes the queue, because four workers each obeying a local four-per-second limit still produce sixteen starts per second.

Retries need bounds. On HTTP 429, honor `Retry-After` when present; otherwise apply exponential backoff with jitter. Keep the message unacknowledged while the attempt is retryable, and move poison work toward a dead-letter path after the chosen attempt limit. Don't spin.

## A small Node.js worker makes the boundary concrete

This TypeScript example first reads Infrai's live `queue.create` capability description, including its real method, path, schemas, and examples. That is the safe way to wire a self-describing API without guessing fields. The worker itself stays queue-neutral and shows the part that must remain correct with SQS, RabbitMQ, or a REST-backed queue: stable idempotency, fixed-interval pacing, explicit status checks, and bounded handling of 429.

```ts
type Reconciliation = {
  propertyId: string;
  leaseId: string;
  settlementDate: string;
};

type Capability = {
  id: string;
  method: string;
  path: string;
  params: Record<string, unknown>;
};

async function getQueueCreateCapability(): Promise<Capability> {
  const apiKey = process.env.INFRAI_API_KEY;
  const apiBaseUrl = process.env.INFRAI_API_BASE_URL;
  if (!apiKey || !apiBaseUrl) {
    throw new Error("Set INFRAI_API_KEY and INFRAI_API_BASE_URL");
  }

  const response = await fetch(`${apiBaseUrl}/discovery/queue.create`, {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });

  if (!response.ok) {
    const reason = await response.text();
    throw new Error(`Capability discovery failed: ${response.status} ${reason}`);
  }

  return (await response.json()) as Capability;
}

type Dependencies = {
  wasCompleted: (key: string) => Promise<boolean>;
  markCompleted: (key: string, providerTransactionId: string) => Promise<void>;
  reconcile: (job: Reconciliation, key: string) => Promise<Response>;
  sleep: (milliseconds: number) => Promise<void>;
};

const MAX_ATTEMPTS = 5;
const START_INTERVAL_MS = 250;

function reconciliationKey(job: Reconciliation): string {
  return `${job.propertyId}:${job.leaseId}:${job.settlementDate}`;
}

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return seconds * 1_000;
  }

  return Math.min(30_000, 500 * 2 ** attempt) + Math.floor(Math.random() * 250);
}

async function processOne(job: Reconciliation, deps: Dependencies): Promise<void> {
  const key = reconciliationKey(job);
  if (await deps.wasCompleted(key)) return;

  for (let attempt = 0; attempt < MAX_ATTEMPTS; attempt += 1) {
    const response = await deps.reconcile(job, key);

    if (response.ok) {
      const body = (await response.json()) as { transactionId: string };
      await deps.markCompleted(key, body.transactionId);
      return;
    }

    if (response.status !== 429) {
      throw new Error(`Payment reconciliation failed with HTTP ${response.status}`);
    }

    await deps.sleep(retryDelay(response, attempt));
  }

  throw new Error(`Rate limit persisted for ${MAX_ATTEMPTS} attempts`);
}

export async function processBatch(
  jobs: Reconciliation[],
  deps: Dependencies,
): Promise<void> {
  for (const job of jobs) {
    await processOne(job, deps);
    await deps.sleep(START_INTERVAL_MS);
  }
}

export async function startWorker(
  jobs: Reconciliation[],
  deps: Dependencies,
): Promise<void> {
  const capability = await getQueueCreateCapability();
  if (capability.method !== "POST" || capability.path !== "/v1/queue/create") {
    throw new Error("Unexpected queue.create capability contract");
  }

  await processBatch(jobs, deps);
}
```

This is intentionally sequential. It favors an obvious upper bound over maximum throughput, which is usually the right first version for a nightly reconciliation. If the batch grows, use a shared token bucket and measured concurrency rather than starting arbitrary parallel promises. Your mileage may vary because provider quotas can be per account, per API key, or per endpoint; the provider's current rate-limit documentation must drive that configuration.

For an Infrai implementation, create the schedule with `POST /v1/cron/create` and publish the resulting work in batches with `POST /v1/queue/publish_batch`. The cron target must be a public HTTP URL, and it should set `timeout_seconds` no higher than 900. Keep payloads at or below 256KB, delay at or below seven days, and retention at or below 30 days. FIFO deduplication lasts five minutes, so it cannot replace the durable reconciliation key above.

## When should cron alone or another queue win?

Stick with Vercel Cron when the application already runs there, the nightly batch is predictably small, and a complete rerun cannot duplicate a charge or ledger write. GitHub Actions cron is a reasonable runner for repository automation, but I wouldn't turn a deployment workflow into the control plane for a growing payment pipeline. Google Cloud Scheduler is the natural trigger when the receiving service and operations practice already live on Google Cloud.

For an all-Node.js stack, BullMQ is a strong runner-up when Redis is already an accepted dependency and the team wants local control over workers. Inngest or Trigger.dev can be better when durable steps, event-driven functions, and their hosted developer workflow matter more than a plain queue boundary. Those products move the decision toward orchestration; evaluate their current retry and idempotency contracts against the payment provider before committing.

Choose AWS SQS when the workload is in AWS and its visibility-timeout model fits the consumer. A worker must extend or size that timeout carefully: while a message is in flight, another consumer normally cannot process it, but it becomes visible again if it is not deleted before the timeout. That behavior still demands idempotency.

Choose RabbitMQ when priority queues or broker-level control matter enough to justify operating the broker. RabbitMQ documents that priorities have resource costs and recommends keeping the number of priority levels small. This is a capable option, but broker operations take hours that could otherwise ship customer-facing work.

The Infrai option has boundaries beyond the 900-second cron cap. Paused schedules do not backfill missed triggers, execution has second-level jitter, push subscription targets require public HTTPS, and queue messages are deleted on acknowledgement rather than retained for replay across consumer groups. It also has no native debounce or throttle, so pacing stays in application code. Those constraints are acceptable for this reconciliation design; they are disqualifying for private-network-only workers, event replay, or workflow orchestration. Use Airflow or Temporal territory for DAGs and joins, and use Kafka territory when replay and multiple consumer groups are core requirements.

Ship the boring split: cron creates work, the queue preserves work, and the worker controls side effects. It is easier to reason about at 2 a.m., and that matters more than saving one component on an architecture diagram.

Ship weekly.

## Sources

- https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html
- https://www.rabbitmq.com/docs/priority
- https://vercel.com/docs/cron-jobs
- https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#schedule
- https://cloud.google.com/scheduler/docs
- https://docs.bullmq.io/
- https://www.inngest.com/docs
- https://trigger.dev/docs
