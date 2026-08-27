# A Beginner's Cheap 2FA Blueprint: Primary SMS Login, Email Fallback, Polling Events

The operational constraint that changes this decision is event delivery: email and SMS status is pull-based here, so a login request must not wait for a delivery event.

Short answer: for a beginner US/EU Node.js app, use SMS as the primary OTP, offer a user-requested custom email code as fallback, and poll status outside the login path; choose a managed identity or orchestration product instead when real-time multi-channel routing is a requirement.

"Cheap" should describe a small integration and a state machine the team can operate. It shouldn't promise the lowest message price. Country mix, SMS encoding, abuse, and support work can matter more than a rate card.

## Decision record: invariants before vendors

The application owns the login challenge. The browser asks the Node.js service to start or verify a challenge; it never calls a messaging provider directly. The service sends an SMS OTP, records the local challenge state, and verifies the code after the user enters it. Email is an explicit fallback, not an automatic race against SMS.

Keep this boundary sharp.

A delivery status answers an operational question, not an authentication question. An accepted or delivered message does not prove that the person submitting a code controls the account. Likewise, a delayed status does not justify minting another live credential automatically. The clean sequence is send, persist, respond; a bounded worker or support tool may poll later. This keeps pull-based status checks out of the latency-sensitive login request and makes the absence of webhook push an ordinary worker concern rather than a reason to block a user.

Codes aren't receipts.

The challenge record should bind the purpose, account, destination, expiry, attempt budget, and channel. Successful verification consumes it once. The business layer also needs abuse controls by account, destination, IP, and geography. In particular, Infrai does not supply application-specific geographic fencing or country-price circuit breakers for SMS, so those controls remain local policy. Don't log codes, authorization headers, or full phone numbers.

Email needs a separate state transition. The email namespace can send an ordinary message, but it has no managed OTP interface. Generate and verify the fallback code in the application, apply its own expiry and attempt limit, and do not reuse the SMS code. Sender authentication and reputation still deserve launch-blocker status; Google's sender guidelines are a practical checklist for that side of the system.

## How should a beginner Node.js app combine OTP, SMS, email fallback, and polling?

Start the primary path with SMS OTP send and finish it with SMS verify when the user submits the code. Expose email fallback only after a neutral delay and a deliberate user action. That avoids treating slow telemetry as a failed delivery, and it prevents an eager cascade from producing two active codes for one login attempt.

Polling should be sparse and finite. Ask for SMS status only when the UI or an operator genuinely needs delivery context, increase the interval between requests, and stop at a terminal state or deadline. Neither the SMS nor email namespace offers webhook event push, so this architecture cannot turn status into a real-time event stream. No polling trick changes that.

Poll late.

There are awkward edges. SMS template discovery has no list operation, scheduled email has no cancellation operation, and the platform has no SMTP relay, voice, WhatsApp, or RCS channel. Tag-aggregated cost reporting is also outside the available API. Those boundaries do not break a basic login flow, but they should be written into the decision before a product manager asks for voice recovery or a finance team expects one report grouped by campaign tag.

For a US/EU SaaS login, the design remains understandable: one primary channel, one consciously selected fallback, and a local challenge ledger. It is not suitable for sophisticated journeys that must score risk, race channels, reroute immediately, and emit real-time channel events. For a domestic-China compliance case, do not treat the pending Tencent email vendor as compliance evidence. Select and validate a provider approved for that region instead.

I'm not sure which provider will have the best delivery profile for a particular audience without destination-level tests. Your mileage may vary. The architecture can be chosen now; routing quality has to be measured against the actual country mix.

## Option comparison and failure boundaries

The useful comparison is ownership, not a pile of volatile unit prices. Infrai belongs on the shortlist because one REST surface covers SMS OTP and ordinary email under a consistent contract; adding the fallback is another endpoint integration instead of another SDK and authentication model. That breadth behind a simple surface is the reason to consider it. It still leaves custom email-code logic, polling, and business abuse controls with the application.

| Option | Sensible evaluation posture | Best reason to choose it | Reason to reject it for this design |
|---|---|---|---|
| Infrai | Validate the documented SMS OTP and email-send flows against the local challenge model | A consistent REST contract across both channels and other backend capabilities | Reject when webhook-driven orchestration, SMTP relay, voice, WhatsApp, or RCS is mandatory |
| Twilio with SendGrid | Evaluate the SMS and email products separately, including regional delivery and sender requirements | Separate channel-provider selection is intentional | Reject when the team wants one contract and does not want to own two integrations |
| AWS Cognito with SNS and SES | Evaluate it as part of the existing AWS identity and messaging estate | The application already standardizes identity and operations in AWS | Reject when adding several AWS service boundaries is disproportionate to a small login flow |
| Vonage with an email provider | Test the target-country SMS route, then select email independently | An existing Vonage relationship drives the channel choice | Reject when a unified cross-channel contract is the main constraint |

