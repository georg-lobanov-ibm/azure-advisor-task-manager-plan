# Jira MCP Server

Research notes on the server this project will use to create and sync Jira tickets (see [PLAN.md](../PLAN.md), section 3).

## What it is

Jira is the ticketing target: the agent proposes `create`/`update`/`skip`/`close` decisions for Advisor findings, and (after human approval) writes them as Jira issues. Two real MCP options exist for talking to Jira. This project uses the self-hosted one, for a concrete architectural reason explained below — not just as a style preference.

## Server options compared

| | **Official Atlassian Remote MCP Server** | **`sooperset/mcp-atlassian`** (used here) |
|---|---|---|
| Hosting | Always Atlassian-hosted; no self-hosting option | Self-hosted (your own process/container) |
| Transport | Streamable HTTP (`https://mcp.atlassian.com/v2/mcp`); legacy SSE being retired | stdio by default; optional SSE/streamable-HTTP for multi-user setups |
| Auth | OAuth 2.1 (interactive consent), or an admin-enabled API token/API key | API token (Cloud) or Personal Access Token (Server/DC); OAuth 2.0 3LO also supported |
| Jira Cloud / Server-DC | Cloud only (no Server/Data Center mention anywhere in its docs; architecture is built around a Cloud `cloudId`) | Both — Cloud and Server/Data Center (v8.14+) |
| Tool catalog | 215+ tools across 10+ Atlassian products (Jira, Confluence, Bitbucket, JSM, Compass, Loom, Teamwork Graph...), dynamically discovered (`discover` + `executeRead`/`executeWrite`/`executeDestructive`) | ~98 tools, static flat list, Jira + Confluence only, conventional per-tool definitions |
| Maturity | GA, actively versioned (v2 current; v1 sunsetting by March 2027) | Actively maintained (v0.23.1), MIT-licensed, ~6k GitHub stars |

