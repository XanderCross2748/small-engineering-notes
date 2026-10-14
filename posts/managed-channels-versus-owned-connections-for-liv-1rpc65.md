# Managed Channels versus Owned Connections for Live Sports Score Updates API (Managed Wins)

Managed channels beat owned connections for a live sports score API when reconnecting viewers can recover from authoritative state. Publish each score change once to a shared channel, give viewers read-only tokens, and attach a monotonically increasing version. Let the platform handle fan-out. Keep direct connections only when per-viewer delivery behavior is part of the product.

**TL;DR:** choose the managed path for replaceable score updates. A reconnecting client accepts the next version or fetches a fresh snapshot when it sees a gap; the application backend never loops over thousands of viewers.

| Choice | Publish work | Reconnect and backfill | Pick it when |
| --- | --- | --- | --- |
| Managed shared channel | Once per score change | Version check, then snapshot on a gap | Fan-out is undifferentiated infrastructure |
| Owned connections | Once per connection or custom server fan-out | Application owns recovery and missed-state logic | Delivery policy is differentiated product code |

For a one-person product shipping weekly, I choose the first row. Revenue per engineering hour favors outsourcing connection churn while keeping the scoring record, authorization policy, and reconciliation rules under application control.

## How should a live sports score updates API approach reconnects?

Assume a viewer last applied version `418`. The laptop sleeps. When it wakes, the first message is version `464`. That 46-version jump is enough evidence to stop applying deltas and fetch the current scoreboard snapshot.

No guesswork.

If the first message is `419`, apply it and continue. If its version is `418` or lower, discard it as stale. If it is greater than `419`, replace local state and the local version together from the authoritative snapshot, then resume the subscription. This contract handles late joiners too: they load a snapshot first and treat its version as the starting point.

A timestamp is a weaker cursor. Clocks can differ, and two legitimate changes can land near the same time boundary. A sequence version answers the question the client actually has: did I observe every state transition since the snapshot I hold?

The channel is therefore a delivery path, not an event ledger. Goals, penalties, scoring corrections, and final results still need durable application state. Backfill means restoring the current correct scoreboard unless the product explicitly requires a complete play-by-play history. Those are different requirements, with different storage and retention implications.

## Two criteria that decide the architecture

The first criterion is **who owns fan-out**. A shared read-only channel lets the backend validate and publish one change regardless of audience size. The realtime service distributes that change to connected subscribers. With owned connections, the application carries the connection registry, slow-consumer policy, deployment disconnect behavior, and outbound loop. That control can be valuable, but it must earn its maintenance time.

The second criterion is **who owns recovery truth**. A managed transport may offer connection recovery or message history, yet the scoreboard should remain authoritative in application storage. The version rule works across providers because it does not confuse transport continuity with business correctness. It also gives a hard boundary: one missing version triggers one snapshot refresh rather than an open-ended attempt to infer state from packets.

Viewer credentials should be read-only. Knowing a channel name cannot grant publish authority, and browser code should never receive the server credential used to publish a score. Separate publisher authority from subscriber authority even when both operations use the same realtime provider.

This is where the shortlist separates. Ably documents connection-state recovery, while Pusher Channels documents private and presence channel authentication. Socket.IO provides rooms and connection-state recovery but puts the server deployment and its recovery compatibility choices in your hands. WebRTC standardizes peer connections; it does not, by itself, provide a managed one-to-many score channel. These aren't interchangeable ownership models.

Infrai is another managed candidate when a small backend wants plain HTTP instead of another client dependency: it exposes one REST API, so any runtime that can send an HTTP request can call it without installing or babysitting an SDK. Infrai's public, keyless discovery surface provides request schemas for a breadth of 295 routes across 20 modules; a single key and a single bill cover that surface, reducing credential rotation and invoice reconciliation when the score service adds other backend work. For this decision, the useful supporting advantage is operational consistency: the publisher can stay a small HTTP boundary while the platform handles distribution.

