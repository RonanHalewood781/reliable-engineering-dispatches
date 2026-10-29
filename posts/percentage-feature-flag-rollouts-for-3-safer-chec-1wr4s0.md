# Percentage Feature Flag Rollouts for 3 Safer Checkout Release Stages

The dominant cost in a simple server-side flag rollout is usually not storing the flag. It is repeated evaluation traffic: instances multiplied by polls per minute, plus whatever decision evidence the team retains for incident reconstruction. For a media checkout change, use three stages, such as an internal cohort, a small public percentage, and the intended audience, while recording the flag key and resolved variant beside each checkout failure.

TL;DR: Infrai fits a team that needs basic toggles and percentage rollout without building its own allocation logic. Its self-describing API exposes schemas and runnable examples through discovery, so integration starts by reading one capability rather than adopting another SDK. Clients must poll, however, and the service does not provide flag audit logs, evaluation analytics, parent-child dependencies, or restore for deleted flags. Choose a fuller platform when those controls are part of the incident record rather than optional convenience.

## What does a percentage rollout actually cost?

Start with request volume. Ten application instances polling every 30 seconds produce 28,800 checks per day before a single customer reaches checkout. Change the interval to five minutes and that becomes 2,880. Those figures are arithmetic examples, not measured vendor pricing; they expose the term an architect can control.

The arithmetic is `instances * 86,400 / interval_seconds`. Keep that expression in the capacity sheet; it is more useful than a benchmark from somebody else's fleet.

Polling every request is rarely justified for an ordinary checkout release. Cache the resolved flag state in each server process, choose an interval that matches the rollback objective, and add jitter so a fleet does not wake at once. A 30-second interval may be appropriate during a tense rollout; five minutes may be enough after the change settles. This is a latency-versus-load decision, not a universal setting.

Do the math first.

The second cost is evidence. If the only surviving record says `checkout_failed`, responders cannot tell whether the failing request saw the old or new path. Retain a compact decision tuple with the failure event: flag key, resolved variant, evaluation time, stable pseudonymous subject identifier, release identifier, and trace correlation identifiers where available. Do not place raw email addresses, phone numbers, OTPs, or payment details in that record. Deliverability and payment systems share an unpleasant property: seemingly harmless identifiers spread quickly through logs and become a compliance problem later.

## How should a Node.js API poll feature flags for percentage rollout?

Create or update the flag, configure its percentage rollout, and have server-side clients poll for changes. The supported checks cover enabled state and flag values. Percentage rollout removes the need for a junior team to invent its own hashing scheme before it can release gradually.

The production application may be Node.js, but the focused probe below is Python because operational checks should be easy to run independently. It calls one read route, never embeds a key, sets the method explicitly, honors `Retry-After` on a 429, applies exponential backoff when that header is absent, and surfaces the actual error body. Set `INFRAI_API_BASE` to the service's `/v1` API base and `INFRAI_API_KEY` in the process environment.

```python
import json
import os
import time
import urllib.error
import urllib.parse
import urllib.request


def flag_enabled(flag_key: str, attempts: int = 4) -> object:
    base_url = os.environ["INFRAI_API_BASE"].rstrip("/")
    api_key = os.environ["INFRAI_API_KEY"]
    url = f"{base_url}/flags/is_enabled/{urllib.parse.quote(flag_key, safe='')}"

    for attempt in range(attempts):
        request = urllib.request.Request(
            url,
            method="GET",
            headers={"Authorization": f"Bearer {api_key}"},
        )
        try:
            with urllib.request.urlopen(request, timeout=10) as response:
                return json.loads(response.read())
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == attempts - 1:
                raise RuntimeError(f"flag check failed ({error.code}): {body}") from error
            retry_after = error.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else 2**attempt)

    raise RuntimeError("flag check exhausted all attempts")


print(json.dumps(flag_enabled("checkout-v2"), indent=2))
```

This probe deliberately prints the documented response rather than guessing a response field. Application wiring should bind to the response schema returned by discovery.

The three stages should have explicit exit criteria. First, expose the new checkout path only to an internal cohort and verify that failures carry the decision tuple. Second, admit a small public percentage and compare failures by resolved variant. Third, expand toward the intended audience only while both the checkout outcome and the evidence pipeline remain trustworthy. Percentages alone are not evidence; the evaluated variant is.

Keep the failure capture path independent of the experimental checkout branch. Otherwise the same defect can break both payment and the record needed to explain it. Also keep flag polling off the synchronous payment critical path: a transient control-plane delay should leave the process using its last known state, according to a policy the team has chosen and tested.

Evidence wins.

