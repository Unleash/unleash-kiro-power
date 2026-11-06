# Feature flag conventions

## Customize this file for your team

- **Naming convention:** default is `{domain}-{feature}` (e.g. `checkout-stripe-integration`), with kill switches named `disable-{domain}-{feature}`. Replace with your team's pattern.
- **Flag types in scope:** default lists `release`, `experiment`, `operational`, `kill-switch`, `permission`. Trim or extend.
- **When a flag is required:** default is the tier table below (Critical and High require a flag, Medium asks, Low skips). Some teams require flags for *all* user-facing changes — move rows between tiers to match your bar.
- **Change-size bands and exemptions:** default bands are under 20, 20 to 100, and over 100 changed lines; tests and docs are exempt. Tune both.

When Kiro reads this file, it should apply the rules as written. The bullets above are for the humans editing the file.

---

# Feature flag conventions

These conventions govern how feature flags are named, created, and used in this codebase. Apply them whenever code changes might benefit from a feature flag.

## 1. When to require a feature flag

<!-- CUSTOMIZE: move change types between tiers, or change the size bands and exemptions, to match your team's bar -->

A change takes the tier of the riskiest thing it does. Judge what the change does, not the words in it: fixing a typo in a login error message is Low, not Critical.

| Tier | Changes | Flag |
|---|---|---|
| Critical | Authentication, authorization, sessions; payments, billing; security-sensitive code (encryption, hashing, input sanitization, CSRF/XSS protection); database schema changes, migrations, destructive queries | Required |
| High | New user-facing features; user-visible changes to existing behavior; new or changed API endpoints and contracts; calls to external services, webhooks, message queues; large changes | Required |
| Medium | New async work or background jobs; changes to shared state; performance-sensitive paths (caching, rate limiting, batching); medium changes | Ask the user, recommending a flag when the change reaches production traffic |
| Low | Bug fixes and refactors that keep behavior; small changes | Not needed |

- **Change size:** small is under 20 changed lines, medium 20 to 100, large over 100. The more lines a change touches, the more likely it hides a bug that only shows in production, where a flag can turn it off without a redeploy.
- **Exempt:** test code and documentation, such as `*.test.*`, `*_test.go` or `docs/`: they never run in production. Config files are exempt unless they change runtime behavior in a Medium or higher category, such as a rate limit or a cache setting.
- A change in a domain listed in `high-risk-domains.md` always requires a flag, whatever its tier.

To apply this table to a change, follow `evaluate-change.md`. Don't use the MCP server's `evaluate` tool for this — it checks who gets an existing flag, not whether a change needs one.

## 2. Naming convention

<!-- CUSTOMIZE: replace the pattern below with your team's flag naming convention -->

Use the format **`{domain}-{feature}`** for all new flags. Examples:

- `checkout-stripe-integration`
- `auth-sso-google`
- `api-rate-limiting`
- `search-vector-ranking-v2`

Rules:

- Lowercase only, kebab-case (no underscores, no camelCase).
- `domain` matches the top-level directory in `src/` where the change lives.
- `feature` is a short phrase describing what the flag controls.
- An experiment is one flag whose values are the arms, so arms don't go in the name.
- **Kill switches** are named after what they turn off: `disable-{domain}-{feature}`, such as `disable-payments-stripe`.
- **Unleash's own rule wins:** a project can enforce a naming pattern. Call `inspect` (`kind: project`) and follow its pattern when it has one.

## 3. Flag types

<!-- CUSTOMIZE: adjust the expected lifetimes below to match your team's internal guidelines. The defaults shown are Unleash's out-of-the-box lifetimes (set per type in the Unleash admin UI under Configure → Feature flag types). -->

Pick the type that matches the flag's purpose. The MCP server's `manage_flag` tool accepts these five types, plus `sunset`.

| Type | Purpose | Default expected lifetime |
|---|---|---|
| `release` | Gradual feature rollouts | **40 days** — temporary; remove after rollout stabilizes |
| `experiment` | A/B tests, multivariate experiments | **40 days** — temporary; remove after the experiment concludes |
| `operational` | Short-term system behavior toggles (caching, batching, rate limits) | **7 days** — short-lived; replace with config or remove after the operational change ships |
| `kill-switch` | Emergency shutdown for external integrations or high-risk paths | **Permanent** — no expected lifetime; meant to live indefinitely |
| `permission` | Role- or tier-based access (paid features, beta access) | **Permanent** — no expected lifetime; ties to user attributes, not a rollout |

Lifetimes come from Unleash's product defaults and are used by Unleash to mark flags "potentially stale" once they exceed the threshold for their type. Teams customize these per project in the Unleash admin UI.

When in doubt, use `release`. A change that calls an external dependency also gets a `kill-switch`, if the call has none yet, so the dependency can be cut off on its own.

## 4. Prefer reuse over creation

Look for an existing flag before creating one, following `find-existing-flag.md`. Reuse cuts flag sprawl and keeps related code paths consistent.

`release`, `kill_switch` and `experiment` create a flag that doesn't exist yet, with its type's defaults and no description. Create a new flag with `manage_flag` (`action: define`) first, so it gets the type from this file and a description. It starts off in every environment.

## 5. Wrap the change

Write the guard code yourself, following `wrap-change.md`: the check runs on every request, matches how the codebase already checks flags, and has a real off path.

## 6. Remove after rollout

Feature flags are temporary by design. See `remove-flag.md` for taking a flag out of the code, and `audit-cadence.md` for when a flag is due.

## Tools available from this Power

The Unleash MCP server (configured in `mcp.json`) exposes the following tools:

`search`, `inspect`, `evaluate`, `audit`, `docs`, `manage_flag`, `release`, `kill_switch`, `experiment`, `configure`, `send_feedback`.

Read-only tools (`search`, `inspect`, `evaluate`, `audit`, `docs`) are pre-approved. Write tools require explicit user confirmation. Use `search` for inventory discovery before creating new flags — it also drives the flag-audit hook.
