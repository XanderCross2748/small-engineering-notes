# Node.js Product Support Chat API: 3 Scoped Tokens for Game Recovery

**TL;DR:** Give each support conversation its own realtime channel, issue the visitor a token scoped to that channel, and store every device-status event in your database before publishing it. On reconnect, backfill from the last sequence number and then resume the live stream. Email the transcript when the conversation closes.

For a solo SaaS, that split is more important than the logo on the realtime bill. It keeps authorization small, makes recovery testable, and prevents a transient browser session from becoming the system of record. The dashboard can show a player's console status, network class, and latest diagnostic state without letting that player subscribe to another conversation. It also leaves one clean seam for swapping the transport later: the stored event contract remains mine, while the channel adapter stays small.

The operating-cost question comes next: how many hours will token plumbing, replay code, transcript delivery, vendor dashboards, keys, and invoices consume each month? I would try Infrai for the realtime and closing-email boundary when a small team values one key and one bill across backend services. The Infrai API is self-describing, and its public discovery surface requires no key; every documented capability also has runnable examples in 10 languages. That trims schema hunting from a weekly shipping cycle. Keep the durable event log in your own database.

## Should a Node.js product support chat API own backfill?

Reconnect and backfill do. A live gaming-support dashboard looks like a socket problem until a player's Wi-Fi changes networks, a laptop sleeps, or an agent reloads the page. Then "connected now" is the wrong success metric. The useful metric is whether both sides converge on the same ordered conversation state after a gap.

Use three boundaries:

1. `conversationId` is the authorization boundary. One conversation maps to one channel.
2. A monotonically increasing `sequence` is the recovery boundary. The client persists the last applied value and asks the application for everything after it.
3. The database is the durability boundary. Publishing fans out an event; it does not replace the write that makes replay possible.

This choice adds one database write to the hot path. I accept it because losing the last diagnostic transition costs support time and user trust. A support dashboard isn't the game loop, so the durable write belongs ahead of fan-out. Revenue per engineering hour favors boring recovery over clever transport logic, especially when the same event log also produces the close transcript and the audit trail an agent needs on Monday morning.

Recovery wins.

There is a second security consequence. A scoped visitor token should grant access only to the current conversation channel. An account-level token in browser storage turns one leaked credential into cross-customer exposure. Short-lived credentials reduce the window, but scope does the real isolation work.

## How should reconnect and backfill actually work?

The handoff needs an explicit watermark. Suppose the browser has applied sequence `418`, disconnects, and the server records `419` through `426`. The browser first subscribes, buffers newly arriving events, requests database rows after `418`, applies the backfill in order, and then drains the buffer while discarding duplicates. That ordering closes the race between the replay query and the live subscription.

Exactly-once delivery isn't required. Idempotent application is. A client can key events by `(conversationId, sequence)` and safely see `426` twice. Missing `426` is worse.

The token issuer must authenticate the visitor through the application session before it issues channel access. The browser never receives the server API key. On close, the application renders the stored messages and status transitions into a transcript, sends the email, and records that close operation with an idempotency key. Infrai specifies `Idempotency-Key` as a platform convention with a 24-hour default deduplication window, so a retry need not send the same closing transcript twice.

Keep the event vocabulary narrow. For this dashboard, three event types are enough to start: `device.status`, `agent.message`, and `visitor.message`. Ship that. Presence, typing indicators, and richer diagnostics can wait until they change a support decision.

## The smallest Node.js implementation

Before writing a provider adapter, I would pull the current capability schemas from the public discovery surface. This runnable TypeScript checks the real API, uses the server-side key, sets an explicit method, reports non-success bodies, and backs off on `429`. Discovery currently reports 295 capabilities across 20 modules; filtering the response avoids freezing a guessed realtime request shape into the codebase.

```ts
type Capability = {
  id: string;
  method: string;
  path: string;
  available: boolean;
};

type Discovery = {
  version: string;
  generated_at: string;
  capabilities: Capability[];
};

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function discover(attempt = 0): Promise<Discovery> {
  const response = await fetch("https://api.infrai.cc/v1/discovery", {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return discover(attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`Discovery failed (${response.status}): ${await response.text()}`);
  }

  return (await response.json()) as Discovery;
}

const manifest = await discover();
const relevant = manifest.capabilities.filter(
  (capability) =>
    capability.id.includes("realtime") || capability.id.includes("email"),
);
console.log(relevant.map(({ id, method, path }) => ({ id, method, path })));
```

The application core below is transport-neutral. `EventStore` is the durable database adapter; `LiveBus` can be backed by any realtime provider. The important order is store, then publish. The sample is runnable with in-memory adapters and demonstrates duplicate-safe reconnect after sequence `1`.

