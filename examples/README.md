# Examples

Alternative configurations that you can swap in for the defaults shipped at the top of this Power.

## `mcp.remote.json` — Remote HTTP transport

Use this configuration when:

- Your org disallows running local MCP server processes (no `npx`, no Node.js on developer machines).
- You prefer a centrally hosted MCP server you control through Unleash admin settings.
- Your Unleash instance admin has enabled the remote MCP endpoint and your identity provider permits OAuth 2.0 Dynamic Client Registration.

### How to swap

```bash
cp examples/mcp.remote.json mcp.json
```

Then open `mcp.json` and replace `https://your-instance.getunleash.io` with your actual Unleash host.

### Required setup on the Unleash side

1. Sign in to Unleash as an instance admin.
2. Go to **Admin settings → Remote MCP server**.
3. Toggle **Enable Remote MCP Server for this instance** to **Enabled**.
4. Confirm with your identity provider admin that OAuth 2.0 Dynamic Client Registration (DCR) is allowed for the Unleash app.

### Auth flow

The remote endpoint authenticates via `Authorization: Bearer ${UNLEASH_PAT}`. Personal access tokens issued through the browser-based exchange flow are valid for 24 hours; long-lived PATs created through the Unleash admin UI do not expire until you rotate them.

### Trade-offs

| | Local stdio (default) | Remote HTTP |
|---|---|---|
| Node.js 18+ required | ✅ | — |
| Admin must enable a server-side toggle | — | ✅ |
| OAuth DCR required | — | ✅ |
| Works against any Unleash instance | ✅ | Only instances with remote MCP enabled |
| Centralized server (no per-laptop process) | — | ✅ |

If you're unsure, start with the default local mode and switch later — your steering files, hooks, and credentials work identically with both transports.
