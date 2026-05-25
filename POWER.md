---
name: unleash
displayName: Unleash · FeatureOps Template
description: Automate feature flag management with Unleash from inside Kiro. Evaluate code changes for risk, create flags with consistent naming, generate SDK wrapping code, manage rollouts, and clean up stale flags. Ships as a customizable FeatureOps template — adapt the steering files to your team's conventions.
author: Unleash
keywords:
  - feature flag
  - feature toggle
  - feature management
  - rollout
  - release
  - experiment
  - kill-switch
  - canary
  - progressive delivery
  - featureops
  - unleash
  - flag audit
  - flag inventory
  # CUSTOMIZE: add domain-specific terms your team uses (e.g. "checkout flag", "billing rollout")
---

## Customize this Power for your team

This Power ships as a template. The defaults work out of the box, but feature management conventions vary widely across teams and industries. Edit the following before rolling it out broadly:

- **High-risk domains** in `steering/high-risk-domains.md` — replace the payments example with your own domains (auth, billing, data migrations, public API, etc.).
- **Naming convention** in `steering/feature-flag-conventions.md` — the default is `{domain}-{feature}-{variant}`; swap for whatever pattern your team uses.
- **Rollout milestones** in `steering/rollout-guidance.md` — default is 10% → 50% → 100% with 24-hour holds; adjust for your release cadence.
- **Cleanup cadence** in `steering/cleanup-cadence.md` — default flags-this-stale threshold is 14 days at 100%; adjust for your tolerance.
- **Hook file patterns** in `hooks/*.kiro.hook` — defaults target `src/payments/**/*.ts`; change to match your highest-risk directories.

Every customizable spot in the steering files is marked with an HTML comment: `<!-- CUSTOMIZE: ... -->`. Search for `CUSTOMIZE` to find them all.

# Onboarding

Run these steps in order the first time this Power activates. If the user has completed a step already, skip it.

## Step 0: Choose your transport

The Unleash MCP server is available in two modes. Ask the user which one they want, then continue with the matching path:

| Mode | When to use | Setup cost |
|---|---|---|
| **Local stdio** (default) | Individual developer with Node.js 18+ and permission to run `npx`. Works against any Unleash instance. | Set 3 env vars, no admin action needed. |
| **Remote HTTP** | Org disallows local MCP processes, or you prefer Unleash-hosted. Requires admin to enable remote MCP on the instance and the org to permit OAuth 2.0 DCR. | One env var, but admin must toggle "Remote MCP server" in Admin settings first. |

If the user picks **remote**, copy the contents of `examples/mcp.remote.json` over `mcp.json` and replace `https://your-instance.getunleash.io` with their instance URL. Verify reachability with `curl -I https://<instance>/api/admin/mcp` — a `401 Unauthorized` is expected (proves the endpoint exists; auth happens in the next step).

If the user picks **local** (default), keep `mcp.json` as-is and verify Node with `node --version` (needs 18+).

## Step 1: Verify prerequisites

| Requirement | Mode | Verify with |
|---|---|---|
| Unleash instance (Cloud or self-hosted) | both | `curl -I ${UNLEASH_BASE_URL}` returns 200 or 401 |
| Personal Access Token with flag-management permissions | both | Manage in Unleash Admin → API access → Personal access tokens |
| Node.js 18+ | local only | `node --version` |
| Remote MCP enabled on the Unleash instance | remote only | Unleash Admin → Admin settings → Remote MCP server is **Enabled** |
| OAuth 2.0 Dynamic Client Registration allowed | remote only | Confirm with your org's identity provider |

## Step 2: Set environment variables

Both modes use `${VAR}` expansion. Create a credentials file and source it from the shell that launches Kiro:

```bash
mkdir -p ~/.unleash
cat > ~/.unleash/mcp.env << 'EOF'
UNLEASH_BASE_URL=https://your-instance.getunleash.io/api
UNLEASH_PAT=your-personal-access-token
UNLEASH_DEFAULT_PROJECT=default
EOF
chmod 600 ~/.unleash/mcp.env
```

Add to `~/.zshrc` or `~/.bashrc`:

```bash
if [ -f ~/.unleash/mcp.env ]; then
  source ~/.unleash/mcp.env
  export UNLEASH_BASE_URL UNLEASH_PAT UNLEASH_DEFAULT_PROJECT
fi
```

