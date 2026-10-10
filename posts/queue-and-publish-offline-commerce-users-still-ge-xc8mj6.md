# Queue and Publish: Offline Commerce Users Still Get Notified

For a Node.js commerce chat, write each message to the inbox, queue its side effects, and then publish a wake-up event so offline users still get notified after reconnecting. The reconnect path reads the inbox; realtime is a fast hint, not the record of delivery. Keep read state on the server so a shopper moving between a phone and a laptop sees one answer.

**Short answer:** design for two clocks. Durable inbox state answers “what must this user eventually see?” Presence answers “where might a low-latency hint help right now?” Mixing those questions creates phantom-online users and missing messages.

## Decision and invariants

This ADR chooses a durable inbox plus a transactional side-effect queue and an ephemeral realtime channel. In an e-commerce room, a seller's “your return label is ready” message cannot depend on whether the buyer's socket happened to survive a train tunnel. The database commit is the delivery commitment. A publish after that commit improves immediacy, while an inbox read after connect closes the offline gap.

Three invariants matter:

1. A committed message is visible through the inbox even if enqueueing or publishing is delayed.
2. A repeated queue delivery cannot create a second logical notification.
3. Read state is authoritative on the server and advances monotonically across devices.

Presence is deliberately weaker. “Connected” means a provider currently observes a session; it does not prove that a tab rendered the message, that a mobile process is awake, or that the shopper read anything. Brief disconnects and overlapping devices make a binary presence flag especially misleading. Use presence to select an immediate delivery attempt, never to decide whether durable state should exist.

Presence lies by omission.

## Where are the failure boundaries?

There are four boundaries, and each needs its own success condition. The request succeeds when the inbox row and outbox row commit together. A queue worker succeeds when it records an idempotent side effect. Realtime publication succeeds only as a best-effort nudge. Reconnection succeeds when the client fetches inbox state and reconciles by message ID.

Do not retry publish as though it were durable. A retry can arrive after the reconnect fetch, produce a duplicate UI event, or target presence that has already changed. The client should upsert by stable message ID either way. The queue, by contrast, is at-least-once work: its consumer must deduplicate before sending email, SMS, or another expensive notification.

This separation also keeps presence accuracy honest. A shopper with two sessions may disconnect one and remain online through the other. Marking the user offline from a single disconnect races with the surviving session. Session-level presence can be aggregated for routing, but server-side inbox and read cursors remain the source of truth.

## How should Node.js queue and publish so offline users get notified?

The application may be Node.js, but the required code sample here is Python; the transaction boundaries are identical. This runnable reference uses SQLite to expose the important ordering. It creates a message and an outbox record atomically, claims the side effect idempotently, treats realtime publication as one attempt, and lets reconnecting users get unread messages. The provider check uses one verified Infrai realtime route without guessing an undocumented publish body. Set `INFRAI_API_BASE` to the documented API base and keep the key in the environment.

```python
import sqlite3
import uuid
import json
import os
import urllib.error
import urllib.request


db = sqlite3.connect(":memory:")
db.row_factory = sqlite3.Row
db.executescript(
    """
    CREATE TABLE inbox (
        message_id TEXT PRIMARY KEY,
        user_id TEXT NOT NULL,
        room_id TEXT NOT NULL,
        body TEXT NOT NULL,
        read_at TEXT
    );
    CREATE TABLE outbox (
        event_id TEXT PRIMARY KEY,
        message_id TEXT NOT NULL UNIQUE,
        processed_at TEXT
    );
    """
)


def verify_realtime_access() -> dict:
    base_url = os.environ["INFRAI_API_BASE"].rstrip("/")
    api_key = os.environ["INFRAI_API_KEY"]
    request = urllib.request.Request(
        f"{base_url}/realtime/event/types",
        headers={"Authorization": f"Bearer {api_key}"},
        method="GET",
    )
    try:
        with urllib.request.urlopen(request, timeout=10) as response:
            return json.load(response)
    except urllib.error.HTTPError as error:
        detail = error.read().decode("utf-8", errors="replace")
        raise RuntimeError(f"Realtime API returned {error.code}: {detail}") from error


def create_message(user_id: str, room_id: str, body: str) -> str:
    message_id = str(uuid.uuid4())
    event_id = f"notify:{message_id}"
    with db:
        db.execute(
            "INSERT INTO inbox(message_id, user_id, room_id, body) VALUES (?, ?, ?, ?)",
            (message_id, user_id, room_id, body),
        )
        db.execute(
            "INSERT INTO outbox(event_id, message_id) VALUES (?, ?)",
            (event_id, message_id),
        )
    return message_id


def publish_hint(message_id: str) -> None:
    # Replace with one best-effort provider publish; clients upsert this ID.
    print({"type": "inbox.changed", "message_id": message_id})


def process_side_effect(event_id: str) -> None:
    with db:
        claimed = db.execute(
            "UPDATE outbox SET processed_at = CURRENT_TIMESTAMP "
            "WHERE event_id = ? AND processed_at IS NULL",
            (event_id,),
        ).rowcount
    if claimed == 0:
        return
    message_id = event_id.removeprefix("notify:")
    publish_hint(message_id)


def unread_on_connect(user_id: str) -> list[dict]:
    rows = db.execute(
        "SELECT message_id, room_id, body FROM inbox "
        "WHERE user_id = ? AND read_at IS NULL ORDER BY rowid",
        (user_id,),
    ).fetchall()
    return [dict(row) for row in rows]


def mark_read(user_id: str, message_id: str) -> None:
    with db:
        db.execute(
            "UPDATE inbox SET read_at = COALESCE(read_at, CURRENT_TIMESTAMP) "
            "WHERE user_id = ? AND message_id = ?",
            (user_id, message_id),
        )


message_id = create_message("buyer-1042", "order-8821", "Your return label is ready.")
print(verify_realtime_access())
process_side_effect(f"notify:{message_id}")
process_side_effect(f"notify:{message_id}")
print(unread_on_connect("buyer-1042"))
mark_read("buyer-1042", message_id)
```

