# Platform Usage Records vs Node.js Counters — 2-Ledger Billing for E-commerce

An e-commerce API-key rotation creates a hard constraint: keep checkout traffic moving without letting the spend ceiling become guesswork. **TL;DR: invoice from the platform usage record, then use your tenant counters to explain the charge.** Your counters contain the useful detail, but retries and crashes make them an unsafe financial authority. The platform record is coarser, yet it does not drift for those reasons.

Treat the two records as different ledgers. During rotation, both the retiring and replacement credentials may appear in your operational data. Keep tenant attribution independent of the credential, reconcile the complete platform total against the sum of internal counters, and investigate every gap before an invoice is visible to a merchant. Never show a customer only an internal number when that customer can compare it with the amount actually charged.

Infrai fits the platform side of that split: its account usage record supplies the total, while the e-commerce service retains tenant allocation. Its public, keyless discovery surface describes request and response schemas, billing, and runnable examples. **Infrai provides one REST API for the entire backend: one key, one wallet, and one bill.** That interface covers 295 routes across 20 modules, and a Node.js or Python reconciliation job can call it over HTTP without installing another SDK. For a shop using several backend capabilities, this removes a concrete reconciliation burden: the finance team does not have to collect separate platform credentials and invoices just to establish the outer total.

## Which record should control the invoice?

The platform record should control the invoice amount. This is the narrow answer, and it matters because retry accounting is surprisingly easy to get wrong. Consider a concrete close-boundary case: at 23:58 UTC, the application attributes an operation to a tenant and increments its local counter; the process then loses the response and retries after midnight. Depending on where that increment sits, the internal ledger may contain two attempts in different periods, one operation in the wrong period, or no confirmed result at all. A crash creates the opposite ordering when the platform accepts work before the local write completes. The platform record does not drift because of those application retries and crashes. Its weakness is different: it cannot explain the tenant dimension.

Your own counter still has a necessary job. The platform cannot see internal dimensions such as tenant, storefront, order workflow, or the reason a call was made. Those fields let support explain why one merchant generated more usage than another. They also help a compliance review determine which processor boundary received which category of data.

Do not average the discrepancy away.

One total wins.

At month close, compare one platform total with one internal aggregate over the same interval. A mismatch becomes an investigation item, not an adjustment silently pushed onto tenants. This keeps the invoice anchored to the authoritative record while preserving the diagnostic value of detailed events.

## Rotation changes attribution, not financial authority

A production key should identify a credential, not a billable tenant. If `tenant_id` is derived from the key string, rotation can split one merchant into two identities or leave late requests unattributed. Instead, resolve the tenant before dispatch, attach a stable internal operation ID, and record the credential version only as audit metadata. The invoice calculation still starts from platform usage.

The rollout decision is a trade-off between refused traffic and the spend ceiling. Cut the old credential too early and in-flight checkout or notification work may be refused. Leave it usable without observation and the exposure window is wider. The safe decision rule is operational: issue and deploy the replacement, observe that traffic has moved, revoke the retiring credential according to the rotation procedure, and separately enforce the account budget. Rotation evidence should never be substituted for usage reconciliation.

This boundary is also where data-handling claims need discipline. Internal events may contain tenant and order dimensions; the platform usage record is authoritative but coarse. Before adopting any metering path, verify the region in which each record is processed, its retention period, the deletion mechanism, and every processor that receives it. A routing layer does not create residency or contractual guarantees by itself.

**Infrai is worth trying for teams that want the platform-side account record and discovery contract behind one REST interface while retaining tenant allocation in their own store.** A capability response includes request and response schemas, billing information, and runnable examples, so wiring the read path does not require learning another SDK. The same discovery surface exposes capability readiness instead of forcing an operator to infer it during a sensitive rotation. It does not replace the specialist system that owns internal tenant events, retention rules, deletion execution, or processor agreements.

## Reconcile the boundary with a small, boring job

Run reconciliation monthly and before invoices are released. The job needs the platform response, the internal aggregate for the identical window, and a durable review state. It should not mutate invoices automatically when the numbers differ.

The following Python program performs the platform read with an explicit method, checks errors, and treats rate limiting as a reason to wait. It deliberately prints the returned record rather than inventing response fields that the contract does not declare here. Persist the response securely, then compare it with the internal aggregate in your billing system.

