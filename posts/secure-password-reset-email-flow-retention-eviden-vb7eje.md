# Secure Password Reset Email Flow: Retention Evidence Beyond Token Expiry

TL;DR: Build a secure password reset email flow by storing a digest of a random token, expiring it quickly, and redeeming it with one atomic database update. For a customer-support team that must prove a notice was handled correctly, retain the event record and delivery state, not the secret, reset URL, message body, or new password. The recurring bill is driven less by one outbound email than by how many copies of sensitive evidence survive across the primary database, logs, queues, analytics, and backups.

A useful first estimate is `records per day x retained days x copies x bytes per record`. Suppose the deliberately small audit row below is about 300 bytes before database and index overhead. At 100,000 recovery requests per day, 400 days, and three retained copies, its raw payload is 36 GB. That is an illustrative capacity calculation, not a measured production benchmark. Message bodies, verbose transport payloads, and indexed URLs can make the retained footprint much larger. The change that moves the dominant term is data minimization: keep stable event facts and discard bearer material.

The security boundary is crisp. A recovery email can be observed, delayed, forwarded, or scanned. Its token must therefore be random, short-lived, single-use, and useless if the token table leaks. NIST treats account recovery as part of authenticator lifecycle management and requires protected channels when personal information is exchanged. A delivery receipt alone does not prove that the account owner read the message or completed the reset.

No secret survives.

## How should a secure password reset email flow redeem a token?

Start with the question an auditor or support lead can legitimately ask: did the service accept a recovery request, dispatch the corresponding notice, and allow at most one valid redemption before expiry? The evidence model should answer that without recreating the credential.

Keep an opaque event ID, account ID, request time, expiry time, token digest, redemption time, dispatch state, and a provider-neutral delivery reference when one exists. A coarse request-origin risk result may be retained if policy requires it; raw IP addresses and full user-agent strings should not be collected by reflex. Define their purpose and retention first. Never store the plaintext token, complete reset URL, submitted password, or email body in the audit row.

Do not equate states that mean different things. `accepted` means the mail handoff accepted the request. `delivered` may mean a receiving system reported delivery. `redeemed` means the application consumed the recovery capability. None proves human readership. Customer support should see those distinctions instead of a green check mark that overclaims certainty.

Google's sender guidance adds an operational boundary: authenticate mail, use TLS for transmission, keep spam rates low, and make sender identity and formatting accurate rather than deceptive. Those controls improve the trustworthiness of dispatch. They do not replace token controls or application-side evidence.

## Make redemption the only authority

Generate at least 256 random bits with a cryptographically secure generator. Put the raw value only in the outbound URL, and persist a SHA-256 digest. A random high-entropy token does not need a slow password hash: the database stores no human-chosen secret to defend against guessing. Compare digests through an indexed equality lookup, then enforce expiry and unused status in the same write that marks the row redeemed.

Here is the core in Python and SQL expressed through a generic database interface. It is intentionally smaller than a mail-provider integration because the database transition, not an SDK callback, grants authority.

```python
import hashlib
import secrets
from datetime import datetime, timedelta, timezone


def issue_recovery(db, account_id: str) -> tuple[str, str]:
    event_id = secrets.token_hex(16)
    token = secrets.token_urlsafe(32)
    digest = hashlib.sha256(token.encode("ascii")).hexdigest()
    now = datetime.now(timezone.utc)
    expires_at = now + timedelta(minutes=15)

    db.execute(
        """
        INSERT INTO recovery_events
          (event_id, account_id, token_digest, requested_at, expires_at, dispatch_state)
        VALUES (%s, %s, %s, %s, %s, 'pending')
        """,
        (event_id, account_id, digest, now, expires_at),
    )
    return event_id, token


def redeem_recovery(db, token: str, new_password_hash: str) -> bool:
    digest = hashlib.sha256(token.encode("ascii")).hexdigest()
    now = datetime.now(timezone.utc)

    with db.transaction():
        row = db.fetch_one(
            """
            UPDATE recovery_events
               SET redeemed_at = %s, token_digest = NULL
             WHERE token_digest = %s
               AND redeemed_at IS NULL
               AND expires_at > %s
         RETURNING account_id, event_id
            """,
            (now, digest, now),
        )
        if row is None:
            return False

        db.execute(
            "UPDATE accounts SET password_hash = %s WHERE account_id = %s",
            (new_password_hash, row["account_id"]),
        )
        db.execute(
            "UPDATE user_sessions SET revoked_at = %s WHERE account_id = %s AND revoked_at IS NULL",
            (now, row["account_id"]),
        )
        return True
```