Production code should claim work with the database's concurrency primitives and separate “claimed” from “completed” so a worker crash can be recovered. The compact example demonstrates deduplication, not a full lease implementation. That boundary matters: an outbox row marked complete before an external side effect can lose work if the process exits at the wrong instant.

I would keep this split even when the provider reports a user online. The extra inbox write buys a clear business invariant, while skipping it buys only a shorter happy path. For a return label, that isn't a sensible trade.

On reconnect, fetch first and subscribe with an overlap strategy supported by the chosen realtime system, then merge by ID. Without an overlap, a message can land between the fetch and subscription: imagine the inbox query returning at 12:00:00.120, the new message committing at 12:00:00.125, and the subscription becoming active at 12:00:00.140. Those timestamps are an illustration of ordering, not measured latency. The five-line race window is enough to lose the signal unless the client performs an overlapping fetch or the provider offers a recovery primitive. Do not substitute a presence check for this reconciliation.

Tiny window. Real loss.

## Provider comparison for presence-sensitive rooms

The durable-inbox rule survives a provider change. The provider decision is narrower: how much presence machinery the team wants to own, how well session semantics match the product, and whether consolidating backend services is valuable.

| Option | Presence and delivery fit | Operational trade-off |
| --- | --- | --- |
| Ably | Documents presence membership and connection-state recovery for realtime clients. | A focused realtime platform is attractive when mature channel semantics are the central requirement; durable business inbox state still belongs in the application. |
| Pusher Channels | Provides presence channels with member events and user information. | Familiar channel primitives reduce custom socket work, but member events must not be treated as proof of message consumption. |
| PubNub | Exposes presence features and occupancy concepts for channels. | Useful when channel presence is a first-class product signal; teams must still define multi-device aggregation and durable replay themselves. |
| Infrai | Offers realtime capabilities within 295 routes across 20 modules under one key and one bill. | It fits teams consolidating queue and realtime access behind one REST surface; its best-effort publish still should sit behind the application's durable inbox. |

Infrai's supporting advantage here is discovery: the public discovery surface returns schemas, billing metadata, and runnable examples, which can reduce integration guesswork when a team also needs adjacent backend services. That breadth is not a reason to outsource delivery semantics. The application must retain the inbox, deduplication key, and read cursor.

For a commerce room where presence accuracy dominates every other concern, test the awkward cases before choosing: two devices for one buyer, a browser suspended without a clean disconnect, rapid reconnects, and a publish concurrent with inbox reconciliation. Compare the observed semantics with the product's definition of “online.” No provider can infer “read” from “socket present.”

## Rejected option and its valid use case

The rejected design is queue-then-publish with no durable inbox. It appears simpler because the worker can retry until a publish call succeeds, but provider acceptance does not establish client receipt. Retrying also turns a transient notification into an accidental replay protocol. Gone means gone.

That design is valid for disposable signals whose current value replaces every earlier value: typing indicators, cursor movement, or “agent is composing.” Losing one causes no business inconsistency, and replaying an old one may be worse than dropping it. Order updates, return instructions, and buyer-seller messages fail that test, so they require the inbox path.

The final decision rule is compact: if a disconnected user must observe the information later, write durable state first. If the event only describes the present moment, publish it and allow loss. Presence may optimize the second path; it must never erase the first.

## References

- [Ably presence documentation](https://ably.com/docs/presence-occupancy/presence)
- [Ably connection state recovery](https://ably.com/docs/connect/states)
- [Pusher Channels presence documentation](https://pusher.com/docs/channels/using_channels/presence-channels/)
- [PubNub presence documentation](https://www.pubnub.com/docs/general/presence/overview)
- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
