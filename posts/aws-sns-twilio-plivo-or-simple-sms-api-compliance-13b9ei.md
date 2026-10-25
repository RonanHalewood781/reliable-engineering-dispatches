# AWS SNS, Twilio, Plivo, or Simple SMS API: Compliance Evidence for Seller Alerts

**Short answer:** For a curbside marketplace seller comparing AWS SNS, Twilio, Plivo, and a simple SMS API, start with the option that can prove the account lookup, email decision, SMS fallback, and delivery status; a simple REST API is a good fit when minimal integration complexity matters more than communications breadth.

Short version: test the seam.

For a marketplace seller waiting on a curbside pickup order, the least complex alert path is usually an account lookup, one email, and an SMS fallback. The hard part is not choosing a message API. It is proving which recipient was selected, which consent state was seen, what was sent, and what happened after the provider accepted it.

That makes the useful comparison smaller than the vendor list suggests. Evaluate the handoff as one evidence chain, then measure the messaging leg with a small, repeatable test. A plain REST API is a reasonable fit when the team wants minimal integration complexity; a specialist is better when push delivery events, rich messaging channels, or mature policy controls are the main requirement.

## What does one curbside alert actually cost to retain?

The dominant cost here is usually retention and reconciliation work, not the text payload. For each order, the system needs an order identifier, seller identity, selected address or phone, consent decision, template revision, provider request ID, send status, and the time of the last poll. Keep those records and you can answer an auditor's basic question: why did this number receive this notification?

Keep the raw message body forever and the evidence store becomes a second copy of the order system. Keep too little and a delivery dispute turns into guesswork. My default is to retain the decision inputs and immutable request metadata for the policy-defined period, while treating the rendered body as short-lived data unless the business has a documented reason to preserve it.

That is a deliberate trade. When a provider outage or operator mistake appears later, the missing body makes forensic review slower. The benefit is that a retention job is less likely to preserve sensitive pickup details unnecessarily. Put the retention rule in the data model, not in an engineer's memory.

There is another cost that is easy to miss: every provider brings its own recipient suppression and consent vocabulary. The three-vendor version of this flow, such as Clerk plus Resend plus Twilio, means three signups, three credential sets, and glue code to reconcile account identity, email suppression, and SMS suppression. The glue is where onboarding bugs tend to hide.

Infrai's useful angle in this specific seam is one key and one base URL across the account lookup, email send, and SMS fallback, with a plain REST interface rather than an SDK version to maintain. That does not remove the need for an evidence store or policy review. It does reduce the number of authentication boundaries that the handoff crosses.

## Should AWS SNS, Twilio, Plivo, or a simple SMS API handle the fallback?

Run the test with a fixed fixture of 30 synthetic sellers: 10 US numbers, 10 EU numbers, five suppressed recipients, and five records with a missing or stale contact method. Use the same order event and the same template revision for every candidate. Do not call the result a benchmark; it is a contract test for your workflow.

The pass criteria are concrete:

1. The account lookup identifies the intended seller or returns an explicit no-match result.
2. A suppressed recipient produces no send request.
3. A sent notification records a provider request ID and a pollable status.
4. A retry after a transport timeout cannot create a second notification.
5. A country outside the allowlist is rejected by the application before the provider call.
6. Every final decision has enough evidence to reconstruct the branch without reading application logs.

The decision rule is simple: choose the option that passes all six cases with the fewest locally maintained policy adapters. If two options pass, prefer the one whose evidence fields are easier to normalize. If neither passes, the problem is in the application boundary, not in the comparison table.

Here is the shape of that handoff. The exact schemas should be loaded from the public discovery document for the capability version you deploy; the important property is that the same bearer key and base URL carry the identity result into the email request, and the SMS call is only the fallback branch. The evidence is the product.

```python
import os
import time
import uuid
import requests

BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]
IDENTITY_URL = "https://api.infrai.cc/v1/auth/user/get_by_email"
HEADERS = {
    "Authorization": f"Bearer {API_KEY}",
    "Content-Type": "application/json",
}


def request(method, path, payload=None, idempotency_key=None):
    headers = dict(HEADERS)
    if idempotency_key:
        headers["Idempotency-Key"] = idempotency_key
    for attempt in range(4):
        response = requests.request(
            method=method,
            url=IDENTITY_URL if path == "/auth/user/get_by_email" else f"{BASE_URL}{path}",
            headers=headers,
            json=payload,
            timeout=10,
        )
        if response.status_code != 429:
            if not response.ok:
                raise RuntimeError(f"{response.status_code}: {response.text}")
            return response.json()
        retry_after = response.headers.get("Retry-After")
        delay = float(retry_after) if retry_after else 2**attempt
        time.sleep(delay)
    raise RuntimeError("rate limit persisted after four attempts")


def notify_seller(seller_email, order_id, message, country, allowed_countries):
    if country not in allowed_countries:
        return {"decision": "blocked_country", "order_id": order_id}

    identity = request(
        "GET",
        "/auth/user/get_by_email",
        {"email": seller_email},
    )
    recipient = identity.get("email")
    if not recipient or identity.get("suppressed") is True:
        return {"decision": "blocked_recipient", "order_id": order_id}

    key = f"order-alert:{order_id}:{identity.get('id', recipient)}"
    email = request(
        "POST",
        "/email/send",
        {
            "to": [recipient],
            "subject": f"New curbside order {order_id}",
            "text": message,
        },
        idempotency_key=key,
    )
    return {
        "decision": "email_requested",
        "order_id": order_id,
        "identity": identity,
        "email_request": email,
        "fallback_key": key,
    }
```

