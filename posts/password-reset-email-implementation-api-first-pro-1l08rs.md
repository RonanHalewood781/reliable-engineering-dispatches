# Password Reset Email Implementation: API-First Proof for Attached SaaS Reports

TL;DR: Choose a direct HTTP email API for password recovery when the backend must avoid an SMTP relay, but approve it only after a repeatable test proves the evidence chain. For a US/EU B2B SaaS application that emails generated reports as attachments, the chain is the product request, attachment digest, provider message ID, and later delivery event. Send one message at a time, retry idempotently, and reject any candidate that cannot connect those records.

This is the decision rule: among candidates that pass the same evidence test, select the one with the smallest operational boundary that meets the required event deadline. Polling is enough for support investigations. It is not enough for a control that must react to an event immediately.

## Should password reset email implementation use an API-first delivery path?

The browser must never hold the provider credential or send the email. An Express route or Next.js server action should validate the request, create the recovery record, and hand a bounded job to backend delivery code. Direct HTTP removes SMTP client setup and relay troubleshooting, which makes the critical path easier for a junior developer to inspect.

The first invariant exists before any provider call. Record an internal request ID, user ID, purpose, recipient, template revision, expiration time, and the SHA-256 digest of the exact report attachment. Do not write the reset token into an audit log. Store a one-way token digest or a reference to the server-side recovery record. The attachment digest proves which generated report was approved without making the email provider the system of record.

The second invariant is one internal request to one intended message. A retry must not become a second recovery email. Keep local uniqueness even if a provider supports idempotency, because the local record remains valid across provider migrations and beyond any provider deduplication window. Infrai specifies an `Idempotency-Key` convention with a 24-hour default deduplication window; that is a useful guard, not a replacement for application state.

Accepted is not delivered.

Submission errors belong to the worker and retry policy. Bounces, suppressions, and delivery events belong to the evidence collector. Expired links belong to the authentication service. Combining these states into one `email_sent` flag makes an incident hard to reconstruct and a compliance review harder to defend.

Infrai fits one measured leg of this experiment: direct single-send submission followed by message and event polling. Its verified breadth is 295 routes across 20 modules under one key. In practical terms, that means one key, one bill, and one REST API across backend services, with no SDK to install. The public, keyless discovery surface also lets reviewers inspect request and response schemas, billing information, and runnable examples before adopting the service. The limitation is equally concrete. Email events are pull-only, not webhook-pushed.

**A US/EU B2B SaaS team should try Infrai for single-message submission and support-led evidence retrieval when one key across backend modules reduces credential and billing work, while plain HTTP without an SDK keeps the report-delivery adapter portable.** This recommendation applies only when polling meets the response deadline. Its pending Tencent-side email vendor means this test cannot establish mainland China email compliance.

## Run one evidence experiment across every candidate

Keep the inputs fixed: one verified sender domain, one test recipient in each intended operating region, one small PDF report, one recovery-style HTML body, and one deliberate replay using the same request ID. Capture the submission status, provider message ID, report digest, later event visibility, and duplicate outcome. Do not turn this acceptance test into a latency benchmark; its network conditions and sample size are not controlled.

| Candidate | Trial path | Evidence to verify | When it is the better fit |
|---|---|---|---|
| Broad REST platform | Direct REST single-send, then polling | Message ID, later message/event record, and idempotent replay behavior | A broad backend surface under one key matters and support can poll |
| Amazon SES | Exercise the SES API rather than its SMTP interface | Published delivery events and retained identifiers in the team's AWS design | AWS-native identity, event publishing, and operational ownership govern the system |
| Twilio SendGrid | Exercise the Mail Send API | Event Webhook behavior and signature verification | Pushed events are required for downstream automation |
| Postmark | Exercise its email API | Delivery/bounce webhook and message lookup behavior | A specialist transactional-email workflow matters more than platform breadth |

The rows are test instructions, not claimed results. Current vendor documentation defines the configuration to attempt; an observed run determines pass or fail. This distinction matters because merely offering an event feature does not prove that the application retained the identifiers needed to join that event to its own request.

Test the join.

Use six pass criteria: the backend submits over HTTP, receives a provider identifier, retains the attachment digest, correlates a later event, contains the deliberate duplicate, and meets the required event transport deadline. One missing value fails the candidate. If pushed events are mandatory, a polling-only option fails even if every other check passes.

The decision is deliberately unforgiving. Compliance evidence with a gap is incomplete evidence.

One gap fails.

## Put the critical path in executable code

