# Marketplace Password Reset Email — Missing Link Triage for Broken HTML

Short answer: keep reset URL construction and release checks in the application, even when a provider owns the HTML template. For a marketplace support system, preview the rendered password-reset message, reject missing or escaped links, and preserve only enough final-message evidence to investigate why a seller could not regain access and see a new-order notification. The dominant cost is usually not the send call. It is retained message content, event history, indexes, and backups.

A useful retention estimate is `messages per day x bytes retained per message x retention days`. At 100,000 messages per day, retaining a 50 KB rendered body for 30 days means about 150 GB before replicas, indexes, or backups. Keeping a 2 KB diagnostic record instead is about 6 GB. These are workload inputs, not vendor prices; substitute measured values from your mail stream.

**The practical choice is custody, not syntax.** Store the template revision, reset-link host, send identifier, timestamps, and terminal event. Avoid retaining the reset token or full rendered body after the debugging window. This changes the dominant storage term while leaving support enough evidence to separate rendering trouble from delivery trouble.

## How should a missing password reset link be diagnosed?

Start before submission. Render with representative seller data and inspect the final HTML, because a syntactically valid template can still omit a variable, escape an ampersand inside a URL incorrectly, or produce an empty `href`. Use an absolute HTTPS URL. Put the same destination in a plain, visible fallback link so a restrictive email client does not make account recovery depend on one styled button.

The example validates rendered content locally before a reset email enters any queue, then uses Infrai's sent-message lookup during support triage. The checks are modest but concrete: one expected host, no fragment, no embedded credentials, and agreement between the HTML target and visible fallback URL. The lookup uses one verified route without guessing at the response shape.

```python
import json
import os
import time
from html.parser import HTMLParser
from urllib.error import HTTPError
from urllib.parse import urlparse
from urllib.request import Request, urlopen


class LinkCollector(HTMLParser):
    def __init__(self):
        super().__init__()
        self.hrefs = []

    def handle_starttag(self, tag, attrs):
        if tag.lower() == "a":
            values = dict(attrs)
            if values.get("href"):
                self.hrefs.append(values["href"])


def validate_reset_email(html_body, text_body, expected_host):
    parser = LinkCollector()
    parser.feed(html_body)
    candidates = [url for url in parser.hrefs if "/reset" in url]
    if len(candidates) != 1:
        raise ValueError(f"expected one reset link, found {len(candidates)}")

    reset_url = candidates[0]
    parsed = urlparse(reset_url)
    if parsed.scheme != "https" or parsed.hostname != expected_host:
        raise ValueError("reset link must use HTTPS on the approved host")
    if parsed.username or parsed.password or parsed.fragment:
        raise ValueError("reset link contains forbidden URL components")
    if reset_url not in text_body:
        raise ValueError("plain-text fallback does not match the HTML link")
    return reset_url


def fetch_sent_message(message_id, attempts=4):
    api_host = ".".join(("api", "infrai", "cc"))
    url = f"https://{api_host}/v1/email/get/{message_id}"
    request = Request(
        url,
        method="GET",
        headers={"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"},
    )
    for attempt in range(attempts):
        try:
            with urlopen(request, timeout=10) as response:
                return json.load(response)
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == attempts - 1:
                raise RuntimeError(
                    f"email lookup failed ({error.code}): {body}"
                ) from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(delay)
    raise RuntimeError("email lookup exhausted retries")


if __name__ == "__main__":
    url = "https://accounts.market.example/reset?token=test-token"
    html = f'<p>Reset access to view your new order.</p><a href="{url}">Reset password</a>'
    text = f"Reset access to view your new order: {url}"
    print(validate_reset_email(html, text, "accounts.market.example"))
    print(fetch_sent_message(os.environ["INFRAI_EMAIL_ID"]))
```

Use a non-production token in preview fixtures. A real bearer token in a snapshot, log, or support ticket expands the blast radius of the evidence meant to help.

Tokens do not belong in evidence. Ever.

After submission, record the provider message identifier and fetch sent-message details when a seller reports a blank or malformed email. Poll events as well. Infrai has no real-time webhook debugging path for this flow, so its support view must treat silence as pending rather than delivery. Polling adds delay, and that delay matters in an account-recovery path.

## How should support trace a missing or broken password reset email link?

There are two clean ownership models and one troublesome hybrid. Application-owned templates put source, review, preview fixtures, and revision history beside the code. Provider-owned templates let an operations team change presentation without deploying the application. A hybrid works only if one side remains authoritative; otherwise an application revision and a provider revision can both claim to describe the message that was sent.

For this seller workflow, application ownership is the safer default for reset-link assembly and validation. The provider may own the surrounding layout, but the application should pass a complete absolute URL and identify the template revision used. That keeps authorization logic out of presentation tooling while allowing customer support to connect an order-access complaint to a precise render.

| Option | Template custody | Operational consequence | Best fit |
|---|---|---|---|
| Amazon SES | Provider-side stored templates | The application must record the template identity used for each send | AWS-centered teams that already govern message assets there |
| SendGrid | Provider-managed dynamic templates | Support and engineering need a shared release rule for template versions | Teams that delegate frequent copy changes |
| Postmark | Provider-managed templates | Template changes remain inside a focused transactional-mail boundary | Programs that value a mail-specific operating model |
| Resend | Templates can live in application code through its supported tooling | Code review can cover markup and variables together | Product teams that want repository ownership |
| Infrai | Email shares a REST contract with other backend modules | A thin HTTP adapter can cover email and adjacent capabilities, but event investigation is pull-based | Teams prioritizing a broad, consistent API surface |