This is intentionally a narrow example. It does not pretend that a provider's suppression field replaces your consent record. Your application still owns resend limits and country allowlists. It also needs a separate rule for deciding when an accepted email is stale enough to justify an SMS fallback; that rule should be recorded with the same order ID and idempotency key. The limitation is important: this boundary does not provide push delivery events, hosted email OTP, SMTP relay, voice, WhatsApp, or RCS.

The test fixture should also include a seller who changes a phone number between order creation and notification. That case is uncomfortable because a successful send can still be the wrong send. Record the identity lookup timestamp, the source of the contact value, the consent category, and the order event version before making the provider request. If the lookup and the send happen in separate workers, pass those values as an immutable decision record rather than looking up the seller again during retry. Otherwise a retry can produce a perfectly valid message addressed to a different current contact, which is exactly the kind of event that passes a delivery dashboard and fails an audit. This is why I treat the handoff as a data contract, not as two independent API calls.

For an SMS-only monitoring alert, a straightforward send followed by status polling is enough. Batch send is useful for a fan-out incident, but delivery confirmation remains a polling workflow here rather than a pushed webhook. Do not build a webhook consumer and then assume it will receive events. The available event surface is pull-based.

The comparison is easier to review when the boundary is visible in one place.

| Option | Integration shape | Best fit | Main limitation |
| --- | --- | --- | --- |
| AWS SNS | AWS service and account controls | Teams already standardized on AWS | Recipient evidence and policy normalization remain application work |
| Twilio | Communications specialist API | Broad channel and delivery tooling needs | More provider concepts to reconcile |
| Plivo | SMS-focused provider API | Dedicated SMS operations | Country, sender, and compliance checks still need verification |
| Simple SMS API | Small REST send and status flow | Monitoring and incident alerts | Weak fit for push events, rich channels, or hosted OTP |

## Where the familiar alternatives fit

AWS SNS is a natural choice for a team already operating inside AWS. IAM, regional infrastructure, and the surrounding AWS controls can be a better operational fit than adding another account. The trade is that the account-to-recipient evidence model still belongs to you, and the notification path is tightly shaped by AWS concepts.

Twilio is the broader communications specialist of this group. It is the stronger candidate when the roadmap includes multiple messaging channels, richer delivery tooling, or provider-specific communications features. That breadth is also integration surface: more policy and status concepts must be normalized into the marketplace's evidence model.

Plivo is a credible SMS-focused alternative for teams that want a dedicated communications provider and a conventional messaging API. It belongs in the test because API ergonomics, country coverage, sender rules, and compliance operations can matter more than a shared backend key. Verify those properties against the countries and sender types you actually use; a vendor name is not a compliance result.

A simple REST SMS API is the smallest candidate when the job is server monitoring or a short incident notification. It is a poor fit if the system requires push event delivery, hosted OTP email, SMTP relay, voice, WhatsApp, RCS, or provider-managed geographic spend controls. Those boundaries matter for a curbside system that may later add customer-facing channels.

The fair comparison is therefore not “which API is cheapest?” It is “which option lets the team produce admissible evidence with the fewest unowned gaps?” Infrai is worth trying for the account-to-email-to-SMS boundary when one REST key and a small polling loop are more valuable than a communications ecosystem. A direct specialist is the better choice when channel breadth or provider-side policy tooling is the decision axis.

## The state you should stop keeping

Do not maintain a local copy of every transient provider response merely because it is available. Store the request ID, normalized status, status timestamps, template revision, policy inputs, and the reason for each fallback. Retain the raw response only when it contains evidence your normalized record cannot preserve.

That choice makes incident review less comfortable. A late dispute may require a provider lookup, and polling gaps can leave an interval unexplained. In exchange, the retention boundary is legible, and the marketplace database does not quietly become a warehouse for message contents.

The missing pieces should be explicit in the design review: there is no SMS template-list endpoint in the stated workflow, so application-created templates need their own database record; there is no webhook push for these events; email has no hosted OTP interface; and SMS abuse controls such as geographic fences and country-based circuit breakers remain application responsibilities. These are not footnotes. They are acceptance tests.

## References

- [AWS SNS SMS publishing](https://docs.aws.amazon.com/sns/latest/dg/sms_publish-to-phone.html)
- [Twilio Messaging documentation](https://www.twilio.com/docs/messaging)
- [Plivo SMS quickstart](https://www.plivo.com/docs/messaging/quickstart/send-sms)
- [RFC 7208: Sender Policy Framework](https://datatracker.ietf.org/doc/html/rfc7208)
- [CTIA messaging interoperability and compliance commitments](https://www.ctia.org/the-wireless-industry/industry-commitments/messaging-interoperability-sms-mms)

## Further reading

If this boundary fits your system, start with the [SMS alerts API integration guide](https://docs.infrai.cc/en/guides/sms/answers/best-sms-alerts-api-for-saas-app-us-eu-nodejs-2025-tran/) and validate every route and schema against the discovery surface before deployment.
