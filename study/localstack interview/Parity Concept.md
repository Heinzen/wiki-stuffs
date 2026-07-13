# The Parity Concept

## Concept

"Parity" refers to how closely LocalStack's emulated behavior matches real AWS's actual behavior for a given API call. This is arguably the single most important concept for a LocalStack TAM to understand deeply, because it sits at the center of both technical credibility and account risk conversations.

**Key ideas:**
- **LocalStack does not claim 100% parity.** No emulator can perfectly replicate a system as large and constantly evolving as AWS. Parity is always partial and improving over time, service by service.
- **Parity gaps come in different flavors:**
  - *Not-yet-implemented* — an API call or feature simply isn't built yet
  - *Behavioral difference* — the call exists but responds slightly differently than real AWS (e.g., error message wording, edge-case validation)
  - *Timing/consistency differences* — real AWS has certain eventual-consistency behaviors that are hard to replicate exactly locally
  - *Enforcement gaps* — services like IAM historically had looser permission enforcement locally than in production AWS, meaning a test could pass locally but fail against real AWS
- **Parity gaps are discoverable, not embarrassing.** Since Terraform, the CLI, and SDKs all talk to AWS through its public API, any mismatch in LocalStack's implementation becomes immediately visible to anyone using standard tooling — this is expected and part of why the parity roadmap is a constantly evolving, public conversation with the community.
- **Why customers care:** a parity gap can silently pass in local testing and then fail in production, which undermines the core value proposition (catching issues *before* they reach AWS). This makes parity gaps a legitimate churn/trust risk, not just a minor annoyance.

**Why this matters for a TAM:** you are the translator between "this doesn't work exactly like AWS" (a customer's frustration) and "here's where this sits on the roadmap and here's the workaround in the meantime" (a manageable technical risk conversation). Mishandling parity conversations is one of the fastest ways to erode customer trust.

---

## Knowledge Checklist

- [ ] Can explain in plain language why 100% AWS parity isn't a realistic goal
- [ ] Can describe the different types of parity gaps (not-implemented, behavioral, consistency, enforcement)
- [ ] Understands why parity gaps are especially risky for customers (local pass, prod fail)
- [ ] Knows the general workflow for how a parity gap gets logged, prioritized, and communicated back to a customer
- [ ] Can hold a calm, credible conversation with a frustrated customer about a parity gap without over-promising a fix date
- [ ] Aware of at least one or two historically known gap areas (e.g., IAM enforcement nuances) as a talking point
