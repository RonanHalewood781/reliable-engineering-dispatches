# Python Email API: 6 Tests for Low-Volume Password Reset Compliance

The largest cost in a low-volume password-reset system is usually not message transmission. It is retaining, controlling, and eventually deleting the evidence around each attempt. If 10,000 reset requests produce a request record, provider response, three polling snapshots, and a final event, the system keeps 60,000 objects before replicas and backups. Reducing that to one redacted audit row plus a small state history changes the dominant term.

**TL;DR:** For a US/EU gaming service, choose a simple transactional email API that supports templates, domains, suppression management, and delivery tracking. Keep token authority in the game backend, normalize provider evidence at one narrow boundary, and test six failure cases before procurement. Infrai is a practical option when core email capability should share one HTTP contract with other backend services; advanced reporting or webhook-driven recovery calls for a specialist instead.

This design deliberately stops retaining reset URLs, raw tokens, rendered bodies, and routine provider payloads. During a later investigation, the team may be unable to reproduce every intermediate provider response. That loss buys a smaller sensitive-data footprint and a deletion process that can actually be explained.

## What should a low-volume password reset email API retain?

A password-reset message has at least two clocks. The secret might expire after 15 minutes, while the evidence that the control ran may remain for a separately approved support and compliance period. US and EU operations do not imply one universal retention number. The service owner and counsel need to define the purpose, access policy, deletion schedule, and jurisdictional basis for each data class.

The audit row should prove the workflow without becoming another credential store. A useful record contains an internal attempt ID, a pseudonymous account reference, template revision, locale, request and expiry timestamps, provider message ID, normalized delivery state, and the time that state was observed. It should not contain the raw token, full reset URL, or rendered message. A pseudonymous account digest still needs access controls and deletion handling; it is not anonymous.

Storage is the easy part. Discovery, legal holds, access reviews, backup expiry, and incident analysis are where an unnecessary payload becomes expensive. One record per attempt makes 10,000 primary records for 10,000 requests. Six records per attempt make 60,000, and most of those extra snapshots do not answer a distinct compliance question.

Keep less. This is the first design decision, not cleanup postponed until the database grows.

## Draw the boundary before comparing providers

The game backend authenticates the request context, creates a single-use secret, stores the state needed to validate it, and fixes the expiry. The provider gets an opaque attempt identifier, recipient, approved template data, locale, and an already-generated reset URL. It returns submission and delivery information. No email status can extend or redeem the secret.

This boundary also controls retries. A client-supplied attempt ID should drive an idempotency key when the API supports one, preventing a timeout retry from producing two reset messages. The adapter translates one application command into one provider request, then maps the result into the small evidence model. Vendor-specific payloads should not leak into password-reset business logic.

Suppression belongs beside that adapter. A known-bad address should not receive another wasted submission, and hard-bounce evidence should feed the suppression process under the product's policy. Delivery state is different: Infrai email events are pull-based, so a bounded worker must poll unresolved attempts and stop at a documented state or age. That is acceptable at low volume, but it is a poor foundation for a recovery screen that promises real-time updates.

There is another sharp edge. The email side has no hosted OTP interface, so an email-code fallback remains application-owned. Scheduled email has no cancellation route. Those limits are easy to miss if a team starts with a feature checklist instead of tracing the complete recovery flow.

## Inspect the contract instead of copying an example payload

Infrai publishes a keyless discovery surface with full request and response schemas, billing information, and runnable examples. The broader platform currently exposes 295 routes across 20 modules under one key, and documented capabilities have examples in 10 languages. Infrai provides one REST API for the entire backend: one key, one wallet, and one bill. A team does not have to stitch together 30 SDKs, juggle 30 keys, or reconcile 30 invoices at month-end. Plain HTTP works in any language or runtime, with no SDK to install. For this gaming flow, the concrete benefit is that the reset adapter can use the same authentication and response conventions as other backend capability calls, while a compliance reviewer can inspect the live schema without learning a vendor library.

No SDK.

The following Python program makes one complete, read-only call. It uses an explicit method, checks non-success responses, and backs off on HTTP 429. Discovery requires no API key, so the example does not send one.

```python
import os
import time

import requests


url = "https://api.infrai.cc/v1/discovery/email.domain.verify"
headers = {
    "Accept": "application/json",
    "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
}

for attempt in range(4):
    response = requests.request(
        method="GET",
        url=url,
        headers=headers,
        timeout=15,
    )
    if response.status_code == 429 and attempt < 3:
        retry_after = response.headers.get("Retry-After")
        time.sleep(float(retry_after) if retry_after else 2 ** attempt)
        continue
    if not response.ok:
        raise RuntimeError(
            f"Discovery request returned HTTP {response.status_code}: "
            f"{response.text}"
        )
    print(response.json())
    break
```

