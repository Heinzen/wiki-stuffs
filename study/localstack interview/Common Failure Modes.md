# Common Failure Modes

## Concept

Recognizing the handful of recurring reasons LocalStack "doesn't work" for a customer lets a TAM triage quickly and confidently — routing genuine product bugs to engineering while resolving configuration issues directly.

**The most common categories:**
1. **Version mismatches** — a customer's tooling, extension, or documentation reference is out of sync with the LocalStack version they're actually running (especially relevant given the monthly, calendar-versioned release cadence). Symptoms: something "used to work" and now doesn't after an upgrade, or a feature described in docs doesn't behave as expected.
2. **Auth token misconfiguration** — since the March 2026 shift to mandatory auth tokens, a large share of "won't start" or "access denied" issues trace back to a missing, expired, or incorrectly scoped token, or a tier that doesn't entitle access to a needed service.
3. **Service not yet supported at required fidelity** — a genuine parity gap: the customer's exact use case touches an API call or behavior LocalStack doesn't yet emulate accurately. This is the category that actually needs product/engineering involvement.
4. **Container resource limits** — insufficient memory/CPU allocated to the Docker container (common on constrained CI runners or older laptops), leading to slow starts, timeouts, or crashes that look like product bugs but are host/environment issues.
5. **Networking/configuration issues** — port conflicts, incorrect endpoint URLs in IaC tooling, containers on mismatched Docker networks — these are Docker/config fundamentals issues, not LocalStack bugs.

**The TAM's job**: quickly sort an incoming issue into "configuration/environment" (resolvable directly or with light guidance) vs. "genuine parity/product gap" (needs to be logged and routed to engineering with a clear reproduction case). Misclassifying either direction either burns unnecessary engineering time or leaves a customer's real blocker unresolved.

---

## Knowledge Checklist

- [ ] Can list the five common failure mode categories from memory
- [ ] Can ask the right diagnostic questions to distinguish a config issue from a genuine parity gap
- [ ] Understands why version mismatches are especially relevant given LocalStack's monthly release cadence
- [ ] Knows auth token issues are a leading cause of "won't start" tickets post-March 2026
- [ ] Can recognize resource-constraint symptoms (slow start, crash under load) as environment issues, not bugs
- [ ] Comfortable triaging an issue to "resolve directly" vs. "escalate to engineering with repro steps"
