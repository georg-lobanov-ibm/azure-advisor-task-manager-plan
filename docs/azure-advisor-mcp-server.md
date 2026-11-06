# Azure Advisor MCP Server

Research notes on the server this project will use to read Azure Advisor findings (see [PLAN.md](../PLAN.md), section 3).

## What it is

Azure Advisor is Azure's recommendation engine: it analyzes a subscription's resource configuration and usage and produces actionable recommendations across five categories (Cost, Security, Reliability/HighAvailability, Performance, Operational Excellence).

The **Azure MCP Server** is Microsoft's official MCP server for Azure (`azmcp`), developed in the [microsoft/mcp](https://github.com/microsoft/mcp) repository (`Azure.Mcp.Server`; the older `Azure/azure-mcp` repo is archived and redirects there). It exposes dozens of Azure services as MCP tools, grouped into namespaces. `advisor` is one such namespace, under "Management and governance". Its description: *"List and summarize Azure Advisor recommendations, explore recommendation metadata, and apply recommendations to infrastructure-as-code files."*

Despite the general product page mentioning the ability to "mark recommendations as completed, dismiss, or postpone", **Microsoft's tool reference** for the `advisor` namespace lists exactly **5 tools**, all read-only, with no mutating action. The table below is Microsoft's documentation, not what the version this project pins (3.0.0) registers; see "What the pinned 3.0.0 server registers" below it:

| Tool (CLI form) | Purpose |
|---|---|
| `azmcp advisor metadata get` | Fetch the catalog entry for one recommendation *type* (display name, category, impact, remediation guidance). |
| `azmcp advisor metadata list` | Browse the catalog of all recommendation *types* Advisor can produce, with filters (category, impact, resource type, etc.). Useful for "what could Advisor flag" even before any resource exists. |
| `azmcp advisor recommendation list` | List active, individual recommendations for a subscription (one row per finding). |
| `azmcp advisor recommendation summary` | Aggregate active recommendations into counts, grouped by category/impact/resource-type/recommendation-type. |
| `azmcp advisor recommendation apply` | Return IaC (ARM/Terraform) authoring rules for a resource type, to bake recommendations into templates. Does not touch live infrastructure. |

Microsoft's reference annotates every one of these five tools `Read Only: true`, `Destructive: false`.

### What the pinned 3.0.0 server registers

The 3.0.0 server's debug log, as recorded by the project (the log itself is not in either repo), lists **six** advisor commands. That is not Microsoft's five: there is no `recommendation apply`, and there are two commands Microsoft's table lacks (`recommendation update`, `remediation get`):

| Command (as logged) | Purpose |
|---|---|
| `advisor_recommendation_list` | List active recommendations. |
| `advisor_recommendation_summary` | Aggregate counts of active recommendations. |
| `advisor_recommendation_update` | **Change a recommendation's state** (New, Postponed, Dismissed or Completed, per the command's option descriptions in the source). |
| `advisor_metadata_list` | Browse the catalog of recommendation types. |
| `advisor_metadata_get` | Fetch one recommendation type. |
| `advisor_remediation_get` | Get remediation guidance for a recommendation type. |

