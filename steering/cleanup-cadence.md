# Cleanup cadence

## Customize this file for your team

- **Stale-flag threshold:** default is 14 days at 100% rollout. Some teams use 7 days for fast-moving services or 30 days for regulated environments.
- **No-activity threshold:** default is 90 days untouched. Tune for your team's release cadence.
- **Cleanup ownership:** default assumes the engineer who created the flag also removes it. If your team has a centralized FeatureOps function, adjust the language accordingly.
- **Permanent-flag exceptions:** default assumes `operational`, `kill-switch`, and `permission` flags are long-lived. List any other types your team treats as permanent.

---

# Cleanup cadence

Feature flags are temporary by design. Stale flags accumulate technical debt, confuse future readers, and increase the risk that an unrelated change accidentally re-enables old code. This file is loaded when the conversation involves removing flags or auditing for cleanup opportunities.

## When a flag is ready for cleanup

<!-- CUSTOMIZE: adjust these thresholds for your team's tolerance -->

A flag is a cleanup candidate when **any** of these conditions hold:

- It has been at **100% rollout in production for 14 or more days** with no regressions.
- It has not been **modified or referenced in code for 90 or more days**.
- The feature it gated has been **fully launched** and the legacy code path is no longer needed.
- The experiment it controlled has **concluded** and a winning variant has been picked.

Permanent flags are exempt. By default, `operational`, `kill-switch`, and `permission` flag types are treated as long-lived — review them annually for relevance but do not auto-flag them for removal.

## Removal workflow

When the user asks to clean up a flag, follow this sequence:

### 1. Get the full picture

Call `cleanup_flag` with the flag name. The tool returns:

- Every file and line number where the flag is referenced.
- The current rollout state across all environments.
- The "enabled" branch — the code path that becomes the default after removal.
- A short cleanup checklist for the user's language/framework.

### 2. Confirm the keep path

Before editing anything, confirm with the user which branch should be preserved:

- For a `release` flag at 100%: keep the enabled branch, remove the conditional.
- For an `experiment` flag: keep the winning variant, remove all others and the conditional.
- For a `kill-switch` flag that's being decommissioned: keep the "enabled" path (the new behavior).

Ask the user explicitly if there is any ambiguity. Do not guess.

### 3. Remove the conditional code

For each location returned by `cleanup_flag`:

- Delete the `if (unleash.isEnabled(...))` check.
- Promote the "kept" branch to the surrounding scope.
- Delete the fallback/legacy branch entirely.
- Remove any now-unused imports (`unleash-client`, helper functions, mock paths in tests).

Do not leave the conditional in place with a TODO. The point of cleanup is to eliminate the branching, not document it.

### 4. Run the relevant tests

Run the test suite for the touched code paths. Pay special attention to tests that may have been written against the legacy branch — they should be deleted, not patched to pass.

### 5. Delete the flag from Unleash

Only after the code change is merged, delete the flag from Unleash. If the flag is deleted from Unleash while production code still references it, every call to `isEnabled()` returns `false` — silently switching production to the fallback path. Sequence matters.

### 6. Document the cleanup

If your team tracks technical-debt removal, mention the flag name and merge SHA in the PR description so the rollout history is searchable later.

## Periodic audits

Run a flag audit on a recurring cadence (the `flag-audit.kiro.hook` template in `hooks/` automates this). The audit should:

- List all flags in the default project (or whatever project the user specifies).
- Check rollout state for each via `get_flag_state`.
- Flag any that exceed the thresholds above.
- Prioritize by age × rollout percentage so the longest-stale fully-rolled-out flags surface first.

A weekly audit is reasonable for most teams. Run it more frequently during release-heavy periods.

## What not to clean up automatically

Never auto-execute `cleanup_flag` followed by code removal without the user reviewing each step. Flag cleanup touches production code paths and can re-introduce bugs if the wrong branch is preserved. Always present the plan, get confirmation, then act.

## Related

- For naming and creation: `feature-flag-conventions.md`
- For rollout state checks: `rollout-guidance.md`
- For automating the audit on demand: `hooks/flag-audit.kiro.hook`
