# Cleanup cadence

## Customize this file for your team

- **Stale signal:** default trusts Unleash's built-in `stale` marker plus a "no SDK has evaluated this flag in N days" rule (default N = 30). Tune N for your team's traffic patterns — low-traffic services may legitimately go longer between evaluations.
- **Age-vs-type thresholds:** defaults follow Unleash's product defaults (release 40d, experiment 40d, operational 7d). If your team customized these in the Unleash admin UI, update the numbers here to match.
- **Permanent-flag exceptions:** Unleash treats `kill-switch` and `permission` as permanent (no expected lifetime). If your team treats other types as permanent, list them here.
- **Cleanup ownership:** default assumes the engineer who created the flag also removes it. If your team has a centralized FeatureOps function, adjust accordingly.

---

# Cleanup cadence

Most feature flags are temporary by design. Stale flags accumulate technical debt, confuse future readers, and increase the risk that an unrelated change accidentally re-enables old code. This file is loaded when the conversation involves removing flags or auditing for cleanup opportunities.

## When a flag is ready for cleanup

<!-- CUSTOMIZE: adjust the no-activity threshold (last-seen window) for your team's traffic patterns -->

A flag is a cleanup candidate when **any** of these conditions hold (signals listed in order of confidence):

1. **Unleash marked it `stale` or `potentially_stale`.** This is the strongest signal — a person marked it overdue, or Unleash's own staleness logic did based on type and age.
2. **No SDK has evaluated the flag in 30+ days** in any enabled environment (`inspect` shows when SDKs last saw it in each one). If clients aren't checking it, removing it from code won't change behavior.
3. **Age exceeds the type's expected lifetime** (see the table below). A `release` flag older than 40 days, or an `operational` flag older than 7 days, is overdue by Unleash's defaults.
4. **The feature has fully launched** and the legacy code path is no longer needed (user judgment, not metadata).
5. **The experiment has concluded** and a winning variant has been picked.

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

## Removal workflow

When the user asks to clean up a flag, follow this sequence:

### 1. Get the full picture

Call `inspect` (`kind: flag`) for the flag's type, lifecycle stage and rollout state in every environment. Then search the codebase yourself. Build a list of:

- Every file and line where the flag is referenced — including indirect usages through wrappers, hooks and config (see the patterns in step 3).
- The "enabled" branch — the code path that becomes the default after removal.

Real codebases rarely have raw `unleash.isEnabled('flag-name')` calls everywhere. Find how this codebase reads flags first, then trace every caller of that wrapper.

### 2. Confirm the keep path

Before editing anything, confirm with the user which branch should be preserved:

- For a `release` flag at 100%: keep the enabled branch, remove the conditional.
- For an `experiment` flag: keep the winning variant, remove all others and the conditional.
- For a `kill-switch` flag that's being decommissioned: keep the "enabled" path (the new behavior).

Ask the user explicitly if there is any ambiguity. Do not guess.

### 3. Remove the flag's surface area

The shape of "remove the conditional" depends entirely on how the flag is used. Real codebases rarely have raw `unleash.isEnabled('name')` calls scattered through business logic — that's the textbook example, not the common case. Be ready for any of these patterns:

