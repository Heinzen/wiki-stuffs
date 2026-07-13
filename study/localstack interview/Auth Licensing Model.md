# Auth & Licensing Model

## Concept

As of the March 2026 packaging change, LocalStack's licensing and authentication model is one of the most important — and most recently changed — things a TAM needs to know cold, since it's the subject customers will ask about most directly.

**What changed:**
- **Single unified Docker image** — the old split between a free "Community" image and a paid "Pro" image was eliminated. `localstack/localstack` and `localstack/localstack-pro` now contain the identical image; what a customer can actually access is determined entirely by their **auth token's entitlements**, not which image they pulled.
- **Auth token now required to start LocalStack at all** — fully anonymous usage is no longer possible, even for free/hobby use cases. This is a significant shift customers migrating from older setups need to be walked through.
- **New pricing tiers:**
  - **Hobby** — free, for non-commercial use, roughly equivalent to the old Community image
  - **Base** — a discounted paid tier
  - **Ultimate** — top tier, available with a 45-day free trial
- **CI credit limits removed** — across all tiers, a change from the historical model where CI usage was metered — a major win for CI-heavy customers
- **New CLI** — a rebuilt CLI automatically handles authentication and pulls the latest image, reducing manual config errors

**Why this matters for a TAM:** licensing/auth questions are the single most common "how do I..." conversation a TAM will have, especially with legacy customers migrating off the old Community-image model. Getting the tier-to-entitlement mapping wrong in a customer conversation directly damages trust and can create real commercial friction (e.g., a customer discovering mid-project that their tier doesn't cover a service they need).

---

## Knowledge Checklist

- [ ] Can explain the shift from split Community/Pro images to a single unified image
- [ ] Understands that entitlements are now tied to the auth token, not the image pulled
- [ ] Knows the three current tiers (Hobby, Base, Ultimate) and roughly what each is for
- [ ] Can explain what changed with CI credits and why it matters to CI-heavy customers
- [ ] Can walk a customer through migrating from the old Community-image model to the new token-based model
- [ ] Knows the new CLI handles auth/image-pulling automatically
