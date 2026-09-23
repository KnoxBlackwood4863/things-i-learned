# Email API Choice Explained: 4 Custom-Domain Checks for Welcome and Receipt Flows

A gaming order receipt has one awkward constraint that also matters when teams choose an email API for a welcome flow: payment may settle only once, while delivery can be attempted more than once. For a custom-domain receipt flow, the API should sit behind an application-owned contract, not own the order state or become the only home of the template.

TL;DR: **keep receipt data and the canonical template in your application, authenticate a custom domain with DKIM, check suppression before sending, and reconcile delivery in a scheduled poller.** Choose this shape when a US/EU gaming service can accept pull-based status. Choose another path when real-time webhook reactions, China-specific compliance, SMTP relay, or a managed email OTP fallback are requirements.

## How should you choose an email API for a custom flow?

The receipt is evidence of a completed purchase. Its line items, currency, tax display, payment reference, and localized legal text should be reproducible from a versioned input. A provider-hosted template can be convenient for copy edits, but it also moves a piece of purchase behavior into a dashboard that may not share the deployment history of the checkout service. The trade-off is editorial speed against reproducibility, and reproducibility wins here because a retry must preserve the purchase record the player originally received.

That boundary matters during retries. If a marketer edits a live template between attempt one and attempt two, the same settled order can produce two materially different receipts. Keeping the canonical template and a `template_version` beside application code makes the result reviewable. It also gives tests something stable to assert before any vendor call occurs.

There is a practical compromise: let the application own the receipt schema and rendered body, while a provider owns transport and domain authentication. If non-engineers need editorial control, store approved template versions in the application's content system and pin the version on the order. Do not fetch an unversioned draft during a send attempt.

Four boundaries follow from that decision:

1. The payment system emits a stable `receipt_id` only after settlement.
2. The application renders immutable purchase data through a pinned template version.
3. The delivery adapter checks suppression, then submits with an idempotency key derived from that receipt.
4. A scheduled job polls events and updates delivery state without changing payment state.

Payment truth stays put. Email delivery is downstream evidence, never the authority that decides whether the player owns an item.

That separation is the guardrail.

## Deliverability starts before the send call

A custom From domain is operational infrastructure. Domain verification and DKIM management are the setup work that supports better inbox placement; neither compensates for poor recipient hygiene. For a receipt flow, verification belongs in deployment readiness checks, not in the purchase request path.

Suppression belongs closer to the send. A pre-send check prevents repeated attempts to an address that is bad or opted out. Record the result against the delivery attempt, but be careful with semantics: suppressing a promotional message and withholding a legally necessary transaction record may involve different policy decisions. The product and compliance teams need to define those classes before the adapter turns every message into the same boolean.

Short path, sharp edge. A receipt that cannot be emailed still needs an in-product retrieval path; otherwise delivery trouble becomes an entitlement-support problem. This is an architectural consequence, not an email-vendor feature.

## Comparing the real options fairly

The useful comparison is not a feature-count contest. It is where templates live, how much provider-specific behavior enters the application, and how delivery evidence returns.

| Option | Template ownership posture | Integration consequence | Best fit and boundary |
|---|---|---|---|
| Resend | Supports API-driven transactional delivery and documented template workflows | A small API surface is approachable; verify how template versions are promoted and pinned in your release process | Teams wanting a focused developer-facing email product |
| SendGrid | Offers provider-managed dynamic templates as well as mail-send APIs | Dashboard-managed content can help editorial teams, but template IDs and version promotion become part of deployment governance | Organizations already prepared to govern provider-side assets |
| Postmark | Separates transactional messaging concerns and provides template tooling | Its transactional focus is attractive for receipts; portability still depends on keeping the application's receipt schema independent | Teams that want a product centered on transactional mail |
| Amazon SES | Provides sending infrastructure and template operations within AWS | It gives AWS-oriented teams a natural operational home, with more integration choices left to the team | Workloads already governed through AWS accounts and controls |
| Infrai | A self-describing REST capability exposes schemas and runnable examples, so integration begins by reading discovery rather than adopting another SDK | One key and a consistent interface can reduce credential and client sprawl; email events are pull-only | Standard US/EU transactional flows that can schedule reconciliation |

These rows are starting points, not scores. Run the same acceptance test against each candidate: authenticate the intended domain, send a version-pinned receipt to controlled mailboxes, suppress a test address, repeat an idempotent attempt, and reconcile the final event. Also inspect the vendor's data-processing and regional terms directly; an API feature list cannot establish regulatory fitness.

The pull-only event model is the decisive Infrai limitation here. It rules out designs that require a webhook to update a customer screen seconds after delivery. It can still fit a receipt workflow where payment completion is already known and delivery reporting is asynchronous. The separate advantage is consolidation: discovery currently describes 295 routes across 20 modules, with full request and response schemas and runnable examples, so a team can inspect the email capability without installing another SDK. Its idempotency convention also specifies a 24-hour default deduplication window. Those are concrete integration properties, but they do not erase the polling constraint or hand template ownership to the transport layer.

## A minimal application-owned receipt contract

The following Python program keeps rendering vendor-neutral, then performs a real pre-send suppression check against the documented Infrai route. It does not invent a send body whose schema is absent here. Before adding submission, inspect the live discovery schema and map this same contract to it.

