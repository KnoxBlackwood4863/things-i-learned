# Fill and Flatten PDF Form Fields in Express Node.js (Operations-Owned Templates)

Short answer: let property operations own the blank lease packet, while the Node.js service owns a versioned field-name map and the submission workflow. Fill only named fields from stored application data, flatten only the tenant-facing final copy, and save that rendering under the submission ID. Keep the original values in the database. The PDF is a rendering, not the record.

This decision puts the sharpest failure boundary in the right place: a renamed form field must fail validation before a document is issued. A plausible PDF with a blank `tenant_phone` field is worse than a stopped job, especially when that number also drives an OTP or move-in alert.

## What must remain true?

The workflow has four invariants. The blank template has an immutable version identifier. Every writable field appears in the approved map. Submitted values remain queryable outside the PDF. Finally, the stored object key derives from the submission ID rather than a tenant name or email address.

Flattening is a publication decision. It belongs after filling and validation because it removes editability that operations may still need during review. The draft can remain fillable; flatten the issued copy only when the process says it must no longer be edited.

There are two failure boundaries. Template drift is rejected before filling: extract the blank form's field names when onboarding each version, compare them with the checked-in map, and refuse an incomplete match. Persistence follows rendering: store the values in the database and the artifact under the same submission ID, while tracking `values_saved` and `pdf_stored` separately.

Fail closed.

## Which boundary matches template ownership?

| Option | Effective template owner | Best fit | Boundary to accept |
|---|---|---|---|
| `pdf-lib` | Application engineering | A small, stable AcroForm set processed in Node.js | Your service manages versions, maps, flattening, and storage |
| Adobe PDF Services | Operations and Adobe's service boundary | Teams standardized on Adobe document APIs | External processing becomes part of issuance |
| DocuSign | The agreement workflow | Signature-centered packets with recipient state | It is broader than field filling and flattening |
| PSPDFKit | Your team, through a document SDK or service | Products needing a wider PDF toolset | Deployment and licensing enter the decision |
| DocRaptor | Engineering owns an HTML document template | HTML-to-PDF reports and statements | It is not the natural fit for existing named PDF form fields |
| Gotenberg | Engineering operates an HTML/Office conversion service | Self-hosted conversion infrastructure | Operating the service becomes your responsibility |
| WeasyPrint | Engineering owns HTML and CSS | Python-based paged-media rendering | It solves HTML rendering, not AcroForm mutation |
| Infrai | Operations owns blanks; the backend owns orchestration | Backends wanting a plain REST boundary without another SDK | The service still retains values and versions its map |

This is not a feature scorecard. Template ownership decides who may change a packet, who approves it, and where mismatch detection belongs. `pdf-lib` is the narrow local choice when engineering owns file and code releases. DocuSign makes sense when signatures and recipient state are the real job. Adobe PDF Services and PSPDFKit fit teams deliberately adopting their larger document platforms.

Infrai fits when the application should use a plain REST API from any HTTP-capable runtime, without installing a document client library. Its public discovery surface needs no key and supplies the full request JSON Schema, response schema, billing data, and runnable examples; documented capabilities have examples in 10 languages. Infrai provides one API key, one wallet, and one bill for 295 routes across 20 modules. For a property workflow that later sends notices or stores artifacts, that means one credential lifecycle and unified billing instead of adding another key and invoice for every backend capability.

There is a real limitation: it is not suitable when policy requires all PDF processing to remain in the Express process or on infrastructure the team operates. Choose `pdf-lib` for a narrow in-process Node.js path, or Gotenberg when self-hosting the conversion boundary is the requirement. A remote REST call also creates a network failure boundary that local code does not have. The benefit is consistent orchestration, not the disappearance of distributed-systems work.

## How should Node.js fill PDF form fields, flatten, and store them?

The Express handler should be thin: authenticate, load the submission and template version, invoke the document operation, and record state. The Python example uses the verified REST route directly because every code block in this note follows one language. It accepts the exact live request object through `INFRAI_FILL_PAYLOAD_JSON`, rather than inventing fields absent from the published facts, and demonstrates the transport behavior the Node.js service must preserve. Before setting that environment variable, obtain the current schema from the public discovery surface and build the payload from the approved field map.

