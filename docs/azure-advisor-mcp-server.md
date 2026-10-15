# Azure Advisor MCP Server

Research notes on the server this project will use to read Azure Advisor findings (see [PLAN.md](../PLAN.md), section 3).

## What it is

Azure Advisor is Azure's recommendation engine: it analyzes a subscription's resource configuration and usage and produces actionable recommendations across five categories (Cost, Security, Reliability/HighAvailability, Performance, Operational Excellence).

The **Azure MCP Server** is Microsoft's official MCP server for Azure (`azmcp`), developed in the [microsoft/mcp](https://github.com/microsoft/mcp) repository (`Azure.Mcp.Server`; the older `Azure/azure-mcp` repo is archived and redirects there). It exposes dozens of Azure services as MCP tools, grouped into namespaces. `advisor` is one such namespace, under "Management and governance". Its description: *"List and summarize Azure Advisor recommendations, explore recommendation metadata, and apply recommendations to infrastructure-as-code files."*

Despite the general product page mentioning the ability to "mark recommendations as completed, dismiss, or postpone", the tool reference for the `advisor` namespace lists exactly **5 tools**, all read-only, with no mutating action:

| Tool (CLI form) | Purpose |
|---|---|
| `azmcp advisor metadata get` | Fetch the catalog entry for one recommendation *type* (display name, category, impact, remediation guidance). |
| `azmcp advisor metadata list` | Browse the catalog of all recommendation *types* Advisor can produce, with filters (category, impact, resource type, etc.). Useful for "what could Advisor flag" even before any resource exists. |
| `azmcp advisor recommendation list` | List active, individual recommendations for a subscription (one row per finding). |
| `azmcp advisor recommendation summary` | Aggregate active recommendations into counts, grouped by category/impact/resource-type/recommendation-type. |
| `azmcp advisor recommendation apply` | Return IaC (ARM/Terraform) authoring rules for a resource type, to bake recommendations into templates. Does not touch live infrastructure. |

Every one of these tools is annotated `Read Only: true`, `Destructive: false` in the server's own tool metadata. This confirms what the plan already assumed in section 2 (non-goal: "any write access to Azure") — there is no tool in this namespace to dismiss, postpone, or mark a recommendation resolved. Any such action has to happen in the Azure portal/API directly, outside this agent's scope.

## `recommendation list`: what it returns and what it already filters

This is the tool our agent will call each run. Key behavior:

- Returns only recommendations with status **`New`**. Dismissed and postponed recommendations are excluded by the tool itself.
- Supports filters: `category`, `impact`, `resource-type`, `resource` (name or ARM ID substring), `search` (free-text over the problem description), `top` (max 100, default 50), `resource-group`.
- No single "get by ID" tool exists — the agent must `list` (optionally paging) and diff against what it already knows, rather than re-fetching one recommendation by ID.

**This resolves part of the plan's "exempted findings" question (section 4/11):** a *dismissed* or *postponed* recommendation already never appears in `recommendation list`'s output — Advisor's own API does this filtering, not just this MCP tool. The agent does not need its own filter for that case. What's still open is whether an **Azure Policy exemption** (a different mechanism — see Azure Policy, not Advisor) also suppresses the matching Advisor recommendation, or whether that needs separate handling.

## Identifiers, and the critical stability question

The plan's section 11 flagged `advisor_id` stability as `[CRITICAL]` for the database primary key. Here's what the underlying Advisor REST API (which this MCP tool wraps) actually returns per recommendation:

```json
{
  "id": "/subscriptions/.../providers/Microsoft.Advisor/recommendations/<GUID>",
  "name": "<GUID>",
  "properties": {
    "recommendationTypeId": "<type-GUID>",
    "resourceMetadata": { "resourceId": "/subscriptions/.../resourceGroups/.../providers/..." },
    "lastUpdated": "2017-02-24T22:24:43Z",
    "category": "...", "impact": "...",
    "suppressionIds": ["<uuid>", "..."]
  }
}
```

