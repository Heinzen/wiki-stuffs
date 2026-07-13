# Release Cadence

## Concept

LocalStack ships new releases roughly monthly, using the calendar versioning scheme (`YYYY.MM.patch`). For a TAM, the release cadence isn't just a technical detail — it's a recurring communication responsibility.

**Why the cadence matters operationally:**
- **Predictable but frequent** — monthly releases mean there's always something new to track, but also means customers can reasonably plan around a known rhythm rather than being surprised by ad hoc updates
- **Breaking change risk** — not every release is purely additive; some releases change behavior in ways that can break a customer's pipeline if they're not prepared, especially given LocalStack's history of significant packaging shifts (e.g., the March 2026 unified image/auth-token change)
- **Customer awareness varies widely** — some customers actively track release notes; many don't, and only discover a change when something breaks. A proactive TAM closes this gap rather than waiting for a reactive ticket

**The TAM's responsibilities tied to release cadence:**
- **Proactive communication** — summarizing relevant changes for at-risk or high-value accounts before they hit a breaking change, rather than after
- **Risk flagging** — identifying which upcoming or recent releases are likely to affect a specific customer's setup (e.g., a customer relying on a service that changed behavior)
- **Upgrade guidance** — helping customers think through when and how to upgrade, including version-pinning strategy, rather than leaving them to figure it out via trial and error
- **Feeding customer feedback back** — release cadence also creates a recurring opportunity to gather feedback ("did this release address the issue you raised last quarter?") that feeds the Voice of Customer process

**Why this matters for a TAM:** proactively managing the release cadence relationship is one of the easiest ongoing ways to demonstrate value between formal QBRs — it shows the customer you're paying attention to their specific setup, not just running a generic playbook.

---

## Knowledge Checklist

- [ ] Knows LocalStack ships on roughly a monthly cadence
- [ ] Can explain the risk of customers being surprised by breaking changes they didn't track
- [ ] Understands the TAM's role in proactively communicating relevant release changes to at-risk/high-value accounts
- [ ] Can describe how to identify which customers are likely affected by a given release
- [ ] Understands version-pinning as part of upgrade guidance
- [ ] Can connect release cadence communication back to ongoing trust-building between QBRs