- **Custom wrapper functions:** `featureFlags.isEnabled('name')`, `flags.get('name')`, or project-specific helpers. Removing the flag means removing the call sites *and* potentially the wrapper if this was its only caller.
- **Dependency-injected flag service:** `this.flagService.isEnabled('name')` or constructor-injected `featureFlags`. Check whether the injection can be simplified after removal (e.g., a service that no longer needs the flags dependency).
- **React/Vue hooks and HOCs:** `useFlag('name')`, `useFeatureFlag('name')`, `withFeatureFlag('name')`. Removing the hook usage also means unwrapping the conditional rendering and possibly the HOC.
- **Decorators or annotations:** `@FeatureFlag('name')`, `[FeatureGate("name")]` in some Java/.NET stacks. The decorator itself disappears; whatever it guarded becomes unconditional.
- **Config-driven gating:** flag values mapped into a config object at startup (`config.features.newCheckout`). Remove the config entry, then trace every reader.
- **Server-side rendering / edge:** flag context passed into render functions, middleware, or CDN edge logic. Cleanup may span multiple deployment artifacts.
- **A/B routing:** experiment flags that route traffic to different controllers or templates. Cleanup means deleting the losing branch's code, not just the routing logic.
- **Test fixtures and mocks:** mock providers, fixture flag-state objects, `jest.mock('unleash-client')` setups. These often outlive the production code.
- **Analytics and logging:** flag name as a tag, dimension, or log field. Search for the literal flag-name string, not just the SDK call.

If you find a pattern you can't remove cleanly, show it to the user — don't silently work around it.

After removing all references:

- Promote the "kept" branch to the surrounding scope.
- Delete the fallback/legacy branch entirely.
- Remove now-unused imports, wrappers, hooks, decorators, mocks, and config entries.
- Search the codebase for the literal flag name string (including dashes/underscores variants) to catch references in non-code locations like analytics events.

Do not leave the conditional in place with a TODO. The point of cleanup is to eliminate the branching, not document it.

### 4. Run the relevant tests

Run the test suite for the touched code paths. Pay special attention to:

- Tests that exercised the fallback/losing branch — they should be deleted, not patched to pass.
- Mock providers that listed the removed flag — these need updating.
- Snapshot tests that captured the old conditional rendering — re-snapshot intentionally.

### 5. Complete and archive the flag in Unleash

When the code change is ready, call `manage_flag` with `action: complete` and the outcome (`kept`, with the kept value for an experiment, or `discarded`). This records the result and moves the flag to the cleanup stage; SDKs are not affected.

Only after the code change is merged and deployed to production, archive the flag with `manage_flag` (`action: archive`). If the flag is archived while production code still references it, every call to `isEnabled()` returns `false` — silently switching production to the fallback path. Sequence matters.

The MCP server can't delete flags. Archiving keeps the flag's history, including the incident log of `kill-switch` flags.

### 6. Document the cleanup

If your team tracks technical-debt removal, mention the flag name and merge SHA in the PR description so the rollout history is searchable later.

## Periodic audits

Run a flag audit on a recurring cadence (the `flag-audit.kiro.hook` template in `hooks/` automates this):

1. **Enumerate flags** with `search` (`kind: flags`, the project id). Archived flags only come back when `stages` lists `archived`, so pass every stage: `define`, `develop`, `production`, `cleanup`, `archived`. Page with `offset` and `limit` until you reach `total`.
2. **Fetch each flag's details** via `inspect` (`kind: flag`) for what the list rows don't include, such as when SDKs last saw it in each environment.
3. **Skip** flags of type `kill-switch` or `permission` — these are permanent by design.
4. **Surface** flags where any of these are true:
   - Staleness is `stale` or `potentially_stale` (a `search` with the `staleness` filter finds these directly)
   - Lifecycle stage is `archived`, but code references may still exist (search the codebase)
   - SDKs last saw it more than 30 days ago in every enabled environment (no SDK is hitting it)
   - Its age exceeds the expected lifetime for its type (overdue by Unleash's defaults)
5. **Prioritize** stale flags first, then archived, then by oldest last-seen time, then by how far past its expected lifetime.

A weekly audit is reasonable for most teams. Run it more frequently during release-heavy periods.

## What not to clean up automatically

Never remove flag code, or complete or archive a flag, without the user reviewing each step. Flag cleanup touches production code paths and can re-introduce bugs if the wrong branch is preserved. Always present the plan, get confirmation, then act.

## Related

- For naming and creation: `feature-flag-conventions.md`
- For rollout state checks: `rollout-guidance.md`
- For automating the audit on demand: `hooks/flag-audit.kiro.hook`
