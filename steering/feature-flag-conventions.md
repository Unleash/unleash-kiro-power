# Feature flag conventions

## Customize this file for your team

- **Naming convention:** default is `{domain}-{feature}-{variant}` (e.g. `checkout-stripe-integration`). Replace with your team's pattern.
- **Flag types in scope:** default lists `release`, `experiment`, `operational`, `kill-switch`, `permission`. Trim or extend.
- **What counts as "high-risk":** default mentions payments, authentication, data migrations, external integrations. Add the categories that matter for your business.
- **Mandatory vs. optional evaluation:** default is "evaluate before implementing high-risk changes". Some teams require flags for *all* user-facing changes — adjust to your bar.

When Kiro reads this file, it should apply the rules as written. The bullets above are for the humans editing the file.

---

# Feature flag conventions

These conventions govern how feature flags are named, created, and used in this codebase. Apply them whenever code changes might benefit from a feature flag.

## 1. Evaluate risk before implementing

<!-- CUSTOMIZE: list the domains, file paths, or change patterns that should always trigger evaluation -->

Before implementing a change, check whether it falls into any of these categories:

- Payment processing or billing logic
- Authentication, authorization, or session handling
- Data migrations or schema changes
- External integrations (third-party APIs, webhooks, message queues)
- Performance-sensitive paths (caching, rate limiting, batching)
- Anything user-visible that changes existing behavior

If it does, it needs a flag: proceed to the detect-and-create steps below. If you're unsure, call the Unleash MCP server's `docs` tool for Unleash's guidance on when to flag a change. Don't use `evaluate` for this — it checks who gets an existing flag, not whether a change needs one.

## 2. Naming convention

<!-- CUSTOMIZE: replace the pattern below with your team's flag naming convention -->

Use the format **`{domain}-{feature}-{variant}`** for all new flags. Examples:

- `checkout-stripe-integration`
- `auth-sso-google`
- `api-rate-limiting`
- `search-vector-ranking-v2`

Rules:

- Lowercase only, kebab-case (no underscores, no camelCase).
- `domain` matches the top-level directory in `src/` where the change lives.
- `feature` is a short verbal phrase describing what the flag controls.
- `variant` is the rollout strategy or experiment arm (omit for simple release flags).

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

When in doubt, default to `release` for new features and `kill-switch` for external-dependency integrations.

## 4. Prefer reuse over creation

Always call `search` (`kind: flags`) with key words from the new flag's intent before creating one. If it returns a suitable flag, reuse it. Reuse reduces flag-sprawl and keeps related code paths consistent.

If nothing matches, create the flag with `manage_flag` (`action: define`). It starts off in every environment.

## 5. Wrap with the right SDK pattern

After `manage_flag` creates the flag, write the guard code yourself. Find how the codebase already checks flags — the SDK client, a wrapper function, a hook — and use the same pattern. Call `docs` if you need SDK details.

For a TypeScript Express endpoint, expect output like:

```typescript
if (unleash.isEnabled('checkout-stripe-integration', context)) {
  return stripeService.processPayment(request);
} else {
  return legacyPaymentService.process(request);
}
```

Keep the fallback branch realistic — if there is no legacy path, the fallback should be a clear failure message, not a silent no-op.

## 6. Clean up after rollout

Feature flags are temporary by design. See `cleanup-cadence.md` for the removal workflow.

## Tools available from this Power

The Unleash MCP server (configured in `mcp.json`) exposes the following tools:

`search`, `inspect`, `evaluate`, `audit`, `docs`, `manage_flag`, `release`, `kill_switch`, `experiment`, `configure`, `send_feedback`.

Read-only tools (`search`, `inspect`, `evaluate`, `audit`, `docs`) are pre-approved. Write tools require explicit user confirmation. Use `search` for inventory discovery before creating new flags — it also drives the flag-audit hook.