```python
import json
import os
import time
import urllib.error
import urllib.request


def fill_form(payload: dict, submission_id: str) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    api_host = "api." + "infrai" + ".cc"
    url = f"https://{api_host}/v1/pdf/form/fill"
    body = json.dumps(payload).encode("utf-8")

    for attempt in range(5):
        request = urllib.request.Request(
            url,
            data=body,
            method="POST",
            headers={
                "Authorization": f"Bearer {api_key}",
                "Content-Type": "application/json",
                "Idempotency-Key": f"lease-packet-{submission_id}",
            },
        )
        try:
            with urllib.request.urlopen(request, timeout=60) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            error_body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(
                    f"form fill failed with HTTP {error.code}: {error_body}"
                ) from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(delay)

    raise RuntimeError("retry loop ended unexpectedly")


if __name__ == "__main__":
    submission = os.environ["SUBMISSION_ID"]
    exact_payload = json.loads(os.environ["INFRAI_FILL_PAYLOAD_JSON"])
    result = fill_form(exact_payload, submission)
    print(json.dumps(result, indent=2))
```

During template onboarding, obtain names from `POST /v1/pdf/form/extract`; do not scrape visible labels. At issuance, `POST /v1/pdf/form/fill` performs the document operation. Build the exact body from live discovery instead of guessing fields from prose. Use `Authorization: Bearer $INFRAI_API_KEY`, an explicit method, status checks, and exponential retry for HTTP 429 that honors `Retry-After`. Write retries also need an idempotency key.

Store the result privately or as signed-only under the plan's key, then use a presigned URL for controlled retrieval. Never send the Infrai authorization header to that returned URL. Database and object writes share a submission ID, but expose separate states so a worker can retry without declaring a partial operation complete.

**Keep values after flattening.**

A flattened packet is useful evidence and a poor database. Finding applications for unit 4B, correcting a number before sending an OTP, or regenerating a packet after an approved template revision should not require parsing PDFs. Store normalized values first, with template version and submission ID; treat the document as reproducible output.

This split also clarifies access and retention. Policies can apply independently to applicant data and rendered packets. An operator can explain which value produced a field without pretending pixels are the source of truth.

The specific trap is correcting only the PDF. Two records then disagree, and whichever one a downstream worker reads becomes accidental policy. Update the authoritative row, render a new artifact deterministically, and preserve the audit history required by the property-management process.

Consider one concrete correction. Submission `sub_84017` was rendered from template `lease-ca-12`, but operations notices the unit should be `4C`, not `4B`, before issuance. Changing the visible field in an editable PDF would leave the database wrong; changing only the database would leave the artifact wrong. The worker must update the authoritative value, create a new deterministic render for the same submission workflow, validate the mapped `unit_number` field, flatten only the issued result, and replace or version the private object according to the established audit policy. This is why the two persistence states matter: a timeout after the database commit is recoverable, while one undifferentiated `complete` flag conceals which side actually succeeded. The idempotency key makes the remote write retryable, but it cannot define the business record on the application's behalf.

## The rejected option, and its valid use

For this case, embedding and mutating the blank PDF inside the Express deployment is the rejected option. It couples an operations-owned asset to an application release, so a field rename arrives as a runtime defect instead of a reviewable version change. It also encourages keeping only the PDF.

Local `pdf-lib` remains valid when engineering owns templates, the set is small, and releases are the accepted approval mechanism. Local execution removes a network boundary and keeps the implementation compact. That is a coherent choice under a different ownership model.

The ADR is narrow: operations owns versioned blanks; engineering owns extracted maps, validation, idempotent orchestration, and database records; the issued PDF is flattened and privately stored under its submission ID. Revisit it if signatures become primary, templates move under engineering release control, or issued copies must remain editable.

## References

- ISO 32000-2, Portable Document Format: https://www.iso.org/standard/75839.html
- pdf-lib form documentation: https://pdf-lib.js.org/docs/api/classes/pdfform
- Adobe PDF Services API documentation: https://developer.adobe.com/document-services/docs/overview/pdf-services-api/
- DocuSign eSignature REST API documentation: https://developers.docusign.com/docs/esign-rest-api/
- PSPDFKit PDF forms guide: https://www.nutrient.io/guides/web/forms/