The 3.0.0 source ([`AdvisorSetup.cs`](https://github.com/microsoft/mcp/blob/Azure.Mcp.Server-3.0.0/tools/Azure.Mcp.Tools.Advisor/src/AdvisorSetup.cs)) registers the same six commands: `recommendation` {`list`, `update`, `summary`}, `metadata` {`list`, `get`}, `remediation` {`get`}.

With `--namespace advisor` these six are not exposed as six separate MCP tools. In namespace mode the server exposes **four** MCP tools: `advisor` (a router that takes the sub-command), `group_list`, `group_resource_list` and `subscription_list`.

So the claim that no tool in this namespace can dismiss, postpone or complete a recommendation holds for Microsoft's documentation but **not** for 3.0.0: `advisor_recommendation_update` does exactly that, and the command's unit test in the 3.0.0 source asserts it is not read-only and is destructive. The plan's non-goal ("any write access to Azure", PLAN.md section 1) therefore depends on `--read-only` removing it:

- `--read-only` is described as "If true, no write operations will be allowed" ([`ServerStartOptions.cs`](https://github.com/microsoft/mcp/blob/Azure.Mcp.Server-3.0.0/core/Microsoft.Mcp.Core/src/Areas/Server/Options/ServerStartOptions.cs)), and the namespace tool loader ([`NamespaceToolLoader.cs`](https://github.com/microsoft/mcp/blob/Azure.Mcp.Server-3.0.0/core/Microsoft.Mcp.Core/src/Areas/Server/Commands/ToolLoading/NamespaceToolLoader.cs)) filters out sub-commands that are not read-only and refuses them at execution time. So the source indicates `--read-only` removes `advisor_recommendation_update`.
- **Unverified:** the project has not confirmed this against a live tool list. Until it does, treat the Azure side as read-only by expectation, not by observation.

Dismissing, postponing or completing a recommendation is outside this agent's scope either way; it has to happen in the Azure portal/API directly.

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

This project uses the PyPI package [`msmcp-azure`](https://pypi.org/project/msmcp-azure/), pinned to an exact version (currently `3.0.0`) and run with `uvx`. Node.js and npm are not needed; `uv` is the only prerequisite, and the Jira server needs it too. The pin is bumped deliberately, with the tool list re-checked, so a new release is never picked up unreviewed.

Microsoft's own Python sample (`get-started/languages/python`) does exactly this with the standard `mcp` package. This is **Microsoft's npm sample**, which launches the npm package with `npx`:

```python
from mcp import StdioServerParameters
from mcp.client.stdio import stdio_client

server_params = StdioServerParameters(
    command="npx",
    args=["-y", "@azure/mcp@latest", "server", "start", "--namespace", "advisor", "--read-only"],
    env=None,
)
```

The **project's command** (`src/advisor_agent/azure_mcp.py` in the project repo) starts the same server from PyPI, pinned, and with an explicit environment instead of `env=None`:

```python
server_params = StdioServerParameters(
    command="uvx",
    args=["--from", "msmcp-azure==3.0.0", "azmcp", "server", "start",
          "--namespace", "advisor", "--read-only"],
    env=env,  # minimal environment plus AZURE_* variables; see "Auth and access"
)
```

The agent uses the same `mcp` stdio client, not an agent framework's own MCP option: `stdio_client` starts each server (Azure and Jira alike) as a local subprocess and speaks MCP to it over stdin/stdout, and the `anthropic` SDK's `async_mcp_tool` turns each MCP tool into a tool its tool runner (`client.beta.messages.tool_runner`) can call. The model sees each tool under the name the server reports, with no prefix. See PLAN.md section 3 for how this is wired into the agent, including the approval gate (Jira tools only; the Azure tools run without approval).

## Auth and access

Uses the caller's Azure credential via Azure RBAC — no separate server credential to manage. A **Reader** role at the subscription (or management group) scope is enough for every tool above.

**The default credential chain, and its exact order, matter for this project** because the subprocess's environment is whatever the agent passes to it, not an interactively-logged-in developer machine. The agent does not hand the server its own environment. It starts each server with a **minimal environment**: `HOME`, `PATH`, proxy and CA-certificate settings, `TERM` and similar, plus only the `AZURE_*` variables that are set in the agent's process. `ANTHROPIC_API_KEY` and every other variable never reach the server. `az login` needs no forwarding: its token cache is found through `HOME`.

The chain is a fixed sequence in the Azure MCP Server binary itself (via the Azure Identity SDK); the server has no notion of which client is driving it. Per the server's own [Authentication guide](https://github.com/microsoft/mcp/blob/main/docs/Authentication.md), for local/stdio use (how this project connects — see "Direct MCP connection from the agent" above) the server tries credential sources in this order, first success wins:

1. `EnvironmentCredential` — the service-principal env vars: `AZURE_TENANT_ID`, `AZURE_CLIENT_ID`, `AZURE_CLIENT_SECRET`
2. `VisualStudioCredential` — a cached Visual Studio sign-in
3. `VisualStudioCodeCredential` — VS Code's Azure Account extension cache
4. `AzureCliCredential` (`az login`)
5. `AzurePowerShellCredential` — an Azure PowerShell sign-in cache
6. `AzureDeveloperCliCredential` — an `azd` (Azure Developer CLI) sign-in cache
7. `InteractiveBrowserCredential` — an interactive browser login

(A broker check — OS broker/WAM on Windows — runs before this chain if broker mode is enabled.)

Two of these steps behave differently from what one might assume for a non-interactive agent run:

- **Step 7 is not skipped when nothing is pinned, and it can hang.** In [`CustomChainedCredential.cs`](https://github.com/microsoft/mcp/blob/Azure.Mcp.Server-3.0.0/core/Microsoft.Mcp.Core/src/Services/Azure/Authentication/CustomChainedCredential.cs) (tag `Azure.Mcp.Server-3.0.0`), the browser credential is added to the chain unless a specific credential is pinned. It is built while the chain is created, and on Linux building it calls `XOpenDisplay(":1")` ([`WindowHandleProvider.cs`](https://github.com/microsoft/mcp/blob/Azure.Mcp.Server-3.0.0/core/Microsoft.Mcp.Core/src/Services/Azure/Authentication/WindowHandleProvider.cs)), which ignores `DISPLAY`. The project observed that when the X socket `@/tmp/.X11-unix/X1` does not accept connections, that call never returns and every Azure tool call hangs with no error (project PR #10, checked by the maintainer in a real run; the 3.0.0 source shows the call, not an unbounded wait). So step 7 does not "always fail"; it can **hang**.
- **Step 3 can come first.** When `VSCODE_PID` is set and `AZURE_TOKEN_CREDENTIALS` is not, the server puts the VS Code credential at the front of the chain.

Steps 2, 5 and 6 only look at local sign-in caches. Whether one of them succeeds depends on the machine the agent runs on.

Three things worth correcting against an easy but wrong assumption:

- **Environment variables win, not `az login`.** If the server's environment has both a service-principal env var set and an ambient CLI login, the env vars are used — `az login` (step 4) only gets a chance if step 1 fails.
- **Managed identity is not in this default chain at all.** It only appears if `AZURE_TOKEN_CREDENTIALS=prod` is explicitly set, which *restricts* the chain to "Environment, Azure Pipelines (when configured), Workload Identity, and Managed Identity credentials only" — the right setting for a hosted/scheduled run (PLAN.md section 9, Phase 4), where the agent should not fall through to a developer-tool credential by accident. For a user-assigned managed identity, also set `AZURE_CLIENT_ID`.
- **No automatic tenant-ID fallback is documented.** For the service-principal path, `AZURE_TENANT_ID` must be set explicitly; the server does not infer it from an `az account set` context the way one might expect.

**For this project:** the agent pins the credential on purpose. `AZURE_TOKEN_CREDENTIALS` defaults to `AzureCliCredential` (the `az login` session) when it is unset or empty. Naming a credential makes the server use only that credential and skips the browser credential (the source adds it only when the variable is unset, `dev` or `InteractiveBrowserCredential`). To use something else, set `AZURE_TOKEN_CREDENTIALS` yourself:

| Setting | Use | Needs |
|---|---|---|
| `AzureCliCredential` (agent default) | Developer machine with `az login` | an `az login` session |
| `EnvironmentCredential` | Service principal | `AZURE_TENANT_ID`, `AZURE_CLIENT_ID`, `AZURE_CLIENT_SECRET` |
| `prod` | Managed identity or workload identity (hosted runs) | the platform identity; `AZURE_CLIENT_ID` for a user-assigned managed identity |

Do **not** set it to `dev` or leave it empty in the server's environment: both bring the browser credential, and with it the hang, back. (The agent replaces an empty value with the default, but a server started some other way would not.)

Server options relevant to us: `--namespace advisor` (expose only this namespace) and `--read-only` (only read-only commands are exposed; see "What the pinned 3.0.0 server registers" above). The project does not use `--tool`.

Keep `--read-only`: on Microsoft's five-tool documentation it would be redundant, but in 3.0.0 `advisor_recommendation_update` is registered. The source indicates `--read-only` removes it; that is **unverified** against a live tool list, so re-check the tool list whenever the pinned version changes.

Status: the Azure server was confirmed to start and answer with the pinned credential (project PR #10). The agent's wiring to the Anthropic API against the live servers has **not been verified**.

## Open items this doesn't resolve

- Whether `--read-only` removes `advisor_recommendation_update` from the live tool list (the 3.0.0 source indicates it does; not confirmed live). Re-check the tool list whenever `AZURE_MCP_VERSION` is bumped.
- The Microsoft tool table and the "`recommendation list`: what it returns and what it already filters" section above come from Microsoft's documentation and were not re-checked against 3.0.0.
- Azure Policy exemptions vs. Advisor suppressions (different systems; still needs a decision — see PLAN.md section 11).
- Pagination behavior of `recommendation list` beyond `top=100` (the docs warn aggregate counts from `list` can undercount; use `recommendation summary` for counts, and plan to page `list` with repeated calls / filters for completeness rather than relying on a single call).
- Whether `suppressionIds` ever appears non-empty on a `New`-status recommendation (would indicate partial/scheduled suppression); not observed in the sampled docs.

## Sources

- Microsoft Learn: [Azure MCP Server tools for Azure Advisor](https://learn.microsoft.com/en-us/azure/developer/azure-mcp-server/tools/azure-advisor)
- Microsoft Learn: [Azure MCP Server tools (index)](https://learn.microsoft.com/en-us/azure/developer/azure-mcp-server/tools/)
- Microsoft Learn: [Advisor REST API — Recommendations: List](https://learn.microsoft.com/en-us/rest/api/advisor/recommendations/list)
- [microsoft/mcp](https://github.com/microsoft/mcp) (current home of the Azure MCP Server; the former `Azure/azure-mcp` repo is archived)
- [microsoft/mcp — Authentication guide](https://github.com/microsoft/mcp/blob/main/docs/Authentication.md) (default credential chain order, `AZURE_TOKEN_CREDENTIALS` override, managed identity handling)
- microsoft/mcp at tag `Azure.Mcp.Server-3.0.0` (the pinned version): [`AdvisorSetup.cs`](https://github.com/microsoft/mcp/blob/Azure.Mcp.Server-3.0.0/tools/Azure.Mcp.Tools.Advisor/src/AdvisorSetup.cs) (registered advisor commands), [`ServerStartOptions.cs`](https://github.com/microsoft/mcp/blob/Azure.Mcp.Server-3.0.0/core/Microsoft.Mcp.Core/src/Areas/Server/Options/ServerStartOptions.cs) (`--namespace`, `--read-only`, `--tool`), [`NamespaceToolLoader.cs`](https://github.com/microsoft/mcp/blob/Azure.Mcp.Server-3.0.0/core/Microsoft.Mcp.Core/src/Areas/Server/Commands/ToolLoading/NamespaceToolLoader.cs) (read-only filtering), [`CustomChainedCredential.cs`](https://github.com/microsoft/mcp/blob/Azure.Mcp.Server-3.0.0/core/Microsoft.Mcp.Core/src/Services/Azure/Authentication/CustomChainedCredential.cs) (credential chain), [`WindowHandleProvider.cs`](https://github.com/microsoft/mcp/blob/Azure.Mcp.Server-3.0.0/core/Microsoft.Mcp.Core/src/Services/Azure/Authentication/WindowHandleProvider.cs) (`XOpenDisplay`)
- PyPI: [`msmcp-azure`](https://pypi.org/project/msmcp-azure/)
