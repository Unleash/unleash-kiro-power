# Hooks

Kiro hooks fire on IDE events (file save, file create, user trigger, scheduled) and can ask the agent to act on the trigger. This Power ships two hook templates that pair with the Unleash MCP server.

**Both hooks ship with `"enabled": false`.** Hooks fire automatically and consume Kiro credits, so they should never auto-install. Edit the file pattern (or other config) to match your project, then flip `enabled` to `true`.

## What's here

| File | Trigger | What it does |
|---|---|---|
| `evaluate-high-risk-save.kiro.hook` | File save in a high-risk path | Asks Kiro to evaluate the change per `evaluate-change.md`, look for an existing flag, then offer `manage_flag` and guard code if needed |
| `flag-audit.kiro.hook` | Manual (user-triggered from Hooks panel) | Uses `search` (every lifecycle stage, archived included) to enumerate every flag in the project, then evaluates each against the signals in `audit-cadence.md` (staleness, archived, last seen, age-vs-type) |

## Install

Copy the hook files into your workspace `.kiro/hooks/` directory:

```bash
mkdir -p .kiro/hooks
cp hooks/evaluate-high-risk-save.kiro.hook .kiro/hooks/
cp hooks/flag-audit.kiro.hook .kiro/hooks/
```

Reload Kiro (or use **Developer: Reload Window**) and confirm the hooks appear in the Hooks panel. They will be disabled until you customize and enable them.

## Customize before enabling

### `evaluate-high-risk-save.kiro.hook`

| Field | Default | Change to |
|---|---|---|
| `when.filePattern` | `src/payments/**/*.ts` | The glob(s) for *your* high-risk directories. Use multiple patterns by changing the field to an array, e.g. `["src/auth/**", "src/payments/**"]` |
| `then.prompt` | References `evaluate-change.md`, `high-risk-domains.md`, `find-existing-flag.md` and `wrap-change.md` | Add domain-specific guidance if you have multiple high-risk areas with different naming patterns |
| `enabled` | `false` | `true` once the pattern is right |

**Credit cost warning:** File-save hooks fire on *every* save in the matched pattern. For files saved many times a day, this can be expensive. Mitigations:

- Use **narrow** globs. `src/payments/checkout/**/*.ts` is better than `src/**/*.ts`.
- Exclude test files: `["src/payments/**/*.ts", "!src/payments/**/*.test.ts"]` if your team saves tests frequently.
- For very-high-edit-frequency directories, consider replacing this with a **file-create** or **pre-commit** hook instead, which fire less often.

### `flag-audit.kiro.hook`

| Field | Default | Change to |
|---|---|---|
| `then.prompt` thresholds | Last seen 30 days; age-vs-type lifetime (release/experiment 40d, operational 7d, kill-switch/permission permanent) | Match the thresholds in your customized `audit-cadence.md` |
| `then.prompt` project | The project the user names, else asks | Write a fixed project ID into the prompt to always audit the same project |
| `then.prompt` skipped types | `kill-switch`, `permission` | Add or remove types your team treats as permanent |
| `enabled` | `false` | `true` once the prompt matches your conventions |

No credit-cost concern — this hook only fires when the user clicks it. The hook makes one `search` call per page of flags plus an `inspect` call per non-permanent flag.

## Removing hooks

If you decide a hook isn't worth the credit cost, either:

- Flip `enabled` to `false` (preserves the hook for later).
- Delete the file from `.kiro/hooks/`.

## Related

- For the conventions the hooks reference: `../steering/evaluate-change.md`, `../steering/find-existing-flag.md`, `../steering/wrap-change.md`, `../steering/high-risk-domains.md`, `../steering/audit-cadence.md`
- For Kiro's hook documentation: https://kiro.dev/docs/hooks/
- For Kiro's hook trigger types: https://kiro.dev/docs/hooks/types
