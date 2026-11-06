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
- **Naming convention and when to flag** in `steering/feature-flag-conventions.md` — the default name is `{domain}-{feature}` (kill switches `disable-{domain}-{feature}`), and a tier table decides which changes need a flag; swap in your team's pattern and bar.
- **Rollout milestones** in `steering/rollout-guidance.md` — default is 10% → 50% → 100% with 24-hour holds; adjust for your release cadence.
- **Audit cadence** in `steering/audit-cadence.md` — default no-activity window is 30 days, and lifetimes follow Unleash's per-type defaults; adjust for your tolerance.
- **Hook file patterns** in `hooks/*.kiro.hook` — defaults target `src/payments/**/*.ts`; change to match your highest-risk directories.

Every customizable spot in the steering files is marked with an HTML comment: `<!-- CUSTOMIZE: ... -->`. Search for `CUSTOMIZE` to find them all.

# Onboarding

Run these steps in order the first time this Power activates. If the user has completed a step already, skip it.

## Step 0: Choose your transport

The Unleash MCP server is available in two modes. Ask the user which one they want, then continue with the matching path:

| Mode | When to use | Setup cost |
|---|---|---|
| **Local stdio** (default) | Individual developer with Node.js 24+ and permission to run `npx`. Works against any Unleash instance. | Set 2 env vars, no admin action needed. |
| **Remote HTTP** | Org disallows local MCP processes, or you prefer Unleash-hosted. Requires admin to enable remote MCP on the instance and the org to permit OAuth 2.0 DCR. | One env var, but admin must toggle "Remote MCP server" in Admin settings first. |

If the user picks **remote**, copy the contents of `examples/mcp.remote.json` over `mcp.json` and replace `https://your-instance.getunleash.io` with their instance URL. Verify reachability with `curl -I https://<instance>/api/admin/mcp` — a `401 Unauthorized` is expected (proves the endpoint exists; auth happens in the next step).

If the user picks **local** (default), keep `mcp.json` as-is and verify Node with `node --version` (needs 24+).

The bundled `mcp.json` runs `@unleash/mcp@latest`. This Power's steering files and hooks call the tools of the Unleash MCP server v2 (0.6.0 and later).

## Step 1: Verify prerequisites

| Requirement | Mode | Verify with |
|---|---|---|
| Unleash instance (Cloud or self-hosted) | both | `curl -I ${UNLEASH_BASE_URL}` returns 200 or 401 |
| Personal Access Token with flag-management permissions | both | Manage in Unleash Admin → API access → Personal access tokens |
| Node.js 24+ | local only | `node --version` |
| Remote MCP enabled on the Unleash instance | remote only | Unleash Admin → Admin settings → Remote MCP server is **Enabled** |
| OAuth 2.0 Dynamic Client Registration allowed | remote only | Confirm with your org's identity provider |

## Step 2: Set environment variables

Both modes use `${VAR}` expansion. Create a credentials file and source it from the shell that launches Kiro:

```bash
mkdir -p ~/.unleash
cat > ~/.unleash/mcp.env << 'EOF'
UNLEASH_BASE_URL=https://your-instance.getunleash.io
UNLEASH_PAT=your-personal-access-token
EOF
chmod 600 ~/.unleash/mcp.env
```

Add to `~/.zshrc` or `~/.bashrc`:

```bash
if [ -f ~/.unleash/mcp.env ]; then
  source ~/.unleash/mcp.env
  export UNLEASH_BASE_URL UNLEASH_PAT
fi
```

Restart the shell, then restart Kiro. When Kiro shows the "unapproved environment variables" popup, click **Allow** for `UNLEASH_BASE_URL` and `UNLEASH_PAT`. Kiro only expands variables that have been explicitly approved.

**Remote mode only needs `UNLEASH_PAT`** (the URL is in `mcp.json` directly), but setting both is harmless and lets the user switch transports later without re-doing the env work.

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
| Naming a new flag, picking its type, the team's bar for when a change needs a flag | `steering/feature-flag-conventions.md` |
| Deciding whether a change needs a flag, before implementing it | `steering/evaluate-change.md` |
| Before creating any flag: looking for one that already covers the feature | `steering/find-existing-flag.md` |
| Putting code behind a flag | `steering/wrap-change.md` |
| Planning a gradual rollout, configuring percentages, setting halt conditions | `steering/rollout-guidance.md` |
| Taking a flag out of the code | `steering/remove-flag.md` |
| Finding flags due for removal, auditing stale flags | `steering/audit-cadence.md` |
| Working in a high-risk directory (payments, auth, data migrations, external integrations) | `steering/high-risk-domains.md` |
| Locating flag checks in code, or telling what a check covers | `steering/flag-usage-patterns.md` |

For tasks that span multiple workflows (e.g., "create a flag and plan the rollout"), load each relevant file in turn.

# Tool reference

The Unleash MCP server exposes the following tools.

| Tool | Description | When to use |
|---|---|---|
| `search` | Finds projects, environments, flags, segments, context fields, tags, release templates and event types. Filters flags by tag, lifecycle stage, staleness, creator or creation date. | Discovering projects; finding an existing flag before creating one; auditing flag inventory |
| `inspect` | Shows a project, flag, context field or segment in full: who gets a flag in each environment and when SDKs last saw it there. | Debugging, status checks, before changing a rollout |
| `evaluate` | Tells whether a user or context gets a flag in an environment, with which value, and what blocks it. | Answering "does user X get flag Y?" |
| `audit` | Reads what changed in Unleash, who changed it and when, over at most 31 days. | Incident review, tracing a rollout change |
| `docs` | Answers feature flag and Unleash questions from the current Unleash docs. | Practice and SDK questions |
| `manage_flag` | Creates a flag (off in every environment) or changes its type, description, tags and links; marks it completed or stale; archives or revives it. | Creating flags; recording the end of a rollout; archiving |
| `release` | Turns a flag on or off in an environment, for everyone, a percentage, audiences, segments, or a release template. Creates a missing flag. | Gradual releases, per-environment activation, pausing |
| `kill_switch` | Adds a permanent kill switch or flips one in an environment. | Emergency shutdown of an integration or high-risk path |
| `experiment` | Runs or stops an A/B test across values such as control and treatment. | Experiments |
| `configure` | Creates or changes projects, release templates, context fields, segments and tag types. | Shared setup that many flags use |
| `send_feedback` | Reports to Unleash where the MCP server fell short. Asks the user for consent itself. | When a tool fails or can't do what the user asked |

The read-only tools — `search`, `inspect`, `evaluate`, `audit`, and `docs` — are pre-approved (`autoApprove`) in `mcp.json` so the agent can use them without prompting. Write tools — `manage_flag`, `release`, `kill_switch`, `experiment`, and `configure` — always require explicit user approval. Keep it that way unless the user has a specific reason to lower the bar.

The server has no tool to judge a code change's risk, write SDK guard code, or find a flag's references in code. The agent does these itself, following `steering/evaluate-change.md`, `steering/wrap-change.md` and `steering/remove-flag.md`. Note that `evaluate` checks who gets a flag; it does not judge whether a change needs one.

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
