# Freight Contact Forms in Node.js: Custom Template, Suppression, and Delivery Polling

Use the smallest piece of machinery that still proves the email reached a mailbox: one database row per notification, a custom template rendered on the server, a suppression list checked before submission, and a small polling worker for delivery status. That is enough for a Next.js or Node.js contact form that has to acknowledge a shipper, a driver, or a claims adjuster and tell them which support queue owns their case.

No webhook receiver. No event bus.

## The failure mode is a duplicate case, not a missing email

A freight contact form carries a tracking number, a service level, an issue type — damaged pallet, missed pickup window, invoice dispute — and a country. A router turns that into a queue assignment: claims, dispatch, billing. The transactional email that goes back out carries the case ID and the queue that owns it, which makes it the receipt for a decision the system already made. When it never lands, the customer fills the form in again. Now two cases exist for one damaged pallet, in two different queues, on two different response clocks, and the weekly report says volume is up.

Nobody escalates a missing welcome email. Everybody escalates a missing case ID.

So the send has to be a state transition rather than a call bolted onto the end of the insert. Reserve the notification row first, under a key derived from the case itself — `ack:<case-id>:<template-revision>` works fine. Evaluate suppression and region policy against that row. Render the template on the server, submit once, store whatever message identifier comes back, and let every retry read the existing row instead of creating a second one. That ordering covers the ugly case where the provider accepted the message and your process died before the local commit: a deploy at the wrong second, a serverless function hitting its wall clock, a browser that double-posted because the shipper got impatient. Without the reservation, a retry is indistinguishable from a first attempt, and the second acknowledgement arrives quoting a different case ID than the first.

Client-supplied idempotency keys on the provider call help too. They don't replace the row.

## How should a Node.js worker poll transactional email delivery for a support queue?

Submit from a server route, persist the returned message identifier, and let a separate worker ask for status on a schedule. Don't hold the request open while a mailbox decides; a form post that waits on delivery is a form post that times out behind a corporate mail gateway. Don't park the follow-up in an in-process timer either, because a serverless instance can disappear between the send and the check.

Polling is the lower-integration-effort option, and that's the honest reason to start there. A webhook receiver means a publicly reachable endpoint, signature verification, replay protection, out-of-order events, and a second thing to keep deployed and monitored. A poller is a cron entry, a bounded query, and a retry budget.

The catch is volume and latency.

Every open notification costs a request, and a status you learn 60 seconds late is a status an agent can't act on while the customer is still on the phone. Past a few thousand messages an hour — or once agents need bounce information inside the console in real time — the webhook receiver stops being extra work and becomes the cheaper design. Somewhere between those two shapes there's a crossover point that depends on your open-notification count, and I'd measure it rather than guess.

The worker below is deliberately narrow: standard library only, explicit method, credentials from the environment, bounded attempts, and it honours `Retry-After` when the provider answers `429`.

```python
import json
import os
import random
import time
import urllib.error
import urllib.parse
import urllib.request

BASE = os.environ["MAIL_STATUS_BASE"]        # region-pinned host, e.g. mail-eu.internal.example
TOKEN = os.environ["MAIL_STATUS_TOKEN"]
TERMINAL = {"delivered", "bounced", "complained", "rejected"}


def backoff(headers, attempt):
    retry_after = headers.get("Retry-After")
    if retry_after:
        try:
            return max(0.0, float(retry_after))
        except ValueError:
            pass
    return min(2 ** attempt, 30) + random.uniform(0, 0.5)


def fetch_status(message_id, attempts=5):
    url = f"{BASE}/messages/{urllib.parse.quote(message_id, safe='')}"
    for attempt in range(attempts):
        req = urllib.request.Request(url, method="GET", headers={
            "Authorization": f"Bearer {TOKEN}",
            "Accept": "application/json",
        })
        try:
            with urllib.request.urlopen(req, timeout=15) as resp:
                return json.loads(resp.read().decode("utf-8"))
        except urllib.error.HTTPError as err:
            if err.code != 429 or attempt == attempts - 1:
                raise
            time.sleep(backoff(err.headers, attempt))
    raise RuntimeError("status lookup exhausted its retry budget")


def poll_open_acks(db, now):
    # One pass per tick. The query decides what is still open, not the process.
    for row in db.due_notifications(limit=200, sent_before=now - 300):
        payload = fetch_status(row.message_id)
        db.record_raw(row.id, payload)          # keep the provider payload verbatim
        state = payload.get("state")
        if state in TERMINAL:
            db.settle(row.id, state)
            if state in {"bounced", "complained"}:
                db.suppress(row.recipient, reason=state, evidence=payload.get("id"))
                db.reroute_case(row.case_id, queue="callback")
        else:
            db.defer(row.id, next_check=now + min(300, 30 * (row.checks + 1)))
```

Two things it refuses to do. It never maps an unfamiliar state onto `delivered` or `bounced` because the local enum has no better slot, and it never treats an old row as settled — a notification that has been open past its budget is an alert, not a delivery. Keeping the raw payload is what lets you rebuild the mapping later when a provider adds a state you've never seen.

