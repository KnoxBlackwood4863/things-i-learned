# Real Estate Listing Image Pipeline API: Node.js Upload-Time Beats On-Demand for 6 Sizes

Choose upload-time processing for the six public image slots in a real estate listing, and retain the immutable original behind a private object key. The deciding constraint is consistency: search results, listing pages, agent dashboards, email alerts, and social previews should not discover a new crop, orientation, or encoder result while serving a request. On-demand rendering remains useful for uncommon editorial exports, but it should not define the public listing contract.

Short answer: accept one source, validate and normalize it once, publish named derivatives only after every required variant succeeds, and let Node.js return stored metadata rather than resize pixels in the request path. This makes a missing derivative a visible workflow state instead of a late surprise to a buyer opening a listing.

This is an architecture decision, not a preference for a particular image service. The API owns stable slot names and state transitions. A replaceable worker owns decoding and encoding. Object storage owns bytes; the database owns which exact objects are allowed to appear.

## Should a real estate listing image pipeline API resize at upload?

Yes, for a finite set of public sizes. The source is immutable. Replacing a kitchen photo creates a new asset version rather than overwriting an existing key, because an overwritten object can leave caches and database rows referring to different bytes under the same identity. The previous version can be retired by policy, but it must not quietly mutate. This approach keeps crops consistent while originals are kept apart from public derivatives.

Orientation is applied before width, height, and crop coordinates are calculated. This matters because JPEG camera metadata can describe an orientation that is not represented by the stored pixel matrix. If one path honors that metadata and another ignores it, the same upload can become portrait in the agent preview and landscape in search.

Six is deliberate.

The public slots are named by use, not by an arbitrary requested width:

1. `search_square`, 480 by 480, cover crop
2. `search_card`, 640 by 480, cover crop
3. `listing_main`, 1280 by 960, contain
4. `listing_zoom`, 1920 by 1440, contain
5. `email_card`, 600 by 450, cover crop
6. `social_share`, 1200 by 630, cover crop

Those dimensions are an example contract, not a universal real estate standard. The useful property is the finite, versioned set. A team can change the numbers after measuring its layouts, but a change creates a new recipe version and new object keys. Old pages keep resolving while the backfill runs.

Publication is atomic at the application boundary: a listing photo becomes `ready` only when the database records the original, the recipe version, and all required derivative keys. Partial work stays `processing` or becomes `failed`; it never leaks a mixture of generations into a gallery. Keep the original private, strip unnecessary metadata from public derivatives, and authorize any source download separately. Property photographs can contain GPS EXIF data, names in filenames, or other information that has no place in a public response.

## Decision table

| Concern | Upload-time derivatives | On-demand derivatives |
| --- | --- | --- |
| First view of a new size | Processing happens before publication | Processing may occur on the first request |
| Public output consistency | Pinned to an explicit recipe and asset version | Requires equally strict cache keys and recipe pinning |
| Rare one-off exports | Creates files that may never be used | Generates only when requested |
| Failure location | Worker queue, visible before `ready` | Request or edge path, visible to the viewer unless shielded |
| Capacity shape | Bursty after uploads and bulk imports | Coupled to read traffic and cache misses |
| Rollback | Point metadata back to an older complete generation | Preserve the old recipe namespace and cached results |

For public listing slots, the left column has the better failure boundary. A broker import with 2,000 photos can create a queue spike, but that load is measurable and can be throttled without slowing every listing read. Backpressure belongs at ingestion. Reads then remain boring: fetch one manifest, select a named slot, and emit the URL plus intrinsic dimensions. I favor that operational shape because it separates an agent's bursty write workload from a buyer's latency-sensitive read path; the choice is explicit, and so is the reason.

The limitation of upload-time processing is real: it duplicates bytes, spends compute before demand is known, delays publication, and requires a backfill plan whenever a recipe changes. Those drawbacks are acceptable only for a small, high-hit set of outputs. Prebuilding dozens of speculative dimensions would reverse the decision, as would a workflow dominated by one-off exports.

That is the trade-off.

## The critical path is a manifest, not a resize endpoint

The upload API should issue an asset identifier and a constrained upload destination. After the source arrives, a worker verifies the actual file signature, decodes within configured pixel and byte limits, applies orientation, and writes derivatives under content-addressed or immutable versioned keys. Filename extensions and client-provided MIME types are hints, not validation; OWASP recommends validating uploads with multiple controls and storing files outside the webroot.

The following Python sketch shows the important boundary. It deliberately omits storage and queue vendor calls. A Node.js API can produce and consume the same JSON-shaped manifest while an isolated Python worker uses a mature imaging library. The recipe is data, and the worker rejects unknown operations rather than accepting arbitrary transform parameters.

