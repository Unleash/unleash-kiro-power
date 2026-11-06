# Audit cadence

## Customize this file for your team

- **Stale signal:** default trusts Unleash's built-in `stale` marker plus a "no SDK has evaluated this flag in N days" rule (default N = 30). Tune N for your team's traffic patterns — low-traffic services may legitimately go longer between evaluations.
- **Age-vs-type thresholds:** defaults follow Unleash's product defaults (release 40d, experiment 40d, operational 7d). If your team customized these in the Unleash admin UI, update the numbers here to match.
- **Permanent-flag exceptions:** Unleash treats `kill-switch` and `permission` as permanent (no expected lifetime). If your team treats other types as permanent, list them here.
- **Cleanup ownership:** default assumes the engineer who created the flag also removes it. If your team has a centralized FeatureOps function, adjust accordingly.

---

# Audit cadence

Most feature flags are temporary by design. Stale flags accumulate technical debt, confuse future readers, and increase the risk that an unrelated change accidentally re-enables old code. This file is loaded when the conversation involves auditing for flags due for removal. Taking a flag out of the code is in `remove-flag.md`.

## When a flag is due for removal

<!-- CUSTOMIZE: adjust the no-activity threshold (last-seen window) for your team's traffic patterns -->

A flag is due for removal when **any** of these conditions hold (signals listed in order of confidence):

1. **Unleash marked it `stale` or `potentially_stale`.** This is the strongest signal — a person marked it overdue, or Unleash's own staleness logic did based on type and age.
2. **It was completed:** someone recorded the feature as kept or discarded with `manage_flag` (`action: complete`), which moves it to the `cleanup` lifecycle stage.
3. **No SDK has evaluated the flag in 30+ days** in every environment where it is enabled (`inspect` shows when SDKs last saw it in each one). If clients aren't checking it, removing it from code won't change behavior.
4. **Age exceeds the type's expected lifetime** (see the table below). A `release` flag older than 40 days, or an `operational` flag older than 7 days, is overdue by Unleash's defaults.
5. **It serves everyone in production**, so the off path never runs there. Confirm with the user that the old path is no longer needed.
6. **The experiment has concluded** and a winning variant has been picked (user judgment, not metadata).

### Type-based expected lifetimes

<!-- CUSTOMIZE: replace with your team's customized lifetimes if you changed them in the Unleash admin UI -->

| Type | Default lifetime | Cleanup behavior |
|---|---|---|
| `operational` | 7 days | Audit aggressively — these are meant to be short-term |
| `release` | 40 days | Audit for removal after the rollout stabilizes |
| `experiment` | 40 days | Audit for removal after the experiment concludes |
| `kill-switch` | Permanent | **Skip** — no expected lifetime, meant to live indefinitely |
| `permission` | Permanent | **Skip** — ties to user attributes, not a rollout |

Only `kill-switch` and `permission` are treated as permanent by default. If your team uses operational flags long-term, change the type to `kill-switch` or update the lifetime in the Unleash admin UI.

## Removing a flag

To take a flag out of the code once it is due, follow `remove-flag.md`.

## Periodic audits

Run a flag audit on a recurring cadence (the `flag-audit.kiro.hook` template in `hooks/` automates this):

1. **Enumerate flags** with `search` (`kind: flags`, the project id). Archived flags only come back when `stages` lists `archived`, so pass every stage: `define`, `develop`, `production`, `cleanup`, `archived`. Page with `offset` and `limit` until you reach `total`.
2. **Fetch each flag's details** via `inspect` (`kind: flag`) for what the list rows don't include, such as when SDKs last saw it in each environment.
3. **Skip** flags of type `kill-switch` or `permission` — these are permanent by design.
4. **Surface** flags where any of these are true:
   - Staleness is `stale` or `potentially_stale` (a `search` with the `staleness` filter finds these directly)
   - Lifecycle stage is `cleanup` (the flag was completed)
   - Lifecycle stage is `archived`, but code references may still exist (search the codebase)
   - SDKs last saw it more than 30 days ago in every enabled environment (no SDK is hitting it)
   - Its age exceeds the expected lifetime for its type (overdue by Unleash's defaults)
5. **Prioritize** stale flags first, then archived, then completed, then by oldest last-seen time, then by how far past its expected lifetime.

A weekly audit is reasonable for most teams. Run it more frequently during release-heavy periods.

## What not to clean up automatically

Never remove flag code, or complete or archive a flag, without the user reviewing each step. Flag cleanup touches production code paths and can re-introduce bugs if the wrong branch is preserved. Always present the plan, get confirmation, then act.

## Related

- For taking a flag out of the code: `remove-flag.md`
- For naming and creation: `feature-flag-conventions.md`
- For rollout state checks: `rollout-guidance.md`
- For automating the audit on demand: `hooks/flag-audit.kiro.hook`
