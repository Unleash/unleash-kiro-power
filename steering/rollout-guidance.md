# Rollout guidance

## Customize this file for your team

- **Starting percentage:** default is 10%. Lower for very high-risk changes; higher for low-risk internal features.
- **Hold duration between milestones:** default is 24 hours. Shorten for fast-moving teams; lengthen for regulated industries.
- **Halt thresholds:** defaults are >1% error rate and >20% p95 latency increase. Tune to your SLOs.
- **Milestone count and percentages:** default is 10% → 50% → 100% (three steps). Some teams prefer 1% → 10% → 25% → 50% → 100% (five steps) for higher-traffic services.
- **Environments included:** default assumes `development`, `staging`, `production`. Add `canary`, `preview`, or region-specific environments if you use them.

---

# Rollout guidance

This file is loaded when the conversation involves rolling out feature flags. Use it to configure `release` calls.

## Default rollout strategy

<!-- CUSTOMIZE: replace these milestones with your team's standard rollout pattern -->

For new release flags, use this three-step progression unless the user specifies otherwise:

| Milestone | Percentage | Hold duration | Advance when |
|---|---|---|---|
| **Canary** | 10% | 24 hours | Error rate < 1%, p95 latency stable |
| **Beta** | 50% | 48 hours | Error rate < 0.5%, no new regression reports |
| **GA** | 100% | — | All metrics healthy for 48 consecutive hours |

For high-risk flags (payments, auth, data migrations), use a more conservative ramp: 1% → 10% → 25% → 50% → 100% with 48-hour holds at each step.

## Halt conditions

<!-- CUSTOMIZE: tune these thresholds to match your SLOs and alert thresholds -->

Pause the rollout immediately and re-evaluate if any of the following are observed during a hold period:

- **Error rate** exceeds 1% (or 0.5x baseline, whichever is higher).
- **p95 latency** increases by more than 20% versus the pre-rollout baseline.
- **User reports** of broken functionality through any inbound channel.
- **Downstream system errors** spike (database, message queue, third-party API).

To pause a rollout, call `release` with `flag.state: disabled` for the affected environment. `release` replaces the environment's audience, so pass the current targets from `inspect` to keep them for a restart. Do not archive the flag — pausing preserves the rollout history for the postmortem.

## Per-environment activation

<!-- CUSTOMIZE: list the environments your team uses; rename or add as needed -->

Use `release` to enable a flag in one environment at a time:

1. **`development`** — Enable as soon as the flag is created. Default-on for local testing.
2. **`staging`** — Enable after integration tests pass against the new code path.
3. **`production`** — Enable only after staging has run the new code path for 24 hours with no regressions.

Never enable a flag in production before staging unless the user explicitly confirms (e.g., for a kill-switch you're pre-positioning).

## Targeting strategies

`release` supports targeting beyond simple percentage rollouts. Each call takes a list of targets, each with an optional `rollout` percentage, constraints, segments and `stickiness`. Common patterns:

- **Internal-first:** Target by user email domain (e.g., `@your-company.com`) or by a `userType=employee` context attribute. Use this before any external exposure.
- **Beta users:** Target a named segment (create one with `configure`). Useful when you have a curated list of early-access customers.
- **Geographic:** Target by region for regulatory or capacity reasons. Useful for staged regional rollouts.
- **Sticky percentage:** Hash on `userId` so the same user sees consistent behavior across requests.

When configuring targeting, always include a sticky identifier so the user experience is consistent across sessions.

## Tools to use

- `release` — enable or disable a flag in an environment and set its percentages and targeting. Each call replaces the environment's targets; to drop unused ones, send the trimmed list.
- `inspect` — check the current rollout state before changing it.
- `kill_switch` — flip a `kill-switch` flag on or off.
- `experiment` — run an A/B test instead of a plain rollout.
- `configure` — create segments and release templates that rollouts share.
- `audit` — see who changed a rollout and when.

## Related

- For naming: `feature-flag-conventions.md`
- For creating the flag: `find-existing-flag.md`
- For removing the flag after rollout reaches 100%: `remove-flag.md`
- For domain-specific overrides: `high-risk-domains.md`