Three different identifiers are in play, with different stability:

| Field | What it identifies | Stable across refreshes? |
|---|---|---|
| `id` / `name` (the per-finding GUID) | This specific cached recommendation instance | **No** — Advisor recomputes recommendations (`generateRecommendations`) and `lastUpdated` tracks when it last revalidated one. The per-instance GUID is not documented as persistent across recalculation cycles. |
| `properties.recommendationTypeId` | The recommendation *rule* (e.g., "right-size this VM SKU") | **Yes** — the MCP docs explicitly call this "the stable Advisor recommendation type ID ... the join key shared with individual recommendations." It is shared by every resource that triggers the same rule. |
| `properties.resourceMetadata.resourceId` | The Azure resource the finding is about | Yes, for the life of the resource (standard ARM resource ID). |

**Conclusion for the plan:** don't use the per-finding `id`/`name` GUID as the database primary key — treat it as a transient reference useful only for logging/debugging a single run. Use the composite key **`(recommendationTypeId, resourceId)`** instead. This is stable: if an issue is fixed and later reappears on the same resource for the same rule, it will produce a new per-finding GUID but the same `(recommendationTypeId, resourceId)` pair, so the database still recognizes it as "the same finding" and finds the existing Jira link.

This replaces the single `advisor_id` field proposed in PLAN.md section 2 with a composite key. **Applied**: PLAN.md sections 2, 5, 10 and 11 now use `(recommendation_type_id, resource_id)` as the database key instead of a single `advisor_id`.

## Direct MCP connection from the agent

The agent connects to the Azure MCP Server **directly as an MCP client** — not by shelling out to the `azmcp` CLI per call. Microsoft documents no remotely-hosted Azure MCP Server endpoint: every connection method (NuGet `Azure.Mcp`, NPM `@azure/mcp`, PyPI `msmcp-azure`, Docker, IDE integrations) spawns the server as a **local subprocess** that talks MCP over stdio.

Microsoft's own Python sample (`get-started/languages/python`) does exactly this with the standard `mcp` package:

```python
from mcp import StdioServerParameters
from mcp.client.stdio import stdio_client

server_params = StdioServerParameters(
    command="npx",
    args=["-y", "@azure/mcp@latest", "server", "start", "--namespace", "advisor", "--read-only"],
    env=None,
)
```

The Claude Agent SDK's `mcp_servers` option on `ClaudeAgentOptions` wraps this same pattern — it spawns the subprocess and speaks MCP to it directly, identically for the Jira MCP server. See PLAN.md section 3 for how this is wired into the agent, including tool naming (`mcp__<server_name>__<tool_name>`) and the approval gate for Jira writes.

## Auth and access

Uses the caller's Azure credential via Azure RBAC — no separate server credential to manage. A **Reader** role at the subscription (or management group) scope is enough for every tool above.

**The default credential chain, and its exact order, matter for this project** because the subprocess's environment is whatever the agent's process passes down — not an interactively-logged-in developer machine. This chain is a fixed sequence hardcoded in the Azure MCP Server binary itself (via the Azure Identity SDK) — it's the same seven steps regardless of what spawned the stdio subprocess, whether that's VS Code, the `azmcp` CLI, or (as in this project) the Claude Agent SDK; the server has no notion of "which client is driving me." Per the server's own [Authentication guide](https://github.com/microsoft/mcp/blob/main/docs/Authentication.md), for local/stdio use (how this project connects — see "Direct MCP connection from the agent" above) the server tries credential sources in this order, first success wins:

