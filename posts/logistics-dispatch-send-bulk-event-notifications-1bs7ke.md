# Logistics Dispatch: Send Bulk Event Notifications in 4 Email-SMS Batches

TL;DR: Treat each logistics contact submission as one durable intent, resolve its support queue before fan-out, and give email and SMS separate delivery jobs. A worker may claim small batches, but every attempt needs an idempotency key, an explicit retry time, and a terminal state. Cron should recover stranded work, not act as the primary sender. This adds a little schema work; it removes the far harder problem of explaining why a carrier received two escalation texts while the customs desk received none.

A logistics contact form looks harmless until the messages concern a refrigerated load, a customs hold, or a delivery-window change. Then routing accuracy and delivery timing matter more than raw throughput. The lowest-integration design is not a loop that calls two gateways. It is a transactional outbox plus channel-specific workers, with policy decisions made before anything leaves the system.

## How should workers send bulk email and SMS event notifications?

The four controls are routing, consent, idempotency, and bounded retries. They belong in the application boundary because a provider can transport a message but cannot infer which support queue owns shipment `CN-48217`, whether an SMS is appropriate for that contact, or whether a repeated form POST represents a new request.

That boundary matters.

Start by normalizing the submission and selecting exactly one queue from stable business data: issue type, shipment region, account tier, and operating hours. Store the routing result alongside the original intent. Do not ask the worker to recalculate it later; rules may change between acceptance and delivery, producing an email for one desk and a text for another.

Consider a form submitted for shipment `CN-48217`. Its issue is a customs document, its region is East Asia, and the routing rules assign it to the customs queue. The intake transaction stores that decision and creates an email acknowledgment plus, only when policy permits, an SMS escalation. If the region map changes ten minutes later, both pending jobs still point to the customs queue selected at intake. Recomputing inside each worker looks easier because it avoids storing a route, but it creates a split-brain result: two channels representing one request can disagree. I would spend the extra column and transaction work here. It is a small integration trade-off for a stable answer to the question support staff will ask first: where did this request go?

Consent is channel-specific. An email address and a phone number are destinations, not permission. Record the policy result that applied when the job was created, including the policy version and the reason a channel was allowed or suppressed. Compliance reviews become a database query instead of an exercise in reconstructing application logs.

Idempotency closes two duplicate paths: a browser retry at intake and a worker retry after an ambiguous gateway response. A practical key combines the immutable contact intent ID, channel, recipient, and template revision. Put a unique constraint on it. The constraint, rather than timing luck, decides whether a second job exists.

Duplicates are worse.

Retries must be finite. Temporary transport failures can move through scheduled retry states with exponential delay and jitter; permanent address, policy, or content failures should stop immediately. Keep the original job row and append attempt records so the support team can distinguish never attempted, accepted for transport, rejected, and exhausted.

## Shape the data before adding workers

Three records are enough for a useful boundary: `contact_intent`, `delivery_job`, and `delivery_attempt`. The intent owns the route. Each job owns one channel and recipient. Attempts preserve transport evidence without turning the job row into an ever-changing log blob.

The following Python sketch shows the transaction that matters. The database call is deliberately generic; the important behavior is that the intent and its jobs commit together.

```python
def accept_contact(db, submission, policy):
    intent_id = submission["id"]
    route = policy.route(
        issue=submission["issue"],
        region=submission["shipment_region"],
    )

    with db.transaction() as tx:
        tx.insert_intent(intent_id, route, submission)
        for channel, recipient in policy.allowed_destinations(submission):
            tx.insert_job(
                intent_id=intent_id,
                route=route,
                channel=channel,
                recipient=recipient,
                idempotency_key=(
                    f"{intent_id}:{channel}:{recipient}:contact-v3"
                ),
                state="ready",
            )
```

Keep message rendering deterministic. Save the template revision and structured variables, then render from that immutable pair. If a customs reference is corrected, create a new intent or an explicit superseding job; silently mutating a queued payload ruins the audit trail.

Email authentication is part of this boundary too. DKIM signs selected message headers and the body so a verifier can validate the responsible signing domain and detect alteration covered by the signature. It does not choose the support queue or grant consent. Treat signing, SPF authorization, and DMARC policy as domain operations owned by the mail program, while the application supplies stable identity and content fields.

SMS has a different edge. Message encoding affects segmentation: GSM-7 commonly permits 160 characters in a single segment, while UCS-2 permits 70, and concatenated messages have lower per-segment limits because headers consume space. A curly quote or non-GSM character can therefore change segment count. Measure the rendered body with the same encoding rules used at dispatch, and keep urgent logistics texts short enough that a tracking link or reference does not unexpectedly cross the boundary.

## Claim batches without merging outcomes

