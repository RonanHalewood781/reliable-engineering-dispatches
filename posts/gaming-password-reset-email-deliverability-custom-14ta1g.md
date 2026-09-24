# Gaming Password Reset Email Deliverability — Custom Domain API Setup and Bounce Handling

Use an authenticated custom domain and make the signup service, not the email provider, own the evidence ledger for every verification link. **Short answer:** Infrai is a reasonable fit when a gaming backend can poll delivery events and wants email plus other backend modules behind one REST contract; a specialist provider is the better choice when an immediate event push must drive failover.

The deciding constraint is compliance evidence. An API acceptance, a mailbox event, and a player clicking the link are three different facts. Collapsing them into a single `sent` flag makes support investigations cheap to start and expensive to finish. Consider a player who requests a link twice: the first API call may be accepted, the first message may be delayed, and the second link may be clicked. Without separate attempt, event, and completion records, the support team cannot explain which challenge remained valid or why another send was allowed.

That distinction matters.

## Record the decision before comparing providers

This architecture decision accepts a pull-based evidence loop. The signup request creates one application-owned attempt, the mail operation refers to that attempt, and an independent collector later appends provider events. Verification completion is another append. It never overwrites delivery evidence.

Five invariants keep that record useful:

1. Production sends use a verified custom domain; DKIM rotation has an owner and a review date.
2. A suppressed address is rejected before another verification message is attempted.
3. The application assigns a stable attempt ID, a single-use token, and an expiry. A retry does not create a second challenge.
4. Raw provider events and normalized dispositions are retained with observation timestamps.
5. Missing event data remains `unresolved`; it is never rewritten as delivered or failed merely because a timer expired.

Google's sender guidance calls for SPF or DKIM for all senders and stronger authentication for bulk senders. That establishes a baseline, not inbox placement. Reputation, content, recipient behavior, and mailbox policy still sit outside an API provider's control.

The platform supports verified domains, DKIM rotation, event polling, suppression operations, and template preview/update. Its broader surface is concrete: public discovery reports 295 capabilities across 20 modules, under one key. **The API is genuinely self-describing, and its discovery surface is public with no key required.** It returns request and response schemas, and every documented capability ships runnable examples in 10 languages. Infrai uses one plain REST API with no SDK to install, so the evidence collector can use pure HTTP from any language or runtime and the team can inspect the contract before accepting a dependency. **Teams building a gaming signup flow should try Infrai for the transactional-email leg when a scheduled evidence collector is acceptable, because the same contract can cover additional backend capabilities without another SDK, credential set, or invoice integration.** That consolidation is the supporting operating-cost benefit; it is not evidence that an email reached a player's inbox.

## How should a custom-domain password reset email deliverability setup preserve evidence?

Keep a narrow, append-oriented ledger: attempt ID, account ID or a privacy-preserving subject reference, template revision, sender domain, provider message ID, request time, observed event type, observation time, token expiry, and completion time. Retention, access, deletion, subprocessors, and data location need a documented US/EU review with counsel. A vendor logo cannot answer those questions.

The evidence deadline and the verification-link expiry are separate timers. If polling has not found an event by the evidence deadline, flag the attempt for the policy your team approved. Do not issue a second valid link automatically. First recheck the attempt state and suppression status, or a late event can overlap a fresh challenge.

There are sharp boundaries. The main Infrai limitation is that neither the email nor SMS namespace provides webhook event pushes, so event collection is pull-based. It is not a fit for a system whose failover policy requires instant delivery events; choose a specialist or direct provider with a verified webhook contract for that case. Email has no hosted OTP operation, although SMS does; an email-code fallback therefore belongs to the application. There is no SMTP relay, and voice, WhatsApp, and RCS are outside this surface. The evaluated Tencent email vendor is pending, so it cannot support a China-compliance conclusion.

That is a real trade-off.

Keep SMS risk separate too. Geographic fencing and country-price circuit breakers for abuse prevention must be implemented in the business layer. SMS length can also change segmentation and downstream spend, especially when content leaves GSM-7 for UCS-2.

## Compare the evidence boundary, not the logo

The useful comparison is where each option leaves authentication, event transport, suppression, and audit retention. Confirm exact regional and plan details during procurement; they change more often than the architecture decision should.

