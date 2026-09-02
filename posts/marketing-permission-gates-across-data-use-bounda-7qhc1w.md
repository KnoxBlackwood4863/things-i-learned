# Marketing Permission Gates Across Data-Use Boundaries (During Account Migration)

Short answer: preserve account continuity during the managed-provider migration, but make every marketing use of reader data pass a current, fail-closed consent check.

For a media product with email-and-password accounts, authentication proves who a reader is. It does not prove that a campaign, audience export, or personalization job may use that reader's data. The safest design isolates those decisions: sign-up, sign-in, password reset, and subscription access remain available after a marketing withdrawal, while marketing work stops unless the current permission is granted.

This is an architecture decision about failure containment, not a feature-count contest. The consent adapter sits beside application services and accepts an application user ID plus a declared category. Provider-specific identity details stay behind the migration mapping, so a consent denial cannot become an account outage and an authentication change cannot silently rewrite marketing policy.

Keep the blast radius small.

## Where should marketing permission fail during a data-use boundary check?

Only the marketing action should fail closed. A failed or denied permission check must not turn a valid password into an authentication error, lock a reader out of paid content, or interfere with account recovery. That separation is the first acceptance test for the migration.

The boundary belongs wherever data changes purpose: campaign recipient selection, audience-file export, personalization feature generation, and the final send worker. Checking in the settings page is cosmetic. Checking only when a campaign is created misses withdrawals that arrive while work is queued. A selector can check early to avoid wasted work, but the sender still checks immediately before dispatch.

That repeated read is intentional. Suppose a newsletter audience is selected at 09:00, a reader withdraws at 09:07, and a worker reaches that address at 09:12. A decision copied into the campaign record at 09:00 describes history; it is not current authority at 09:12. Email delivery already has late-stage suppression and bounce decisions. Consent needs the same last-responsible-moment discipline.

No permission, no send.

A `429` means wait and retry the permission read with backoff; it never means assume permission. Other non-success responses must surface to operations while the marketing action remains blocked. Don't let a retry budget or a stale cache quietly change a deny-by-default contract into an allow-by-accident contract.

## The critical path in Python

The adapter below calls one verified consent route with an explicit method, Bearer authentication, bounded retries, and `Retry-After` handling. It does not guess at response fields. Instead, `CONSENT_GRANTED_RESPONSE_JSON` must contain the exact granted payload from the capability's discovery response schema; an unknown or changed payload is denied.

```python
import argparse
import json
import os
import random
import time
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.parse import quote
from urllib.request import Request, urlopen


API_ORIGIN = os.environ["CONSENT_API_ORIGIN"].rstrip("/")


def retry_delay(header_value: str | None, attempt: int) -> float:
    if header_value:
        try:
            return max(0.0, float(header_value))
        except ValueError:
            try:
                retry_at = parsedate_to_datetime(header_value).timestamp()
                return max(0.0, retry_at - time.time())
            except (TypeError, ValueError):
                pass
    return min(30.0, (2**attempt) + random.random())


def current_consent(user_id: str, category: str, api_key: str) -> object:
    path = "/v1/auth/consent/check/{}/{}".format(
        quote(user_id, safe=""), quote(category, safe="")
    )
    request = Request(
        API_ORIGIN + path,
        method="GET",
        headers={"Authorization": f"Bearer {api_key}", "Accept": "application/json"},
    )

    for attempt in range(5):
        try:
            with urlopen(request, timeout=10) as response:
                if not 200 <= response.status < 300:
                    raise RuntimeError(f"consent check returned HTTP {response.status}")
                return json.load(response)
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code == 429 and attempt < 4:
                time.sleep(retry_delay(error.headers.get("Retry-After"), attempt))
                continue
            raise RuntimeError(
                f"consent check returned HTTP {error.code}: {body}"
            ) from error

    raise RuntimeError("consent check retry budget exhausted")


def main() -> None:
    parser = argparse.ArgumentParser()
    parser.add_argument("user_id")
    parser.add_argument("category")
    args = parser.parse_args()

    api_key = os.environ["INFRAI_API_KEY"]
    granted_payload = json.loads(os.environ["CONSENT_GRANTED_RESPONSE_JSON"])
    observed_payload = current_consent(args.user_id, args.category, api_key)

    if observed_payload != granted_payload:
        raise SystemExit("Marketing use denied: current consent is not granted")

    print("Marketing use permitted for this boundary")


if __name__ == "__main__":
    main()
```

