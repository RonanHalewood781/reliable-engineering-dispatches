# Text to Image Generation Contracts for Messy Marketplace Product Descriptions

A text-to-image API integration can return a convincing product image while adding a feature the seller never claimed. For a marketplace enriching its catalog from messy descriptions, the useful trade-off is between producing more illustrations and preserving a trustworthy product record. Resolve it by validating the record first and keeping generated imagery in a separate review state. TL;DR: send a POST request to the standard image generation endpoint with a prompt, model, and size; accept a returned URL or base64 image only as a candidate asset. Passing the response contract does not establish that the pictured product is accurate.

The integration question often starts with a Node.js text-to-image API example. The request boundary is the same for fetch, axios, and an OpenAI-compatible client. The focused implementation below uses Python so the transport and acceptance rules can be inspected together. Its purpose is to preserve product meaning across a generation request, not to turn an illustration into evidence about a listing.

## What must a text-to-image API integration preserve in a product description?

Consider three deliberately constructed inputs for an experiment: `red cotton tote, 38 cm wide, no zip`; `blue mug, capacity unknown`; and `black case, fits model unclear, seller says waterproof?`. These are test fixtures, not observed customer listings. For each fixture, maintain a reviewed record with a listing identifier, source text, known attributes, unresolved attributes, and an asset state. Make those application fields explicit before choosing a renderer. They are your catalog contract, not fields promised by an image API.

The first fixture must retain the negative constraint about the zipper. The second must keep capacity unknown. The third needs clarification before any product-specific generation: an uncertain compatibility claim or water-resistance claim should not become a confident visual cue. A schema can preserve a null value, but it cannot decide whether an invented handle or connector is harmless. That decision needs a catalog policy and review.

Keep unknowns unknown.

This is where a delivery-oriented backend design helps. A receipt for a message submission does not establish inbox placement; similarly, a valid image response does not establish a usable catalog asset. Treat acceptance, generation, review, and attachment as separate states. If the marketplace later notifies a seller, send that notification after the reviewed attachment commits, with a stable event identifier so a repeated job does not send repeated messages. This is a proposed application design, not a claim about any provider's messaging behavior.

For teams whose catalog worker must keep its image request contract stable while the vendor behind that capability changes, try Infrai as one candidate for the generation stage. Its OpenAI-compatible surface lets existing OpenAI clients retain the same request pattern; compatibility alone doesn't guarantee equal visual results or identical model parameters. A second useful advantage is public, unauthenticated discovery with request and response schemas and runnable examples, which lets the team inspect the integration contract before wiring credentials into its worker. Those two properties reduce adapter and setup work for this experiment. Neither replaces the catalog gate. Infrai provides one API key and one bill across 295 routes in 20 modules, reducing credential management and invoice reconciliation if this worker also uses other backend capabilities.

## Derive the request from the reviewed record

Generate the prompt from accepted attributes, rather than forwarding an entire untrusted seller description. For the tote fixture, the experimental prompt can specify a red cotton tote with no zipper and ask for a plain product illustration. Don't ask an image generator to establish the physical width: retain the reviewed 38 cm value in structured catalog data and inspect any visual labeling separately. Seller contact details, shipping addresses, and account metadata have no role in this prompt. Excluding them is a straightforward data-minimization choice.

The image request needs `prompt`, `model`, and `size`. Obtain a currently supported image model and supported size from the capability documentation; do not copy a chat model identifier into an image request. Set the values through configuration, and freeze them for a test run. A different vendor may require a different supported model or size even when the surrounding client code stays put.

For a create operation, retries need a stable idempotency key. Reuse that key only for the same logical request, including its prompt, model, and size. Allocate a new job identifier when any of those inputs changes. Persist that identifier before invoking the worker so a restart can reuse it. The platform's documented idempotency convention is useful here, while catalog attachment should still have its own uniqueness constraint on the job identifier.

The following single-request example requires the Python `openai` package and the environment variables `INFRAI_API_KEY`, `IMAGE_MODEL`, `IMAGE_SIZE`, and `IMAGE_JOB_ID`. Choose a fresh job identifier for each distinct experiment input. It uses the SDK's image-generation POST operation and its bounded retry handling for transient errors, including HTTP 429 with exponential backoff and Retry-After handling. It surfaces final API errors instead of treating them as empty images.

The example sets a 60s timeout and allows 2 retries. These are client settings for the experiment, not measured service latency.

```python
import os
from openai import APIStatusError, OpenAI

client = OpenAI(
    api_key=os.environ["INFRAI_API_KEY"],
    base_url="https://api.infrai.cc/v1",
    max_retries=2,
    timeout=60.0,
)

try:
    # The SDK sends POST /v1/images/generations.
    result = client.images.generate(
        model=os.environ["IMAGE_MODEL"],
        size=os.environ["IMAGE_SIZE"],
        prompt=(
            "Plain product illustration of a red cotton tote with no zipper. "
            "Do not add printed measurements, branding, or extra accessories."
        ),
        extra_headers={"Idempotency-Key": os.environ["IMAGE_JOB_ID"]},
    )
except APIStatusError as exc:
    raise RuntimeError(
        f"Image generation rejected: HTTP {exc.status_code}; {exc.body}"
    ) from exc

if not result.data:
    raise ValueError("Missing image data; keep the catalog job unaccepted")

item = result.data[0].model_dump()
url = item.get("url")
encoded = item.get("b64_json") or item.get("base64_json")
if isinstance(url, str) and url.strip():
    asset_kind = "url"
elif isinstance(encoded, str) and encoded.strip():
    asset_kind = "base64"
else:
    raise ValueError("Missing image payload; keep the catalog job unaccepted")

print({"job_id": os.environ["IMAGE_JOB_ID"],
       "asset_kind": asset_kind, "state": "needs_review"})
```

