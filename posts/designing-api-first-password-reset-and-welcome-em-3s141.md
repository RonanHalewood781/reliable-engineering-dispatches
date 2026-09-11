# Designing API-First Password Reset and Welcome Email Delivery (With Compliance Evidence)

Short answer: choose the email service whose API, event stream, and suppression controls let your fintech team prove why each password reset or welcome email was sent, what happened to it, and why a failed recipient won't be retried blindly. A dedicated sending domain is useful isolation, but it cannot compensate for weak bounce tracking or an unauditable suppression workflow.

That changes the selection exercise. The best fit isn't the service with the longest feature sheet. It's the one that preserves a clean chain from an application decision to a delivery event and then to the next send decision — without making SMTP the hidden integration boundary.

## Security starts with a traceable authorization record

Start with evidence, not throughput. For any attempted send, an operator should be able to connect four things: the application event that authorized the message, the recipient and message class, the provider's accepted identifier, and the later delivery or bounce event. The stored record should also explain whether the address entered suppression and which rule caused that state.

This matters more in fintech than a pretty template editor. A password reset is security-sensitive and time-sensitive; a welcome email is usually less urgent, but it still reaches the same mailbox and can damage future delivery if the system keeps retrying an invalid address. If both flows share a sender, one careless retry policy can contaminate the evidence for both.

The dedicated domain question belongs here. Treat the domain as a boundary for reputation, authentication, ownership, and operational change. Require the team to document who controls its DNS, who may change sending configuration, and how a migration would be staged. Don't award a checkbox merely because a vendor accepts a custom domain.

No mystery state.

The API-first requirement should be equally literal. The application submits a typed request over an authenticated API and receives an identifier that it persists. Delivery events then update that record through a verified event channel. SMTP may be a valid compatibility mechanism elsewhere, but if the requirement is “no SMTP,” reject designs that quietly relay through SMTP behind an SDK or internal adapter. The boundary you operate should match the boundary you selected.

## How can password reset email stay reliable when bounce tracking arrives late?

The send call is the easy part. The difficult part is deciding, under retries and delayed events, whether another call is allowed.

Put that decision in a small delivery service owned by your application. It receives a business event, classifies the message as `password_reset` or `welcome`, checks the current recipient state, creates an immutable attempt record, and only then calls a provider adapter. The adapter translates your internal request into the selected API shape. It should not contain business policy.

Bounce events travel in the opposite direction. Verify the event, store the raw evidence with access controls appropriate to recipient data, normalize the provider-specific type, and update suppression idempotently. A duplicate event must not create a second policy transition. An older event must not overwrite newer recipient state merely because it arrived late. These rules are application invariants; keeping them outside a vendor-specific callback makes a later provider change much less invasive.

Here is a deliberately small Python model. The event names are an internal contract, not a claim about any commercial API:

```python
from dataclasses import dataclass
from datetime import datetime, timezone
from enum import Enum


class RecipientState(str, Enum):
    ACTIVE = "active"
    SUPPRESSED = "suppressed"


@dataclass(frozen=True)
class DeliveryEvent:
    event_id: str
    message_id: str
    recipient: str
    outcome: str
    occurred_at: datetime


def apply_delivery_event(event, attempts, recipients, seen_events):
    if event.event_id in seen_events:
        return "duplicate"

    attempt = attempts.get(event.message_id)
    if attempt is None or attempt["recipient"] != event.recipient:
        return "quarantine"

    seen_events.add(event.event_id)
    attempt["events"].append(event)

    if event.outcome == "permanent_bounce":
        current = recipients.setdefault(
            event.recipient,
            {"state": RecipientState.ACTIVE, "changed_at": datetime.min.replace(tzinfo=timezone.utc)},
        )
        if event.occurred_at >= current["changed_at"]:
            current["state"] = RecipientState.SUPPRESSED
            current["changed_at"] = event.occurred_at

    return "recorded"
```

