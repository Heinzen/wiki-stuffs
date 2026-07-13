# Versioning Scheme

## Concept

LocalStack switched from traditional semantic versioning (major.minor.patch, where numbers signal the size/riskiness of a change) to **calendar versioning** (`YYYY.MM.patch`) as part of the March 2026 packaging overhaul. Understanding why this matters — and how to talk customers through it — is a small but frequently relevant piece of TAM knowledge.

**Key ideas:**
- **Format**: a release is labeled by the year and month it shipped, plus a patch number — e.g., the March 2026 release is version `2026.3.0`. A subsequent patch to that same release would be `2026.3.1`.
- **What this signals**: calendar versioning tells you *when* something shipped, not how big or risky the change is (unlike semver, where a major version bump signals breaking changes). This means customers can't infer "this is a safe patch update" just from the version number the way they might with semver — they need to actually read release notes.
- **Monthly release cadence**: LocalStack ships on roughly a monthly cycle, meaning there's a steady, predictable stream of new versions to track — relevant both for customers deciding when to upgrade and for a TAM proactively flagging what's in each release.
- **Version pinning**: because releases are frequent and calendar-based, many customers pin a specific version in their CI/CD configs (rather than always pulling `latest`) to avoid unplanned behavior changes disrupting their pipelines. Helping customers think through a sensible pinning/upgrade cadence is a legitimate TAM value-add.

**Why this matters for a TAM:** version and release conversations come up constantly — "should we upgrade," "what changed in this release," "why did our pipeline break after an update." Being fluent in how the versioning scheme works (and proactively summarizing what's new/breaking in recent releases) is one of the easiest ways to demonstrate ongoing value between QBRs.

---

## Knowledge Checklist

- [ ] Can explain the `YYYY.MM.patch` versioning format and give the current example (e.g., `2026.3.0`)
- [ ] Understands why calendar versioning doesn't signal risk/breaking-change size the way semver does
- [ ] Knows LocalStack ships on roughly a monthly cadence
- [ ] Can explain why customers pin specific versions in CI/CD rather than always using `latest`
- [ ] Comfortable proactively summarizing "what's new" from recent release notes for a customer conversation
- [ ] Can explain the risk of *not* pinning a version in a CI pipeline
