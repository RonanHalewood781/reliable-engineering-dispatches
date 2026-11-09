# Contact Form Routing — Queue Assignment Before Email and SMS Notification APIs

A contact form can be quick to connect to an email API, but adding SMS before deciding who owns the request creates a second integration with no clear routing contract. For a B2B SaaS support intake, resolve that trade-off by saving the request and its queue assignment before scheduling either message. TL;DR: set up the notification system around a durable routing decision, then let separate email and SMS workers execute that decision through small API adapters.

This note addresses a Node.js application receiving contact-form events. The focused Python example below specifies the routing policy; implement the same policy in the application's event handler. The engineering objective is fewer decisions inside each transport integration, with enough evidence to explain why a particular support queue received a particular alert.

## How should a contact-form notification system use email and SMS APIs?

Start with ownership. In this proposed design, a form submission becomes a support request with a request ID, an authenticated account ID when available, a topic, and an assigned queue. Treat free-form text as request content rather than routing authority. An unauthenticated sender typing an enterprise account name should not gain access to that account's escalation path. Likewise, a phone number submitted in a form should not automatically become an SMS destination.

Keep destinations separate.

Use an application-controlled directory to map support queues to approved recipients and their channel preferences. For the example policy, billing questions go to the billing queue, account-access questions go to account support, and everything else goes to general support. Only an explicitly enabled queue receives an SMS alert, and that alert contains a request identifier rather than the customer's message. These are proposed policy choices, not claims about a particular service. Their purpose is to keep the first integration small while avoiding accidental disclosure through a second channel.

The request record should preserve the queue chosen at intake and the policy version that made the choice. Otherwise, changing a routing rule while work is pending leaves an awkward question: should the worker notify the old owner or the new one? Choose a rule deliberately. For this design, workers use the saved assignment; an explicit reassignment creates a new notification intent.

## Make the routing decision independently testable

Define a notification intent as one requested action for one request, queue, channel, and recipient. Give it a stable identity before making a network call. A transport adapter should receive the resolved recipient and rendered content; it should not reinterpret the contact form to decide which team owns it. That separation makes the policy testable without credentials, outbound traffic, or a live recipient directory.

The following executable Python example models planning only. It leaves persistence and network calls to the application. Its 2 topic mappings, policy version, and destinations are illustrative configuration.

```python
from dataclasses import dataclass


POLICY_VERSION = 1
TOPIC_QUEUES = {
    "billing": "billing",
    "account_access": "account-support",
}


@dataclass(frozen=True)
class Queue:
    email: str
    sms: str | None = None


@dataclass(frozen=True)
class Intent:
    key: tuple[str, int, str, str, str]
    channel: str
    destination: str
    message: str


def plan(request_id: str, topic: str, queues: dict[str, Queue]):
    queue_name = TOPIC_QUEUES.get(topic, "general-support")
    queue = queues[queue_name]
    message = f"Support request {request_id} assigned to {queue_name}."
    intents = []
    for channel, destination in (("email", queue.email), ("sms", queue.sms)):
        if destination is None:
            continue
        key = (request_id, POLICY_VERSION, queue_name, channel, destination)
        intents.append(Intent(key, channel, destination, message))
    return queue_name, intents


queues = {
    "billing": Queue("billing@example.com"),
    "account-support": Queue("account-support@example.com"),
    "general-support": Queue("support@example.com"),
}
assigned_queue, planned = plan("contact-1042", "billing", queues)
assert assigned_queue == "billing"
assert [intent.channel for intent in planned] == ["email"]
```

In the Node.js handler, validate the input and resolve the recipient directory before committing the request, queue assignment, and intents in one database transaction. A dispatcher can then claim pending intents and call the appropriate email or SMS adapter. Keep API credentials in deployment configuration, and make the adapter translate its service's documented response into your own attempt record. No transport-specific payload needs to escape that boundary.