## What the suppression list has to model for support agents

A hard bounce on a contact-form address means the acknowledgement cannot reach that person at all, so the case needs a different channel — a callback queue, an SMS to the phone number on the shipment, or a note on the customer record. That's why the suppression list belongs in your database and readable by the agent console, not only inside a provider dashboard. The agent looking at a two-day-old claim needs to see "we tried, the domain rejected it" without opening a second tool.

Check it before submission, not after. An address that has already bounced twice this week doesn't earn a third attempt just because someone filled the form in again.

Keep the scopes separate and decide the overlap on purpose. Marketing consent, service updates, and case receipts are three different legal and operational categories, and plenty of platforms apply account-level suppression across all of them — which is exactly the behaviour you want for complaints and exactly the behaviour that will silently swallow a claims receipt if you dump a marketing purge into the same list. Complaint feedback arrives in a standard format (ARF), so parse it rather than screen-scraping a dashboard export. Publish DMARC for the sending domain, keep the one-click unsubscribe header on the marketing stream where it belongs, and leave it off the transactional receipt.

## What the US and EU split actually costs you

Pick the region at ingestion, not at send time. The form body is customer data — addresses, invoice numbers, sometimes photos of a crushed pallet — and if the case lands in an EU store, the notification, the rendered template, and the provider event history should stay on that side too. A `region` column on the notification row is a routing input, not evidence of compliance; the evidence is your retention schedule, your deletion path, and a processor agreement that covers wherever message bodies actually sit.

Run a separate sending subdomain per region with its own DKIM selector and reputation. It costs a DNS record and saves you from a US bulk problem dragging EU case receipts into a spam folder. And if the claims portal ever mails a one-time code instead of a signed link, treat that code as an authenticator with a short lifetime and a rate limit — NIST's digital identity guidance is the baseline worth reading before choosing the numbers.

## Three integration shapes, and where they differ in daily work

Compare the shapes on operating surface, because that's what integration effort really means once the first version ships.

| Shape | What you write | What you operate | Where it stops fitting |
|---|---|---|---|
| Self-hosted relay | SMTP client, queue, bounce parser | IP warmup, PTR and DKIM records, feedback processing, blocklist appeals | Small teams without a deliverability owner |
| Managed API with pull status | HTTP call, notification row, poller | Cron worker, retry budget, suppression mirror | Real-time agent consoles at high volume |
| Messaging platform with template editor and webhooks | Event handler, signature check | Public endpoint, template governance, vendor-shaped data model | Teams that expect to move providers later |

The decision rule I'd apply to a freight support desk: if one engineer owns notifications part-time, the middle row is the sane default, and the poller is a day of work rather than a subsystem. Move up to the platform row when non-engineers need to edit the custom template and the queue routing rules without a deploy — that's a real capability, and rebuilding it yourself is worse. Stick with the self-hosted relay when volume, cost structure, or contractual data handling make an outside processor the wrong pick, and accept that you've hired a deliverability problem along with it.

Each row also has a boundary worth naming out loud. A managed API that doesn't support the transport you need — SMTP relay for a legacy WMS, or voice fallback for a dispatch escalation — is the wrong tool no matter how clean the JSON is, and a platform whose template model has no export path is a migration you'll pay for later.

## Rollout: cut over the transport without losing a case

Build the ledger and the renderer first with sending switched off, then send only to an internal cohort while the router runs live. Compare rows against real mailboxes before anyone outside sees an acknowledgement.

```bash
*/5 * * * * cd /srv/support && python -m workers.poll_acks >> /var/log/poll_acks.log 2>&1
```

The test matrix that catches the actual regressions is short: an empty display name, accented characters in a consignee address, a 40-character tracking reference, an address already on the suppression list, and a deploy that lands between submission and the local commit. Add an alert for notifications open longer than your poll budget, and one for a queue whose acknowledgement rate drops while its case count holds steady — that pattern usually means a template revision broke rendering for one locale, not that customers went quiet.

During a provider migration, keep both paths available but make the choice atomic per case, so one contact form submission can never produce two receipts. Freeze the template revision during the cutover; changing content and transport in the same week makes every deliverability question unanswerable.

## References

- [Amazon SES Developer Guide](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [NIST SP 800-63B: Digital Identity Guidelines, Authentication and Lifecycle Management](https://pages.nist.gov/800-63-3/sp800-63b.html)
- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://www.rfc-editor.org/rfc/rfc7489)
- [RFC 5965: An Extensible Format for Email Feedback Reports (ARF)](https://www.rfc-editor.org/rfc/rfc5965)
- [RFC 8058: Signaling One-Click Functionality for List Email Headers](https://www.rfc-editor.org/rfc/rfc8058)
- [RFC 9110: HTTP Semantics (Retry-After, 429 handling)](https://www.rfc-editor.org/rfc/rfc9110)
- [Regulation (EU) 2016/679 (GDPR), consolidated text](https://eur-lex.europa.eu/eli/reg/2016/679/oj)