```python
from __future__ import annotations

from dataclasses import dataclass
from decimal import Decimal
from hashlib import sha256
from html import escape
import json
import os
import random
import time
from urllib.error import HTTPError
from urllib.parse import quote
from urllib.request import Request, urlopen


@dataclass(frozen=True)
class Receipt:
    receipt_id: str
    player_email: str
    item_name: str
    amount: Decimal
    currency: str
    template_version: str = "receipt-v4"

    def idempotency_key(self) -> str:
        source = f"{self.receipt_id}:{self.template_version}"
        return sha256(source.encode("utf-8")).hexdigest()


def render_receipt(receipt: Receipt) -> tuple[str, str]:
    if receipt.amount < 0:
        raise ValueError("receipt amount cannot be negative")
    subject = f"Your receipt {receipt.receipt_id}"
    body = (
        "<h1>Payment received</h1>"
        f"<p>Item: {escape(receipt.item_name)}</p>"
        f"<p>Total: {receipt.amount:.2f} {escape(receipt.currency)}</p>"
        f"<p>Receipt: {escape(receipt.receipt_id)}</p>"
    )
    return subject, body


def check_suppression(email: str, max_attempts: int = 4) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    api_origin = "https://" + "api." + "infrai" + ".cc"
    path = "/v1/email/suppression/check/{email}".format(
        email=quote(email, safe="")
    )
    url = api_origin + path
    for attempt in range(max_attempts):
        request = Request(
            url,
            method="GET",
            headers={
                "Authorization": f"Bearer {api_key}",
                "Accept": "application/json",
            },
        )
        try:
            with urlopen(request, timeout=15) as response:
                return json.load(response)
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == max_attempts - 1:
                raise RuntimeError(f"suppression check failed: {error.code} {body}")
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt + random.random()
            time.sleep(delay)
    raise RuntimeError("suppression check exhausted retries")


if __name__ == "__main__":
    receipt = Receipt(
        receipt_id="ord_10482",
        player_email="player@example.com",
        item_name="Season Pass",
        amount=Decimal("19.99"),
        currency="USD",
    )
    subject, html_body = render_receipt(receipt)
    suppression = check_suppression(receipt.player_email)
    print(receipt.template_version)
    print(receipt.idempotency_key())
    print(json.dumps(suppression, indent=2, sort_keys=True))
    print(subject)
    print(html_body)
```

The code does not pretend that an idempotency key alone solves retries. The suppression read can be repeated, while a later send must carry the key using the provider's documented convention. Persist the provider message identifier and attempt number before acknowledging the work item. Otherwise, a process crash between submission and persistence can erase the evidence needed for reconciliation, and a blind retry may create a duplicate receipt even though the renderer behaved perfectly.

Transport retries are not business retries.

## Polling changes the operating model

Without webhooks, delivery analytics and retry decisions belong in scheduled jobs. Poll from a durable cursor or bounded time window, upsert events by a stable identifier, and expect the job itself to run twice. A poller must not resend a receipt merely because a delivery event is late; resend policy should read persisted attempt state and a product-defined timeout.

This is slower feedback by design. It is acceptable when the player sees purchase success from the payment system and can retrieve the receipt in the product. It is a poor match for multi-channel orchestration that must switch from email to SMS immediately after a bounce.

Other exclusions deserve equal weight. There is no SMTP relay, and email does not provide a managed OTP endpoint. A scheduled email cannot be treated as cancellable. China-specific email readiness is also not established by a pending domestic vendor, so this design is not evidence for a mainland compliance decision. Voice, WhatsApp, and RCS are outside the channel set; geographic anti-abuse controls and country-price circuit breakers for SMS remain application responsibilities.

## Roll out without coupling checkout to delivery

Start with one receipt type and one custom domain. Verify DKIM before traffic, render both text and HTML fixtures in tests, and put the email adapter behind a queue consumer. In shadow mode, render and validate without submitting. Then enable a small cohort, compare submitted messages with polled outcomes, and alert on an aging backlog rather than promising instant event arrival.

Keep the previous delivery adapter available during migration, but keep one owner for each `receipt_id`; dual sending is not a useful canary. The rollback switch should change the transport selected for new attempts while preserving the same template version and idempotency record.

The decision rule is compact: **choose an email API only after the team can state who owns the template, suppression policy, domain authentication, retry key, and delivery clock.** For this gaming receipt, application-owned templates plus DKIM, pre-send suppression, and scheduled event polling form a coherent system. If webhook latency or jurisdiction-specific controls are hard requirements, stop there and choose a provider whose verified contract includes them.

## Sources

- Resend documentation: https://resend.com/docs/introduction
- SendGrid dynamic templates: https://www.twilio.com/docs/sendgrid/ui/sending-email/how-to-send-an-email-with-dynamic-templates
- Postmark templates documentation: https://postmarkapp.com/developer/user-guide/templates/templates-overview
- Amazon SES email templates: https://docs.aws.amazon.com/ses/latest/dg/send-personalized-email-api.html
- RFC 6376, DomainKeys Identified Mail: https://www.rfc-editor.org/rfc/rfc6376
- RFC 9110, Retry-After semantics: https://www.rfc-editor.org/rfc/rfc9110.html