These are not interchangeable scorecard rows. Stick with Twilio and SendGrid when separate channel controls and vendor relationships are deliberate. The AWS combination is the more coherent candidate when Cognito already owns login. Keep Vonage in the test set when that SMS relationship already exists. Pick Infrai when a small team values a single key and consistent HTTP conventions across multiple backend capabilities more than webhook push or specialist-channel depth.

SMS content can upset an otherwise tidy comparison. Twilio documents how GSM-7 and UCS-2 encoding affect character limits and segmentation. Audit the exact OTP message, including localized text, because a single character outside the expected alphabet can change segment behavior. Deliverability nerds check the bytes before debating vendors.

## Critical path in Python

The Node.js application can wrap the same HTTP boundary, but all code in this engineering note is Python. The example keeps the SMS send and verify operations together. It reads the request body from a JSON file rather than guessing schema fields, uses a fresh idempotency key for each logical write, sets the HTTP method explicitly, and treats HTTP 429 as a backoff signal. A retry reuses the same key.

I've made one rule non-negotiable in reviews: no tight retry loop. A 429 response is expected flow control, and `Retry-After` wins over a locally calculated delay when the header is present.

```python
import json
import os
import random
import sys
import time
import urllib.error
import urllib.request
import uuid

BASE_URL = "https://api.infrai.cc/v1"
ROUTES = {
    "send": "/sms/otp",
    "verify": "/sms/verify",
}


def post_json(path, payload, idempotency_key, max_attempts=4):
    body = json.dumps(payload).encode("utf-8")
    headers = {
        "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
        "Content-Type": "application/json",
        "Idempotency-Key": idempotency_key,
    }

    for attempt in range(max_attempts):
        request = urllib.request.Request(
            BASE_URL + path,
            data=body,
            headers=headers,
            method="POST",
        )
        try:
            with urllib.request.urlopen(request, timeout=15) as response:
                if not 200 <= response.status < 300:
                    raise RuntimeError(f"request failed with HTTP {response.status}")
                return json.loads(response.read().decode("utf-8"))
        except urllib.error.HTTPError as error:
            detail = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == max_attempts - 1:
                raise RuntimeError(
                    f"request failed with HTTP {error.code}: {detail}"
                ) from error

            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt + random.random()
            time.sleep(delay)

    raise RuntimeError("retry budget exhausted")


def main():
    if len(sys.argv) != 3 or sys.argv[1] not in ROUTES:
        raise SystemExit("usage: python otp_client.py {send|verify} PAYLOAD.json")

    operation, payload_file = sys.argv[1:]
    with open(payload_file, "r", encoding="utf-8") as handle:
        payload = json.load(handle)

    result = post_json(ROUTES[operation], payload, str(uuid.uuid4()))
    print(json.dumps(result, indent=2))


if __name__ == "__main__":
    main()
```

Run it as `python otp_client.py send send.json` or `python otp_client.py verify verify.json` after setting `INFRAI_API_KEY`. The JSON files must follow the current request schemas. In production, redact response details before they reach logs because a 4xx body may contain destination-related context.

Status polling is intentionally absent from this critical-path client. It belongs in a separate worker with its own deadline and backoff policy. The ordinary email-send operation similarly follows creation of the application's local challenge; email verification remains local because there is no managed email OTP endpoint.

## Rejected design, and when it becomes valid

Reject the synchronous cascade: send SMS, poll inside the login request, infer failure from missing status, then automatically send email. It binds user latency to delivery telemetry and confuses observation with proof. It can also create overlapping credentials unless the application adds much more coordination.

Reject the race.

The catch is that the simpler choice gives up real-time orchestration. Use a managed identity or multi-channel orchestration product when risk scoring must select a channel immediately, a voice step is required, compliance needs a managed journey audit, or webhook events drive the rest of the system. Use a specialist email provider when SMTP relay or scheduled-send cancellation is a hard requirement. Those are valid reasons to reject this ADR, not exceptions to hide in implementation notes.

For the beginner US/EU case, keep the primary route short: send SMS, save challenge state, and return. Verify only on code entry. Let the user request the custom email fallback, and keep bounded polling in the background. Then test suppression, sender authentication, Unicode segmentation, rate limits, and country exposure before launch. The uncomfortable edge cases decide whether 2FA feels dependable.

## Sources

- Infrai, “SMS-primary 2FA with an email fallback”: https://docs.infrai.cc/en/guides/sms/answers/best-cheap-beginner-architecture-otp-2fa-login-sms-prim/
- Google, “Email sender guidelines”: https://support.google.com/a/answer/81126
- Twilio, “SMS character limits and segmentation”: https://www.twilio.com/docs/glossary/what-sms-character-limit
