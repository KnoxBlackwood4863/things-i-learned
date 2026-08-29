# SMS OTP Delivery Failure in US/EU: Carrier Filtering, Sender Registration, and 2FA Routes

Short answer: treat SMS OTP delivery as a chain of policy decisions, not a single API call; register the sender where required, keep dedicated and shared routes observable, and make anti-fraud controls fail closed without trapping legitimate users.

An OTP can be generated correctly and still never reach a handset. The message crosses an application queue, a messaging provider, one or more carrier gateways, and a handset policy layer. US and EU carriers apply different filtering signals, and a sender that is acceptable in one country can be throttled or rejected in another. The useful unit of reliability is therefore a country-and-carrier path, measured by outcome, not an aggregate “sent” count.

## What actually fails between login and the handset?

Start with a failure map. Application errors are the easy part: an expired code, a duplicate request, or a queue timeout is visible in your logs. Carrier filtering is less polite. A submission can be accepted upstream while the handset never displays it, so a provider-side success response is not proof of delivery. In a real incident review, I want to line up the timestamps for the login attempt, queue handoff, provider acceptance, carrier status, and code entry; the missing interval usually tells us which owner can act. If the carrier status is absent, that is an evidence gap to record, not a reason to label the request delivered. The same dashboard should show the denominator for each country and route, because a single successful test number says little about a mixed US/EU population.

Start with the carrier.

The message body is one signal. Repeated templates, suspicious URLs, unusually high bursts, and sender identity that does not match registration records can increase filtering risk. Shared routes add another variable: traffic from unrelated senders shares a route reputation, and a neighbor's abuse can affect latency or filtering for your traffic. That is a route property, not evidence that your OTP code is wrong.

US application-to-person traffic may require sender registration and campaign information before normal throughput is available. EU traffic is not one policy either; each destination country and carrier can apply its own sender rules, content filters, and consent expectations. “US” and “EU” are useful dashboard dimensions, but they are not delivery guarantees.

There is a security boundary here. A provider should not disclose the exact filtering rule for a rejected message, because that information helps fraud operators tune around it. Your system should expose a neutral user state such as “try another verification method,” while retaining the detailed provider status for operators.

## How should a 2FA login architecture handle carrier filtering and sender registration?

I use four invariants in an architecture decision record:

1. The code is generated and verified by the authentication service, never by the messaging adapter.
2. Every send has an idempotency key tied to the login attempt, destination, and purpose.
3. Delivery telemetry is partitioned by destination country, carrier when known, sender, route class, and registration state.
4. A fallback method is selected by policy, not by blindly retrying the same SMS path.

The critical path stays small. The adapter receives a prepared message, submits it once, and records the provider's correlation identifier. Verification remains possible only within a short expiry window and after an attempt-specific rate check.

```python
from dataclasses import dataclass
from datetime import datetime, timedelta, timezone
import hashlib
import hmac
import secrets


@dataclass(frozen=True)
class OtpAttempt:
    attempt_id: str
    destination: str
    country: str
    code_digest: str
    expires_at: datetime
    idempotency_key: str


def create_attempt(user_id: str, destination: str, country: str) -> tuple[OtpAttempt, str]:
    code = f"{secrets.randbelow(1_000_000):06d}"
    salt = secrets.token_bytes(16)
    digest = hashlib.scrypt(code.encode(), salt=salt, n=2**14, r=8, p=1).hex() + ":" + salt.hex()
    now = datetime.now(timezone.utc)
    attempt_id = secrets.token_urlsafe(12)
    return (
        OtpAttempt(
            attempt_id=attempt_id,
            destination=destination,
            country=country,
            code_digest=digest,
            expires_at=now + timedelta(minutes=5),
            idempotency_key=f"otp:{user_id}:{attempt_id}",
        ),
        code,
    )


def submit_sms(attempt: OtpAttempt, code: str, sender: str) -> dict:
    # The real adapter should use your provider's HTTPS client and persist its response.
    return {
        "to": attempt.destination,
        "from": sender,
        "body": f"Your login code is {code}. It expires in 5 minutes.",
        "idempotency_key": attempt.idempotency_key,
        "country": attempt.country,
    }
```

