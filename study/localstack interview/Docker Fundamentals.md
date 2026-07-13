# Docker Fundamentals

## Concept

Docker is the containerization technology LocalStack is built on and distributed through — you cannot meaningfully support LocalStack customers without a working grasp of how containers behave.

**Core concepts to hold:**
- **Image vs. container** — an image is a static, versioned snapshot (the blueprint); a container is a running instance of that image. LocalStack ships as an image (`localstack/localstack`) that customers run as containers.
- **Docker Compose** — a way to define multi-container setups (or a single LocalStack container plus supporting services like a database) in a single YAML file, so a whole environment starts with one command (`docker-compose up`). Many customers manage LocalStack this way rather than raw `docker run` commands.
- **Volumes** — persistent storage mounted into a container so data survives beyond the container's lifecycle. Relevant when customers want LocalStack state to persist across restarts.
- **Networking** — containers get their own internal network namespace by default. Customers need to expose ports (e.g., `-p 4566:4566`) to reach LocalStack from the host, and networking issues (port conflicts, containers on different Docker networks unable to reach each other) are a very common source of "it's not working" tickets.
- **Environment variables** — LocalStack's behavior (which services are active, auth token, debug mode, persistence settings) is largely controlled through environment variables passed at container start, not a config file.
- **Resource limits** — Docker containers can be capped on CPU/memory; under-provisioned CI runners or laptops are a frequent cause of LocalStack instability or slow starts.

**Why this matters for a TAM:** most "LocalStack won't start" or "can't connect to LocalStack" tickets trace back to a Docker fundamentals issue (port already in use, wrong network, insufficient memory) rather than a LocalStack bug. Recognizing this quickly lets you triage confidently instead of always escalating.

---

## Knowledge Checklist

- [ ] Can explain the difference between a Docker image and a container
- [ ] Understands Docker Compose and can read a basic `docker-compose.yml`
- [ ] Knows how port mapping works and can recognize a port-conflict symptom
- [ ] Understands volumes and why persistence needs to be explicitly configured
- [ ] Knows LocalStack is primarily configured via environment variables at container start
- [ ] Can identify resource-constraint symptoms (slow start, crashes) as a Docker/host issue rather than a product bug
- [ ] Comfortable reading basic `docker logs` output to spot an obvious startup error