The 15-minute lifetime is an example policy choice, not a universal standard. Choose it from the threat model, expected mail latency, support burden, and the sensitivity of the account. Persist an absolute UTC expiry and let the database predicate decide. Application timers disappear on restart and cannot settle a race between two requests.

Expiry is not consumption.

There is another race before redemption. Repeated requests can create several live tokens unless issuance invalidates earlier unredeemed rows for that account in the same transaction. Decide the policy explicitly: either one active recovery capability per account, or multiple independently traceable capabilities. The first is easier to explain to support and narrows exposure, but an older email will stop working.

Return the same public response for known and unknown addresses, and keep timing differences from becoming an easy account-enumeration signal. Rate limits belong at several boundaries: source, account, and dispatch queue. They protect recipients from message floods as much as they protect infrastructure.

## How should delivery and retries affect the audit trail?

Commit the recovery event before attempting delivery. Then enqueue a message containing the event ID and a template identifier; do not place the plaintext token in ordinary application logs. The worker can resolve the protected payload, send once per idempotency key, and update dispatch state. If queue policy requires the URL in a payload, encrypt that payload, restrict access, and expire it no later than the recovery capability.

Retries need a ceiling and a reason code. A timeout is ambiguous: the receiver may have accepted the message even though the sender did not receive confirmation. Reusing the same logical event and idempotency key prevents an ambiguous retry from becoming a second recovery grant. The audit row can record attempt count, last attempt time, and a normalized outcome while detailed transport diagnostics age out sooner.

This separation matters during incidents. A support agent can say that event `R7K...` was requested, handed off, and never redeemed. The agent still cannot open the reset link. **Good evidence reduces privilege.**

Delivery is not redemption.

Test the awkward paths: two simultaneous redemptions, redemption exactly at expiry, resend after a worker timeout, an unknown address, a bounced mailbox, queue replay, database rollback after password hashing, and session revocation failure. Use a controlled mailbox domain in preproduction, but keep assertions centered on state transitions. Inbox placement can vary outside the application boundary.

## Retain the record, discard the capability

Retention should be field-specific. Clear the digest during successful redemption; delete it after expiry for unused events. Expire queued protected payloads and redact URLs from HTTP access logs, error trackers, tracing attributes, and analytics. Keep the smaller event ledger only for the documented support, security, or legal period, then delete it through the same scheduled and auditable process used for other customer records. Backups need matching expiry handling because deleting the primary row does not erase historical copies immediately.

The trade-off is real. Dropping message bodies and transport payloads means a later investigation cannot reproduce the exact MIME content or recovery URL. That is intentional: those artifacts carry more disclosure risk than evidentiary value. Preserve the template version, policy version, timestamps, normalized outcomes, and immutable event ID instead. If exact rendered content is legally required, retain a content digest or a separately controlled template snapshot rather than every personalized message.

Keep that boundary.

Cost follows the boundary. Short-lived queue payloads need fast, protected storage; compact audit rows need predictable retention and indexes on event ID, account ID, and token digest while active. After digest deletion, an archival projection can omit the lookup index and operational detail. Measure row size, index size, daily event volume, replica count, backup duration, and log duplication in the real system before setting a budget. Egress and mail charges still exist, but a design review should not ignore years of copied evidence because one send is easy to count.

The final rule is narrow: **retain enough to demonstrate request, dispatch, expiry, and single redemption, while making retained data incapable of authorizing another reset.** You lose packet-level reconstruction and personalized message replay. When something goes wrong, that can lengthen diagnosis; it also prevents the diagnostic estate from becoming a credential store.

## Further reading

- Google, Email sender guidelines: https://support.google.com/a/answer/81126
- NIST SP 800-63B, Digital Identity Guidelines: https://pages.nist.gov/800-63-3/sp800-63b.html