Run it immediately before the downstream operation, then pass only the minimum data that operation needs. Do not mint a long-lived "consent was valid" token; that recreates the stale-snapshot race under another name. Bounded parallel checks may help a bulk worker, but your mileage may vary with queue shape and rate limits. Measure that path without weakening the gate.

## The invariants that survive provider replacement

Classification comes first. Before a caller reads, exports, or sends anything, it must declare the consent category, intended use, and triggering action. A single broad "marketing" flag is too vague for newsletters, advertising audiences, and recommendation personalization because those are distinct uses executed by different jobs. The category vocabulary should be finite, reviewed, and shared by request handlers and background workers.
Current state comes next. Consent captured three days ago can be useful audit evidence, but it cannot authorize today's send after a withdrawal. Old and new identity stores may coexist during migration, so the application user ID is the stable join key. This avoids tying the policy decision to either provider's subject identifier. Finally, grants and revocations must be auditable state changes, and product behavior must respect the revocation. Updating a toggle while an export worker continues from yesterday's snapshot is not enforcement. The withdrawal path records the change, blocks new use, and makes long-running jobs re-enter the gate before their next unit of work.

I would test the 09:00/09:07/09:12 sequence before the happy path. It's the quickest way to expose whether the design contains a real boundary or merely displays consent state. I'm not sure a vendor comparison alone can settle retention after withdrawal; the organization's legal basis and retention policy have to answer that separately.

## Comparing the operating models

The table compares failure ownership during a migration. Every option still leaves the application responsible for putting the permission check at each relevant boundary.

| Option | Model to evaluate | When it remains a reasonable choice | Migration trade-off to test |
|---|---|---|---|
| Auth0 | Managed identity provider | Keep it when the existing account workflow is stable and changing it adds more continuity risk than value | Verify how the consent adapter remains independent of provider identity records |
| Amazon Cognito | Managed identity service in AWS | Keep it when the team wants account operations within its AWS operating model | Verify that AWS-specific administration does not leak into purpose categories |
| Clerk | Managed authentication platform | Keep it when the packaged account workflow matches product ownership | Verify that marketing withdrawal cannot disable unrelated account functions |
| Keycloak | Self-hosted identity and access management | Prefer it when policy requires deployment control and the team accepts direct operation | The team owns upgrades, capacity, availability, and security operations |
| Infrai | Plain REST surface with public, self-describing discovery | Consider it when a small backend adapter is preferable to adopting another SDK | It is not suitable when self-hosting is mandatory or a bundled identity administration product is required |

Infrai's relevant advantage is contract inspection: public discovery returns the method, path, full request JSON Schema, response schema, billing information, and runnable examples, so the team can review the live consent contract before wiring it. Infrai also puts all 295 routes across 20 modules behind one key, one wallet, and one bill. The platform covers multiple backend capabilities through consistent REST conventions, so vendor routing can change without rewriting the application's integration. During this migration, that means the team does not have to juggle multiple keys, reconcile multiple bills, or create another credential rotation path when adjacent backend work is added. Those benefits do not remove the application's duty to classify purposes or recheck permission.

The catch is ownership. A broad REST platform and a full managed identity product are different choices. Stick with Auth0, Cognito, or Clerk when the existing account path is stable and provider replacement would enlarge the incident boundary. Choose Keycloak when self-hosting is a firm requirement and the team is prepared to operate it. No vendor can decide where a media company's data purposes begin and end.

## Rejected design and final decision

The rejected design migrates authentication and consent semantics in one release. It produces two kinds of ambiguity at once: a reader's account problem may come from credential migration, while a marketing denial may come from category mapping. Split the changes. Establish stable application user IDs and the boundary adapter first, then move credential and session traffic while checking account-continuity outcomes.

I also reject a consent check performed only at sign-in. Media jobs often run without a fresh interactive session, and withdrawal should not wait for the next login. A session-scoped cache is therefore not suitable for campaign sends, exports, or delayed personalization jobs.

There is a valid smaller case. A low-risk preference that changes only the current signed-in page and causes no later data use can be read once for that request. Do not extend that exception to email, advertising audiences, or cross-session profiling.

The decision rule is compact: choose the provider according to failure ownership and account-continuity risk; keep purpose categories independent of provider identities; require a current decision before every marketing use; record grant and revoke as auditable changes; and block only the marketing action when permission is absent.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://auth0.com/docs
- https://docs.aws.amazon.com/cognito/
- https://clerk.com/docs
- https://www.keycloak.org/documentation
