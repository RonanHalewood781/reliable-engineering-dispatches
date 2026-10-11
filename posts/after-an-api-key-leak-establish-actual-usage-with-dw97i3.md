# After an API Key Leak: Establish Actual Usage Within a Spend Ceiling

A leaked API key creates two competing risks for a B2B SaaS account platform: leaving the credential active can expand exposure, while revoking it blindly can refuse legitimate account, email, SMS, or OTP traffic. Resolve that trade-off by setting a short evidence-collection window under a hard spend ceiling, replacing the credential, and measuring the suspected key against a known-good baseline. **The blast radius is the intersection of identity, time, action, and affected object, not the total request count.**

TL;DR: preserve the evidence you already have, stop new use of the exposed secret, and build a usage series grouped by credential identity and operation. Treat missing attribution as uncertainty. If the ceiling is reached before attribution is complete, favor containment and explicitly record which legitimate traffic may be refused.

## What can logs establish after an API key leak?

Start with the credential itself. Record its stable identifier, owner, intended environment, expected callers, privileges, creation time, and rotation state. Do not copy the secret value into the incident record. OWASP's secrets-management guidance calls for lifecycle handling that includes creation, rotation, revocation, expiration, and auditing; those controls make an investigation possible without turning the incident notebook into another secret store.

Next, preserve access records covering a defensible interval before discovery and continuing through replacement. Useful fields are event time, credential identifier or a nonreversible lookup value, authenticated principal, operation, target account or tenant, result, source context, and request correlation identifier. The records must answer who did what and when. A status code alone cannot tell you which customer object was read or changed.

Keep two clocks in view: event time and ingestion time. A request can occur before revocation yet arrive in an analytics store afterward. Mark the extraction boundary and rerun the same query after the normal ingestion interval rather than silently treating the first result as final.

Sparse telemetry is still evidence, but it is evidence of a gap. Do not translate “no matching row” into “no access.”

## Derive the boundary before choosing the response

The first useful artifact is an operation-by-time series for the suspected credential, with denied and successful outcomes kept separate. Compare that series with the key's declared purpose. An account provisioning key that suddenly performs credential-management operations deserves attention even if its total volume is small; a burst of expected reads may instead reflect a retrying legitimate worker. Volume helps prioritize. Semantics establish impact.

For a multi-tenant service, enumerate distinct affected tenant and object identifiers from successful events, then join them to the operation class. Keep reads, mutations, administrative actions, and outbound-message requests separate because their follow-up differs. An OTP send may create delivery and abuse concerns. A profile read has a confidentiality question. Collapsing both into “API calls” destroys the decision signal.

Use half-open intervals, such as `start <= event_time < end`, so adjacent investigation windows do not double-count boundary events. Freeze the query inputs in the incident record: timezone, start, end, key identifier, included environments, and the definition of success. Small details matter here.

The following Python sketch operates on already-preserved, normalized records. It deliberately returns sets and counts rather than claiming that a count proves exposure.

```python
from collections import Counter, defaultdict
from datetime import datetime


def summarize(events, key_id, start, end):
    """Build an evidence summary for one credential and half-open UTC window."""
    operations = Counter()
    outcomes = Counter()
    tenants_by_operation = defaultdict(set)
    correlated_requests = set()

    for event in events:
        occurred_at = datetime.fromisoformat(event["occurred_at"])
        if event["key_id"] != key_id or not (start <= occurred_at < end):
            continue

        operation = event["operation"]
        outcome = event["outcome"]
        operations[operation] += 1
        outcomes[outcome] += 1

        if outcome == "success":
            tenants_by_operation[operation].add(event["tenant_id"])
        if event.get("request_id"):
            correlated_requests.add(event["request_id"])

    return {
        "operations": dict(operations),
        "outcomes": dict(outcomes),
        "tenants_by_operation": {
            operation: sorted(tenants)
            for operation, tenants in tenants_by_operation.items()
        },
        "correlated_request_count": len(correlated_requests),
    }
```

This summary is a lead sheet, not a verdict. Validate high-risk mutations against the system of record and follow correlation identifiers into the relevant service logs. Deduplicate retries by a stable request identifier when one exists, but retain the raw event count: retries can explain spend and rate-limit pressure even when they represent one intended action.

## Where should the spend ceiling force containment?

Define the ceiling before the drill. It can be an approved maximum for incremental request volume, outbound-message attempts, analyst time, or another resource the organization already governs. The exact number belongs to the business, so copying a universal threshold would be false precision.

The decision rule should be written plainly: continue the narrow observation window only while doing so materially improves attribution and remains below the approved ceiling. Revoke or disable sooner when the credential can perform high-impact actions, evidence shows unexpected successful operations, or reliable attribution is unavailable. **Known refused traffic is recoverable; unbounded unauthorized activity is not an acceptable experiment.**

There is a practical wrinkle for account platforms. Revocation may interrupt sign-ins, invitations, password resets, or OTP delivery, and a rushed retry policy can amplify the load. Prepare a replacement credential through the normal secret-distribution path, shift known callers, then revoke the exposed value. Monitor refused requests by credential identifier and caller so the recovery team can distinguish an attacker from a missed deployment.

Do not stretch the observation window merely to make a chart look complete. Stop means stop.

## Turn uncertain logs into an explicit blast-radius statement

A useful incident statement has three parts: confirmed impact, excluded scope, and unresolved scope. For example, confirmed impact might name the tenant identifiers and successful mutation classes found in the evidence window. Excluded scope should list only boundaries supported by records, such as an environment the credential was not authorized to enter. Unresolved scope names missing log intervals, absent object identifiers, late events, or callers that share an identity.

That structure prevents two common errors. The first is reporting every request as a compromised customer action. The second is presenting incomplete telemetry as proof that nothing happened. Both are attractive under pressure, and both fail scrutiny.

Build a small evidence matrix in the incident record:

| Claim | Primary evidence | Corroboration | Remaining uncertainty |
|---|---|---|---|
| Credential was used in the window | Authenticated access event | Credential lifecycle record | Late-arriving events |
| Tenant object was changed | Successful mutation event | Current object state or audit record | Side effects outside retained fields |
| Legitimate caller was refused after revocation | Denied event for old key | Deployment and caller identity | Shared caller identity |

Keep denied attempts in the report, but do not count them as successful object access. They remain operationally relevant because they show continued use and can expose missed callers after rotation.

## Roll out the drill without rehearsing a real outage

Begin with a credential whose ownership and expected operations are documented. Confirm that logs expose a credential identifier without exposing its value, then generate a small, authorized sequence containing a read, a mutation, and a denied attempt in a non-production environment. Run the same bounded query the incident team will use, rotate through the approved secret-management process, and verify that the old credential is refused while the replacement serves the intended caller.

Only then exercise production response procedures. Use a synthetic marker that can be located and removed, keep the spend ceiling active, and assign one person to make the containment decision. The drill passes when the team can produce a reproducible evidence window, enumerate confirmed objects and operations, state uncertainty, replace the secret, and identify legitimate refused traffic. A polished dashboard is optional.

Afterward, close telemetry gaps as incident-response defects: missing credential attribution, ambiguous operation names, inconsistent timestamps, or retention shorter than the investigation need. Do not preserve secret values to make future searches easier. The durable fix is accountable secret lifecycle management plus auditable use.

## Sources

- https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