Batching is a database and scheduling optimization, not a delivery guarantee. Claim perhaps 50 ready jobs in one transaction, mark them leased with an expiry, and process each job independently. The exact batch size is an operational setting to tune against gateway limits, database contention, and latency; it is not a correctness constant.

This design has a real limitation: it is a poor fit for a tiny internal form that sends a few non-urgent messages and can tolerate manual recovery. A direct synchronous send may require less integration work there. At the other extreme, teams that already operate a durable event log and streaming consumers may use that substrate instead of database polling. The outbox approach trades extra tables, leases, and reconciliation code for local transactional control; choose it when duplicate or missing contact alerts justify that operational burden.

Short leases are dangerous during slow downstream calls. Very long leases delay recovery after a crashed worker. Pick an initial lease longer than the enforced request timeout, renew it only while the worker is healthy, and expose lease age as a metric. The useful invariant is simple: another worker cannot claim a live lease, but an expired lease does not imply that the previous network request failed. Idempotency must still protect the handoff.

```python
def run_batch(store, transports, now):
    jobs = store.claim_ready(limit=50, lease_seconds=90, now=now)

    for job in jobs:
        try:
            result = transports[job.channel].send(
                recipient=job.recipient,
                payload=job.rendered_payload,
                idempotency_key=job.idempotency_key,
                timeout_seconds=20,
            )
            store.record_acceptance(job.id, result.transport_id, now)
        except PermanentDeliveryError as error:
            store.record_terminal_failure(job.id, str(error), now)
        except TemporaryDeliveryError as error:
            store.schedule_retry(job.id, str(error), now)
```

One awkward case deserves attention: the transport accepts a request, but the worker times out before reading the response. Retrying may duplicate a message if the transport does not honor the supplied idempotency key. Record the attempt before sending, retain any transport identifier returned later, and define an `unknown` outcome that can be reconciled instead of pretending every timeout is a clean failure. For high-consequence SMS escalation, manual review may be preferable to an automatic second send after an ambiguous result.

Backpressure should be per channel and, where appropriate, per destination class. A throttled SMS path must not stop email acknowledgments. Conversely, an email backlog must not consume every database connection while the SMS worker waits. Separate queues or claim predicates are a modest integration cost with a large operational payoff.

## Polling is the safety net

Cron comes later.

Run workers continuously when notification latency matters. A cron-triggered poller can be acceptable at small scale, but its schedule becomes a built-in delay and overlapping executions need the same leasing discipline as permanent workers. Cron is most valuable as a reconciler that scans for expired leases, overdue retries, and attempts stuck in `unknown`.

Do not reset every old row to `ready`. Recovery should compare job state, lease expiry, attempt history, and any available transport receipt, then choose one explicit transition. That makes the repair task safe to run twice.

Observe the system from the intent outward. Useful signals include oldest ready-job age by channel, attempts per job, expired leases, unknown outcomes, suppressions by policy reason, and terminal failures by route. Transport acceptance is not proof that a person read a message, so name the metric `accepted`, not `delivered`, unless a documented delivery receipt supports that state. Logs should carry the intent ID, job ID, route, channel, template revision, and attempt number; avoid putting contact text, email addresses, or phone numbers into routine log fields.

The failure drill is concrete: stop a worker after it sends but before it records acceptance, let the lease expire, and verify that reconciliation does not blindly send again. Then throttle one channel and confirm the other continues. Finally, change a routing rule while old jobs remain queued and confirm those jobs retain their original queue. These tests find gaps that happy-path mocks miss.

## Roll out by route, then widen

Begin with one low-risk support route in shadow mode: create jobs and render payloads, but replace transport sends with recorded suppressions. Compare the chosen queue with the existing workflow. Next, enable email acknowledgments for that route, then SMS only for policy-approved escalation cases. Keep a kill switch per channel and route.

For migration, dual-writing is safer than dual-sending. Let the existing path remain authoritative while the new pipeline records what it would do. Once routing and suppression decisions agree, move a small route to the new sender and watch queue age, duplicate constraints, unknown outcomes, and terminal failures. Expand by route, not by an arbitrary percentage that mixes customs, billing, and urgent shipment traffic.

The integration effort is concentrated where it pays: one transactional write boundary, one generic transport interface, and one reconciliation process. With those pieces in place, swapping a transport changes an adapter. It does not rewrite routing, consent, retry policy, or the evidence needed to explain a missed logistics alert.

## Sources

- RFC 6376, DomainKeys Identified Mail (DKIM): https://datatracker.ietf.org/doc/html/rfc6376
- RFC 7208, Sender Policy Framework (SPF): https://datatracker.ietf.org/doc/html/rfc7208
- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance (DMARC): https://datatracker.ietf.org/doc/html/rfc7489
- SMS character limits and GSM-7/UCS-2 segmentation: https://www.twilio.com/docs/glossary/what-sms-character-limit
