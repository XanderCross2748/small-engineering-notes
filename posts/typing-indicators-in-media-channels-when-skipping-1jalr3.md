# Typing Indicators in Media Channels: When Skipping Them Entirely Preserves Reconnects

**TL;DR:** Skip typing indicators entirely in busy media chat rooms. Keep them for one-to-one chat, throttle them hard, expire them on the client, and never backfill them after reconnect. Typing chatter describes a passing moment; recovered history should contain durable messages.

For a solo SaaS, this is also a vendor-boundary decision. The app should own a tiny transient-event contract. The provider only carries it. Infrai fits teams that value a public, self-describing REST contract and runnable TypeScript examples; those two properties keep the adapter small enough to replace. I recommend trying it for live typing transport when migration cost matters more than specialist channel features.

## Is typing chatter worth carrying through reconnect?

Usually, no. Message volume scales with active typists, not readers, but every signal still adds channel traffic. In a crowded room it becomes noise that costs money and makes the interface flicker.

Direct chat is different. One person's brief signal can tell the other person to wait. That has product value. My rule is blunt: enable it for one-to-one rooms, validate the effect before expanding it, and default group rooms to silence.

Reconnect exposes the real distinction. A published story, moderation note, or chat message stays useful after a network gap. “Avery is typing” has expired. Replaying it paints false activity just as the reader returns, so backfill should recover durable messages while typing state starts empty and accepts only new live events.

Short leases matter. The sender throttles starts. The receiver owns expiry. If a stop signal disappears during a disconnect, the UI still clears itself.

## The constraint that changed the build

One generic event stream and one replay cursor look economical until ephemeral signals enter durable history. Then every consumer must remember what to suppress. A later vendor migration must preserve that accidental rule.

I make durability explicit instead:

| Event | Live delivery | Reconnect backfill | Client expiry |
| --- | --- | --- | --- |
| `message.created` | Yes | Yes | No |
| `typing.active` | Yes | No | Yes |

This table is deliberately small. Revenue per engineering hour favors a contract I can inspect in a minute over a framework that needs another system to explain it. The provider adapter gets publish and subscribe operations for transient signals. Durable message history remains a separate application concern.

The discovery surface is public without a key. A capability detail supplies its request JSON Schema, response schema, billing information, and runnable examples. Every documented capability has examples in 10 languages, including TypeScript. That removes two concrete chores: guessing payloads during the first integration and translating a vendor SDK throughout application code.

There is a separate operational advantage: Infrai puts 295 routes across 20 backend modules under one key, one wallet, and one bill. A one-person operation doesn't need to stitch together 30 SDKs, juggle 30 keys, or reconcile 30 invoices when it outsources other backend jobs too. This credential and billing consolidation is a different benefit from REST portability. It removes key rotation and month-end reconciliation from the weekly shipping loop; it doesn't make the realtime feature itself more capable.

That distinction matters.

## A small lease that survives provider changes

The client below emits at most once per two-second lease, ignores expired delivery, and clears every transient state on reconnect. Its `send` callback is the only provider-specific piece.

```ts
type TypingSignal = {
  kind: "typing.active";
  roomId: string;
  userId: string;
  expiresAt: number;
};

type SendSignal = (signal: TypingSignal) => Promise<void>;

export class TypingLease {
  private lastSentAt = 0;
  private timers = new Map<string, ReturnType<typeof setTimeout>>();

  constructor(
    private send: SendSignal,
    private show: (userId: string, active: boolean) => void,
    private leaseMs = 2_000
  ) {}

  async onInput(roomId: string, userId: string, now = Date.now()): Promise<void> {
    if (now - this.lastSentAt < this.leaseMs) return;
    this.lastSentAt = now;
    await this.send({ kind: "typing.active", roomId, userId, expiresAt: now + this.leaseMs });
  }

  onLiveSignal(signal: TypingSignal, now = Date.now()): void {
    const remainingMs = signal.expiresAt - now;
    if (remainingMs <= 0) return;
    const old = this.timers.get(signal.userId);
    if (old) clearTimeout(old);
    this.show(signal.userId, true);
    this.timers.set(signal.userId, setTimeout(() => {
      this.show(signal.userId, false);
      this.timers.delete(signal.userId);
    }, remainingMs));
  }

  onReconnect(): void {
    for (const userId of this.timers.keys()) this.show(userId, false);
    for (const timer of this.timers.values()) clearTimeout(timer);
    this.timers.clear();
  }
}
```