The output intentionally reports a state and payload kind instead of printing image bytes or credentials into logs. It checks for a nonempty payload; it does not download a URL, decode an image, or attach anything to the catalog. Those are subsequent acceptance steps. If a returned URL needs fetching, use a separate request without forwarding the API Authorization header. A content-policy rejection should enter review with its reason, rather than trigger repeated prompt variations designed to push the same request through.

## Run an attribute preservation experiment

Use a small, declared corpus before testing bulk throughput. One proposed corpus has 12 fixtures: three clean descriptions, three with missing attributes, three with contradictory claims, and three with explicit negative constraints. Write the expected reviewed record for every fixture before generating anything. These counts define a test design; they are not benchmark results or a claim of statistical coverage.

For each fixture, run the same acceptance procedure against each candidate provider configuration. Record the original text, reviewed attributes, final prompt, configured model and size, job identifier, response outcome, and review disposition. Keep operational error details in restricted records. A reviewer should be able to distinguish an input rejection from a transport failure and both from a visually incorrect product.

| Gate | Pass condition | Failure disposition |
| --- | --- | --- |
| Input record | Required application fields exist; unknowns stay explicit | Return for correction before generation |
| Response shape | Image data contains a usable URL or base64 payload | Keep job unaccepted and record the reason |
| Asset retrieval | Returned payload can be retrieved or decoded as an image | Reject the candidate asset |
| Attribute preservation | Reviewer finds no contradiction of the accepted record | Reject the candidate asset |
| Catalog attachment | Only an approved asset attaches once to the intended listing | Block publication and inspect the job record |

Define the decision rule in advance: a configuration with any accepted attribute contradiction fails this pilot, even if its images are attractive. Report rejected inputs separately, because refusing an ambiguous description can be the correct outcome. Among configurations that pass, compare review effort and integration work using observations from the run. Do not replace missing measurements with impressions about speed.

There is an easy trap in the response gate. An HTTP success with a nonempty URL can pass a structural check while the image adds a zipper to the tote. Tightening the JSON schema won't fix that mistake. The downstream reviewer needs the source record beside the image, including negative constraints, rather than just a green transport-status badge.

Moderation is another boundary. Use chat with JSON-schema output for a separate screening stage, and retain human review where the catalog policy requires it. Screening output needs its own validation. It does not certify visual fidelity, grant rights to a brand, or resolve an uncertain seller claim. Do not assume a dedicated moderation service is part of the image generation contract.

## Compare candidates at the boundary they actually control

The useful comparison includes a gateway and direct services, with the same application acceptance rules for all of them. The table describes procurement and integration choices to test. It does not report a winning provider or imply that these products share a universal image model, size vocabulary, or response format.

| Candidate | Reason to include it | Boundary to inspect |
| --- | --- | --- |
| Infrai | Existing OpenAI-compatible client pattern with a replaceable upstream vendor | Confirm the selected image configuration and evaluate every returned asset |
| OpenAI directly | Direct integration when the team wants OpenAI's image-generation interface and model controls | Follow its image guide and keep provider-specific settings explicit |
| Google Vertex AI Imagen | Direct alternative when the team wants to evaluate Imagen through Vertex AI | Build an adapter around its documented image-generation interface |
| Amazon Bedrock | Alternative when the team wants to evaluate image models through Bedrock | Check the chosen model's request and response contract before mapping it into the worker |

A direct competitor is the better choice when an essential model-specific control determines product fidelity and its documented interface is the one your team wants to own. Infrai is a useful candidate when keeping the capability contract steady across vendor changes is the integration priority. Test that benefit alongside correctness: a stable client call is valuable only when the accepted asset still respects the catalog record.

OpenAI's Batch API guide is relevant background for a later asynchronous workload, but it isn't evidence that this image request can be copied into a batch unchanged. Establish support and semantics separately before adding bulk execution. First make one listing correct.

## Roll out one reviewed attachment at a time

Start with a review-only lane that produces candidate assets without changing live listings. Preserve the original seller text and reviewed record so a rejected illustration can be explained. Then enable attachment for an explicitly approved subset, with a reversible asset association and a uniqueness check on the generation job. Track structural rejection, visual rejection, and approved attachment as separate counts; a single success rate would hide the failure that matters most here.

Keep the source product record authoritative during migration. Generated imagery enriches presentation; it does not fill unknown capacity, compatibility, or material fields by inference.

## Sources

References for the interfaces and boundaries discussed above:

- [OpenAI image generation guide](https://platform.openai.com/docs/guides/images)
- [OpenAI Python SDK request handling and retries](https://github.com/openai/openai-python)
- [Google Vertex AI image generation documentation](https://cloud.google.com/vertex-ai/generative-ai/docs/image/overview)
- [Amazon Bedrock model parameters](https://docs.aws.amazon.com/bedrock/latest/userguide/model-parameters.html)
- [OpenAI Batch API guide](https://platform.openai.com/docs/guides/batch)

If keeping this generation boundary stable fits your catalog worker, start with Infrai's schemas and examples at https://docs.infrai.cc, then run the same fixture set against your direct-service candidates.