```python
from dataclasses import dataclass
from io import BytesIO
from typing import Literal

from PIL import Image, ImageOps


Fit = Literal["cover", "contain"]


@dataclass(frozen=True)
class Slot:
    width: int
    height: int
    fit: Fit


RECIPE_VERSION = 3
SLOTS = {
    "search_square": Slot(480, 480, "cover"),
    "search_card": Slot(640, 480, "cover"),
    "listing_main": Slot(1280, 960, "contain"),
    "listing_zoom": Slot(1920, 1440, "contain"),
    "email_card": Slot(600, 450, "cover"),
    "social_share": Slot(1200, 630, "cover"),
}


def render(source_bytes: bytes, slot_name: str) -> tuple[bytes, dict[str, int | str]]:
    slot = SLOTS[slot_name]
    with Image.open(BytesIO(source_bytes)) as opened:
        image = ImageOps.exif_transpose(opened).convert("RGB")
        if slot.fit == "cover":
            result = ImageOps.fit(
                image,
                (slot.width, slot.height),
                method=Image.Resampling.LANCZOS,
            )
        else:
            result = image.copy()
            result.thumbnail(
                (slot.width, slot.height),
                resample=Image.Resampling.LANCZOS,
            )

        output = BytesIO()
        result.save(output, format="JPEG", quality=82, optimize=True)
        return output.getvalue(), {
            "width": result.width,
            "height": result.height,
            "format": "image/jpeg",
            "recipe_version": RECIPE_VERSION,
        }
```

There is a subtle contract in `contain`: a portrait source will fit inside the bounding box and will not magically become 1280 by 960. Clients need the returned intrinsic dimensions so they can reserve the correct aspect ratio and avoid layout shifts. If the product requires an exact canvas, define padding color and placement as part of a separate recipe. Do not leave that choice to whichever worker happens to run.

The encoder settings also belong in the recipe version. MDN documents that JPEG is lossy and well suited to still photographs, while formats such as WebP and AVIF support both lossy and lossless compression. Format negotiation can reduce transfer size, but it expands the test matrix. Start with the formats your supported clients decode reliably, record `Content-Type` correctly, and use an HTML `picture` element when serving alternate encodings rather than changing bytes behind one ambiguous URL.

## Failure handling, compliance, and observability

Retries must be idempotent. Derive every destination key from the asset version, recipe version, slot, and format; then a repeated job either confirms the existing object or writes the same logical result. Never increment “completed slots” blindly. Count distinct validated outputs.

Watch four signals: queue age, decode rejection rate, processing latency by source megapixels, and incomplete manifests by recipe version. Add a fifth alert for a public response that references anything except `ready` media. That last check catches application mistakes that worker dashboards cannot see.

Uploads deserve hostile-input treatment even when they come from authenticated agents. Enforce byte and decoded-pixel ceilings before expensive transforms, allow only formats the decoder is configured to handle, randomize storage keys, and scan according to the organization's threat model. Rate limits should distinguish a single photo replacement from a 500-listing feed import; otherwise a legitimate bulk job can starve an urgent correction. Compliance also needs a deletion workflow that covers originals, derivatives, backups according to retention policy, and database manifests. Deleting only the visible JPEG is not deletion.

Test with more than attractive landscape photos. A useful fixture set includes portrait images with EXIF rotation, an image already smaller than every slot, a transparent input that must be flattened predictably, malformed headers, an extreme pixel count, repeated delivery of the same job, and a source replaced while the previous generation is still running. Assert dimensions, format, recipe version, and state transitions. Pixel snapshots can help with crop regressions, but they should be backed by numeric invariants because encoder upgrades can change bytes without changing the intended display.

Deploy recipe changes as a parallel generation. Write version 4 beside version 3, validate a sample and operational metrics, backfill, then switch manifests in one database operation. Keep the prior generation until rollback risk passes. This avoids an especially nasty state: half the listing gallery using an old crop rule while newly requested images use a new one.

## Why reject on-demand rendering here?

On-demand rendering moves an expensive, failure-prone operation toward the reader. A cache can hide most executions, but cache misses still need strict parameter validation, concurrency control, stable recipe identifiers, and protection against a flood of unique dimensions. Allowing arbitrary `width` and `height` query parameters also turns a finite storage plan into an unbounded workload. Its advantage is equally concrete: the system does not encode or retain a derivative until somebody asks for it, which is a better fit when the possible sizes are numerous and reuse is low.

So reject it for the six canonical listing surfaces.

It is still the right option for low-frequency, authenticated work where the requested shape is not known ahead of time: a print brochure composer, a custom comparative-market-analysis export, or an internal crop-review tool. Put that path behind authorization and a bounded transform vocabulary. Store the result if it will be reused, and never let it overwrite the original or masquerade as a canonical slot. A team that cannot tolerate even a short publication delay should also examine on-demand generation or a hybrid, provided it can absorb cache-miss latency and isolate transform failures from the public response.

The final API contract stays small: clients create an asset, observe its processing state, and read a manifest of named outputs. Node.js coordinates that lifecycle well without becoming the image processor. The clear choice is prebuilt, versioned derivatives for public real estate listings, with on-demand rendering reserved for controlled exceptions whose dimensions cannot be predicted.

## References

- https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types
- https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/picture
- https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html
- https://www.w3.org/TR/exif/
- https://pillow.readthedocs.io/en/stable/reference/ImageOps.html