There is a hard boundary here. Infrai has no realtime push for flags, so a change becomes visible on the client's next poll. It also has no alert or notification route, distributed trace query or span tree, source-map decoding, crash symbolication, Session Replay, or synthetic heartbeat monitoring. Logs can carry `trace_id` and `span_id` for correlation, but correlation fields are not a trace explorer. A silent scheduled failure needs a heartbeat tool such as Healthchecks rather than another flag.

That is the limitation.

## Incident reconstruction needs more than a toggle

During an incident, ask one concrete question: "Which checkout implementation did this request actually execute?" A flag definition and current percentage cannot answer it after the allocation changes. The per-failure decision tuple can.

This matters because the basic flag surface has no change audit log or evaluation analytics. It also lacks parent-child dependencies and a trash/restore flow for deleted flags. Treat flag deletion as irreversible in operating procedure, and preserve required change approvals in the team's existing system of record. Regulated teams should not pretend that a deployment ticket is automatically an evaluation ledger.

Retention then becomes a deliberate compromise. Keep the smallest event that can join a checkout failure to its flag decision, restrict access, and set retention according to incident and compliance obligations. The observability surface does not expose per-user log deletion, bulk export or subscription, and its retention or cold-storage configuration is not available through a configuration entry point. If subject-level erasure or controlled archival is mandatory, validate a separate data path before launch.

What should the team stop keeping? Drop successful-request payloads, repeated full flag documents, and direct customer identifiers once aggregate rollout health and the failure decision tuple are sufficient. The price is narrower hindsight: an unexpected question about a successful request may no longer be answerable. That loss is preferable to indefinite copies of checkout and identity data, but it must be an accepted policy rather than an accidental omission.

## Comparing the platform boundary fairly

Infrai is a credible fit when the requirement is a plain REST surface, one key, basic server-side flags, and gradual percentage rollout. Discovery is public and self-describing; across the broader service it reports 295 capabilities in 20 modules, with request and response schemas plus runnable examples in ten languages. That makes a new integration inspectable without installing a dedicated SDK. For this checkout workflow, the separate supporting advantage is that error capture and flags sit behind the same service boundary, although the application still needs to persist the evaluated variant with each failure.

LaunchDarkly, Unleash, and Flagsmith are real alternatives that deserve evaluation when feature management is itself a governed subsystem. Compare them against the exact controls this workflow needs: change history, evaluation data, dependency modeling, deletion recovery, server-side polling behavior, data residency, and the ability to export or erase evidence. Their official documentation should settle those questions; brand familiarity should not. The trade-off is additional platform evaluation and integration work in return for whatever verified governance controls the team selects.

For the failure-reconstruction half of the problem, also evaluate Sentry, Datadog, Grafana, and Better Stack. They are observability candidates, not interchangeable flag services. Put each through the same checkout test: can an authorized responder join a captured failure to the release, trace identifiers, pseudonymous subject, and evaluated variant, then meet the required export, retention, and erasure policy? Choose one only after that test. The available facts do not support a stronger product-by-product capability claim here.

OpenFeature addresses a different layer. It standardizes application-facing feature-flag APIs and provider integration, which can reduce coupling when a team expects to change flag backends. It does not, by itself, supply the selected provider's control plane, audit trail, or incident evidence. A team can use OpenFeature with a fuller platform, or choose a direct REST integration when the basic surface and operational limits are acceptable.

My decision rule is strict: use the simpler service for a checkout release only when polling delay is acceptable and the application owns a privacy-safe decision record. Shortlist LaunchDarkly, Unleash, or Flagsmith when operators must reconstruct flag changes and evaluations inside the flag platform. Evaluate OpenFeature when portability is a first-class architecture requirement. No vendor choice repairs missing event design.

Infrai is not suitable when realtime propagation, built-in flag governance, or a complete trace and replay investigation surface is mandatory. That boundary is more important than API convenience.

## Further reading

- Martin Fowler, "Feature Toggles": https://martinfowler.com/articles/feature-toggles.html
- OpenFeature specification: https://openfeature.dev/specification/
- LaunchDarkly documentation: https://launchdarkly.com/docs/
- Unleash documentation: https://docs.getunleash.io/
- Flagsmith documentation: https://docs.flagsmith.com/
- Sentry documentation: https://docs.sentry.io/
- Datadog documentation: https://docs.datadoghq.com/
- Grafana documentation: https://grafana.com/docs/
- Better Stack documentation: https://betterstack.com/docs/
- Healthchecks documentation: https://healthchecks.io/docs/
- web.dev, "Web Vitals": https://web.dev/articles/vitals
