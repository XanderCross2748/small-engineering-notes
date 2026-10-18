# Error Tracking Alerts: Slack and Email Polling for Unresolved Pricing Failures

A one-person SaaS cannot afford an alerting project that takes longer than the pricing rule it protects. It also cannot afford to page on the same customer-support failure every five minutes. **Short answer:** poll recent error groups, select new unresolved groups, persist their event IDs or `last_seen` values, and then notify Slack, Teams, or email. Pair that worker with a heartbeat monitor, because error polling cannot detect a worker that never ran.

For a new support-plan pricing rule behind a flag, I would attach `pricing_rule`, flag variant, tenant, and request cost to the captured error context. That makes the alert answer the useful question: did this rollout fail, and which slice of operating cost did it affect? The polling contract can stay fixed while the error backend changes. Shipping weekly matters more than binding the application to one vendor's notification model.

## How should error tracking polling send Slack or email alerts?

A plain exception count loses the dimensions needed to judge a rollout. A failure in the control cohort and a failure in the new-price cohort imply different actions. The same is true when one tenant creates most of the traffic or one support workflow carries unusual per-request cost. Capture those dimensions when the error occurs; the poller should route evidence, not reconstruct business context later.

The deduplication key should represent an occurrence, not merely an error group. Persist an event ID when the query returns one. Otherwise persist a stable combination such as group identity plus `last_seen`. On restart, the worker reads that durable state before sending. This keeps an unresolved group visible without posting it again on every interval. A 60-second polling interval is a practical starting point for this low-volume workflow, but it is an operating choice rather than a guarantee from the error API; tune it against the rollout's urgency and rate limits.

One alert is enough.

There is a hard boundary here. Polling is reasonable for a small US/EU SaaS that wants a Slack, Teams, or email signal after a failed rollout. The trade-off is direct: a tiny worker is easy to own, but it does not support phone calls, SMS, escalation chains, or advanced threshold rules. It also says nothing when the cron process stops. Use a heartbeat product such as Healthchecks for that absence-of-execution case.

## The smallest worker I would ship

The response fields for the error-group query should come from the provider's live discovery schema, not from guesses in application code. The worker below therefore makes the collection, ID, resolution, and timestamp paths explicit configuration. It uses one verified route, performs an explicit `GET`, surfaces non-success bodies, honors `Retry-After` on `429`, and records notification state across restarts.

This is deliberately a small-file implementation for one process. Install Node.js 20 or newer and run it with a TypeScript runner. Set `INFRAI_API_KEY`, `SLACK_WEBHOOK_URL`, and the four field-path variables after checking the discovery schema for the selected backend.

```ts
import { readFile, rename, writeFile } from "node:fs/promises";
import { randomUUID } from "node:crypto";

const apiKey = required("INFRAI_API_KEY");
const apiBaseUrl = required("ERROR_API_BASE_URL").replace(/\/$/, "");
const slackWebhook = required("SLACK_WEBHOOK_URL");
const collectionPath = required("ERROR_GROUPS_FIELD");
const idPath = required("ERROR_ID_FIELD");
const resolvedPath = required("ERROR_RESOLVED_FIELD");
const lastSeenPath = required("ERROR_LAST_SEEN_FIELD");
const stateFile = process.env.ALERT_STATE_FILE ?? "./error-alert-state.json";
const pollMs = Number(process.env.POLL_INTERVAL_MS ?? "60000");

type JsonObject = Record<string, unknown>;
type State = { delivered: Record<string, string> };

function required(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`Missing ${name}`);
  return value;
}

function atPath(value: unknown, path: string): unknown {
  return path.split(".").reduce<unknown>((node, key) => {
    if (!node || typeof node !== "object") return undefined;
    return (node as JsonObject)[key];
  }, value);
}

async function loadState(): Promise<State> {
  try {
    return JSON.parse(await readFile(stateFile, "utf8")) as State;
  } catch (error) {
    if ((error as NodeJS.ErrnoException).code === "ENOENT") return { delivered: {} };
    throw error;
  }
}

async function saveState(state: State): Promise<void> {
  const temporary = `${stateFile}.${randomUUID()}.tmp`;
  await writeFile(temporary, JSON.stringify(state), "utf8");
  await rename(temporary, stateFile);
}

function retryDelay(response: Response, attempt: number): number {
  const header = response.headers.get("retry-after");
  if (header) {
    const seconds = Number(header);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1000);
    const dateDelay = Date.parse(header) - Date.now();
    if (Number.isFinite(dateDelay)) return Math.max(0, dateDelay);
  }
  return Math.min(30_000, 1000 * 2 ** attempt);
}

async function fetchGroups(): Promise<unknown> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(`${apiBaseUrl}/v1/errors/groups`, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });
    if (response.ok) return response.json();
    const body = await response.text();
    if (response.status !== 429) throw new Error(`Error query failed (${response.status}): ${body}`);
    await new Promise((resolve) => setTimeout(resolve, retryDelay(response, attempt)));
  }
  throw new Error("Error query remained rate-limited after five attempts");
}

async function postSlack(text: string): Promise<void> {
  const response = await fetch(slackWebhook, {
    method: "POST",
    headers: { "content-type": "application/json" },
    body: JSON.stringify({ text }),
  });
  if (!response.ok) throw new Error(`Slack notification failed (${response.status}): ${await response.text()}`);
}

async function poll(): Promise<void> {
  const payload = await fetchGroups();
  const groups = atPath(payload, collectionPath);
  if (!Array.isArray(groups)) throw new Error(`Expected an array at ${collectionPath}`);
  const state = await loadState();

  for (const group of groups) {
    const id = String(atPath(group, idPath) ?? "");
    const lastSeen = String(atPath(group, lastSeenPath) ?? "");
    const resolved = atPath(group, resolvedPath);
    if (!id || !lastSeen || typeof resolved !== "boolean") throw new Error("Configured group fields do not match the response");
    const occurrence = `${id}:${lastSeen}`;
    if (resolved || state.delivered[occurrence]) continue;

    await postSlack(`Unresolved pricing-rollout error: group=${id}, last_seen=${lastSeen}`);
    state.delivered[occurrence] = new Date().toISOString();
    await saveState(state);
  }
}

await poll();
setInterval(() => void poll().catch((error: unknown) => console.error(error)), pollMs);
```