The example intentionally leaves the transport generic. The important contract is the idempotency key and the separation between code verification and delivery. Store the salted digest, expiry, attempt status, and provider correlation ID. Do not put the plaintext code or a full phone number in ordinary logs.

For sender registration, make “registered,” “pending review,” and “not eligible” explicit states in configuration. A deployment check should refuse to route US traffic through an unregistered sender when registration is a prerequisite. For EU destinations, keep a country policy table rather than assuming one regional default. Policy data should be versioned so an incident review can answer which rule was active when the message was sent.

## Which route should carry an OTP: dedicated, shared, or fallback?

| Option | Strength | Boundary | Operational proof |
| --- | --- | --- | --- |
| Dedicated sender and route | More control over identity and reputation | Registration work and country-specific throughput limits | Delivery and filtering rates by sender and carrier |
| Shared route | Fast initial coverage and less route administration | Reputation is influenced by other traffic; filtering can be opaque | Route-level latency, status mix, and carrier slices |
| Non-SMS fallback | Avoids a blocked SMS path | Requires an enrolled authenticator, email, or recovery channel | Completion rate and fallback enrollment health |

The rejected option is “retry the same SMS until it arrives.” Retries amplify bursts, can trigger anti-fraud controls, and create multiple valid-looking codes that confuse users. A bounded retry may be valid for a transient submission timeout when the idempotency key is unchanged; it is not a response to an unknown carrier filter. When the route is unsuitable for a destination, switch to an enrolled authenticator app, passkey, or recovery flow and tell the user what to do next.

The catch is that a dedicated route does not remove compliance obligations or guarantee handset delivery. Stay with a shared route when the volume is small, destinations are predictable, and you have adequate carrier-level telemetry. Choose another design when your login population spans many countries and a single SMS path would become a critical dependency.

## How do you test and observe US/EU OTP delivery without leaking secrets?

Measure the whole funnel: code requested, message submitted, provider acceptance, carrier status, handset delivery receipt when available, code entered, and login completed. Keep these events correlated by a non-sensitive attempt ID. Break dashboards down by country, carrier, sender, route class, template version, and registration state. A healthy aggregate can hide one carrier dropping half of its traffic.

Synthetic tests should use consented numbers on representative carriers, with a quiet schedule and a fixed template. Test the negative paths too: an expired code, a second request, a blocked destination, and a user who chooses fallback. Record latency percentiles and completion rate, not just a green API response.

I still treat delivery receipts as evidence with gaps. Some carriers do not provide a handset receipt, and receipt semantics differ across routes. Your mileage may vary; document the evidence level for each destination instead of converting “accepted” into a false certainty.

Anti-fraud checks belong before the send: velocity per account and destination, IP and device risk, SIM-change signals where lawful, and a global budget for repeated attempts. Return the same user-facing error for risky and unknown delivery outcomes so the endpoint cannot become a phone-number or policy oracle. Operators can see the richer reason codes behind that response.

Email and SMS also have different compliance surfaces. Email sending requires authenticated domains, consent handling, and reputation management; SMS adds sender registration, carrier throughput, and local messaging rules. Keep those concerns in separate adapters behind one verification policy, so adding a channel does not weaken the common attempt and audit model.

## What should the decision record say about limits and ownership?

Name an owner for registration data, an owner for routing policy, and an owner for the authentication state machine. Set a review date for each country rule. Define an incident threshold such as a sustained fall in completed logins for one carrier slice, then page the route owner rather than the identity team by default.

This design is not suitable when users have no pre-enrolled fallback and SMS is the only recovery factor; the right fix is enrollment and support planning, not more retries. It is also a poor fit for high-assurance authentication where SIM swap risk is unacceptable. Use passkeys or a hardware-backed authenticator for that class of account, while retaining SMS as a carefully monitored recovery option only if the risk policy allows it.

The practical decision is boring and durable: register senders before scaling, isolate route reputation in telemetry, cap retries, and give the user another way to finish 2FA. That is how an OTP login survives carrier filtering without pretending that “sent” means “received.”

## References

- https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- https://www.twilio.com/docs/messaging/compliance/a2p-10dlc