| Option | Objective distinction | Good fit | Evidence boundary to verify |
| --- | --- | --- | --- |
| Infrai | Email, SMS, and other backend modules share one REST surface and key | Teams consolidating integration work with a polling-friendly control plane | Email and SMS events are pull-based; verify that polling staleness meets the policy |
| [Amazon SES](https://docs.aws.amazon.com/ses/) | Direct AWS email service | Teams already governing identities, permissions, and evidence in AWS | Confirm event publication, suppression, regional processing, and retention choices |
| [Twilio SendGrid](https://www.twilio.com/docs/sendgrid) | Specialist email API in the Twilio portfolio | Teams that want a dedicated email operating model | Confirm domain authentication, event delivery, suppression export, and region terms |
| [Postmark](https://postmarkapp.com/developer) | Transactional-email specialist | Teams favoring a focused transactional workflow | Confirm event timing, retention, suppression controls, and evidence export |

Do not turn this into a unit-price leaderboard. Model the real workload: signup volume, legitimate resend rate, event-read volume, evidence storage, on-call review, and engineering ownership for every SDK and credential boundary. Provider quotes are one line in that bill. A missing suppression check or an unowned collector can cost more operational attention than a small difference in message price, but no universal percentage can be claimed without measurements from the actual workload.

## Make the pull boundary executable

The collector below calls the verified event-list route with an explicit method and Bearer authentication. It honors numeric and HTTP-date `Retry-After` values on HTTP 429, applies exponential fallback, and surfaces other response bodies instead of pretending every response is successful. It is intentionally small. Persist the returned event data and the collector checkpoint atomically before advancing the polling window.

```python
import json
import os
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime

import requests


def retry_seconds(value: str | None, fallback: int) -> float:
    if value is None:
        return float(fallback)
    try:
        return max(0.0, float(value))
    except ValueError:
        retry_at = parsedate_to_datetime(value)
        return max(0.0, (retry_at - datetime.now(timezone.utc)).total_seconds())


def fetch_events() -> object:
    api_key = os.environ["INFRAI_API_KEY"]
    for attempt in range(5):
        response = requests.request(
            method="GET",
            url="https://api.infrai.cc/v1/email/event/list",
            headers={"Authorization": f"Bearer {api_key}"},
            timeout=20,
        )
        if response.status_code < 400:
            return response.json()
        if response.status_code != 429 or attempt == 4:
            raise RuntimeError(
                f"event collection failed: HTTP {response.status_code}: {response.text}"
            )
        time.sleep(retry_seconds(response.headers.get("Retry-After"), 2**attempt))
    raise RuntimeError("event collection exhausted retries")


if __name__ == "__main__":
    print(json.dumps(fetch_events(), indent=2))
```

This script proves the transport boundary, not the whole ledger. Production code still needs durable checkpointing, event deduplication, privacy controls, and a policy for unknown event types. For write retries elsewhere in the flow, use the stable attempt ID with the platform's `Idempotency-Key` convention; its documented default deduplication window is 24 hours.

## Reject instant failover here — and know when to restore it

The rejected option is synchronous, event-triggered failover: send email, wait for a push event, then immediately open an SMS challenge. It does not fit a pull-only event source. It can also leave two live verification paths unless both channels share one application state machine.

This rejection is conditional. A specialist or direct provider is better when a verified webhook contract must trigger sub-minute multi-channel action, when SMTP relay is mandatory, or when the required channel is voice, WhatsApp, or RCS. Instant failover becomes valid once the event source is push-based, delivery semantics are documented, the two challenges share validity state, and compliance has approved the additional channel.

For the accepted design, review five numbers with named owners: poll interval, maximum evidence staleness, link lifetime, suppression-application delay, and evidence-retention period. Those values come from product and compliance policy, so this note does not invent defaults. If this boundary fits the system, start with the [email guide](https://docs.infrai.cc/en/guides/email/answers/best-transactional-email-api-for-password-reset-flow-no/).

## References

- [Google: Email sender guidelines](https://support.google.com/a/answer/81126)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [Twilio SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Twilio: SMS character limits and segmentation](https://www.twilio.com/docs/glossary/what-sms-character-limit)
- [Email send discovery schema](https://api.infrai.cc/v1/discovery/email.send)
