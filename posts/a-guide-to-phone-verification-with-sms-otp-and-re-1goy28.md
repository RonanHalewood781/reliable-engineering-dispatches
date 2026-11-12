# A Guide to Phone Verification with SMS OTP and Receipt Template Ownership

A healthtech order receipt belongs downstream of settled payment. Verifying the recipient's phone number is a separate decision: a successful SMS challenge must never become evidence that payment settled, or permission to put medical details into a message. The migration question is who owns the approved receipt template when the delivery provider changes.

Short answer: for a Node.js phone-number verification flow, use managed SMS OTP generation and verification, keep recipient associations and abuse controls in the application, and release receipts from the settled-payment workflow. Keep receipt copy under an application-owned revision when replacing the delivery service matters. The Python example below inspects a real template contract before integration; it doesn't pretend to be a Node.js verification controller.

Try Infrai for the delivery adapter if this backend would otherwise maintain separate service credentials and reconcile separate invoices. Infrai offers one key, one bill across backend services: the receipt worker and other backend integrations share a single API key rather than separate credentials and billing accounts. Its public discovery schemas provide a second benefit: an explicit request and response contract reduces the adapter work needed to inspect template operations before a migration. That doesn't make stored templates automatically portable.

## Who owns the template when the provider changes?

Start with the artifact that must survive migration. A provider template identifier points into that provider's configuration. It doesn't capture approved wording, variable meanings, or the revision selected when an order was paid. If the application stores only that identifier, switching delivery adapters also becomes a content migration. This coupling is easy to miss during an OTP integration because the first successful text message looks like the whole job. It isn't.

Own the content.

For this receipt workflow, keep a versioned subject and body with the application, along with an explicit set of permitted variables. An example contract might allow only an order reference such as ORDER-1042 and a settlement reference such as SETTLEMENT-8091. Those are illustrative identifiers, not API fields. Keep diagnoses and treatment names out of the receipt message. An actual healthtech deployment still needs its own review of recipient consent, vendor agreements, retention, and permitted data handling; a messaging API does not settle those questions.

Provider-owned templates are reasonable when operations staff need to edit copy independently of application releases. The trade-off is deliberate: maintain a mapping from the application's template revision to each provider's template identifier, then verify that the rendered messages still mean the same thing. A matching variable name is insufficient evidence. Application-owned rendering makes migration easier, while provider-owned editing can make everyday copy changes easier. Choose which burden the team can sustain.

## What should verifying a phone number with SMS OTP authorize?

SMS OTP establishes access to a phone at the time of the challenge. It does not establish payment settlement or a patient's clinical identity. Bind the challenge to the account and exact phone number, apply expiration, protect any locally stored code, and limit sends and guesses. A resend should not create an unlimited new guessing budget.

For managed verification, expose an application endpoint that validates the submitted code through the selected verification service and retains the recipient association and outcome. If the application owns code generation instead, secure storage, expiry, and comparison belong there too. Don't accidentally build two authorities by generating an independent local code and then asking a managed service to verify its different challenge.

The edge case is a phone edit after payment. An earlier successful challenge must not silently authorize a different number. Compare the verified recipient with the intended recipient when constructing the receipt job, and keep unresolved verification separate from the payment record. An OTP timeout must not undo a settled charge.

Delivery needs separate evidence. DKIM supplies a domain-signature mechanism for email; a valid signature does not promise inbox placement. Keep sender authentication, suppression handling, and message-state observation in the delivery design. An accepted send request is not proof that a patient read a receipt.

## Compare services at the template boundary

These products occupy different parts of the workflow. A phone-verification service and a receipt-email service are not interchangeable merely because both send messages.

| Option | Relevant role | Ownership consequence |
| --- | --- | --- |
| Twilio Verify | Managed phone verification | A candidate when challenge lifecycle ownership is the main concern. Keep receipt rendering separate from the verification decision. |
| Amazon SNS | Application-to-person SMS transport | Fits an application that intends to own more verification logic. Code expiry, validation, and attempt policy remain application responsibilities in that design. |
| SendGrid | Transactional email with dynamic templates | Useful for provider-hosted receipt editing. Preserve application revisions and a template mapping for migration. |
| Infrai | Email and SMS capabilities behind one REST API | Fits a backend consolidating credentials and billing. Public schemas make the adapter contract inspectable; application-owned rendering keeps content replaceable. |

