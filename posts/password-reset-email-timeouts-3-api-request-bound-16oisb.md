# Password Reset Email Timeouts: 3 API Request Boundaries for Settled Receipts

Commit the receipt intent in the same database transaction that records payment settlement, then let a worker claim and deliver it outside the API request. **Short answer:** an HTTP timeout must make the outcome uncertain, never trigger a fresh email intent. The deciding constraint is integration effort: one durable uniqueness constraint and one worker are easier to reason about than retry logic spread across a serverless handler, a mail client, and the caller.

This note uses an order receipt after payment settles, but the same boundary applies to password-reset mail. There is one important difference: repeated reset requests may deliberately rotate or reuse credentials according to the security policy, while one settled order should retain one stable receipt identity. Do not confuse a business operation with a network attempt.

## Which timeout actually failed?

“The request timed out” does not identify where work stopped. The caller may have lost the response after the API committed. A function may have run out of execution time while a mail submission was in flight. A mail service may have accepted a message even though its response never reached the worker. SMTP itself separates message transfer from final delivery; acceptance is not evidence that the recipient read, or even received, the message.

That ambiguity is the duplicate-send trap. Retrying transport is necessary, but creating a new logical receipt on each attempt is not.

The bad sequence is concrete. A payment callback commits the order at 10:00:00. The API starts a mail submission, its deadline expires, and the caller receives no useful answer. At 10:00:05 the caller retries with a new request identifier. The first submission may already be accepted, so treating the second request as new work can send two receipts. No unusual outage is required; one lost acknowledgment is enough. A stable business key makes both invocations compete for the same intent instead.

Use three explicit boundaries:

1. Business boundary: the transition to `settled` creates at most one receipt intent for an order.
2. Execution boundary: the API returns after committing local state; mail delivery is not on its latency budget.
3. Provider boundary: a submission attempt has its own stable key and recorded outcome, including `unknown` when the response is lost.

The first boundary prevents duplicate intent. The second prevents a slow dependency from consuming the request deadline. The third gives operators somewhere honest to record uncertainty instead of calling every timeout a failure.

## How should a password reset email API handle a request timeout?

The decision is a transactional outbox keyed by the business event, processed by an at-least-once worker. “At least once” means the worker may execute again after a crash. The design absorbs that fact rather than assuming a serverless runtime will invoke exactly once.

It cannot promise exactly-once delivery.

| Option | Integration effort | Failure boundary | Duplicate control | Decision |
|---|---:|---|---|---|
| Send inside the settlement request | Low initially | API and mail submission share one deadline | Caller retries can repeat the send | Rejected |
| Publish an event after committing settlement | Medium | Crash can occur between commit and publish | Needs reconciliation or a broker transaction | Valid when existing infrastructure closes that gap |
| Insert an outbox row with settlement | Medium | Database commit is the durable handoff | Unique business key suppresses repeated intent | Chosen |

The invariants are small enough to test. A settled order has no more than one `receipt.requested` outbox row. The payload points to immutable receipt data or a versioned snapshot, so a retry cannot silently change totals. A worker never marks an attempt delivered merely because it started. Finally, the public response does not reveal whether an address exists; that matters when the same machinery is adapted to password resets.

Keep credentials and tokens out of logs. For reset flows, the reset secret also needs a short validity window, single-use enforcement, and invalidation under the application’s security policy. Idempotency prevents duplicate work; it does not replace those controls.

## The critical path in code

The following Python uses a generic database and mail adapter. The SQL placeholder style is illustrative, but the transaction shape is the point: settlement and intent either commit together or do not commit at all.