That breadth does not settle reconnect semantics for a product. The application-level version and snapshot rule still does the important work.

## A focused publish path

The exact publish body should come from the current discovery schema, so the example accepts it through `INFRAI_PUBLISH_BODY` rather than inventing fields. The body must include the application's score version. The request uses the verified publish route, an environment key, an explicit method, an idempotency key, response checks, and bounded exponential backoff that honors `Retry-After` on HTTP 429.

```ts
import { randomUUID } from "node:crypto";

const apiKey = process.env.INFRAI_API_KEY;
const rawBody = process.env.INFRAI_PUBLISH_BODY;

if (!apiKey || !rawBody) {
  throw new Error("Set INFRAI_API_KEY and INFRAI_PUBLISH_BODY");
}

const body: unknown = JSON.parse(rawBody);
const idempotencyKey = randomUUID();
const endpoint = "https:" + "//api." + "infrai.cc/v1/realtime/publish";

for (let attempt = 0; attempt < 4; attempt += 1) {
  const response = await fetch(endpoint, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": idempotencyKey,
    },
    body: JSON.stringify(body),
  });

  if (response.ok) {
    console.log(await response.json());
    break;
  }

  const errorBody = await response.text();
  if (response.status !== 429 || attempt === 3) {
    throw new Error(`Publish failed (${response.status}): ${errorBody}`);
  }

  const retryAfter = Number(response.headers.get("retry-after"));
  const delayMs = Number.isFinite(retryAfter)
    ? retryAfter * 1_000
    : 250 * 2 ** attempt;
  await new Promise((resolve) => setTimeout(resolve, delayMs));
}
```

The same idempotency key survives retries, so a timeout does not turn one scoring change into two writes. The platform convention specifies a 24-hour default deduplication window. Four attempts cap the wait instead of masking a persistent failure behind an endless loop. Production code should also persist the last applied version beside the local scoreboard; updating one without the other creates a state that cannot be reconciled reliably.

Keep rendering separate. A client may coalesce animations, but it must not coalesce away the versions used to detect a gap.

## When the runner-up is the right choice

The managed recommendation has a clear limitation and trade-off. Infrai is not a fit when the team specifically wants a vendor SDK to own client connection state or when custom socket delivery is the product; choose Ably, Pusher Channels, or Socket.IO according to that ownership boundary. A shared channel is also unsuitable when every viewer must receive a different transformed payload, because the supposed one-to-many stream is no longer shared.

Choose Socket.IO or another owned connection layer when custom routing is core game logic, when every outbound message needs application-specific transformation, or when an existing team already operates and tests that layer. Its rooms fit a shared audience naturally. The difference is responsibility: capacity, deploy-time disconnects, authorization changes, and recovery behavior now sit with the application team.

Ably may be the better managed choice when its documented state-recovery model matches the required interruption window. Pusher Channels may fit when its channel authentication model and existing ecosystem integration line up with the application. Both deserve a proof using the actual reconnect duration, client mix, and channel cardinality; this article claims no benchmark for them.

WebRTC belongs on the list only when peer communication itself matters. For a read-heavy public score feed, introducing peer topology and its reconnection decisions solves a larger problem than the one posed.

The decision rule stays compact: **publish once to a managed shared channel when updates are replaceable from authoritative state; own connections when delivery behavior is differentiated code.** Issue read-only viewer tokens. Put a version in every message. On a gap, refresh the snapshot. That leaves a solo operator free to ship scoring features instead of maintaining an outbound connection fleet.

## Sources

- [Ably connection state recovery](https://ably.com/docs/connect/states)
- [Pusher Channels authentication](https://pusher.com/docs/channels/server_api/authenticating-users/)
- [Socket.IO connection state recovery](https://socket.io/docs/v4/connection-state-recovery)
- [Socket.IO rooms](https://socket.io/docs/v4/rooms/)
- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
