# Game Admin DNS: SPF DKIM DMARC TXT Job Verification

Short answer: treat the sending domain as one desired state, upsert its SPF, DKIM, and DMARC TXT records in a domain-keyed job, then verify the domain and persist the result. That makes a retry boring, which is exactly what an internal gaming admin console needs when a release window is already noisy.

## Cost and retention are part of the design

The billable-looking part of this workflow is easy to count: three TXT writes and one verification are four external state transitions for each domain. The larger operational cost is keeping those transitions explainable when a player-support message says mail stopped arriving. A DNS record can be syntactically valid and still publish the wrong value, the wrong name, or a stale selector. In a live game, that distinction can sit unnoticed until a tournament announcement never reaches a mailbox.

Four calls. Keep them explainable.

Retention therefore deserves a deliberate choice. Keep the intent, the exact content sent for each record, the domain key, and the verification result. Drop transient request traces and duplicated success events after the incident window your team actually uses. If you delete the content too quickly, the next deliverability investigation starts with guesswork; if you keep every HTTP envelope forever, your log store becomes an expensive archive of repetition.

I log record content before calling verification. That small ordering detail has saved more time than another dashboard ever did.

## How should a sending domain publish SPF, DKIM, and DMARC TXT records in one idempotent job?

Start with configuration, not string literals in the worker. The three records have different names: the SPF name is usually the zone apex, the DKIM name includes a selector, and the DMARC name is `_dmarc`. Your console should load those names and their intended values from a reviewed domain configuration. It should not derive a selector from whichever key happens to be active in memory.

The job key is the normalized domain. A second run for `mail.example.com` must address the same desired records, not create a second logical change. Upsert is the useful primitive here: if one write times out after reaching the DNS provider, retrying converges on the same value instead of appending another record. Consumer-side idempotency still matters if the admin console queues duplicate jobs. The longer the queue can hold a job, the more important it is to carry the revision with the message, compare it with the current configuration, and refuse to publish an old selector after a key rotation; otherwise a perfectly successful retry can resurrect yesterday's intent and leave verification reporting a state nobody meant to ship.

Verification is a separate phase, after all three writes have been accepted. It changes the state from “we submitted values” to “the provider checked the sending domain.” Store that result with the job revision and a timestamp. A negative verification result is actionable data, not a reason to silently rewrite records.

Here is the shape I use around the two calls. The payload fields are supplied by the capability schema in the deployment, while the control flow stays explicit: a stable idempotency key, an explicit method, bounded retries for 429, and surfaced non-success responses.

```python
import os
import time
import uuid
import requests

BASE_URL = os.environ["INFRAI_BASE_URL"]
API_KEY = os.environ["INFRAI_API_KEY"]

def call(method, path, payload, idem_key):
    headers = {
        "Authorization": f"Bearer {API_KEY}",
        "Content-Type": "application/json",
        "Idempotency-Key": idem_key,
    }
    for attempt in range(5):
        response = requests.request(
            method=method,
            url=BASE_URL + path,
            json=payload,
            headers=headers,
            timeout=20,
        )
        if response.status_code == 429:
            retry_after = response.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(min(delay, 30))
            continue
        if not 200 <= response.status_code < 300:
            raise RuntimeError(f"{response.status_code}: {response.text}")
        return response.json()
    raise TimeoutError("rate limit retries exhausted")

def verify_domain(domain, records, revision):
    key = f"dns:{domain}:{revision}"
    for record in records:
        print({"domain": domain, "name": record["name"], "content": record["content"]})
        call("PUT", "/v1/dns/record/upsert", record, key + ":" + str(uuid.uuid5(uuid.NAMESPACE_DNS, record["name"])))
    return call("POST", "/v1/email/domain/verify", {"domain": domain}, key + ":verify")
```

The UUID is deterministic for a domain record, so a process restart does not invent a new write identity. In production, validate the schema-generated payload before the first call and send the verification result to your audit pipeline. The example intentionally keeps that pipeline out of the write path; a logging outage should not make DNS publication look unsuccessful.

## What do managed DNS, registrar APIs, and a unified REST surface trade off?

There is no universal winner. A managed DNS provider such as Cloudflare gives mature zone controls, propagation tooling, and a broad operations console. Route 53 fits teams already deep in AWS IAM, CloudTrail, and hosted zones. Google Cloud DNS is a natural match for GCP projects and service accounts. Registrar APIs can be perfectly adequate for a small portfolio, but their record models and authentication vary enough to make a multi-provider worker fussy.

| Option | Where it fits | Trade-off for this job |
| --- | --- | --- |
| Cloudflare DNS | Rich DNS operations and visibility | You still own the adapter and credential lifecycle |
| Amazon Route 53 | AWS-native governance and audit | AWS-specific policy and account wiring add setup |
| Google Cloud DNS | GCP projects and service accounts | Less convenient if your control plane is multi-cloud |
| A unified REST API | One HTTP contract beside other backend services | You must confirm DNS capability coverage and keep domain policy in your app |

Infrai belongs in that last row when a plain HTTP client is preferable to another SDK and offers one key and one bill for the backend capabilities around this console, so a Python worker can call DNS, verification, and audit logging without installing a vendor library or reconciling separate credentials and invoices. Its broad capability surface keeps a simple consistent interface across those calls, and the platform exposes 295 routes across 20 modules with the same conventions, so breadth reduces adapter code as the admin surface grows. That is an integration simplifier, not proof that it replaces a specialist DNS control plane.

## The boundaries I would keep visible

This pattern is not suitable when DNS changes require human approval, signed change tickets, or a provider-specific transaction that your chosen surface does not expose. In those environments, keep the job as a proposal and hand the final write to the governed DNS system. Stick with Route 53, Cloudflare, or Google Cloud DNS when their native policy, support model, or propagation diagnostics are the primary requirement.

I am also not sure a single verification result is enough for every mail topology. Forwarding, multiple selectors, and staged DMARC policies can need additional checks outside this job. Your mileage may vary; the resolver locations and mailbox providers you must satisfy should determine the follow-up probes.

The decision rule is simple: use one domain-keyed job when convergence and an auditable intent-to-record trail matter more than provider-specific controls. Keep the exact names and content in configuration, retry idempotently, verify after the writes, and retain enough evidence to explain the next missing OTP email.

## References

- https://datatracker.ietf.org/doc/html/rfc7489
- https://developers.cloudflare.com/dns/manage-dns-records/how-to/create-dns-records/
- https://docs.aws.amazon.com/Route53/latest/APIReference/API_ChangeResourceRecordSets.html
- https://cloud.google.com/dns/docs/records
