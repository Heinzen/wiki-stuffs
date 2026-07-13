# CI/CD Pipelines

## Concept

CI/CD (Continuous Integration / Continuous Delivery) pipelines are automated workflows that build, test, and deploy code — and for most LocalStack customers, this is the *primary* environment where the product actually runs, more so than a developer's local laptop.

**How LocalStack fits into a pipeline:**
1. A pipeline job (in GitHub Actions, GitLab CI, Jenkins, CircleCI, etc.) starts a LocalStack container as a service alongside the test job
2. The test suite runs against the LocalStack endpoint instead of real AWS
3. Tests pass/fail based on whether the application behaves correctly against the emulated services
4. The container tears down when the job finishes — meaning **every pipeline run typically starts from a fresh LocalStack instance**, unlike a persistent local dev environment

**Key concepts:**
- **Ephemeral by default** — pipeline runs don't usually persist state between runs, which affects how customers structure their tests (each test often needs to set up its own fixtures)
- **CI credits (historical)** — LocalStack's licensing historically metered usage in CI environments via credits; this changed with the March 2026 pricing overhaul, which removed CI credit limits across all tiers — a meaningful selling point and a common topic in renewal/expansion conversations for CI-heavy accounts
- **Startup time matters** — since containers spin up fresh for every job, slow LocalStack startup directly costs the customer CI minutes (and money, depending on their CI provider's billing model), making performance a real business concern, not just a technical nicety
- **Version pinning** — customers often pin a specific LocalStack image version in their pipeline config to avoid unexpected breaking changes from automatic updates; this intersects directly with the versioning/release cadence topic

**Why this matters for a TAM:** CI/CD is where "adoption depth" actually lives for most customers — it's a better usage signal than login frequency (which barely applies to a tool with no real UI). Understanding pipeline mechanics also helps you correctly diagnose whether an issue is a LocalStack bug vs. a CI configuration problem.

---

## Knowledge Checklist

- [ ] Can describe how a LocalStack container typically gets started and torn down inside a CI job
- [ ] Understands why CI runs are usually stateless/ephemeral by default
- [ ] Knows what changed with CI credits in the March 2026 pricing update and why it matters commercially
- [ ] Can explain why startup time/performance is a business cost issue, not just a technical detail
- [ ] Understands version pinning and why customers do it
- [ ] Can name at least 2-3 common CI platforms (GitHub Actions, GitLab CI, Jenkins) customers are likely using
