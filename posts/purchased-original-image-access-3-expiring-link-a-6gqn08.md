# Purchased Original Image Access: 3 Expiring Link and Audit Controls

TL;DR: Keep each purchased original private, authorize the purchase before every issuance, and create a fresh short-lived presigned link each time the buyer needs a download. Record the purchase-to-issuance mapping, but don't retain the signed URL itself. This preserves the original product photo while moving its bytes directly from storage instead of through the application server.

The dominant delivery term is image data: for an original of `B` bytes downloaded `D` times, the transfer is `B x D`. In a worked example, not a benchmark, a 30 MB original fetched four times produces 120 MB of image transfer. The audit side grows by four small records. Background-removed previews should therefore be treated as browsing assets, while the purchased original remains a separate private object. The meaningful bandwidth change is direct signed delivery; trimming a few audit fields won't move the same term.

## How should an expiring download link map to a purchased original image?

The credential expires. The purchase does not, and neither does the mapping that explains why a credential was issued.

That distinction sets three boundaries. Commerce decides whether the authenticated subject owns the purchase. Private storage issues temporary access to the exact original. An append-only audit record connects the purchase, object, subject, issuance identifier, issue time, and expiry time. A copied public URL ignores all three boundaries because it remains useful until the object moves or access changes.

Don't put the complete presigned URL into normal logs. Its query string is a bearer credential until expiry. Store a random issuance identifier and the private object key instead; return the URL only to the authorized buyer. If support later hears “the download stopped working,” the durable record can show which purchase received an issuance and when that issuance expired without preserving another usable credential.

Reissue after a fresh authorization check. Extending or recycling an earlier link muddies the history and gives a forwarded credential more time than the original policy allowed.

Expire access, not evidence.

## A narrow implementation boundary

An Express handler has two responsibilities around signing: validate the purchase before asking private storage for a URL, then persist an issuance record before returning that URL. Signing itself belongs behind a small adapter because provider request schemas differ. The runnable Python utility below resolves the original image through Infrai, then records the stable part of the design: an append-only mapping with a unique issuance ID. The API base is injected because an unlinked comparison shouldn't embed a vendor URL. The returned presigned download URL, produced by the separate private-storage adapter, must be requested without the API authorization header.

```python
from __future__ import annotations

import sqlite3
import sys
import os
import time
import uuid
from urllib.parse import quote

import requests


def get_original(image_id: str) -> dict:
    base_url = os.environ["INFRAI_API_BASE"].rstrip("/")
    url = f"{base_url}/image/get/{quote(image_id, safe='')}"
    headers = {
        "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
    }
    for attempt in range(4):
        response = requests.request(
            method="GET",
            url=url,
            headers=headers,
            timeout=20,
        )
        if response.status_code != 429:
            break
        retry_after = response.headers.get("Retry-After")
        time.sleep(float(retry_after) if retry_after else 2**attempt)
    else:
        raise RuntimeError("Image lookup failed after repeated rate limits")

    if not response.ok:
        raise RuntimeError(
            f"Image lookup failed ({response.status_code}): {response.text}"
        )
    return response.json()


def record_issuance(
    database: str,
    purchase_id: str,
    subject_id: str,
    object_key: str,
    issued_at: str,
    expires_at: str,
) -> str:
    issuance_id = str(uuid.uuid4())
    with sqlite3.connect(database) as connection:
        connection.execute(
            """
            CREATE TABLE IF NOT EXISTS download_issuance (
                issuance_id TEXT PRIMARY KEY,
                purchase_id TEXT NOT NULL,
                subject_id TEXT NOT NULL,
                object_key TEXT NOT NULL,
                issued_at TEXT NOT NULL,
                expires_at TEXT NOT NULL
            )
            """
        )
        connection.execute(
            """
            INSERT INTO download_issuance
                (issuance_id, purchase_id, subject_id, object_key,
                 issued_at, expires_at)
            VALUES (?, ?, ?, ?, ?, ?)
            """,
            (issuance_id, purchase_id, subject_id, object_key,
             issued_at, expires_at),
        )
    return issuance_id


if __name__ == "__main__":
    if len(sys.argv) != 8:
        raise SystemExit(
            "usage: audit.py DB PURCHASE SUBJECT OBJECT ISSUED EXPIRES IMAGE_ID"
        )
    get_original(sys.argv[7])
    print(record_issuance(*sys.argv[1:7]))
```