Sources: [`atlassian/atlassian-mcp-server` README](https://github.com/atlassian/atlassian-mcp-server), [atlassian.com/platform/rovo-mcp](https://www.atlassian.com/platform/rovo-mcp), [support.atlassian.com get-started guide](https://support.atlassian.com/atlassian-ai-gateway/docs/get-started-with-the-atlassian-remote-mcp-server/), [`sooperset/mcp-atlassian` README](https://github.com/sooperset/mcp-atlassian).

## Why `mcp-atlassian`, self-hosted via stdio

The deciding factor isn't the bigger tool catalog or Atlassian's own hosting — it's authentication fit. The official server's primary auth is **OAuth 2.1, an interactive user-consent flow**. The Claude Agent SDK does **not** drive that flow itself: per its [MCP docs](https://code.claude.com/docs/en/agent-sdk/mcp), an unauthenticated remote server just comes up as `needs-auth`, and someone has to complete OAuth out-of-band and hand the SDK a cached bearer token. That's a poor fit for an unattended, scheduled agent (PLAN.md section 9, Phase 4).

`mcp-atlassian` instead authenticates with a plain **API token or Personal Access Token**, supplied via an environment variable — no interactive step, and it fits the same "secrets from env vars or a secret store, never the repo" rule PLAN.md section 3 already states for Azure. It also runs as a **local stdio subprocess by default**, exactly the pattern this project already uses for the Azure MCP server:

```python
# Same shape as the Azure Advisor server in PLAN.md section 3 — no "type" key means stdio.
"jira": {
    "command": "uvx",
    "args": ["mcp-atlassian"],
    "env": {
        "JIRA_URL": "https://<your-site>.atlassian.net",
        "JIRA_USERNAME": "<email>",
        "JIRA_API_TOKEN": "<token, from env/secret store>",
    },
}
```

(Docker is the other supported way to run it; both are still a local subprocess the Agent SDK spawns and speaks MCP to directly — not a CLI shell-out.)

## Tools exposed

The Jira-relevant subset (Confluence tools exist in the same server but are out of scope here):

| Tool | Purpose | Write? |
|---|---|---|
| `jira_search` | Search issues by JQL | Read |
| `jira_get_issue` | Fetch one issue by key | Read |
| `jira_get_transitions` | List valid status transitions for an issue | Read |
| `jira_create_issue` | Create a new issue | **Write** |
| `jira_update_issue` | Edit an existing issue's fields | **Write** |
| `jira_transition_issue` | Move an issue to a new status | **Write** |
| `jira_add_comment` / `jira_edit_comment` | Add or edit a comment | **Write** |
| `jira_delete_issue` | Delete an issue (doc notes: cannot be undone) | **Write** |
| Agile tools (`jira_get_agile_boards`, `jira_create_sprint`, ...) | Board/sprint management | Mixed |

Every mutating tool's own doc string carries the note *"This is a write tool. Disabled when `READ_ONLY_MODE=true`."* — there's no separate "destructive" tier the way Azure MCP annotates `Destructive: true/false`; `jira_delete_issue` is just another write tool, gated the same as `jira_create_issue`.

**This matters for PLAN.md section 3's PreToolUse hook.** That section's `^mcp__jira__` regex and the idea of blocking "`create_issue`, `update_issue`, transitions" by name were written before the real tool names were known. The actual names are `jira_create_issue`, `jira_update_issue`, `jira_transition_issue`, `jira_add_comment`/`jira_edit_comment`, and `jira_delete_issue` — all under the `mcp__jira__` prefix once attached via the Agent SDK, so the existing `^mcp__jira__` hook matcher still works, but a hook that tries to allow-list read-only Jira tools by exact name should use these names, not the placeholder ones.

Source: [mcp-atlassian tools reference](https://mcp-atlassian.soomiles.com/docs/tools-reference), [jira-issues](https://mcp-atlassian.soomiles.com/docs/tools/jira-issues).

## Auth and access

Unlike the Azure MCP Server (which uses the Azure Identity SDK's multi-step credential chain), `mcp-atlassian` uses straightforward environment variables. No fallback chain — you explicitly configure one authentication method per deployment, depending on your Jira deployment type and auth policy.

**For Jira Cloud with API token:**
- `JIRA_URL` — Cloud site URL, e.g. `https://my-company.atlassian.net`
- `JIRA_USERNAME` — email address registered on the Jira instance
- `JIRA_API_TOKEN` — an API token generated from the user account (Settings → Security → API tokens)

**For Jira Server or Data Center with Personal Access Token (Server/DC v8.14+):**
- `JIRA_URL` — on-premises server URL, e.g. `https://jira.internal.company.com:8080`
- `JIRA_PERSONAL_TOKEN` — a PAT generated from the user account

**For Jira Server/DC with mutual TLS (client certificate):**
- `JIRA_URL`
- `JIRA_CLIENT_CERT` — path to the client certificate file
- `JIRA_CLIENT_KEY` — path to the private key file
- `ALLOW_GLOBAL_CRED_FALLBACK=true` (permits certificate credentials to authenticate requests lacking an `Authorization` header)

**For Jira Cloud with OAuth 2.0 (not recommended for this project):**
- Multiple `ATLASSIAN_OAUTH_*` environment variables (see [mcp-atlassian authentication docs](https://mcp-atlassian.soomiles.com/docs/authentication) for the full set)
- OAuth introduces the same interactive-consent problem as the official Atlassian Remote MCP Server; avoided here in favor of API token/PAT (which fits a non-interactive agent without extra hand-waving to an external consent step).

**For this project:** set `JIRA_URL`, `JIRA_USERNAME`, and `JIRA_API_TOKEN` in the subprocess's environment if targeting Jira Cloud; use `JIRA_URL` and `JIRA_PERSONAL_TOKEN` if targeting Server/DC. All three are passed down from the agent's environment (via PLAN.md section 3's `env` dict on the `mcp_servers` config) — they must never be committed to the repo, per CLAUDE.md's "secrets from env vars or a secret store, never the repo" rule.

Source: [mcp-atlassian authentication docs](https://mcp-atlassian.soomiles.com/docs/authentication).

## Scoping and the read-only/write split

`mcp-atlassian` has direct analogues to the Azure MCP server's `--read-only`/`--namespace` flags, set as environment variables on the subprocess:

- `READ_ONLY_MODE=true` — disables every write tool server-wide. Useful as defense-in-depth, matching how PLAN.md section 3 already treats Azure's `--read-only` ("redundant ... but useful as defense-in-depth").
- `JIRA_PROJECTS_FILTER` — restrict the server to specific project keys.
- `TOOLSETS` / `ENABLED_TOOLS` — allow-list by tool group or exact tool name (closest equivalent of `--namespace`/`--tool`).

None of these replace the human-approval gate. PLAN.md section 3's **PreToolUse hook** on `^mcp__jira__` is still what makes the approval step unbypassable (hooks run before `allowed_tools` evaluation); `READ_ONLY_MODE` and the filters above are an *additional*, server-side layer — the same "both, not either" relationship Azure's `--read-only` has with `allowed_tools` in this project.

Source: [mcp-atlassian configuration docs](https://mcp-atlassian.soomiles.com/docs/configuration).

## Security note on transport

`mcp-atlassian` has a real history of security advisories (~29 published, several critical) — but every HTTP/SSE-transport related one is an **authentication bypass**: e.g. a token verifier that accepted any non-empty token, or the SSE transport skipping auth middleware entirely, letting an unauthenticated caller ride on the operator's own Jira credentials. A separate cluster covers path-traversal/SSRF bugs in attachment-upload tools.

Staying on **stdio** (this project's choice, same as Azure MCP) sidesteps the whole HTTP/SSE-auth-bypass class outright — there's no network listener to attack. The attachment-handling bugs are patched in current releases but worth re-checking against the pinned version before Phase 2 (ticket creation) ships. This is the same defense-in-depth posture `docs/azure-advisor-mcp-server.md` already recommends for the Azure side.

Source: [`sooperset/mcp-atlassian` security advisories](https://api.github.com/repos/sooperset/mcp-atlassian/security-advisories).

## Official Atlassian Remote MCP Server (alternative, not used here)

Documented for completeness, not as the recommendation: Atlassian's own server (`https://mcp.atlassian.com/v2/mcp`, streamable HTTP) is fully managed, has a much larger tool catalog (Confluence, Bitbucket, Jira Service Management, Compass, Loom, and cross-product "Teamwork Graph" search), and needs no self-hosting. Its blockers for this project: OAuth 2.1 as the primary auth path doesn't fit an unattended agent (see above); Jira Cloud only, with no stated Server/Data Center support; and tool-call cost for its enriched search/context tools draws from an organization's shared "Rovo credit" pool rather than a flat rate limit. The admin-enabled API-key alternative could remove the OAuth blocker if this project later needs the broader catalog, but that's a reason to revisit, not a reason to switch now.

Source: [support.atlassian.com — Rovo MCP server settings](https://support.atlassian.com/security-and-access-policies/docs/control-atlassian-rovo-mcp-server-settings/), [get-started guide](https://support.atlassian.com/atlassian-ai-gateway/docs/get-started-with-the-atlassian-remote-mcp-server/).

## Open items this doesn't resolve

- PLAN.md section 3 still has the Jira launch command as "TBD (Phase 0)" and uses placeholder tool names (`create_issue`, `update_issue`, "transitions"). This doc resolves both — the launch command above and the real `jira_*` names — but applying that fix to PLAN.md itself is a follow-up PR, the same way this doc's Advisor counterpart deferred its own PLAN.md fix.
- PLAN.md section 11's "Maturity and tool naming of the Jira MCP server" open question can likely be closed the same way the Advisor-side `[CRITICAL]` item was, once the above fix lands.
- Jira project, issue type, and custom field mapping (PLAN.md section 11) is unaffected by this research and remains open.

## Sources

- [`atlassian/atlassian-mcp-server`](https://github.com/atlassian/atlassian-mcp-server) (official server's client-side README/manifests)
- Atlassian: [Rovo MCP product page](https://www.atlassian.com/platform/rovo-mcp)
- Atlassian support: [Get started with the Atlassian Remote MCP Server](https://support.atlassian.com/atlassian-ai-gateway/docs/get-started-with-the-atlassian-remote-mcp-server/)
- Atlassian support: [Authentication and authorization](https://support.atlassian.com/atlassian-ai-gateway/docs/authentication-and-authorization/) · [Configure authentication via API token](https://support.atlassian.com/atlassian-ai-gateway/docs/configure-authentication-via-api-token/)
- Atlassian support: [Supported tools](https://support.atlassian.com/atlassian-ai-gateway/docs/supported-tools/) · [Control Atlassian Rovo MCP server settings](https://support.atlassian.com/security-and-access-policies/docs/control-atlassian-rovo-mcp-server-settings/)
- [`sooperset/mcp-atlassian`](https://github.com/sooperset/mcp-atlassian) and its docs: [tools reference](https://mcp-atlassian.soomiles.com/docs/tools-reference), [authentication](https://mcp-atlassian.soomiles.com/docs/authentication), [configuration](https://mcp-atlassian.soomiles.com/docs/configuration), [HTTP transport](https://mcp-atlassian.soomiles.com/docs/http-transport), [compatibility](https://mcp-atlassian.soomiles.com/docs/compatibility)
- [GitHub Security Advisories — `sooperset/mcp-atlassian`](https://github.com/sooperset/mcp-atlassian/security/advisories)
- Claude Agent SDK: [MCP docs](https://code.claude.com/docs/en/agent-sdk/mcp)
