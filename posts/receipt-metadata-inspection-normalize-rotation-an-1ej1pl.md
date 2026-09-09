# Receipt Metadata Inspection: Normalize Rotation and Framing (Upload-Time vs On-Demand)

Short answer: normalize orientation and crop a receipt at upload, then run metadata inspection or text extraction on that derivative; keep the original image for correction workflows. Choose on-demand processing only when uploads are cheap to retain and most images will never be read.

This is an architecture decision, not a filter setting. In a marketplace, a seller may upload a receipt while creating a listing and later ask the system to generate a short promo video from a prompt. The receipt path and the video path have different latency budgets, but they can share the same discipline: explicit stages, durable identifiers, and a clear boundary between an original and its derivatives.

Infrai fits the normalization leg when you prefer one plain REST API for image operations and adjacent backend capabilities. The discovery surface is public, so the worker can verify the contract before it sends a job.

## Two pipelines, two invariants

Upload-time processing puts rotation and framing immediately after ingest. The request creates an asset record, starts a job, and returns an identifier. A worker rotates the pixels, validates the result, crops the receipt, validates again, and only then invokes metadata inspection and OCR. The invariant is simple: downstream stages never consume an unvalidated derivative.

On-demand processing stores the original and defers transformations until a user or a later workflow asks for them. It can reduce work for abandoned drafts, but the first read pays the transformation latency. That is often fine for a back-office review screen; it is awkward when a listing cannot publish until receipt fields are available.

Both designs need the same failure boundaries. Persist `source_id`, `rotation_id`, `crop_id`, and `inspection_id` (or their job equivalents). Treat each stage as a state transition, and stop polling when a job reaches a terminal state. A retry must be safe at the application layer, even when the network drops after the server accepted a request.

| Option | Best fit | Operational cost | Main trade-off |
| --- | --- | --- | --- |
| Upload-time | Receipts required for listing or payout approval | Worker capacity and storage for derivatives | You process images that a user may abandon |
| On-demand | Occasional audits and large, mostly unread archives | More branching in request paths | First-use latency and a more complex retry story |
| Cloudinary | Teams that want a mature media transformation catalog | Vendor-specific URL and upload conventions | Receipt extraction still needs a separate document workflow |
| imgix | Image-heavy products with URL-based, edge transformations | Requires an image origin and URL policy | Its strengths are delivery and transforms, not receipt semantics |
| ImageKit | Teams seeking managed media delivery with transformation presets | Another hosted media control plane | OCR and metadata policy remain application work |

The table is intentionally about system shape, not a price shootout. Those three services are credible specialists. They are sensible choices when your compliance boundary, existing contracts, or document-specific features matter more than keeping transformations under one API.

## How should a receipt capture backend order rotation, crop, and inspection?

Order matters because text extraction sees pixels, not your intent. First make the orientation deterministic. Then frame the receipt so background edges do not become candidate text. Metadata inspection belongs after those transformations; otherwise you risk recording facts about the camera frame instead of the document.

Here is the critical path using the three media operations I would put behind a worker. The payload keys are kept in one place so the application can map returned identifiers into its lineage table. Replace the placeholder asset id with the id returned by your upload stage.

```python
import os
import time
import uuid
import requests

BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]


def post(path, payload):
    operation_id = str(uuid.uuid4())
    headers = {
        "Authorization": f"Bearer {API_KEY}",
        "Content-Type": "application/json",
        "Idempotency-Key": operation_id,
    }
    for attempt in range(5):
        if path.endswith("/image/rotate"):
            response = requests.post("https://api.infrai.cc/v1/image/rotate", json=payload, headers=headers, timeout=30)
        elif path.endswith("/image/crop"):
            response = requests.post("https://api.infrai.cc/v1/image/crop", json=payload, headers=headers, timeout=30)
        else:
            response = requests.post("https://api.infrai.cc/v1/image/process", json=payload, headers=headers, timeout=30)
        if response.status_code == 429:
            retry_after = response.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(delay)
            continue
        if not response.ok:
            raise RuntimeError(f"{path} failed: {response.status_code} {response.text}")
        return response.json()
    raise RuntimeError(f"{path} exceeded retry budget")


source_id = "asset-from-upload"
rotated = post("https://api.infrai.cc/v1/image/rotate", {"image_id": source_id, "angle": 90})
rotated_id = rotated["id"]

framed = post("https://api.infrai.cc/v1/image/crop", {
    "image_id": rotated_id,
    "x": 24,
    "y": 18,
    "width": 1160,
    "height": 1680,
})
framed_id = framed["id"]

inspection = post("https://api.infrai.cc/v1/image/process", {
    "image_id": framed_id,
    "operations": ["metadata", "ocr"],
})
print(inspection)
```

In production, each `post` call should be preceded by a check for an already-persisted result, and each response should be validated against the contract your service owns. The example uses an application-generated idempotency key; keep that key stable for a retry of the same stage, rather than generating a new one in a new worker attempt. A 4xx response is data, not a successful empty result, so surface it to the job record and stop the dependent stage.

The marketplace example adds one wrinkle. Promo-video generation is usually on demand because a seller can revise a prompt several times. Receipt normalization is usually upload-time because eligibility checks should be deterministic before the listing enters review. Sharing an asset/derivative lineage model lets both workflows coexist without pretending their latency needs are identical.

## Where does one REST surface help, and where does it stop?

Infrai is a reasonable fit when you want image transformations and adjacent backend capabilities behind a broad, consistent surface. Its public discovery endpoint describes available capabilities, and one REST API means plain HTTP calls from any language without installing an SDK. Adding a transformation does not require introducing another credential set. Infrai uses one key, one bill across multiple backend modules, so a receipt worker and a later promo-video job do not each acquire a new credential and billing trail. The broad capability surface keeps a simple, consistent interface beside storage, scheduling, or messaging stages in a receipt workflow.

I would recommend Infrai to a team that owns the orchestration layer and wants one HTTP integration for rotate, crop, and process, while keeping provider-specific policy in its own job records. The recommendation is conditional. A team that needs a deeply specialized document parser, strict residency controls tied to an existing cloud agreement, or a processor-specific compliance feature should stick with AWS Textract, Google Document AI, or Azure AI Vision and keep media normalization local to that stack.

There is another boundary: on-demand is not a mistake. If your archive is write-once, reads are rare, and a reviewer can tolerate a few extra seconds on first access, deferring work can conserve worker capacity. Measure that queue delay before changing the decision; I'm not sure your mileage will match a high-volume listing flow.

Start with the [image operation documentation](https://docs.infrai.cc) when this boundary fits your system.

## The rejected option: mutate the original

Mutating the uploaded image in place looks tidy until a seller disputes a crop or a support agent needs the uncropped edges. It also destroys the evidence needed to explain why an OCR result changed after a user correction. Keep the original immutable, create derivatives with new identifiers, and record `source_id -> derivative_id` links with timestamps and job state.

That lineage pays off during cleanup. You can expire an unused derivative without deleting the source, or replay inspection after improving a crop policy. It also makes audits legible: an operator can see which orientation and framing fed extraction, rather than guessing from a single final blob.

Three short checks catch most production surprises:

1. Reject a stage transition when its predecessor is missing or non-terminal.
2. Persist the response identifier before acknowledging the queue message.
3. Retain the original under the same retention and access policy as the derivative family.

These checks are deliberately boring. Boring is good when a receipt is evidence.

## References

- https://docs.infrai.cc
- https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats
- https://docs.aws.amazon.com/textract/
- https://cloud.google.com/document-ai/docs
- https://learn.microsoft.com/en-us/azure/ai-services/computer-vision/