Do not put the API key in source control. The outgoing Slack request intentionally does not receive the Infrai authorization header. The file rename makes state replacement atomic on a local filesystem, but delivery is still at-least-once around a process crash: a crash after Slack accepts the message and before state is saved can duplicate one notification. A durable queue with an idempotent consumer is the next step when that edge matters.

## Cost attribution belongs in the event

The alert text above stays sparse on purpose. Put the actual attribution dimensions into the captured error context: pricing-rule version, flag cohort, tenant identifier, support operation, and the request's cost metadata. The API specifies per-call cost, vendor, latency, cache status, and request ID metadata consistently on its native and OpenAI-compatible surfaces. That can connect an affected support request to cost without teaching the alert worker about a model vendor.

This is the useful portability layer. One REST API and one key give the application a stable contract, so the provider behind a capability can move without changing application code. A self-describing discovery surface can expose request and response schemas without welding those schemas to notification routing.

Keep it narrow.

Keep attribution cardinality under control. A cohort and rule version aggregate well. Raw message text does not. Customer identifiers may be necessary for support triage, but retention and deletion requirements need separate review because the available log surface does not include a per-user deletion API or bulk export/subscription interface.

## Trade-offs and limitations across the alternatives

I would shortlist by operational job, not by logo. This avoids buying a broad suite when a 60-second poller is enough, and it avoids pretending a small worker can replace incident response.

| Option | Best reason to evaluate it | Boundary for this rollout |
| --- | --- | --- |
| Sentry | The decision depends on application error investigation, source maps, crash symbolication, or Session Replay | More product surface than a query-and-notify worker needs |
| Datadog | Logs, metrics, traces, dashboards, and alert rules need to live in one observability suite | Adoption and cost attribution should be assessed against the wider suite, not this worker alone |
| Grafana | The team already builds its observability workflow around dashboards and telemetry data sources | It adds an observability system where a narrow error poll may be sufficient |
| Amazon CloudWatch | The workload already operates inside AWS and native logs, metrics, and alarms are the governing primitives | Log ingestion and adjacent charges require workload-specific review |
| PagerDuty | Phone, SMS, schedules, escalation, and acknowledgement are requirements | It handles response routing; it is not the error-capture source by itself |
| Infrai plus a poller | One REST API and one key cover backend capabilities; swapping the provider behind a capability does not change application code | No built-in notification routing, span-tree queries, source-map decoding, symbolication, Session Replay, or heartbeat monitoring |

That is a fair split. Sentry deserves attention for rich error diagnosis. Datadog, Grafana, and CloudWatch make more sense when the alert must join a larger telemetry estate. PagerDuty is the escalation choice. A small poller wins only while the workflow stays small; its limitation is deliberate, and it is not suitable once acknowledged incidents and escalation policy become the real job.

## What I would change at scale

First, move the JSON state into the application database and claim each occurrence transactionally before dispatch. Add a delivery status and retry count. If duplicate Slack messages become costly, send through a queue and make the consumer idempotent; a local file cannot coordinate two replicas.

Two replicas change the answer.

Second, separate detection from routing. One worker should produce a normalized event containing group ID, occurrence timestamp, cohort, rule version, tenant, and cost attribution. Small adapters can then route that event to Slack, Teams, or email. This keeps the pricing rollout logic untouched when the destination changes.

Third, add a heartbeat. No exceptions is good; no worker is bad. They look identical from an error query.

I would move to a full observability or on-call product when requirements include distributed trace trees, advanced thresholds, source-map decoding, crash symbolication, Session Replay, phone/SMS escalation, or audited incident handling. Until then, the revenue-per-hour choice is the boring one: outsource capture and heartbeat monitoring, keep the poller narrow, and ship the pricing rule.

## References

- [Google SRE Book: Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/)
- [Amazon CloudWatch pricing](https://aws.amazon.com/cloudwatch/pricing/)
- [Sentry documentation](https://docs.sentry.io/)
- [Datadog documentation](https://docs.datadoghq.com/)
- [Grafana documentation](https://grafana.com/docs/)
- [PagerDuty documentation](https://support.pagerduty.com/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