Restart the shell, then restart Kiro. When Kiro shows the "unapproved environment variables" popup, click **Allow** for `UNLEASH_BASE_URL`, `UNLEASH_PAT`, and `UNLEASH_DEFAULT_PROJECT`. Kiro only expands variables that have been explicitly approved.

**Remote mode only needs `UNLEASH_PAT`** (the URL is in `mcp.json` directly), but setting all three is harmless and lets the user switch transports later without re-doing the env work.

## Step 3: Install hook templates (optional)

This Power ships two hook templates in `hooks/`. They are **not installed automatically** — hooks fire on real IDE events and consume Kiro credits, so they should be opt-in and customized to the user's project.

Ask the user: *"Want me to install the file-save and flag-audit hook templates into `.kiro/hooks/`? You should edit the file pattern in the file-save hook before enabling it."*

If yes, copy both files into `.kiro/hooks/`:

```bash
mkdir -p .kiro/hooks
cp hooks/evaluate-high-risk-save.kiro.hook .kiro/hooks/
cp hooks/flag-audit.kiro.hook .kiro/hooks/
```

Then open `hooks/README.md` for guidance on which fields to customize before enabling.

# When to load steering files

Kiro should load steering files contextually based on the task at hand. Map workflows to files:

| User intent | Load |
|---|---|
| Setting up feature flags, deciding when to flag a change, naming a new flag | `steering/feature-flag-conventions.md` |
| Working in a high-risk directory (payments, auth, data migrations, external integrations) | `steering/high-risk-domains.md` |
| Planning a gradual rollout, configuring percentages, setting halt conditions | `steering/rollout-guidance.md` |
| Removing a feature flag, cleaning up after a successful rollout, auditing stale flags | `steering/cleanup-cadence.md` |

For tasks that span multiple workflows (e.g., "create a flag and plan the rollout"), load both relevant files.

# Tool reference

The Unleash MCP server exposes the following tools.

| Tool | Description | When to use |
|---|---|---|
| `evaluate_change` | Analyzes a code change and determines whether it should be behind a feature flag. | Before implementing risky changes |
| `detect_flag` | Searches for existing flags that match a description to prevent duplicates. | Before creating new flags |
| `create_flag` | Creates a new feature flag with proper naming, typing, and metadata. | When no suitable flag exists |
| `wrap_change` | Generates framework-specific code to guard a feature behind a flag. | After creating a flag |
| `list_projects` | Lists Unleash projects available to the configured token, with optional pagination. | Discovering available projects |
| `list_flags` | Lists feature flags in a project (active by default; set archived=true for archived flags). | Auditing flag inventory; discovering existing flags before creating new ones |
| `get_flag_state` | Returns the current state, strategies, and metadata for a flag. | Debugging, status checks |
| `set_flag_rollout` | Configures rollout percentages and activation strategies. | Gradual releases |
| `toggle_flag_environment` | Enables or disables a flag in a specific environment. | Testing, staged rollouts |
| `remove_flag_strategy` | Deletes a rollout strategy from a flag. | Simplifying flag configuration |
| `cleanup_flag` | Returns file locations and instructions for removing a flag after rollout. | After full rollout |

The read-only tools — `get_flag_state`, `detect_flag`, `evaluate_change`, `list_projects`, and `list_flags` — are pre-approved (`autoApprove`) in `mcp.json` so the agent can use them without prompting. Write operations like `create_flag`, `set_flag_rollout`, and `cleanup_flag` always require explicit user approval — keep it that way unless the user has a specific reason to lower the bar.

---

## Power metadata

- **License:** Apache-2.0
- **Privacy Policy:** https://www.getunleash.io/privacy-policy
- **Support:** support@getunleash.io · GitHub Issues at https://github.com/Unleash/unleash-kiro-power/issues
- **Source:** https://github.com/Unleash/unleash-kiro-power

### MCP server attribution

This Power configures the **Unleash MCP server**:

- Package: [`@unleash/mcp`](https://www.npmjs.com/package/@unleash/mcp) on npm
- Source: https://github.com/Unleash/unleash-mcp
- License: Apache-2.0
- Privacy: https://www.getunleash.io/privacy-policy
- Maintainer: Unleash (Bricks Software AS)
- Transport: local stdio via `npx`, or remote Streamable HTTP at `https://<your-instance>/api/admin/mcp`
