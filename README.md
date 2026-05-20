# Unleash · FeatureOps Template for Kiro

A [Kiro Power](https://kiro.dev/docs/powers/) that brings feature flag management into Kiro's spec-driven workflow. It configures the [Unleash MCP server](https://github.com/Unleash/unleash-mcp) and ships a set of steering files and hook templates that encode FeatureOps practices.

The defaults work out of the box, but feature management conventions vary widely across teams and industries. This Power is designed as a **customizable template** — fork it, adapt the steering files to your team's domains and conventions, and roll it out.

> Want the background? Read the [high-level post](https://www.getunleash.io/blog/kiro-unleash-spec-driven-feature-flags) on why specs + feature flags pair well, and the [hands-on deep-dive](https://www.getunleash.io/blog/building-unleash-power-for-kiro) on how this Power was built.

## What's inside

| File | Purpose |
|---|---|
| `POWER.md` | Entry point — Kiro reads this on activation. Frontmatter, onboarding, steering map, tool reference, metadata footer. |
| `mcp.json` | Default MCP server config — local stdio transport via `npx`. |
| `examples/mcp.remote.json` | Alternative MCP config — remote HTTP transport for Unleash-hosted MCP. Swap into `mcp.json` if your org disallows local servers. |
| `examples/README.md` | When and how to swap to the remote transport. |
| `steering/feature-flag-conventions.md` | Universal naming, types, and lifecycle rules. |
| `steering/rollout-guidance.md` | Default rollout milestones, hold durations, halt conditions. |
| `steering/cleanup-cadence.md` | Thresholds and workflow for removing stale flags. |
| `steering/high-risk-domains.md` | Worked example (payments) + template for adding your own domains. |
| `hooks/evaluate-high-risk-save.kiro.hook` | File-save hook that evaluates risky changes. Ships disabled. |
| `hooks/flag-audit.kiro.hook` | Manual-trigger flag audit. Ships disabled. |
| `hooks/README.md` | Install, customize, and credit-cost guidance for the hooks. |
| `LICENSE` | Apache-2.0. |

## Install

### Prerequisites

- **Kiro** (IDE or CLI), with MCP enabled in settings.
- An **Unleash instance** (Cloud or self-hosted) with API access.
- A **Personal Access Token** with permissions to create and manage flags.
- For local transport: **Node.js 18+** on developer machines.
- For remote transport: the Unleash instance admin must enable **Admin settings → Remote MCP server**, and your identity provider must allow OAuth 2.0 Dynamic Client Registration.

### From this repository

1. Open Kiro and go to the **Powers panel**.
2. Click **Add power from GitHub**.
3. Paste this repository URL.
4. Kiro auto-registers the MCP server in `~/.kiro/settings/mcp.json` under a namespaced name.
5. Follow the onboarding prompts (`POWER.md` walks Kiro through transport choice, env vars, and optional hook install).

### From a local checkout (for customization)

```bash
git clone https://github.com/Unleash/unleash-kiro-power.git
# edit the steering files for your team
```

Then in Kiro: **Powers panel → Add power from Local Path → select the cloned directory.**

### Environment variables

Both transports use `${VAR}` expansion. Set:

```bash
export UNLEASH_BASE_URL=https://your-instance.getunleash.io/api
export UNLEASH_PAT=your-personal-access-token
export UNLEASH_DEFAULT_PROJECT=default
```

Kiro requires explicit per-variable approval — click **Allow** on the popup when prompted.

## Customize for your team

This Power ships as a template. Edit these files to match your team's conventions:

1. **`steering/high-risk-domains.md`** — the file you'll edit most. Replace the payments example with your team's actual high-risk areas.
2. **`steering/feature-flag-conventions.md`** — naming convention, flag types in scope, what counts as "high-risk".
3. **`steering/rollout-guidance.md`** — default milestones, hold durations, halt thresholds.
4. **`steering/cleanup-cadence.md`** — stale-flag thresholds.
5. **`hooks/evaluate-high-risk-save.kiro.hook`** — `filePattern` should match your high-risk directories.
6. **`POWER.md` frontmatter** — add domain-specific keywords so the Power activates for your team's vocabulary.

Every customizable spot in the markdown files is marked with an HTML comment: `<!-- CUSTOMIZE: ... -->`. Run `grep -r CUSTOMIZE .` from the repository root to find them all.

## Compatibility

- **Kiro IDE** — primary target. Tested with workspace-scoped MCP configuration.
- **Kiro CLI** — shares the same `.kiro/settings/mcp.json` and steering files as the IDE.
- **Local stdio transport** — works against any Unleash Cloud or self-hosted instance with API access.
- **Remote HTTP transport** — requires the Unleash instance admin to enable remote MCP and your IDP to permit OAuth 2.0 DCR.

The Unleash MCP server is currently labeled experimental in the [Unleash docs](https://docs.getunleash.io/integrate/mcp); the surface API has been stable, but breaking changes between minor versions are possible. Pin a specific `@unleash/mcp` version in `mcp.json` if your team needs determinism.

## License and policies

- **License:** [Apache-2.0](./LICENSE)
- **Privacy Policy:** https://www.getunleash.io/privacy-policy
- **Support:** support@getunleash.io · file an issue at https://github.com/Unleash/unleash-kiro-power/issues
- **Maintainer:** Unleash (Bricks Software AS)

### MCP server attribution

This Power configures the [Unleash MCP server](https://github.com/Unleash/unleash-mcp) ([`@unleash/mcp`](https://www.npmjs.com/package/@unleash/mcp) on npm), Apache-2.0 licensed and maintained by Unleash.

## Related

- [Unleash MCP server](https://github.com/Unleash/unleash-mcp) — the underlying server this Power wraps.
- [Unleash Kiro integration docs](https://docs.getunleash.io/integrate/kiro) — full reference documentation.
- [Kiro Powers documentation](https://kiro.dev/docs/powers/) — how Powers work in Kiro.
- [Kiro community powers repository](https://github.com/kirodotdev/powers) — other community-built powers.
- [FeatureOps](https://featureops.io) — the broader practice this Power supports.
