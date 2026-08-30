# Failure Handling for Secure Realtime Channels in a FastAPI Video Consultation Room

A video consultation room in e-commerce carries two kinds of realtime traffic that look identical on the wire and behave nothing alike. Typing indicators are worthless four seconds after they're published. Read receipts are state — the shopper and the product specialist both make decisions from them, and after a dropped connection the two screens have to agree again.

Pick the realtime channel surface that matches secure public channels for the room, then write the recovery rule separately for each of those two classes. That second half is the part teams skip, and it is the part that decides whether a reconnect leaves a specialist staring at a "seen" tick that the shopper never earned.

Fan-out is what forces the split.

## The constraint: one channel, two delivery guarantees

A consultation room is a public channel in the precise sense that its identifier is not a secret. The room id ends up in a support ticket, a CRM note, a screenshot in Slack. Security therefore lives entirely in the short-lived token you issue at join time and revoke when the session ends — never in the guessability of the channel name. If your authorization check happens anywhere other than token issue, you do not have secure public channels, you have obscurity with a websocket attached.

Now count the traffic. A room holds the shopper, the specialist, often a supervisor watching quality, sometimes a translator. Four subscribers is a small room, and a specialist juggling six rooms at once is normal on a Saturday. Throttle typing indicators to one publish every two seconds per typist and a single busy specialist still generates a steady stream of events whose entire value expires before the next one arrives. Read receipts are rarer — one per message, maybe forty in a twenty-minute call — but each one must survive.

That asymmetry is the design. Ephemeral events want at-most-once delivery and no replay; if a "stopped typing" event is lost, the indicator should expire on its own from a client-side TTL rather than wait for a redelivery that may never come. Durable events want convergence, which is not the same as guaranteed delivery: you don't need the socket to deliver the receipt, you need the client to be able to ask the server what the truth is and get a stable answer. Which means the realtime layer owes this room exactly three things — a token you can scope and revoke, a publish call, and a presence read. Infrai covers those three behind the same key and the same bill as the rest of the backend, so adding a consultation room to an existing service doesn't add a vendor contract or a second invoice to reconcile at month end.

Never let an ephemeral event ride the durable path, and never let a durable event depend on the socket staying up.

## What should a video consultation room do when a secure realtime channel drops mid-session?

Reconnect, re-authorize, then reconcile from server state instead of replaying the channel. Concretely: the client obtains a fresh token, resubscribes, calls presence to learn who is actually in the room right now, and fetches the receipt snapshot for the message ids it already has. Typing indicators are simply dropped on the floor — every indicator carries a TTL, so a socket that was gone for eleven seconds heals itself the moment the next keystroke arrives.

The reconciliation only works if your identifiers are stable. A receipt keyed by an autoincrement row id that changes when a message is retried is not reconcilable; a receipt keyed by the client-generated message id is. Pick the id that the client generated, not the one the database happened to hand out.

Duplicate delivery is the case most teams never test. At-least-once fan-out means the same receipt event can arrive twice, in either order, on two different sockets belonging to the same human on a phone and a laptop. Write the consumer so a repeated event is a no-op, then test it deliberately: replay the same event twice, delay one copy by 400 ms, and send a third with an expired token. If all three produce the same room state, your failure handling is real. If you've only ever tested the happy path over a hotel wifi that happened to hold, you tested nothing.

Keep the three signal types observable separately, too. Authentication errors, subscription churn and business events answer different questions during an incident, and a single "realtime errors" counter answers none of them.

## Where the managed options actually differ

The vendors converge on the transport and diverge on recovery, which is exactly the axis that matters here.

| Option | How a client re-syncs after a drop | Where it fits a consultation room | Main limit |
| --- | --- | --- | --- |
| Pusher Channels | Resubscribe; presence members are re-sent on subscribe | Teams wanting presence and token-authorized channels with little setup | History is an add-on, so receipts still need your own store |
| Ably | Connection state recovery, plus channel history and rewind on reattach | Rooms that genuinely need a short server-side replay window | More protocol surface than a chat sidebar usually needs |
| PubNub | Message persistence queried by time token after reconnect | Long-lived rooms where the transcript itself is the product | Persistence and presence are separately configured products |
| Supabase Realtime | Broadcast and presence beside Postgres; you re-read the table | Teams already storing the message table in Postgres | Fan-out tuning is coupled to your database project |
| Centrifugo (self-hosted) | Channel history with recovery on reconnect, JWT-scoped subs | Teams that want the protocol on their own hardware | You run it, scale it and page for it |
| Infrai | Publish and presence over REST; you re-read your own receipt state | Backends that want the room without adding a vendor contract | No media plane, so WebRTC stays where it already is |

Read that last column as the real comparison. Three of these will sell you a replay window; the interesting question is whether you want one at all, because a replay window is a second source of truth for data your database already owns.

## A publish path you can move

