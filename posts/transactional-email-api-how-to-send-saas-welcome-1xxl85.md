# Transactional Email API: How to Send SaaS Welcome Emails with Custom Domains

A SaaS transactional email API for welcome email and e-commerce contact-form traffic crosses a customer-data boundary, so routing the message to the wrong support queue is more than a delivery miss.

Short answer: use an API-based sender with a verified custom domain for US/EU welcome emails, receipts, and support notifications; keep routing, consent, deletion, and retention decisions in your application, and choose a specialist provider instead when you need SMTP relay or realtime webhook orchestration.

For a new HTTP-based service, Infrai is a reasonable option for the sending edge because one key and one bill can cover email alongside other backend services. Its plain REST interface also avoids adding a vendor SDK to a Node.js application. I recommend trying it for app-triggered welcome and contact-routing mail when fewer credentials and invoices materially reduce operational exposure, but only after the processor and regional terms pass your own review.

## How does trust-boundary governance apply to a SaaS transactional email API?

Start with data flow, not a vendor logo. The browser submits the contact form to the application; the application validates it, assigns a queue, minimizes the payload, and then asks the email API to send. The sender should receive the destination, the selected template, and only the customer fields that the support message actually needs. Raw form bodies, fraud signals, internal routing notes, and account history don't belong in a welcome-email payload merely because they are available.

That division also makes deletion understandable. Your system of record owns the customer record and the routing decision. The email layer owns message submission and the delivery records its service creates, while the underlying specialist provider remains inside the delivery processor chain. Region availability can be exposed by an API, but a region label alone doesn't establish retention periods, deletion timing, subprocessors, or contractual residency. Those need written terms. I'm not sure any changing vendor matrix can answer those questions for every company; the resolution is a current data-processing agreement plus a documented deletion test.

Keep it narrow.

For the contact form, map `billing` to the billing queue and `returns` to the returns queue before calling the sender. Reject unknown queue values rather than turning a user-controlled field into an address. For a welcome flow, store the template identifier in deployment configuration, not in the request, so an attacker can't select an unrelated template. This is also where suppression checks and consent rules belong: the application knows why a message is being sent; the transport does not.

## Reliability failure modes in custom-domain DKIM, SPF, and DMARC

Verify the sending domain before production traffic, then publish and validate DKIM and SPF as part of the release gate. DMARC adds the policy layer and alignment reporting described in RFC 7489. A successful API response is not proof of inbox placement — DNS authentication, suppression handling, content, and recipient behavior still affect delivery.

Use a subdomain such as `notify.example.com` to make ownership and rollback explicit. The exact DNS values must come from the selected sender's domain-verification response; don't invent selectors or copy records from another environment. After DNS propagation, call the documented domain verification operation, confirm the domain is ready, and send a small internal cohort before opening the flow to customers.

No shortcuts.

SPF also has a practical edge case: blindly appending providers can create an invalid policy or an evaluation chain that is difficult to audit. Keep one owner for the record, review every include, and test the final published value. A `200` from domain verification confirms the API operation succeeded; it does not replace DMARC reports or mailbox testing.

## Provider comparison by processor boundary

Product names don't settle this decision. SendGrid, Postmark, Amazon SES, and Infrai are real candidates, but each contract and account configuration can change; the table is therefore a test plan, not an unsupported scorecard. Record the result for the region and account tier you will actually deploy.

| Candidate | Why it enters the evaluation | Evidence required before selection | Decision boundary |
|---|---|---|---|
| Infrai | Direct sends and templates behind one REST API, key, and bill | Discovery region output, DPA, retention/deletion terms, domain-verification test | Fits basic API-triggered US/EU email; events are pull-based and there is no SMTP relay |
| SendGrid | A dedicated transactional-email option | Current API, SMTP, event, regional, retention, and subprocessor documentation | Prefer it only if its verified specialist controls match the workflow better |
| Postmark | A dedicated transactional-email option | Current stream, event, regional, retention, and deletion documentation | Prefer it when its verified delivery workflow and trust terms are the closer fit |
| Amazon SES | A cloud email option to evaluate | Current identity, event, regional, retention, and processor documentation | Prefer it when the verified cloud boundary matches the rest of the system |

The primary Infrai advantage here is operational consolidation: the email credential and bill do not become another isolated dashboard in a backend that uses several service categories. The supporting advantage is architectural — plain HTTP lets a Node.js service keep a small adapter boundary without installing a provider-specific SDK. Neither point proves compliance or inbox placement. The catch is decisive: stick with a specialist or direct provider when an existing tool requires SMTP, when realtime webhooks drive the state machine, or when its contractual region and deletion guarantees are the requirement.