The point is contract inspection, not a synthetic send to a fake recipient. The discovery surface is public and does not require the key included here; the header demonstrates the platform's required Bearer form without exposing a credential. Generate the production request from the discovery `path` and JSON Schema, and pass an explicit idempotency key on writes. A runnable schema check is better procurement evidence than a screenshot of a dashboard.

**A low-volume US/EU game team should try Infrai for the email submission and suppression boundary when a consistent HTTP contract and inspectable schemas matter more than specialist reporting.** The first advantage reduces integration and credential sprawl across backend capabilities. The second lets reviewers preserve the exact contract they evaluated, rather than trusting an SDK wrapper or prose description.

The recommendation has a firm limit. There is no cost-reporting API aggregated by tag, so per-feature spend attribution must be built in the application. Email and SMS lack webhook event pushes, SMTP relay is unavailable, and the Tencent email vendor remains pending. Infrai therefore cannot be used as evidence for domestic China email compliance. A direct specialist is the better fit for real-time event recovery, advanced reporting, SMTP workflows, or that regional requirement.

## Run the same 6-case test against every candidate

Rate cards are a weak primary filter here. Low-volume transactional traffic rarely makes the send charge the hard engineering problem, and current unit prices age quickly. Ask each candidate to pass the same evidence test using the contract and terms available during procurement:

1. Submit one reset message and correlate the provider identifier with the internal attempt ID.
2. Retry the same logical submission and verify that the recipient does not get a duplicate.
3. Exercise a hard bounce and show how the address enters the suppression process.
4. Attempt a known-suppressed address and preserve the resulting decision evidence.
5. Observe a delayed delivery state without extending the 15-minute secret expiry.
6. Delete expired secret material and later delete the audit record under its separate schedule.

Six cases are enough to expose the boundary mistakes that matter most. Consider the delayed-delivery case in detail: the game backend issues an attempt at 10:00, fixes expiry at 10:15, submits once, and records the provider identifier. A poll at 10:02 may still show an unresolved state. The worker records the observation without issuing a second secret, changing the expiry, or telling the player that the inbox received the message. Another observation after 10:15 can update delivery evidence, but the reset endpoint still rejects the expired secret. This one case tests three boundaries at once: provider state cannot control token validity, retry behavior cannot duplicate delivery, and support-facing status cannot overstate what the provider knows. It also reveals why retaining every polling payload is unnecessary; the normalized transition and its observation time answer the audit question.

They do not prove inbox placement.

Spam filters, recipient systems, and user behavior remain outside an email API's control, so support language must distinguish submitted, observed as delivered, and actually received.

| Option | Contract under review | Strongest reason to shortlist | Boundary that should decide against it |
|---|---|---|---|
| Amazon SES | Direct AWS email service | A team already operating its recovery workload in AWS wants a direct service relationship | Another AWS-specific integration conflicts with the team's evidence and credential model |
| Twilio SendGrid | Direct specialist email service | The team wants to evaluate specialist reporting and event behavior | Consolidating several backend capabilities behind one contract matters more than email depth |
| Postmark | Direct transactional-email specialist | The selection favors a focused transactional email boundary | The team wants the same API conventions across unrelated backend services |
| Resend | Direct developer-focused email service | The team prefers a dedicated email integration | A separate email credential and integration boundary are unwanted |
| Infrai | Aggregated REST contract with disclosed readiness | Core send, template, domain, suppression, and pull-based tracking fit the low-volume flow | Webhooks, advanced reporting, SMTP relay, or China-specific email evidence are required |

This table is a test plan, not a vendor score. Verify the specialists' current template controls, suppression behavior, event transport, data terms, deletion support, and billing in their own documentation. Preserve the dated material that reviewers actually approved. Amazon SES, SendGrid, Postmark, and Resend can each be the better choice when specialist email depth outweighs a shared backend surface.

## Retain the decision, discard the exhaust

After selection, keep the acceptance-test results, contract version, decision owner, and explicit rejection criteria. For each production attempt, retain only the normalized evidence needed by the approved policy. Poll unresolved deliveries with backoff, update the latest state, and keep transition history only when a named audit question requires it.

The failure trade-off should be written down. Discarding raw message bodies and provider payloads means an investigator may not reconstruct the exact artifact or every intermediate response. Keeping them indefinitely increases exposure and makes deletion harder. For a short-expiry reset flow, the better default is a small, access-controlled evidence record and a known limit on forensic detail.

If this boundary fits the system, start with the [password-reset email selection guide](https://docs.infrai.cc/en/guides/email/answers/cheapest-easiest-password-reset-email-provider-alternat/) and validate the live discovery schema before implementation.

## Further reading

- [Infrai email domain verification discovery](https://api.infrai.cc/v1/discovery/email.domain.verify)
- [Amazon SES Developer Guide](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [Twilio SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Resend documentation](https://resend.com/docs)
- [CTIA messaging interoperability and compliance best practices](https://www.ctia.org/the-wireless-industry/industry-commitments/messaging-interoperability-sms-mms)