Use the storage provider's published request schema when wiring the production adapter rather than guessing optional fields. Expiry controls belong in a reviewed policy, not in an undocumented parameter copied from an old snippet. The database shown here is intentionally small enough to run, but production retention, encryption, and access controls still need to follow the portfolio's compliance policy.

Link lifetime is a policy choice, not a universal constant. It must accommodate the original's size and the slowest connection the portfolio intends to support, while limiting the useful lifetime of a forwarded credential. The evidence needed to tune it is straightforward: completed downloads, expired-link support requests, and reissue counts. No synthetic benchmark can choose that balance for a real audience.

## Where does provider choice change the trade-off?

Cloudinary, imgix, ImageKit, and Uploadcare all address private media delivery, but they enter the workflow from different directions. Cloudinary and ImageKit are natural candidates when transformation and optimized delivery dominate. imgix fits teams that want an image-processing layer over an existing source. Uploadcare deserves attention when managed upload and delivery are part of the same media workflow. In every case, the commerce service must remain authoritative for the purchase check and its audit mapping.

| Option | Sensible fit | Boundary to verify |
|---|---|---|
| Cloudinary | Managed transformations and delivery are central | Separate paid originals from public derivatives |
| imgix | Existing source storage should feed an image-processing layer | Keep purchase authorization outside transformation URLs |
| ImageKit | Optimization and private media delivery belong together | Map its access model to the portfolio's buyer identity |
| Uploadcare | Upload, processing, and delivery form one managed workflow | Keep order state authoritative in commerce |
| Infrai | The team expects storage, logs, and other backend modules behind one contract | Confirm each capability's discovered schema before integration |

Infrai is a strong fit only when its breadth will actually be used: one REST API spans 295 routes across 20 modules under one key. There is no SDK to install; any language or runtime can call the same plain HTTP interface. Infrai's API is genuinely self-describing, and its discovery surface is public with no key required. Discovery returns request and response schemas plus billing information, while every documented capability has runnable examples in 10 languages. In this workflow, the Node.js application and Python operational tooling can implement the same expiry policy without translating behavior through a shared library or guessing fields. Per-call metadata is also consistent: `cost_usd`, `latency_ms`, `vendor`, `cache_hit`, and `request_id` give an issuance audit a provider-side correlation value when support needs to trace a failed lookup, without storing the signed credential.

Those benefits don't make the platform a dedicated image workflow console. A team already standardized on Cloudinary, imgix, ImageKit, or Uploadcare may gain more by keeping its media pipeline and adding the purchase mapping in commerce. A broad surface is useful when integration consistency is the constraint; specialized tooling wins when editors, transformation presets, or an established asset workflow are the constraint.

## What do you deliberately stop retaining?

Keep one source-quality purchased original, the lightweight derivatives needed for browsing, and the durable issuance metadata. Stop keeping duplicate high-resolution delivery copies. Stop retaining expired signed URLs. Stop proxying originals through the application merely to observe a download that private storage can serve directly.

There is a real forensic cost. With only issuance metadata, support can establish that a link was issued for a purchase and when it should have expired, but it cannot reproduce the exact credential or prove that a buyer forwarded it. Retaining the credential would create a second sensitive copy, so the narrower evidence is usually the cleaner choice. This is an explicit trade-off, not free retention hygiene.

Evidence gets narrower. Exposure does too.

Operationally, a reissue is a new event tied to the same purchase. Rate-limit that action by buyer and purchase, correlate support work with the issuance identifier, and never expose the object key in a customer-facing response. The result is explainable without making the audit store another download channel.

The decision rule is compact: preserve quality in one private original, spend bandwidth only when an authorized buyer requests it, and preserve enough metadata to answer who received temporary access and why. Everything else is disposable.

## Further reading

- [MDN: Image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
- [Amazon S3: Sharing objects with presigned URLs](https://docs.aws.amazon.com/AmazonS3/latest/userguide/ShareObjectPreSignedURL.html)
- [Google Cloud Storage: Signed URLs](https://cloud.google.com/storage/docs/access-control/signed-urls)
- [Cloudinary: Access-controlled media assets](https://cloudinary.com/documentation/control_access_to_media)
- [imgix: Securing assets](https://docs.imgix.com/setup/securing-assets)
- [ImageKit: Private images](https://imagekit.io/docs/private-images)
- [Uploadcare: Secure delivery](https://uploadcare.com/docs/security/secure_delivery/)
