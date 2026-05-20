# High-risk domains

## Customize this file for your team

**This is the file you'll edit most.** The example below uses a "payments" domain because it's universally recognizable, but the value of this Power comes from replacing it with your team's actual high-risk areas.

Decide which directories or modules in your codebase carry enough release risk that every change should be flag-protected by default. Common examples:

- **Payments / billing** — third-party processor calls, ledger updates, refund flows
- **Authentication / authorization** — login flows, session handling, permission checks
- **Data migrations** — schema changes, backfills, dual-write paths
- **Public API** — endpoints external customers depend on
- **External integrations** — webhooks, third-party APIs, message queue consumers
- **Compliance-sensitive paths** — anything touched by SOC 2, HIPAA, GDPR, PCI scope

Then, for each domain, decide:

1. The **directory glob** that scopes the policy (e.g., `src/payments/**/*.ts`).
2. The **default flag type** (often `kill-switch` for external dependencies, `release` for internal logic).
3. The **naming prefix** that flags in this domain must use (e.g., `payments-*`).
4. Whether **all changes** require a flag or only certain change types (new endpoints, behavior changes, etc.).

Duplicate the worked example below for each domain you want to protect.

---

# High-risk domains

This file is loaded when the conversation involves files in a high-risk domain. Apply the matching policy to all changes in scope.

## Worked example: payments

<!-- CUSTOMIZE: replace this entire section with your team's first high-risk domain. Keep the structure (Scope / Default flag type / Naming / Required actions). -->

### Scope

All changes that touch files matching:

- `src/payments/**/*.ts`
- `src/payments/**/*.tsx`
- `src/billing/**/*.ts`

### Default flag type

- **`kill-switch`** for any change that calls an external payment provider (Stripe, PayPal, Adyen, etc.).
- **`release`** for internal payment logic that doesn't cross a network boundary.

The `kill-switch` choice exists because external dependencies can fail at any time, and the ability to disable the integration in seconds is more valuable than a gradual rollout strategy.

### Naming

All flags in this domain must start with `payments-`. Examples:

- `payments-stripe-integration`
- `payments-paypal-checkout`
- `payments-refund-v2`
- `payments-fraud-check`

### Required actions

For every change in this domain:

1. **Evaluate first.** Call `evaluate_change` before writing implementation code. The tool will recommend a flag.
2. **Require a flag.** Do not implement payment changes without a flag, even for "small" or "obvious" changes. Past incidents have shown that small changes in this domain can have outsized impact.
3. **Include a fallback path.** Every external provider call must have a fallback — either a legacy provider, a graceful failure message, or a queue-and-retry pattern. Hand to the user for review if no clear fallback exists.
4. **Test the fallback.** Write at least one test that exercises the fallback path (flag disabled).
5. **Default off in production.** Newly created payment flags must be disabled in production at creation time. Use `toggle_flag_environment` to enable in staging first.

### Rollout overrides

For this domain, override the defaults in `rollout-guidance.md`:

- Start at **1%**, not 10%.
- Hold **48 hours** at each milestone, not 24.
- Add a `25%` milestone between 10% and 50%.
- Halt thresholds are **stricter**: pause if error rate exceeds 0.1% (one tenth of the default 1%).

---

## How to add your own domain

Copy the worked example above and adapt it. The minimum viable policy is four sections:

1. **Scope** — one or more directory globs that define what files this policy applies to.
2. **Default flag type** — pick from the types listed in `feature-flag-conventions.md`.
3. **Naming** — the prefix all flags in this domain must use.
4. **Required actions** — the bullet list of what Kiro must do for changes in this scope.

Optionally add a **Rollout overrides** section if this domain needs different milestones or halt thresholds than the team default.

## Suggested second domain to add

If your team handles user authentication or session data, add an "auth" domain next. Suggested scaffold:

<!-- CUSTOMIZE: uncomment and adapt this block to add an auth domain policy -->

<!--

## Auth

### Scope

- `src/auth/**/*.ts`
- `src/middleware/auth*.ts`
- `src/session/**/*.ts`

### Default flag type

- `release` for new auth methods (SSO, passkeys, MFA enrollment).
- `kill-switch` for identity provider integrations (Auth0, Okta, Cognito).

### Naming

All flags in this domain start with `auth-`. Examples: `auth-sso-google`, `auth-mfa-webauthn`, `auth-session-refresh-v2`.

### Required actions

1. Evaluate first. Auth changes affect every user every session.
2. Require a flag for any change to login, logout, or session validation.
3. Default off in production; require explicit user confirmation before enabling.
4. Roll out internal-first (target by employee email domain).
5. Have a documented rollback plan before enabling beyond 10%.

-->

## Related

- For naming and types: `feature-flag-conventions.md`
- For rollout milestones: `rollout-guidance.md` (this file overrides for high-risk domains)
- For automating evaluation on save: `hooks/evaluate-high-risk-save.kiro.hook`
