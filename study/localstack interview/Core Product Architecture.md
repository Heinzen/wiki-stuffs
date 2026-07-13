# Core Product Architecture

## Concept

LocalStack is, at its core, an **AWS cloud emulator that runs locally**, typically inside a Docker container. Instead of provisioning real AWS resources in a remote data center, developers point their existing AWS tooling (CLI, SDKs, Terraform, CDK, CloudFormation) at a LocalStack endpoint running on their own machine or in a CI pipeline, and LocalStack intercepts those API calls and simulates the corresponding AWS service behavior.

**How it's structured:**
- **Single container model** — LocalStack runs as one Docker container exposing a unified API gateway (traditionally on port 4566) that routes incoming requests to the appropriate emulated service internally, rather than requiring a separate container per AWS service.
- **API-compatible, not infrastructure-identical** — LocalStack doesn't replicate AWS's actual backend infrastructure. It reimplements the *API contract* of each service so that any client speaking the AWS API sees behavior consistent with real AWS, without the underlying implementation being AWS's own code.
- **Lifecycle** — a typical usage flow is: pull the image → start the container (`docker run` or `docker-compose up`, or via the LocalStack CLI) → point tooling at the local endpoint → run tests/dev workflows → tear the container down. This start/stop cycle is often automated inside CI pipelines, spinning up fresh for every test run.
- **State** — by default, state lives inside the running container and is lost on teardown unless persistence is explicitly configured, which matters for customers who want consistent test environments across runs.

**Why this matters for a TAM:** understanding that LocalStack is a single orchestrating container (not a multi-service cluster) helps you reason about resource constraints customers hit (memory/CPU on the host or CI runner), and explains why "container won't start" or "container crashed under load" are common early troubleshooting categories you'll see reported.

---

## Knowledge Checklist

- [ ] Can explain what LocalStack emulates (AWS APIs) vs. what it does not (AWS's actual backend infrastructure)
- [ ] Understands the single-container / unified gateway model and default port (4566)
- [ ] Can describe a typical start → use → teardown lifecycle
- [ ] Knows that state is ephemeral by default unless persistence is configured
- [ ] Can explain, in one sentence, what LocalStack is to both a technical and non-technical audience
- [ ] Understands common resource-related failure modes tied to running a single container (memory, CPU limits)
- [ ] Can distinguish LocalStack's architecture from running a real AWS sandbox account