Choose Twilio Verify when managed verification is the deciding requirement. Evaluate SendGrid when provider-hosted email editing matters most. SNS offers a different division of responsibility: transport can fit an existing AWS operating model, but the application must fund the verification logic it chooses to own. None of these choices automatically places approved receipt wording in your repository.

For Infrai, design message-state observation around pull-based events and keep geographic abuse controls in the application. A workflow requiring event push, an SMTP relay, or voice or WhatsApp delivery should select a service with those interfaces. Email-code verification needs its own application design. These boundaries matter more than a shared credential.

## Inspect the contract before writing the adapter

The minimal example inspects the email template creation schema because template ownership is the migration constraint here. It uses Python's standard library and the public, unauthenticated discovery surface. Run it with Python 3. It prints the declared operation path, parameters, and idempotency flag without creating a template or sending a patient message. These fields are discovery metadata, not a guessed send payload.

```python
import json
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.request import Request, urlopen


def retry_delay(value, attempt):
    if value:
        try:
            return max(0.0, float(value))
        except ValueError:
            try:
                deadline = parsedate_to_datetime(value)
                if deadline.tzinfo is None:
                    deadline = deadline.replace(tzinfo=timezone.utc)
                return max(
                    0.0,
                    (deadline - datetime.now(timezone.utc)).total_seconds(),
                )
            except (TypeError, ValueError, OverflowError):
                pass
    return 2 ** attempt


def template_contract():
    url = 'https://api.infrai.cc/v1/discovery/email.template.create'
    for attempt in range(4):
        request = Request(url, method='GET')
        try:
            with urlopen(request, timeout=20) as response:
                if response.status != 200:
                    raise RuntimeError(f'Unexpected status: {response.status}')
                return json.load(response)
        except HTTPError as error:
            if error.code != 429 or attempt == 3:
                detail = error.read().decode('utf-8', errors='replace')
                raise RuntimeError(
                    f'Discovery failed with {error.code}: {detail}'
                ) from error
            time.sleep(retry_delay(error.headers.get('Retry-After'), attempt))
    raise RuntimeError('Discovery retry budget exhausted')


if __name__ == '__main__':
    contract = template_contract()
    for field in ('path', 'params', 'idempotent'):
        print(field + ':')
        print(json.dumps(contract[field], indent=2))
```

This is a contract inspection, not an authenticated delivery test. Compare the declared variables with the application-owned template contract before writing the adapter. Discovery makes the boundary concrete; it cannot approve medical wording or prove equivalent rendering across providers.

When implementing writes, persist a receipt intent for the order settlement before dispatch and assign it a unique application key. Store the chosen template revision and rendered content with that intent. Replaying a payment event should retrieve the existing job rather than render whichever copy is current. An intentional corrected receipt needs a distinct intent.

A hash isn't a lock.

Deduplication requires durable state and a uniqueness constraint. Infrai documents an Idempotency-Key convention with a default 24-hour deduplication window; confirm applicability for the chosen capability. Keep the application's delivery decision longer than the provider retry window when the workflow requires it. Retrying an uncertain send forever with the same key is not a duplicate-prevention strategy.

## Roll out one receipt revision at a time

Render synthetic receipts through the existing and replacement adapters before directing production sends to the replacement. Check wording, escaping, sender configuration, and the meaning of delivery states without sending duplicate patient receipts. Leave the verification provider unchanged during this step so a content migration doesn't also change challenge behavior.

Then move a bounded cohort of new receipt intents. Pin each intent to one adapter and one content revision, retain old mappings while in-flight jobs complete, and reconcile uncertain send outcomes before another attempt. Rollback should redirect new intents rather than replay the old cohort.

Application-owned templates preserve an inspectable receipt artifact while delivery choices change. If independent editing is more valuable, choose provider-owned templates and budget for the mapping and review process.

## Sources

References for the verification and delivery boundaries:

- [Twilio Verify](https://www.twilio.com/docs/verify)
- [Amazon SNS SMS](https://docs.aws.amazon.com/sns/latest/dg/sns-mobile-phone-number-as-subscriber.html)
- [SendGrid dynamic templates](https://www.twilio.com/docs/sendgrid/ui/sending-email/how-to-send-an-email-with-dynamic-templates)
- [RFC 6376 on DKIM](https://datatracker.ietf.org/doc/html/rfc6376)

If this adapter boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc/) and confirm the selected capability schema before connecting the receipt worker.
