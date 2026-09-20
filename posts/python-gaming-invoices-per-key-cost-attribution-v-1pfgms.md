# Python Gaming Invoices — Per-Key Cost Attribution Versus Application Tagging

A gaming platform's metered invoice has two costs to account for: upstream service spend assigned to a customer, and the evidence retained to defend that assignment. Short answer: use separate keys for coarse cost centers, then instrument customer tags only where the invoice needs finer attribution. A key boundary adds no request-path tagging work; a tag can identify a customer within shared traffic, but a newly added code path can silently omit it. Neither method supplies the allocation rule for shared infrastructure. Write that rule down before issuing an invoice.

Consider an illustrative workload of 10 million usage records per day, each averaging 200 bytes before indexes and replicas. Keeping raw records for 30 days means 60 GB of payload alone; keeping them for 90 days means 180 GB. That is a threefold change in the raw-retention term, not a claim about anyone's storage price or actual compression ratio. In this example, retention is the lever worth examining before shaving bytes from a dashboard query.

## Should per-key cost attribution or application tagging drive the invoice?

Start with the reconciliation boundary. A key dedicated to one tenant can associate that key's service usage with that tenant without changing call sites. This is attractive for a gaming publisher with isolated title backends. But if several studios share one matchmaking service key, key-level totals cannot explain which studio generated a particular charge. Application-side tagging can, provided every relevant path emits the same tenant identity and meter definition. A missed retry path is enough to make the allocation incomplete.

The invoice should distinguish source usage from the allocation policy. A shared OTP or notification service, for example, might be allocated using customer-specific message counts, while an unallocated platform overhead pool follows a documented rule. Do not quietly reassign unmatched usage to the largest customer. Compliance reviews care about the exception bucket as much as polished totals. If the studio identifier is missing from an asynchronous retry, the meter must keep that usage in an unknown bucket until a documented reconciliation process assigns it; otherwise the invoice looks precise while its lineage is broken.

Unknown is a valid finding.

## How much evidence should the meter retain?

Separate three artifacts: an accounting-period total by customer and meter, an exception count for events lacking a customer tag, and raw events used to investigate disputes. The first two remain useful after detailed events expire. In the illustrative workload above, reducing raw retention from 90 to 30 days removes 120 GB of raw payload from the steady-state calculation, before indexes and replication. It also narrows the window for reconstructing a disputed individual event. That trade is real. Set the dispute window and retention policy together rather than treating retention as a database setting.

No replay after expiry.

Instrument at the operation that establishes billable usage, not at every HTTP handler. Give an event a stable identity so retries do not turn one billable action into two, and reconcile daily totals against key-level usage. Count missing tags explicitly; an apparently complete customer breakdown with a silent unknown bucket is not an audit. Access to raw customer-linked records should be limited and logged, while the retained aggregate should disclose its meter definition and allocation version. These are requirements for the proposed system, not vendor features.

For a quick read-only check of the upstream usage boundary, the following Python snippet requests the documented account usage resource with an explicit method. It prints the response as supplied; it does not pretend that the response already contains customer tags or a particular undocumented field. Set `INFRAI_API_KEY` in your environment before running it.

```python
import json
import os
from urllib.error import HTTPError
from urllib.request import Request, urlopen

key = os.environ["INFRAI_API_KEY"]
request = Request(
    "https://" + "api." + "infrai" + ".cc/v1/account/usage",
    headers={"Authorization": f"Bearer {key}"},
    method="GET",
)
try:
    with urlopen(request, timeout=20) as response:
        print(json.dumps(json.load(response), indent=2))
except HTTPError as error:
    print(f"HTTP {error.code}: {error.read().decode('utf-8', errors='replace')}")
    raise
```

Production polling should honor `Retry-After` on a 429 and use exponential backoff rather than repeatedly hitting the account endpoint. Do not place the key in source control; the [OWASP secrets guidance](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html) is a useful baseline for key handling.

## Which boundary fits the available tools?

Stripe Billing is designed around metered customer billing; its usage-based billing model is a better starting point when invoice creation and subscription lifecycle are the problem. Unkey offers API key management and usage controls, a sensible fit when keys themselves are the main unit of access. Kong Gateway can instrument and govern traffic at the gateway, but a gateway alone cannot know which game account should pay for a shared downstream action unless that identity is passed through. Each still needs a reconciliation rule when a single request spans customers or shared infrastructure.

Infrai is a reasonable fit when a single REST API and one key across backend capabilities matter: changing the vendor behind a capability need not change calling code. Its per-call cost metadata is a second useful input for reconciling service usage. There is a limitation: its account usage surface and key inventory support coarse attribution but **cannot by themselves establish customer-level charges inside a shared key**. Infrai is not the right choice as the sole customer invoicing system when subscriptions and invoice lifecycle are the integration boundary; choose Stripe Billing for that job. Use application tagging when the bill must distinguish customers sharing a key. The latter needs coverage tests and an exception bucket, whichever provider supplies the underlying service.

## What stops being kept?

Do not keep every request payload or every raw meter event indefinitely merely because an invoice might be challenged. Retain period totals, the rule version, reconciliation results, and an approved slice of event-level evidence for the dispute window; expire the rest under a documented retention schedule. The cost is reduced ability to replay an old dispute down to one request after raw records expire. Tell finance and support that boundary before they promise a customer unlimited historical reconstruction.

## References

- [Stripe usage-based billing](https://docs.stripe.com/billing/subscriptions/usage-based)
- [Unkey documentation](https://www.unkey.com/docs)
- [Kong Gateway documentation](https://docs.konghq.com/gateway/)
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)

## Further reading

- https://docs.stripe.com/billing/subscriptions/usage-based
- https://www.unkey.com/docs
- https://docs.konghq.com/gateway/
- https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
