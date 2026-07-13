# AWS Service Breadth

## Concept

LocalStack emulates over 100 AWS services, but in practice, a small set of services account for the overwhelming majority of customer usage. A TAM doesn't need mastery of all 100+, but needs fluent, conversational knowledge of the high-frequency ones and a general map of the rest.

**High-priority services to know well:**
- **S3** — object storage; extremely common as a baseline test target since almost every application touches it
- **Lambda** — serverless functions; one of the most complex to emulate well (runtime environments, layers, triggers) and a frequent source of parity discussion
- **DynamoDB** — NoSQL database; common in serverless architectures alongside Lambda
- **SQS / SNS** — queuing and pub/sub messaging; core to event-driven architectures
- **API Gateway** — HTTP API layer in front of Lambda; adds complexity because it chains with Lambda emulation
- **IAM** — permissions and policy simulation; historically a common parity gap area since fine-grained enforcement is hard to replicate perfectly
- **CloudFormation** — AWS's native IaC service; customers testing CloudFormation templates need this working end-to-end
- **ECS / EKS** — container orchestration; increasingly important as customers test containerized workloads
- **RDS, Secrets Manager, KMS, Step Functions, CloudWatch** — common secondary services that show up in more mature, multi-service test suites

**Why breadth matters more than depth for a TAM:** customers rarely use just one service — they're testing an *architecture* (e.g., API Gateway → Lambda → DynamoDB → SQS). Your job is recognizing which services are in a customer's stack, understanding how they chain together, and knowing which of those services tend to have more mature vs. newer/less mature emulation — not writing the emulation code yourself.

---

## Knowledge Checklist

- [ ] Can name the 8-10 most commonly used AWS services in LocalStack customer environments
- [ ] Understands what each of those services does in a real AWS architecture (one sentence each)
- [ ] Can describe a common multi-service architecture pattern (e.g., API Gateway → Lambda → DynamoDB)
- [ ] Knows which services have historically been harder to emulate with full fidelity (Lambda, IAM enforcement)
- [ ] Can ask a customer "which services are you testing against?" and understand the answer without needing it explained
- [ ] Aware that LocalStack's service list is large (100+) but usage is concentrated in a much smaller core set