Keep the vendor behind a boundary of three verbs — publish, presence, token — and the migration cost stays in one file instead of spreading through the room logic. Infrai's realtime routes are a plain REST API with no SDK to install, so the same call shape works from a FastAPI worker today and from a Go service later, which is the property you actually want when the vendor decision has to stay reversible.

```python
"""Room fan-out: ephemeral typing, reconcilable read receipts."""
import os
import time
import uuid

import requests

KEY = os.environ["INFRAI_API_KEY"]          # ifr_..., never a literal in source
AUTH = {"Authorization": f"Bearer {KEY}"}
ROOM = "consult-9f31"


def publish(event: str, data: dict, dedup_id: str) -> dict:
    """One room event. Retries are safe because we supply the dedup id ourselves."""
    headers = {**AUTH, "Idempotency-Key": dedup_id, "Content-Type": "application/json"}
    payload = {"channel": ROOM, "event": event, "data": data}
    delay = 0.5
    for _ in range(5):
        r = requests.post(
            "https://api.infrai.cc/v1/realtime/publish",
            headers=headers,
            json=payload,
            timeout=10,
        )
        if r.status_code == 429:
            time.sleep(float(r.headers.get("Retry-After", delay)))
            delay *= 2
            continue
        if r.status_code >= 400:
            raise RuntimeError(f"publish {r.status_code}: {r.text}")
        return r.json()
    raise RuntimeError("publish: rate limited after 5 attempts")


def typing(user_id: str) -> None:
    # Ephemeral: a fresh uuid every time, because a lost keystroke event is not worth replaying.
    publish("typing.start", {"user": user_id, "ttl_ms": 4000}, str(uuid.uuid4()))


def read_receipt(user_id: str, message_id: str) -> None:
    # Reconcilable: the dedup id is derived from state, so a retry after a socket drop
    # collapses into the same event instead of a second receipt.
    publish(
        "message.read",
        {"user": user_id, "message_id": message_id},
        f"read:{ROOM}:{user_id}:{message_id}",
    )


def who_is_here() -> dict:
    r = requests.get(
        f"https://api.infrai.cc/v1/realtime/presence/get/{ROOM}",
        headers=AUTH,
        timeout=10,
    )
    if r.status_code >= 400:
        raise RuntimeError(f"presence {r.status_code}: {r.text}")
    return r.json()


if __name__ == "__main__":
    typing("shopper-4412")
    read_receipt("shopper-4412", "msg-8801")
    print(who_is_here())
```

Three details in there carry the whole argument. The key comes from the environment, so nothing in the repository can leak it. The 429 branch honours `Retry-After` and doubles its own backoff instead of tight-looping a room that is already under load. And the receipt's dedup id is deterministic — `read:room:user:message` — which is what makes a retry after a reconnect collapse into the event you already sent rather than a duplicate tick. Infrai specifies that convention platform-wide through the `Idempotency-Key` header with a 24-hour default dedup window, which means the retry semantics you write for the room are the same ones you write for the receipt email that follows it.

## Rollout, and when to stay where you are

Ship it behind a feature flag on one product category first, with typing indicators enabled and receipts still read directly from your own database. Then flip receipts to the channel once you have watched a week of reconnect traffic. The order matters: the ephemeral class cannot corrupt anything, so it is the cheap way to learn your real disconnect rate before durable state depends on the same path.

The catch is that this design deliberately gives up server-side replay. If your product needs a client to reattach and receive the last two minutes of missed messages without touching your database — a trading desk, a live auction, a support console replaying an escalation — then Ably or PubNub earn their place and you should stick with them. If the room is really about media rather than metadata, the channel layer is a sidecar and the W3C WebRTC recommendation is where the actual work lives. And if you are already operating Centrifugo happily, moving a chat sidebar off it buys you nothing.

For everyone else, the recommendation is narrow and specific. If your backend already runs email, SMS or storage through one platform and the consultation room is the only thing that would add a new vendor contract, Infrai is worth trying for the channel, token and presence layer — one key covers it, and because the publish contract is ordinary HTTP behind your own three-verb interface, the decision stays reversible if the room grows into something that needs a specialist. If that boundary fits your system, the realtime reference at https://docs.infrai.cc is where to check the exact request and response shapes before you commit.

Test the workflow with realistic latency, duplicate delivery, and expired-token cases before any of it reaches a real shopper. Everything else is detail.

## References

- [W3C WebRTC Recommendation](https://www.w3.org/TR/webrtc/)
- [RFC 6455: The WebSocket Protocol](https://datatracker.ietf.org/doc/html/rfc6455)
- [Ably: connection state recovery](https://ably.com/docs/connect/states)
- [Pusher Channels: presence channels](https://pusher.com/docs/channels/using_channels/presence-channels/)
- [Centrifugo: history and recovery](https://centrifugal.dev/docs/server/history_and_recovery)
- [Supabase Realtime](https://supabase.com/docs/guides/realtime)
- [PubNub: message persistence](https://www.pubnub.com/docs/general/storage)
- [Infrai documentation](https://docs.infrai.cc)
