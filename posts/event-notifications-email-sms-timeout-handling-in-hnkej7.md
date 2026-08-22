# Event Notifications: Email/SMS Timeout Handling in a Node.js Cron Ledger

A send request can finish before an email or SMS reaches a terminal delivery state. **The decision is to persist notification state, let a bounded cron worker poll only due records, and turn an expired delivery deadline into an explicit `unknown` outcome rather than another send.** Use callbacks instead when they are available and dependable, but keep the same ledger either way.

This is an architecture decision about uncertainty, not a promise that polling proves inbox placement. DKIM authenticates a signed email domain and message; it doesn't prove that a person received or read the message. SMS status names also describe stages reported through a delivery chain, not human attention. A useful system records what it actually knows and stops short of inventing certainty.

## Decision, invariants, and failure boundaries

Store one notification intent before calling a channel adapter. The record needs an immutable idempotency key, channel, recipient reference, provider message identifier once accepted, current normalized state, next check time, attempt count, and a delivery deadline. Keep message content out of the polling record when possible. A phone number, email address, or OTP in routine worker logs creates a compliance problem without helping reconciliation.

The first invariant is **one intent, one initial send**. A timeout while submitting does not establish whether the remote system accepted the message, so blindly sending again can produce two OTPs or duplicate event notices. Record the interface's idempotency capability as part of the adapter contract. An ambiguous submission without that capability enters reconciliation or manual review; it never enters the automatic resend queue.

No blind resend.

The second invariant is monotonic state. Normalize channel-specific values into a small internal model such as `accepted`, `in_transit`, `delivered`, `failed`, and `unknown`. Terminal states stay terminal. Late observations may enrich audit metadata, but they must not move `delivered` back to `in_transit` or revive a failed notification.

The third invariant is a deadline tied to product meaning. An OTP can become useless before its carrier status settles. An account-change notice may remain operationally relevant much longer. Once the deadline passes, the worker records `unknown`; a separate policy decides whether to alert an operator, offer another channel, or do nothing. Don't hide that decision inside generic retry code.

Failure boundaries matter here. Sending, reading status, scheduling, and escalation need separate budgets and metrics. Consider a tick that claims 100 records and reaches its status-read ceiling after record 37. The first 36 observations can be committed normally. Record 37 and the unprocessed remainder retain or receive later check times, while a budget-exhausted metric explains the growing queue age. They do not become `failed`, and none of them return to the send queue. If one record instead contains an unfamiliar status, quarantine that record and continue the batch so a new channel value cannot pin the whole worker. Finally, if the process exits after a remote read but before a database update, the next lease may safely read status again because that lookup is observation, not a new send. This is why sending and polling cannot share one generic retry wrapper: identical retry behavior would erase the boundary between a repeatable read and a potentially duplicate notification.

Unknown means unknown.

Short leases make that last property practical. A worker claims a limited set of due rows until `leased_until`, commits the claim, and performs network reads outside the claim transaction. Another worker can recover abandoned rows after the lease expires. The lease must be longer than the normal batch duration but finite. I'm not sure there is a universal value; request latency, batch size, and deployment shutdown time are what settle it.

## How should a Node.js cron worker handle email and SMS delivery status?

Run the cron trigger as a scheduler, not as the source of truth. On each tick, claim records whose `next_check_at` is due, order them oldest first, and stop at a fixed read budget. A Node.js process can implement the worker with its usual database and HTTP libraries; the concurrency model does not change the state rules.

Poll quickly near submission, then widen the interval and add jitter. The exact sequence should come from the provider's status-read limits and the product deadline, not from a copied backoff table. Jitter prevents several workers or tenants created at the same instant from producing synchronized bursts. It also makes capacity planning less brittle — a useful detail when a delayed carrier causes the open set to grow.

Keep two timeout concepts distinct. A request timeout limits how long one status lookup occupies a worker slot. A delivery deadline limits how long the business waits for a terminal observation. Retrying a request timeout can be reasonable within the tick's read budget; extending the delivery deadline as a side effect is not.

No webhook changes the latency and load profile, but it should not change the schema. `next_check_at` is the queue. An index on due, nonterminal records lets cost follow the number of checks scheduled now rather than the total historical notification count. Multiple workers need an atomic claim or database lease so overlapping cron invocations don't query the same row.

Watch the tail.

The primary operational signals are age of the oldest unresolved record, due records not yet claimed, status reads by outcome, lease recovery count, and notifications reaching their deadline. Send-request success is insufficient: it can stay healthy while reconciliation falls behind. Split dashboards and alerts by channel because email authentication trouble and SMS delivery lag call for different responses. For email, validate DKIM signing as part of the sending domain's deployment checks, while treating authentication and delivery state as separate concerns.

## Which status collection design fits the constraints?

All four options can be valid. The choice turns on inbound reachability, acceptable status latency, volume, read quotas, and how costly an unresolved notification is.

