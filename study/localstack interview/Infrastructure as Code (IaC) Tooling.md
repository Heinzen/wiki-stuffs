# Infrastructure-as-Code (IaC) Tooling

## Concept

Infrastructure as Code means defining cloud resources through code/config files rather than manual console clicks. Since LocalStack customers are testing infrastructure, not just application code, IaC tools are central to how they actually use the product.

**Tools to know:**
- **Terraform** — the most widely used IaC tool in the LocalStack customer base. Written in HCL, provider-agnostic, works via `plan`/`apply`. To target LocalStack instead of real AWS, endpoint URLs are overridden — either manually or via `tflocal`, a LocalStack-provided wrapper that handles this automatically.
- **AWS CDK (Cloud Development Kit)** — lets developers define infrastructure using real programming languages (TypeScript, Python, etc.), which CDK then compiles down to CloudFormation templates. LocalStack supports CDK workflows via a similar wrapper concept (`cdklocal`).
- **CloudFormation** — AWS's native, JSON/YAML-based IaC service. Customers testing CloudFormation templates need LocalStack to correctly emulate the CloudFormation service itself, not just the resources it creates — an extra layer of complexity.
- **Serverless Framework** — a higher-level tool popular for Lambda-centric applications, often layered on top of CloudFormation under the hood.

**Why this is central to LocalStack specifically:** these tools all work by talking to AWS's public APIs, which is precisely the mechanism LocalStack intercepts. Customers using IaC are typically testing their *entire environment definition* — not a single service — which raises the stakes: a parity gap in a rarely-used resource type can block an entire `terraform apply` from succeeding, even if 95% of the resources emulate perfectly.

**Why this matters for a TAM:** IaC-heavy customers tend to be your more infrastructure-mature, often higher-value accounts. They test broader surface area, hit parity gaps more often, and care deeply about CI pipeline integration — making IaC fluency one of the highest-leverage technical skills for the role.

---

## Knowledge Checklist

- [ ] Can explain what Terraform is and how `plan`/`apply` works at a basic level
- [ ] Understands how endpoint overrides (or `tflocal`) point Terraform at LocalStack instead of real AWS
- [ ] Can explain, at a high level, what AWS CDK and `cdklocal` do
- [ ] Understands that CloudFormation adds an extra emulation layer (the orchestration service itself, not just resources)
- [ ] Can explain why IaC customers are more exposed to parity gaps than single-service API customers
- [ ] Recognizes IaC usage as a signal of a more technically mature, often higher-value account