Enforce uniqueness for the saved intent identity. The tuple above prevents duplicate planning identities for the same inputs; an actual database constraint must enforce uniqueness under concurrent submissions. Use an opaque, bounded request ID rather than personal information in the identity. For manual reassignment or a deliberate repeat send, record a distinct action identifier so that deduplication does not suppress intended work.

There is a subtle limit here: local uniqueness can't settle an ambiguous remote outcome. If a worker loses its connection after submission, it must not infer that nothing was sent. Where the selected API documents idempotency, preserve the same supported key across retries. Otherwise, record the uncertainty and choose a reconciliation policy before enabling automatic retries for that outcome.

## Delivery evidence needs its own vocabulary

Keep routing status separate from notification status. A support request can be assigned while its email intent remains pending. Similarly, define API acceptance and recipient delivery as different states in the application; only advance to a delivery state when the integration provides documented evidence for it. This prevents a successful submission response from becoming an unsupported claim that an agent has seen the request.

For email, prepare the sender domain before the pilot. RFC 7208 defines SPF authorization of sending hosts for the SMTP MAIL FROM and HELO identities. Check the identity your sending configuration actually uses. Do not treat an SPF authorization result as evidence that a support alert reached an inbox or was read. The distinction matters to this workflow because the queue assignment should remain available in the support application regardless of what the notification channel reports.

For SMS, make recipient authorization an explicit deployment prerequisite. Document who may enable a destination, how the recipient can stop alerts, and which jurisdictional or organizational rules require review. This is a design requirement, not a universal claim that one consent mechanism satisfies every situation. A support form itself supplies no authorization to text the customer.

Send little data.

A request ID and queue name are sufficient for this example; keep the message body behind the support application's access controls. In logs, prefer the intent ID, attempt number, adapter outcome, and elapsed time over raw destinations or submitted text. Track pending age by queue so that a quiet inbox does not conceal an unprocessed intake backlog.

## Compare integration effort after defining the contract

With the policy fixed, compare implementations by the code and operational ownership they add. The following comparison concerns proposed application designs, not commercial products or measured performance.

| Design | Initial work | Work retained by the application |
| --- | --- | --- |
| Send both channels in the request handler | Fewest moving parts in the initial implementation | Coordinate partial outcomes and ambiguous sends during intake |
| Save intents and dispatch in background workers | Add durable records, claiming, and attempt tracking | Operate workers and define retry and reassignment policies |
| Save intents for an existing internal notification service | Map the routing contract into that service | Verify that its evidence, recipient controls, and retry semantics fit support intake |

For a team starting from a contact form, saving intents adds deployment work but gives routing changes a small test surface: input, queue, recipient set, and intent identity. If an internal dispatcher already exists, assess that contract before adding another worker service. Count operational responsibilities as integration effort too. Credentials, sender-domain configuration, delivery-event processing, and recipient management all need an owner even when the outbound API call is short.

Treat cost as a capacity question here. Estimate planned messages from the routing policy and inspect how retry limits and SMS eligibility change that count. There is no need for a unit-price comparison to decide where the routing decision belongs.

## Roll out one queue with observable acceptance criteria

First, exercise the planner with billing, account access, unknown topics, missing queue configuration, and SMS disabled. Then test the worker against controlled adapter outcomes: acceptance, explicit rejection, throttling, timeout after submission, and duplicate delivery-event input. Verify that a repeated intake cannot silently produce another intent and that an unknown delivery outcome stays visible.

Pilot one support queue with email enabled and SMS destinations disabled. Inspect saved assignments against the intake examples, compare pending age with the team's chosen response target, and verify sender authorization. Enable SMS only after the recipient directory and suppression path have been reviewed. During a rollback, pause dispatch while preserving intake and pending intents for inspection; resuming should use the same saved identities.

The release criterion is concrete: every accepted contact form has an assigned queue, every intended alert has a traceable record, and uncertain sends have an explicit handling path. That gives the team a support workflow it can inspect as it expands channel coverage.

## Sources

References:

- RFC 7208: Sender Policy Framework (SPF), the source for the sender-authorization scope discussed above: https://datatracker.ietf.org/doc/html/rfc7208