| Design | Useful when | Main trade-off | Required control |
| --- | --- | --- | --- |
| Signed callbacks into a durable ledger | A public endpoint and callback verification are available | Low status latency, but inbound delivery and replay handling become production dependencies | Signature verification, deduplication, and callback retention |
| Due-record polling with backoff | No callback endpoint exists or reconciliation must be independent | Status latency follows the schedule and reads consume a separate budget | Per-record deadlines, jitter, leases, and a read cap |
| Callbacks plus a sparse reconciliation sweep | Missing a status has meaningful downstream impact | More moving parts and two ingestion paths | One monotonic state transition function for both paths |
| Fixed scan of every open record | Volume is small and operational simplicity dominates | Read load grows with the entire unresolved set | A hard batch cap and measured provider limits |

The hybrid design offers fast observations and recovery from missed callbacks, but it isn't automatically the right answer. It costs more to test, operate, and explain. Stick with callbacks alone when their retry contract, retained event history, and your dead-letter handling meet the consequence of a missed update. Choose due-record polling when inbound HTTP is unavailable or when one controlled reconciliation path is easier for the team to own.

There is another boundary: status reconciliation cannot repair deliverability. A high email failure rate calls for authentication, consent, suppression, and list-hygiene work. A long SMS tail calls for channel- and route-specific investigation. Polling only makes those patterns visible.

## Critical polling path in Python

The adapter below is deliberately generic. Production code would back `Ledger` with transactional storage and implement each `StatusClient` against a documented channel interface. The important part is that a read failure schedules another observation without creating another notification.

```python
from dataclasses import dataclass
from datetime import datetime, timedelta, timezone
from enum import Enum
from typing import Protocol


class State(str, Enum):
    ACCEPTED = "accepted"
    IN_TRANSIT = "in_transit"
    DELIVERED = "delivered"
    FAILED = "failed"
    UNKNOWN = "unknown"


TERMINAL = {State.DELIVERED, State.FAILED, State.UNKNOWN}


@dataclass(frozen=True)
class Claim:
    notification_id: str
    remote_id: str
    state: State
    attempt: int
    deadline_at: datetime


class TemporaryReadError(Exception):
    pass


class StatusClient(Protocol):
    def read(self, remote_id: str) -> State: ...


class Ledger(Protocol):
    def claim_due(self, now: datetime, limit: int) -> list[Claim]: ...
    def settle(self, notification_id: str, state: State) -> None: ...
    def defer(self, notification_id: str, next_check_at: datetime) -> None: ...
    def release(self, notification_id: str) -> None: ...


def next_delay(attempt: int) -> timedelta:
    schedule = (
        timedelta(seconds=15),
        timedelta(minutes=1),
        timedelta(minutes=5),
        timedelta(minutes=20),
    )
    return schedule[min(attempt, len(schedule) - 1)]


def reconcile(ledger: Ledger, client: StatusClient, read_budget: int) -> None:
    now = datetime.now(timezone.utc)
    claims = ledger.claim_due(now, limit=read_budget)

    for claim in claims:
        if claim.deadline_at <= now:
            ledger.settle(claim.notification_id, State.UNKNOWN)
            continue

        try:
            observed = client.read(claim.remote_id)
        except TemporaryReadError:
            ledger.defer(claim.notification_id, now + next_delay(claim.attempt))
            continue

        if observed in TERMINAL:
            ledger.settle(claim.notification_id, observed)
        else:
            ledger.defer(claim.notification_id, now + next_delay(claim.attempt))
```

`settle` must enforce legal transitions in the database, not only in application memory. That closes the race between a callback and a poller, or between two workers whose leases overlap at the edge. `claim_due` should also exclude terminal rows and atomically assign leases. Those are storage guarantees; hiding them inside a friendly repository name doesn't make them optional.

Tests should inject a clock even though the compact example reads system time. Cover an accepted message that remains pending until deadline, a terminal observation, a temporary read error, an unfamiliar status mapped to quarantine, an expired lease, and two collectors racing to settle one record. Then deploy with a small read budget and raise it from observed queue age. Capacity guesses are weaker than backlog evidence.

## Rejected option and its valid use case

The rejected design is waiting synchronously for a terminal delivery result inside the request that creates the event notification. Email and SMS delivery live on timelines the application request doesn't control. Holding the request open consumes connection capacity, couples user latency to an external delivery chain, and still leaves an ambiguous result when the caller disconnects.

It does have a narrow valid use: a local development fake that advances immediately through known states can make an end-to-end test readable. It is not suitable as the production contract. Production submission should return the notification's durable identifier and expose state through a later read, an internal event, or both.

The fixed scan from the comparison table is also a reasonable deliberate simplification for a low-volume internal system after the team measures its read ceiling. The catch is the growth curve. Once unresolved volume rises, every tick rereads the same quiet rows, so moving to `next_check_at` should happen before quota pressure becomes an incident. Your mileage may vary because provider retention and status semantics differ; verify both against the interface contract before setting deadlines.

The durable conclusion is plain: preserve uncertainty, separate sending from observation, and make timeout an auditable state transition. That's what keeps a missing webhook from becoming a duplicate message or a status row that lives forever.

## Sources

- RFC 6376, DomainKeys Identified Mail (DKIM) Signatures: https://datatracker.ietf.org/doc/html/rfc6376
- Twilio SMS documentation: https://www.twilio.com/docs/sms