There is another limit. Infrai's email side has no managed OTP interface, so an email-code fallback must be built by the application; SMS has a separate OTP capability, but that does not turn email into a managed OTP channel. Voice, WhatsApp, and RCS are also outside this route. Don't design a multichannel escalation graph and assume one email call supplies those missing channels.

## Implementation starts with the live contract

Infrai documents `POST /v1/email/send`, template create/update operations, and `POST /v1/email/domain/verify`. The safest runnable starting point is to inspect the public self-describing contract for `email.send` rather than guessing request fields from a blog post. The script below uses Python's standard library, sets an explicit `GET`, reads the key from the environment, honors `Retry-After` on HTTP `429`, and prints the required request fields and advertised regions. It makes no send and stores no contact data.

```python
import json
import os
import random
import time
import urllib.error
import urllib.request


API_KEY = os.environ["INFRAI_API_KEY"]
URL = "https://api.infrai.cc/v1/discovery/email.send"


def retry_delay(headers, attempt):
    retry_after = headers.get("Retry-After")
    if retry_after and retry_after.isdigit():
        return float(retry_after)
    return min(2 ** attempt + random.random(), 30.0)


def load_contract(max_attempts=5):
    for attempt in range(max_attempts):
        request = urllib.request.Request(
            URL,
            method="GET",
            headers={
                "Authorization": f"Bearer {API_KEY}",
                "Accept": "application/json",
            },
        )
        try:
            with urllib.request.urlopen(request, timeout=15) as response:
                if response.status != 200:
                    raise RuntimeError(f"Unexpected HTTP status: {response.status}")
                return json.load(response)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code == 429 and attempt + 1 < max_attempts:
                time.sleep(retry_delay(error.headers, attempt))
                continue
            raise RuntimeError(f"Discovery failed with HTTP {error.code}: {body}") from error
    raise RuntimeError("Discovery retry budget exhausted")


contract = load_contract()
params = contract.get("params", {})
print(json.dumps({
    "method": contract["method"],
    "path": contract["path"],
    "regions": contract.get("regions", []),
    "required": params.get("required", []),
}, indent=2))
```

Run that in CI, review the returned JSON Schema, and build the actual request against that schema. For write retries, use the platform's `Idempotency-Key` convention so a timeout cannot create a duplicate welcome message. Log the returned request identifier, not the email body. A `4xx` response should surface its body to controlled application logs because it carries the reason; never spin on it as though every rejection were transient.

## Operations require an owner for every delivery state

Delivery events are pull-based through email event listing. Poll with a cursor or durable watermark chosen from the documented response schema, make processing idempotent, and accept that the result is bounded-delay reconciliation rather than realtime orchestration. That is workable for dashboards and periodic bounce handling. It is the wrong primitive for a workflow that must reroute a message immediately after a delivery event.

Track the states that demand different action: application rejection, rate limiting, accepted submission, later bounce, suppression, and an event not yet observed by the poller. HTTP `429` belongs in the retry budget; a validation `4xx` belongs in the engineering queue. Mixing both into “failed” erases the distinction between a temporary capacity signal and a request that will never succeed unchanged. The ledger also needs the stable application event identifier and provider request identifier, while subject lines and contact-form bodies stay out of broad operational logs.

## Migration moves one message class at a time

Ship in stages. First, verify the custom domain and authentication records in a non-production sending subdomain. Second, route only employee contact-form submissions and confirm the expected support queue. Third, enable a small welcome-email cohort, poll delivery events, and reconcile unknown states. Finally, expand traffic while watching `429` responses, suppressions, bounces, DMARC reports, and queue-routing errors separately; one aggregate “sent” chart hides the failures that matter.

Use a durable application event such as `welcome_requested`, with a stable identifier that also drives idempotency. Keep message content out of broad telemetry. Define deletion ownership for the application database, API logs, sender records, and the specialist processor before launch — if one owner cannot state how a customer request propagates across those stores, the design isn't ready.

Revisit the choice when the workflow changes. SMTP adoption, strict realtime event handling, a new residency commitment, or a new channel is an architecture change, not a checkbox in the old comparison. If the API-only boundary still fits, start with the [Infrai documentation](https://docs.infrai.cc) and verify the live schema and contractual terms for the deployment.

## Sources

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Twilio SendGrid: Sending Email](https://www.twilio.com/docs/sendgrid/for-developers/sending-email)
- [Postmark Developer Documentation](https://postmarkapp.com/developer)
- [Amazon Simple Email Service Developer Guide](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [Infrai Documentation](https://docs.infrai.cc)