```python
from dataclasses import dataclass
from datetime import datetime, timezone
from typing import Protocol


class Database(Protocol):
    def transaction(self): ...
    def execute(self, statement: str, parameters: dict): ...
    def fetch_one(self, statement: str, parameters: dict): ...


class MailGateway(Protocol):
    def send(self, *, message_key: str, recipient: str, subject: str, body: str): ...


@dataclass(frozen=True)
class ReceiptIntent:
    event_key: str
    recipient: str
    order_number: str
    total: str


def record_settlement(db: Database, intent: ReceiptIntent) -> None:
    with db.transaction():
        db.execute(
            "UPDATE orders SET status = :status WHERE number = :number",
            {"status": "settled", "number": intent.order_number},
        )
        db.execute(
            """
            INSERT INTO outbox (event_key, kind, payload, state)
            VALUES (:event_key, :kind, :payload, :state)
            ON CONFLICT (event_key) DO NOTHING
            """,
            {
                "event_key": intent.event_key,
                "kind": "receipt.requested",
                "payload": {
                    "recipient": intent.recipient,
                    "order_number": intent.order_number,
                    "total": intent.total,
                },
                "state": "pending",
            },
        )


def deliver_one(db: Database, mail: MailGateway, event_key: str) -> None:
    row = db.fetch_one(
        "SELECT payload, state FROM outbox WHERE event_key = :event_key",
        {"event_key": event_key},
    )
    if row is None or row["state"] == "delivered":
        return

    payload = row["payload"]
    mail.send(
        message_key=event_key,
        recipient=payload["recipient"],
        subject=f"Receipt for order {payload['order_number']}",
        body=f"Order {payload['order_number']} settled for {payload['total']}.",
    )
    db.execute(
        """
        UPDATE outbox
        SET state = :state, delivered_at = :delivered_at
        WHERE event_key = :event_key AND state <> :state
        """,
        {
            "state": "delivered",
            "delivered_at": datetime.now(timezone.utc),
            "event_key": event_key,
        },
    )
```

There is still a narrow interval between `mail.send` succeeding and the final update committing. A process crash there can produce another submission. A gateway-supported idempotency key can close that interval only if its documented retention and equivalence rules cover the retry window. Without that contract, suppress automatic retries for an `unknown` outcome and reconcile it using submission records or delivery events. Do not manufacture certainty.

That is the hard edge.

This is also why the event key should be derived from stable business identity, such as `receipt:{order_number}:v1`, rather than a random value generated on every invocation. The database must enforce uniqueness. An in-memory check disappears with a cold start, and a read-then-insert check races under concurrency.

## Retry policy is state, not a loop

Retry only failures classified as transient, and persist the attempt count plus the next eligible time. Exponential backoff with jitter reduces synchronized retries; it does not decide whether another submission is safe. That decision belongs to the recorded outcome.

A practical state model distinguishes `pending`, `submitting`, `delivered`, `retryable`, `unknown`, and `dead`. Put a lease expiry on `submitting` so abandoned work can be recovered. Cap attempts, but alert on age as well as count: one receipt waiting 40 minutes with a single attempt is more useful operational evidence than a busy counter.

Watch four signals: outbox age, attempts by outcome class, provider acceptance latency, and confirmed delivery or bounce events where available. Keep the business key in structured logs, but hash or redact recipient addresses. For compliance work, retention periods and access controls deserve an explicit decision; an outbox is operational data, not a reason to retain message content forever.

Email has another edge: “delivered” in application state usually means accepted by the next hop, not displayed in an inbox. Amazon SES, for example, documents sending, event publishing, and deliverability as distinct concerns. Similar distinctions exist across mail systems. Name the state after what the evidence proves.

## Why reject synchronous sending?

Synchronous sending has a valid use case: an internal tool with low consequence, no automatic caller retry, and an operator present to resolve an ambiguous result. Its attraction is real. There is less infrastructure, and a successful response can include the immediate submission outcome.

It is the wrong fit for a payment receipt endpoint because it couples user-visible latency to DNS, connection setup, remote throttling, and the mail service’s response path. Worse, the client sees the same timeout for two different histories: no submission occurred, or submission occurred and only the acknowledgment vanished. Blind retry cannot distinguish them.

The outbox has clear limitations and trade-offs: it costs a table, a worker, a polling or notification mechanism, and a cleanup policy. It is not suitable when the database cannot participate in the business transaction, and it may be excessive for a low-consequence internal action with a human operator. An existing broker transaction or change-data-capture pipeline can be the better fit when the team already operates it and can prove that committed records are published. For a password-reset endpoint, use the same durable boundary but key requests according to the intended credential lifecycle, return a uniform response, and keep the token valid for only the policy-defined window. For a settled order, key by order and receipt version. Different business rule; same failure analysis.

The durable rule is concise: **retry execution freely only after making intent unique and outcomes explicit.** That gives the API a short, deterministic path while leaving delivery uncertainty visible enough to operate.

## References

- https://www.rfc-editor.org/rfc/rfc9110
- https://www.rfc-editor.org/rfc/rfc5321
- https://www.rfc-editor.org/rfc/rfc5322
- https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- https://microservices.io/patterns/data/transactional-outbox.html
- https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/
- https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- https://www.twilio.com/docs/glossary/what-sms-character-limit