1. `EnvironmentCredential` — the service-principal env vars: `AZURE_TENANT_ID`, `AZURE_CLIENT_ID`, `AZURE_CLIENT_SECRET`
2. `VisualStudioCredential` — **irrelevant here**: checks for a cached Visual Studio sign-in; this project never runs under Visual Studio, so this step always fails and falls through
3. `VisualStudioCodeCredential` — **irrelevant here**: checks VS Code's Azure Account extension cache; not applicable to a Claude Agent SDK-launched subprocess, so this always fails and falls through
4. `AzureCliCredential` (`az login`)
5. `AzurePowerShellCredential` — **irrelevant here**: checks an Azure PowerShell sign-in cache; this project doesn't use PowerShell, so this always fails and falls through
6. `AzureDeveloperCliCredential` — **irrelevant here**: checks an `azd` (Azure Developer CLI) sign-in cache; this project doesn't use `azd`, so this always fails and falls through
7. `InteractiveBrowserCredential` — **irrelevant here**: opens an interactive browser login prompt, which has nothing to attach to in a non-interactive agent run, so this always fails too

(A broker check — OS broker/WAM on Windows — runs before this chain if broker mode is enabled; also not applicable to this project's non-interactive, typically Linux-hosted runs.)

In practice, only steps 1 and 4 can ever succeed for this project — the rest are dead weight that cost a few failed lookups per run but otherwise don't matter. Three things worth correcting against an easy but wrong assumption:

- **Environment variables win, not `az login`.** If the agent's process happens to inherit both a service-principal env var set and an ambient CLI login, the env vars are used — `az login` (step 4) only gets a chance if step 1 fails.
- **Managed identity is not in this default chain at all.** It only appears if `AZURE_TOKEN_CREDENTIALS=prod` is explicitly set, which *restricts* the chain to "Environment, Azure Pipelines (when configured), Workload Identity, and Managed Identity credentials only" — the right setting for a hosted/scheduled run (PLAN.md section 9, Phase 4), where the agent should not fall through to a developer-tool credential by accident.
- **No automatic tenant-ID fallback is documented.** For the service-principal path, `AZURE_TENANT_ID` must be set explicitly; the server does not infer it from an `az account set` context the way one might expect.

**For this project:** set `AZURE_TOKEN_CREDENTIALS=prod` alongside the service-principal env vars (or a managed identity) in the subprocess's environment for any non-interactive run, so the credential actually used is deterministic and auditable rather than whichever of the seven default providers happens to resolve first.

Global server options also relevant to us: `--read-only` (enforce no write tools server-wide — redundant here since Advisor has none, but useful as defense-in-depth if other namespaces are enabled on the same server instance), `--namespace advisor` (expose only this namespace), `--tool azmcp_advisor_recommendation_list` (expose a single tool by name).

## Open items this doesn't resolve

- Azure Policy exemptions vs. Advisor suppressions (different systems; still needs a decision — see PLAN.md section 11).
- Pagination behavior of `recommendation list` beyond `top=100` (the docs warn aggregate counts from `list` can undercount; use `recommendation summary` for counts, and plan to page `list` with repeated calls / filters for completeness rather than relying on a single call).
- Whether `suppressionIds` ever appears non-empty on a `New`-status recommendation (would indicate partial/scheduled suppression); not observed in the sampled docs.

## Sources

- Microsoft Learn: [Azure MCP Server tools for Azure Advisor](https://learn.microsoft.com/en-us/azure/developer/azure-mcp-server/tools/azure-advisor)
- Microsoft Learn: [Azure MCP Server tools (index)](https://learn.microsoft.com/en-us/azure/developer/azure-mcp-server/tools/)
- Microsoft Learn: [Advisor REST API — Recommendations: List](https://learn.microsoft.com/en-us/rest/api/advisor/recommendations/list)
- [microsoft/mcp](https://github.com/microsoft/mcp) (current home of the Azure MCP Server; the former `Azure/azure-mcp` repo is archived)
- [microsoft/mcp — Authentication guide](https://github.com/microsoft/mcp/blob/main/docs/Authentication.md) (default credential chain order, `AZURE_TOKEN_CREDENTIALS` override, managed identity handling)