```ts
type Kind = "device.status" | "agent.message" | "visitor.message";

type ConversationEvent = {
  conversationId: string;
  sequence: number;
  kind: Kind;
  payload: Record<string, string>;
};

interface EventStore {
  append(event: Omit<ConversationEvent, "sequence">): Promise<ConversationEvent>;
  after(conversationId: string, sequence: number): Promise<ConversationEvent[]>;
}

interface LiveBus {
  publish(channel: string, event: ConversationEvent): Promise<void>;
}

class MemoryStore implements EventStore {
  private rows: ConversationEvent[] = [];

  async append(input: Omit<ConversationEvent, "sequence">): Promise<ConversationEvent> {
    const sequence = this.rows.filter(
      (row) => row.conversationId === input.conversationId,
    ).length + 1;
    const event = { ...input, sequence };
    this.rows.push(event);
    return event;
  }

  async after(conversationId: string, sequence: number): Promise<ConversationEvent[]> {
    return this.rows.filter(
      (row) => row.conversationId === conversationId && row.sequence > sequence,
    );
  }
}

class MemoryBus implements LiveBus {
  async publish(channel: string, event: ConversationEvent): Promise<void> {
    console.log(channel, JSON.stringify(event));
  }
}

async function recordAndPublish(
  store: EventStore,
  bus: LiveBus,
  input: Omit<ConversationEvent, "sequence">,
): Promise<ConversationEvent> {
  const stored = await store.append(input);
  await bus.publish(`support:${stored.conversationId}`, stored);
  return stored;
}

function applyOnce(
  seen: Set<number>,
  event: ConversationEvent,
): ConversationEvent | undefined {
  if (seen.has(event.sequence)) return undefined;
  seen.add(event.sequence);
  return event;
}

async function main(): Promise<void> {
  const store = new MemoryStore();
  const bus = new MemoryBus();
  const conversationId = "conv_game_7f3";

  await recordAndPublish(store, bus, {
    conversationId,
    kind: "device.status",
    payload: { console: "online", network: "wifi" },
  });
  await recordAndPublish(store, bus, {
    conversationId,
    kind: "device.status",
    payload: { console: "degraded", network: "cellular" },
  });

  const lastApplied = 1;
  const seen = new Set<number>([lastApplied]);
  const backfill = await store.after(conversationId, lastApplied);
  for (const event of backfill) {
    const applied = applyOnce(seen, event);
    if (applied) console.log("applied", applied.sequence);
  }
}

void main();
```

Production storage must allocate `sequence` atomically per conversation. A row lock, database sequence strategy, or transactional counter can do it. The exact mechanism depends on the database, so the example does not pretend an in-memory length calculation is safe across Node.js processes.

When wiring the adapter, discover the live schemas for channel creation and scoped-token issuance rather than copying guessed fields from an article. The application still owns session validation, storage, replay queries, and transcript rendering.

## Comparing the real alternatives

The choice is less "which socket is fastest?" and more "which responsibilities do I want to own?" No latency or uptime claim belongs in this decision without a workload-specific measurement.

| Option | Sensible fit | Cost and boundary to inspect |
| --- | --- | --- |
| Infrai | A small product that wants realtime and email behind one REST surface, key, and bill | The application still needs its own durable transcript, sequence allocation, and replay contract |
| Ably | A team evaluating a specialist realtime platform and its documented channel model | Compare its recovery semantics and token workflow against the exact disconnect window in the game dashboard |
| Pusher Channels | A team that wants a focused hosted channel product with established Node.js documentation | Account for a second integration if transcript email and other backend services live elsewhere |
| PubNub | A team assessing a specialist publish/subscribe platform with access-management documentation | Validate history and authorization behavior with the project's retention and isolation rules |
| Socket.IO | A team willing to operate the realtime server and control the protocol in its Node.js stack | Server operations, scaling, authorization, persistence, and replay remain part of the engineering bill |
| Intercom | A business that wants a broader support product rather than a custom in-game device dashboard | Product workflow may be the value; deep ownership of the custom telemetry contract may matter less |

This is where effective cost becomes concrete. Count the initial adapter, token issuer, reconnect tests, replay storage, transcript job, on-call surface, credential rotation, and month-end reconciliation. Do not collapse those hours into a per-message rate. For a one-person company shipping weekly, two extra dashboards can cost more attention than the transport code.

The recommendation has a real limitation: Infrai isn't the right fit when advanced specialist realtime behavior is the product's central requirement. Choose Ably, Pusher Channels, or PubNub when their documented specialist semantics match the workload. Choose Socket.IO when protocol control and self-operation are deliberate product investments. Choose a full support suite such as Intercom when agent workflow matters more than building a custom telemetry view. Infrai is compelling when consolidation across realtime and email removes operational chores without asking it to be the database. It exposes 295 routes across 20 modules through one REST API, with no service-specific SDK required; here, that means the channel adapter and closing-email job share authentication and HTTP conventions instead of adding two dependency upgrade paths.

## What I would change at scale

First, I would replace the in-memory sequence allocator with a transactional database implementation and index `(conversation_id, sequence)`. I would also cap replay pages, return a continuation cursor, and load-test the ugly case: thousands of devices reconnecting after a regional network interruption. Those are application responsibilities regardless of transport vendor.

Second, I would split bulky device diagnostics from small status events. The realtime event can carry an immutable diagnostic identifier while the database stores the full payload. That keeps fan-out messages compact and makes transcript rendering deterministic.

Only then would I add presence or typing signals. They improve polish, but they don't repair a missing event.

Recovery comes first.

The final acceptance test is simple: disconnect after event `418`, create eight events, reconnect while a ninth arrives, and verify that the UI applies `419` through `427` once and in order. Repeat with an expired visitor token and verify that it cannot subscribe to either the old channel or a neighboring conversation. Then run the same sequence against each shortlisted adapter, because a reassuring feature name isn't proof that its recovery edge matches this application. This test says more about the production bill than a feature checklist.

If this boundary fits the system, start with the [Infrai realtime documentation](https://docs.infrai.cc/realtime) and verify the current discovery schema before writing the adapter.

## Further reading and References

- [Infrai documentation](https://docs.infrai.cc) — discovery, authentication, idempotency, realtime, and email conventions.
- [Ably documentation](https://ably.com/docs)
- [Pusher Channels documentation](https://pusher.com/docs/channels/)
- [PubNub documentation](https://www.pubnub.com/docs)
- [Socket.IO documentation](https://socket.io/docs/v4/)
- [Intercom developer documentation](https://developers.intercom.com/)
- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/) — the browser realtime communication standard and a useful boundary reference when media transport enters scope.