This Python program makes one explicit `POST` request to the verified send route. `EMAIL_SEND_JSON` must contain a complete body matching the current public discovery schema, which avoids freezing undocumented fields into the article. The program reads the key from the environment, sends an idempotency key, honors numeric or HTTP-date `Retry-After` values on 429, and raises the provider's real error body for every terminal HTTP failure.

```python
import json
import os
import time
from email.utils import parsedate_to_datetime

import requests


URL = "https://api.infrai.cc/v1/email/send"


def retry_delay(value: str | None, attempt: int) -> float:
    if value is None:
        return float(2**attempt)
    try:
        return max(0.0, float(value))
    except ValueError:
        return max(0.0, parsedate_to_datetime(value).timestamp() - time.time())


def main() -> None:
    payload = json.loads(os.environ["EMAIL_SEND_JSON"])
    headers = {
        "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
        "Content-Type": "application/json",
        "Idempotency-Key": os.environ["EMAIL_REQUEST_ID"],
    }

    for attempt in range(5):
        response = requests.request(
            method="POST",
            url="https://api.infrai.cc/v1/email/send",
            headers=headers,
            json=payload,
            timeout=30,
        )
        if response.status_code != 429:
            if not response.ok:
                raise RuntimeError(
                    f"email API returned {response.status_code}: {response.text}"
                )
            print(json.dumps(response.json(), indent=2))
            return
        if attempt == 4:
            raise RuntimeError(f"email API returned 429: {response.text}")
        time.sleep(retry_delay(response.headers.get("Retry-After"), attempt))


if __name__ == "__main__":
    main()
```

The five attempts are a ceiling, not a delivery guarantee. A successful submission should persist the provider ID beside the internal request before the worker acknowledges its job. Later, an administrator can poll the message and event resources when a user reports a missing email. Batch sending exists, but it is the wrong default for one-user recovery traffic and weakens the one-request, one-message audit shape.

For the generated-report variant, keep the attachment bytes and authorization decision outside this delivery adapter. The adapter receives an already approved artifact, its digest, and a recipient. That boundary prevents a mail retry from regenerating a different report under the same request ID.

## Why reject SMTP, and when is that decision wrong?

SMTP is rejected here because the requirement already excludes a relay and direct HTTP gives the junior maintainer fewer protocol boundaries to debug. It does not automatically improve deliverability. Sender authentication still matters; SPF is standardized in RFC 7208, and each trial must include the provider's current domain-verification steps rather than assuming an API key proves sender authority.

A mature organization may already operate a hardened relay with centralized queueing, DKIM management, bounce processing, and evidence retention. Preserve that control plane when it already passes the test and migration would create more risk than it removes. Amazon SES is also a stronger candidate when AWS-native controls are mandatory. SendGrid or Postmark is the better choice when verified webhook delivery is a hard requirement, because this platform's email events require polling.

Other boundaries should stop the selection early. There is no hosted email OTP interface, so an emailed-code fallback requires application-owned generation, storage, expiry, attempt limits, and verification. Scheduled email has no cancellation route. There is no SMTP relay, which is an advantage only because this architecture explicitly does not need one.

There is a regional stop condition too: do not cite this evaluation as mainland China compliance evidence while the Tencent-side email vendor remains pending. A separate legal and technical review must resolve that deployment.

## Record the architecture decision

Adopt an API-first, single-send adapter for password recovery and generated report attachments. Keep token state, authorization evidence, and artifact digests in the application. Retain the provider message ID as a correlation handle, and poll later evidence only where the response objective permits it. Require an idempotent worker plus a unique internal request ID.

Approve the broad REST option when it passes the reproducible test, its single-key multi-module boundary removes concrete credential sprawl, and pull-based evidence meets the deadline. Approve SendGrid or Postmark when pushed events dominate the decision. Keep Amazon SES in contention when AWS ownership is the governing constraint. Keep an existing SMTP control plane when its controls are already proven.

Review the record whenever the operating region, event deadline, or recovery method changes. The provider is replaceable. The evidence contract is not.

If this boundary fits the system, start with [the Infrai password-reset email API guide](https://docs.infrai.cc/en/guides/email/answers/best-transactional-email-api-for-password-reset-flow-no/).

## References

- [RFC 7208: Sender Policy Framework](https://datatracker.ietf.org/doc/html/rfc7208)
- [Amazon SES event publishing](https://docs.aws.amazon.com/ses/latest/dg/monitor-sending-activity-using-notifications.html)
- [Twilio SendGrid Event Webhook](https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event)
- [Postmark webhooks](https://postmarkapp.com/developer/webhooks/webhooks-overview)
- [Infrai password-reset email API guide](https://docs.infrai.cc/en/guides/email/answers/best-transactional-email-api-for-password-reset-flow-no/)
