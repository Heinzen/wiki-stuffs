# Observability Tooling

## Concept

Observability tooling refers to features that help customers understand *what's happening inside* LocalStack while it runs — critical for a product that otherwise behaves like a black box sitting between a customer's tooling and their test results.

**Key feature to know: App Inspector**
- A LocalStack observability layer that surfaces **misconfigurations and missing permissions** *before* they would cause a failure against real AWS.
- Conceptually, it's a diagnostic/insight tool: rather than a customer's test simply failing with an opaque error, App Inspector aims to proactively flag the underlying cause (e.g., an IAM policy that's too permissive, a resource misconfigured in a way that would fail in production).
- This ties directly into the core value proposition of "catch problems locally, before they cost you in production" — observability tooling is what makes that promise tangible rather than aspirational.

**Broader observability concepts relevant to LocalStack:**
- **Logs** — `docker logs` output from the LocalStack container is often the first troubleshooting step for both customers and support
- **Request tracing** — being able to see which API calls were made, in what order, and what responses came back, is essential for debugging complex multi-service test failures
- **CloudWatch Logs emulation** — LocalStack emulates AWS's own CloudWatch Logs service, meaning customers can also observe their *application's* logging behavior locally, not just LocalStack's own container logs

**Why this matters for a TAM:** observability features are a direct value-add talking point — they reduce the time customers spend debugging their own test failures, which is a concrete, demonstrable ROI point for QBRs and expansion conversations. They're also often where you point a customer first before escalating an issue to Support Engineering.

---

## Knowledge Checklist

- [ ] Can explain what App Inspector does and the problem it solves (catching misconfigurations before production)
- [ ] Understands the difference between LocalStack's own container logs and emulated CloudWatch Logs
- [ ] Knows why request tracing matters for debugging multi-service test failures
- [ ] Can position observability tooling as a concrete value/ROI point in a QBR
- [ ] Knows to direct customers to check logs/observability tooling before escalating a "mystery failure" ticket
- [ ] Understands how observability ties back into the core "catch it locally, not in prod" value proposition