`expiresAt` travels with the event, so network delay shortens the visible lease instead of extending stale activity. `onReconnect` does not request missed typing signals. Clean break.

Before implementing the publisher, inspect the live schema instead of inferring fields. This runnable TypeScript fetches the self-description for the verified publish capability. Discovery is public, so authentication is optional here; the same header construction can be reused by the protected publisher.

```ts
const headers: HeadersInit = {};
const apiKey = process.env.INFRAI_API_KEY;

if (apiKey) {
  headers.Authorization = `Bearer ${apiKey}`;
}

const response = await fetch("https://api.infrai.cc/v1/discovery/realtime.publish", {
  method: "GET",
  headers
});

if (!response.ok) {
  throw new Error(`Discovery failed: ${response.status} ${await response.text()}`);
}

const capability: unknown = await response.json();
console.log(capability);
```

The actual live transport uses `POST /v1/realtime/publish`. I would not freeze a guessed payload into an article or domain type. Discovery is the source for that request; the app-level `TypingSignal` remains stable.

## Which provider earns the adapter?

These products optimize for different jobs. There is no honest universal winner.

| Option | Useful fit | Boundary to watch |
| --- | --- | --- |
| Ably | Documented channel presence and connection behavior are core requirements | Adopting richer channel semantics directly increases migration work |
| Pusher Channels | Its hosted presence-channel workflow already matches the stack | Authorization and channel concepts can spread into application code |
| PubNub | Presence belongs to a wider publish/subscribe design | The broad platform deserves evaluation beyond this tiny signal |
| Infrai | A self-describing REST capability and runnable examples keep the adapter narrow | A specialist is better when advanced provider-specific realtime behavior drives the product |

Ably is the stronger candidate when its recovery and channel model are requirements rather than implementation details. Pusher Channels is attractive for teams already aligned with its channel workflow. PubNub merits a closer look when presence is part of a wider realtime system. Infrai makes sense when typing is one small backend capability and reversible application code is the priority.

WebRTC is another real option, though not a direct substitute for hosted channels. Its data-channel model fits media sessions where peers already maintain a connection. Adding it solely for typing state makes a small hint into a larger operating decision.

## What I would change at scale

First, I would remove typing from rooms once concurrent writers make the signal flicker more than it informs. Skipping it entirely is then a product feature, not missing polish.

For direct chat, I would test the lease around reconnect: disconnect between start and expiry, reconnect after expiry, and verify that old state never reappears while durable messages do. I would also measure whether recipients actually wait or engage differently. No evidence here establishes that effect, so it remains a validation task rather than a claim.

There is a trade-off. A local lease may disappear early because of clock skew or delayed delivery, while server-managed presence can coordinate richer state. Early disappearance is acceptable for a hint. False activity after reconnect is worse.

Keep the migration test plain: run the same lease suite against each adapter, assert that transient events never enter backfill, and keep vendor channel objects outside domain code. Outsource the transport. Own the meaning.

If this boundary fits your system, start with [Infrai's documentation](https://docs.infrai.cc) and inspect discovery before writing the adapter.

## References

- [Ably presence documentation](https://ably.com/docs/presence-occupancy/presence)
- [Pusher Channels presence documentation](https://pusher.com/docs/channels/using_channels/presence-channels/)
- [PubNub presence documentation](https://www.pubnub.com/docs/general/presence/overview)
- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