The long paragraph is intentional because this is where reviews tend to miss the real failure mode: a request times out, the caller retries, two message identifiers are created, a bounce arrives for the first attempt after a delivery event for the second, and an operator later sees disconnected rows. An idempotency key derived from the business action can prevent duplicate submission; a unique provider message identifier links callbacks; event identifiers deduplicate delivery evidence; and timestamps plus explicit transition rules keep arrival order from becoming policy. The exact data store can vary. The invariants can't.

I'm not sure any paper evaluation can prove how a service behaves under every reordering pattern. Resolve that uncertainty with a sandbox test and a controlled preproduction rollout, using addresses and data approved for testing.

## Evaluation by event replay exposes policy gaps

Give each candidate the same acceptance test. Create one password-reset request and one welcome request, record the application authorization for each, submit them through the API adapter, and confirm that the returned identifiers survive into your event records. Then replay an identical delivery event and verify that the recipient state changes once. Send an older event after a newer one. Confirm that the audit view remains coherent.

Next, test suppression as a policy boundary. A suppressed recipient should be rejected before the provider call, and the rejection should produce a reason an operator can retrieve. Re-enabling an address should require an explicit, authorized transition rather than deletion of inconvenient history. Retention and access rules need review by the people accountable for your compliance program; the right duration and fields depend on obligations not specified here.

Keep the evaluation table short enough to use in a design review:

| Decision area | Evidence to request | Failure to reject |
|---|---|---|
| API submission | Stable request contract and persisted message identifier | A send cannot be traced to its authorization |
| Bounce tracking | Authenticated events with deduplication inputs | Duplicate or forged events alter recipient state |
| Suppression | Queryable reason, time, and authorized state changes | Invalid recipients are retried without explanation |
| Dedicated domain | Documented DNS ownership and migration procedure | Domain control becomes an operational surprise |
| Operations | Exportable records and a replay test | An incident cannot be reconstructed independently |

It's tempting to score “delivered” as the final state. Don't. Delivery evidence describes what the sending system observed; it does not establish that a human read the message or that every mailbox will treat future mail the same way. Phrase dashboards and support responses accordingly.

## Governance sets the boundary for dedicated domains and SMS fallback

A dedicated domain does not repair stale recipient data, classify bounces, serialize suppression changes, or connect a send to its business authorization. It gives you a useful control boundary. Your application still has to enforce the recipient policy on every attempt.

It is also not suitable when the organization cannot own the DNS and operational work that the boundary creates. In that case, pause the migration or use an already governed sending domain until ownership, change review, and rollback responsibilities are explicit. Likewise, stick with SMTP for a legacy application when replacing that integration would introduce more risk than the API evidence would remove; put the evidence layer around the existing boundary first, then migrate deliberately. “API-first” is an architecture preference, not permission to ignore deployment risk.

SMS fallback needs its own decision path. Email suppression must not silently authorize a text message, because the channel, recipient consent, and compliance evidence differ. CTIA publishes messaging interoperability and compliance best practices for messaging programs; use the applicable guidance when the product introduces SMS rather than treating a phone number as an automatic escape hatch from an email bounce.

## Integration rollout protects the audit trail

Start by defining the internal attempt and event records, then build one provider adapter behind them. Backfill no invented certainty: if an old record lacks a provider identifier or authorization reference, mark that field as unavailable instead of guessing.

Shadow the policy checks before they block sends. Compare the proposed suppression decision with the current path, investigate disagreements, and only then enable enforcement for welcome emails. Password resets deserve a separate rollout because their urgency and security impact change the response to a blocked attempt. Keep a reviewed rollback that changes routing while preserving attempt and event records.

The final selection should be the candidate that passes this evidence test with the least provider-specific policy in your code. Feature breadth can break a tie. It shouldn't erase an ambiguous suppression state.

## References

- https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- https://www.ctia.org/the-wireless-industry/industry-commitments/messaging-interoperability-sms-mms