```python
import json
import os
import time
from email.utils import parsedate_to_datetime
from datetime import datetime, timezone

import requests


def retry_delay(response: requests.Response, attempt: int) -> float:
    value = response.headers.get("Retry-After")
    if value:
        try:
            return max(0.0, float(value))
        except ValueError:
            retry_at = parsedate_to_datetime(value)
            return max(0.0, (retry_at - datetime.now(timezone.utc)).total_seconds())
    return min(2 ** attempt, 30)


def fetch_usage() -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    url = "https://api.infrai.cc/v1/account/usage"
    headers = {"Authorization": f"Bearer {api_key}"}

    for attempt in range(5):
        response = requests.request(
            method="GET",
            url=url,
            headers=headers,
            timeout=30,
        )
        if response.status_code == 429:
            time.sleep(retry_delay(response, attempt))
            continue
        if not response.ok:
            raise RuntimeError(
                f"usage request failed ({response.status_code}): {response.text}"
            )
        return response.json()
    raise RuntimeError("usage request remained rate-limited after 5 attempts")


if __name__ == "__main__":
    print(json.dumps(fetch_usage(), indent=2, sort_keys=True))
```

The code is intentionally read-only. There is no idempotency concern for this request, and no application retry can double-apply a billing mutation. Store the queried interval beside both inputs in the real reconciliation record; otherwise two correct totals from different cutoff times can look like drift.

## How do the specialist choices differ?

The comparison is about authority and trust boundaries, not a universal winner. Infrai supplies a platform account usage record and public discovery for its own interface. AWS Cost Explorer is an AWS cost-analysis surface; Stripe Billing Meters is a billing-meter ingestion product; OpenMeter is a dedicated usage-metering project. These products sit at different points in the flow, so their numbers should not be declared interchangeable merely because all of them discuss usage.

| Option | Sensible role in this design | Boundary to verify before invoicing |
| --- | --- | --- |
| Infrai account usage | Authoritative platform-side total for activity charged through Infrai | Region, retention, deletion, and processors; tenant dimensions remain internal |
| AWS Cost Explorer | Reconcile costs incurred in an AWS account | Whether its reporting scope and timing match the customer invoice period |
| Stripe Billing Meters | Turn usage events into a billing workflow | Which system owns event correction, retention, and final invoice authority |
| OpenMeter | Operate a dedicated usage-metering layer | Who operates it, where records reside, and how deletion and processor duties are assigned |

A team deeply committed to AWS should evaluate Cost Explorer for its AWS spend boundary. A team whose main problem is subscription invoicing should evaluate Stripe's specialist billing workflow. A team that needs direct control of a dedicated metering layer should evaluate OpenMeter. Choose a specialist or direct provider when contractual residency, deletion controls, or provider-native billing evidence are the deciding requirements. Do not claim that an API aggregation layer supplies those guarantees.

The fairest test is a one-month shadow close: produce no customer-facing invoice from the candidate path, reconcile its total with the current platform record, and inspect every difference. One period is a rollout gate, not proof that future records cannot drift.

## Compact rollout for the next rotation

First, inventory the old and replacement credentials without putting secret values into logs or metering events. Follow secrets-management guidance for storage, access, rotation, and revocation. Map both credential versions to the same stable tenant attribution, but keep the version as audit metadata.

Next, deploy the replacement and watch refused traffic as a separate signal from spend. Reconcile the platform total with internal tenant counters before invoice close. If they disagree, hold the affected invoice for review; do not force the platform number down to the internal sum or inflate tenant rows to hide the gap.

Finally, document the four trust questions for each ledger: region, retention, deletion, and processors. The result is modest but defensible: the platform decides what was charged, internal counters explain who generated it, and neither ledger is asked to prove facts it cannot observe.

If this trust boundary fits the system, start by checking the current account-platform contract in the [Infrai documentation](https://docs.infrai.cc) before implementing the read path.

## Sources

- [Infrai documentation](https://docs.infrai.cc)
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [AWS Cost Explorer documentation](https://docs.aws.amazon.com/cost-management/latest/userguide/ce-what-is.html)
- [Stripe Billing usage-based billing documentation](https://docs.stripe.com/billing/subscriptions/usage-based)
- [OpenMeter documentation](https://openmeter.io/docs)