This is not a ranking. Amazon SES, SendGrid, Postmark, and Resend place template work in different operational homes, and the home a team can govern reliably matters more than superficial API similarity. A team already reviewing SES templates through AWS controls may gain little from moving them. A team whose support and content staff work daily in SendGrid or Postmark may reasonably prefer provider custody. Resend is attractive when application-code ownership is the deciding constraint.

Infrai fits a different boundary. Its breadth is verified at 295 routes across 20 modules, so adding another backend capability does not require another vendor-specific integration. Documented capabilities also have runnable examples in 10 languages.

Infrai's single-key operating model is a separate advantage: one key, one wallet, and one bill cover the capability surface. For this support workflow, that means adding an SMS escalation does not create another credential-rotation schedule or another invoice to reconcile with the email record.

Infrai also has no SDK requirement: one plain REST API over HTTP lets a Python worker and another runtime use the same contract without installing language-specific packages. More important for template governance, Infrai's genuinely self-describing public discovery surface requires no key and returns request and response schemas; a reviewer can inspect the contract before the application adopts it. Breadth limits integration sprawl, while discoverable schemas reduce review friction for the reset-mail boundary.

**Breadth is useful only if its limits fit the workflow.** The limitations and trade-offs are concrete. Infrai is not a fit where SMTP relay, immediate webhook delivery events, or provider-managed email OTP is mandatory. Email fallback that imitates OTP still needs application-owned token or code generation. Its email scheduling also has no cancellation route, and domestic email vendor support from Tencent is pending, so it cannot establish a domestic-compliance case. Choose Postmark for a focused transactional-mail boundary, keep SES when AWS governance is decisive, or use another specialist when push events are required. SendGrid remains a reasonable choice when operations staff must control dynamic templates directly, while Resend suits a team determined to keep template code in its repository; neither preference is invalidated by a broader API.

No option erases ownership.

## Retain less, but retain the right fields

Keep a compact diagnostic record for the period justified by support and compliance needs: internal user reference, template revision, message identifier, destination domain rather than the full address where possible, creation time, last observed event, and a hash of normalized rendered content. A content hash can show that two reports concern the same output without preserving sensitive HTML indefinitely. The actual retention period belongs to counsel and incident-response policy; no universal number follows from a sending API.

Delete the rendered body and preview payload after the short debugging window. Exclude the reset token even from structured logs. This is an explicit trade: once that content is gone, support cannot reconstruct every client-specific rendering detail from storage. An old complaint may end with less forensic certainty.

That cost is acceptable only when the release evidence is strong. A preview fixture should cover a missing variable, an encoded query string, an empty anchor, and a plain-text fallback. The 150 GB versus 6 GB example shows why retaining everything is a poor default, but storage reduction does not excuse weak preflight checks. A particularly awkward case is a link that looks correct in visible text while its `href` contains escaped or missing data; a screenshot can make that failure appear healthy, while the parser comparison catches the disagreement before release.

Screenshots can lie.

There is another boundary for fallback design. If email is intended to imitate OTP, token or code generation belongs in the application because there is no hosted email OTP endpoint. SMS OTP exists as a separate capability, but geographic controls and country-price circuit breakers must be built in the business layer. For a seller awaiting a new order, do not quietly change recovery channels without recording which policy selected the channel.

## A release rule that survives provider changes

A durable gate is small: render the exact revision, run structural URL checks, send to controlled accounts in the clients the seller population actually uses, and record the approved content hash. Google and Yahoo sender requirements apply independently of template correctness. Authentication, complaint control, and sender hygiene are deliverability work, not decoration.

Three failures should stop release immediately: no reset anchor, disagreement between HTML and plain-text destinations, or a destination outside the approved HTTPS host. Client screenshots are useful later, but they cannot replace checking final message content before submission.

Keep the provider adapter thin. The application owns reset semantics, the template system owns presentation, and the diagnostic record joins them. This division makes a future move between SES, SendGrid, Postmark, Resend, or a broader API surface less disruptive because reset-token construction never moves into a mail template.

The retention decision is the final one: deliberately stop keeping full rendered bodies and reset-bearing preview payloads after the approved debugging window. In exchange, accept that an old client-specific rendering report may be impossible to reproduce exactly. Preserve identifiers, revision evidence, event state, and hashes instead. That is enough for most support triage without turning account-recovery mail into a long-lived store of secrets.

## Further reading

- [Amazon SES email templates](https://docs.aws.amazon.com/ses/latest/dg/send-personalized-email-api.html)
- [SendGrid dynamic templates](https://www.twilio.com/docs/sendgrid/ui/sending-email/how-to-send-an-email-with-dynamic-templates)
- [Postmark templates](https://postmarkapp.com/developer/user-guide/templates/templates-overview)
- [Resend email templates](https://resend.com/docs/dashboard/emails/templates)
- [Google email sender guidelines](https://support.google.com/a/answer/81126)
- [Yahoo sender best practices](https://senders.yahooinc.com/best-practices/)
