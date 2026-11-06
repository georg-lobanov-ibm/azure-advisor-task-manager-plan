# Azure Advisor Task Manager: Implementation Plan

## 1. Goal & scope

Build a Claude agent that reads Azure Advisor findings through an MCP server and, based on its decision, creates and syncs Jira tickets through a second MCP server.

- **Inputs:** Advisor recommendations across Cost, Security, Reliability, Performance and Operational Excellence.
- **Per-finding decision:** `create`, `update`, `skip` or `close`.
- **Autonomy:** human approval is required before any Jira write.
- **Non-goals:** remediating Azure resources, and any write access to Azure.

## 2. Architecture

A Python app built on the `anthropic` SDK's tool runner (`client.beta.messages.tool_runner`), with two MCP servers attached through the `mcp` package's stdio clients and a database to track Advisor-to-Jira mappings:

| Server | Purpose | Access |
|---|---|---|
| Azure MCP (Advisor tools) | List recommendations | Read-only |
| Jira / Atlassian MCP | Search, create, update, transition issues | Writes gated by approval |
| Database | Track Advisor recommendation ↔ Jira issue links | Read/write |

Flow:

```
fetch Advisor findings -> drop exempted -> normalize -> query db for known mappings -> dedupe
  -> decide -> structured proposal -> human approval -> create/sync in Jira and db -> audit log
```

**Database**: stores the bidirectional link between Advisor recommendations and Jira story/issue keys, plus metadata:
- `recommendation_type_id` + `resource_id` (composite PK): the stable identifiers of the recommendation rule and the Azure resource it applies to. The per-finding `id`/`name` GUID Advisor returns is not stable across its recalculation cycles, so it is not used as a key (see `docs/azure-advisor-mcp-server.md`).
- `jira_issue_key`: e.g., "PROJ-123"
- `created_at`: when the link was first recorded
- `status`: "open", "closed", "paused"
- `fix_evaluation`: notes on whether the fix was effective (populated post-remediation)
- `last_synced`: timestamp of the last agent sync
- `notes`: optional manual annotations

## 3. MCP integration

- **The agent connects to both MCP servers directly as an MCP client** — through the `mcp` package's stdio client, not by shelling out to a CLI. Each server is a locally-spawned stdio subprocess that the agent starts and talks MCP to directly:
  - Azure Advisor: `uvx --from msmcp-azure==3.0.0 azmcp server start --namespace advisor --read-only` (the PyPI package, version pinned and bumped deliberately). Node.js and npm are not needed; `uv` is the only prerequisite, and the Jira server needs it too.
  - Jira: `uvx mcp-atlassian` (see `docs/jira-mcp-server.md`).
- **Runtime:** the `anthropic` SDK's tool runner (`client.beta.messages.tool_runner`) runs the model-and-tools loop, and each MCP tool is turned into a runner tool. The model sees each tool under the name its server reports, with no `mcp__<server>__` prefix; a tool name that two servers both report stops startup. The agent has no built-in file, shell or web tools: its only tools are the Azure and Jira MCP tools.
- **Azure tools:** in namespace mode the server exposes four MCP tools: `advisor` (a router that takes a sub-command), `group_list`, `group_resource_list` and `subscription_list`. The Azure tools run without approval. The 3.0.0 server registers an `advisor_recommendation_update` command that changes a recommendation's state; the server source indicates `--read-only` removes it, but this is **unverified** against a live tool list (see `docs/azure-advisor-mcp-server.md`).
- **Approval gate for Jira writes:** every tool is wrapped in an `ApprovalGatedTool` (`mcp_tools.py`) that asks for a terminal "yes" before the call reaches the server. Only `jira_search`, `jira_get_issue` and `jira_get_transitions` run without approval; every other Jira tool, including any tool the code does not list, needs approval (the gate fails closed). If stdin or stderr is not a terminal, gated calls are denied and the model is told. The gate is in the agent's own tool wrapper, not in a server setting, so a server option or a tool allow-list cannot turn it off.
- **Server environment:** each server gets a minimal environment (`HOME`, `PATH`, proxy and CA-certificate settings, `TERM` and similar) plus only the variables it needs: the set `AZURE_*` variables for Azure, the chosen `JIRA_*` variables for Jira. `ANTHROPIC_API_KEY` never reaches a server.
- **Auth:** the Azure identity needs the Reader role (a managed identity, a service principal, or, in development, the developer's own `az login` session), and Jira needs an API token (Cloud: `JIRA_URL`, `JIRA_USERNAME`, `JIRA_API_TOKEN`; Server/Data Center: `JIRA_URL`, `JIRA_PERSONAL_TOKEN`). `AZURE_TOKEN_CREDENTIALS` defaults to `AzureCliCredential` (`az login`), pinned on purpose: in Azure MCP Server 3.0.0, leaving it unset makes the server build an interactive browser credential, which on Linux can hang every Azure tool call with no error. For a service principal set `EnvironmentCredential` with `AZURE_TENANT_ID`, `AZURE_CLIENT_ID` and `AZURE_CLIENT_SECRET`; for a managed identity set `prod`. Do not use `dev` or an empty value. Secrets come from env vars or a secret store, never from the repo.
- **Status:** the Azure server was confirmed to start and answer with the pinned credential. The agent's wiring against the Anthropic API, and the Jira server, have not been verified live.

## 4. Decision policy

Encoded in the system prompt (`prompts/system.md`) and in deterministic rules (`policy.py`).

- **Inputs per finding:** category, impact (High/Medium/Low), resource, subscription, recommendation ID, potential savings.
- **Create** when impact is at or above the configured threshold and no open ticket exists.
- **Exempted findings are excluded entirely.** If a rule or recommendation is exempted/suppressed in Azure Advisor, it must not appear in the findings at all: not as a proposal, not as a `skip` row, not in reports. This is enforced by a deterministic filter in `policy.py` right after fetch, before the model sees any data, so it never depends on the prompt.
- **Skip** when the finding is postponed in Advisor, below the threshold, or a duplicate.
- **Group** by recommendation type per subscription or resource group to avoid ticket floods.
- **Sync:** if a finding disappears from Advisor, propose closing its ticket. If severity changes, propose an update.
- **Output:** a structured proposal (JSON schema) with action, rationale, and Jira fields (project, issue type, summary, description, labels, priority).

## 5. Dedupe and idempotency

Query the database for an existing mapping by the composite key `(recommendation_type_id, resource_id)` before proposing a new ticket. If a mapping exists, fetch the Jira issue key and link directly. Also store a label or custom field derived from that composite key (e.g. `advisor-key:<hash of the pair>`) on the Jira ticket for manual cross-reference. Re-running the agent must never produce a second ticket for the same finding.

The database is the source of truth for the mapping; Jira labels are redundant but help with manual tracing.

## 6. Human approval flow

- **Phase 1:** proposals are printed as a CLI table. The reviewer approves, rejects or edits each item.
- **Later:** a Jira "Proposed" label, or a Teams/Slack approval message.
- Rejections are recorded so the agent does not re-propose them.

## 7. Safety and guardrails

- Dry-run is the default mode.
- Maximum number of proposed tickets per run.
- Advisor text (descriptions, resource names) is untrusted data and must never be treated as instructions (prompt-injection risk).
- Least-privilege credentials on both servers.
- Full audit log (JSONL): findings seen, decisions, approvals and every tool call.

## 8. Proposed project layout

```
src/advisor_agent/
  main.py       # entrypoint / CLI
  agent.py      # tool-runner setup, conversation session, composes the MCP server configs
  mcp_tools.py  # MCP stdio client helpers, approval-gated tool wrapper, server environment
  azure_mcp.py  # Azure Advisor MCP server config
  jira_mcp.py   # Jira MCP server config, read-tool list
  policy.py     # thresholds, grouping, dedupe rules
  approval.py   # approval UI and rejection store
  audit.py      # JSONL audit log
  database.py   # database schema, queries, and migrations
prompts/system.md
config.yaml
tests/
migrations/   # database schema versions
```

**Database choice**: SQLite (file-based, no external service) for Phase 1-2, with an abstraction layer to swap in PostgreSQL or Azure Cosmos DB for production/multi-instance deployments.

## 9. Phased rollout

| Phase | Deliverable |
|---|---|
| 0 | Spike: connect to both MCP servers, list tools, fetch sample data |
| 1 | Read-only report and dry-run proposals |
| 2 | Approval-gated Jira create with dedupe |
| 3 | Sync: update and close tickets |
| 4 | Scheduled runs (cron or Azure Container Apps job) and notifications |

## 10. Testing and evaluation

- Recorded Advisor payload fixtures, including exempted/suppressed findings.
- A test that exempted findings never reach the agent or any output.
- A mocked Jira MCP server.
- A golden-set eval of decisions (create/skip/update/close).
- Dedupe and idempotency tests (double run produces no new tickets, database state is consistent).
- A prompt-injection test using a malicious recommendation text.
- Database tests: schema creation, migration, mapping CRUD operations, querying by `(recommendation_type_id, resource_id)` and `jira_issue_key`.

## 11. Risks and open questions

- Maturity of the Jira MCP server, and whether its tool names (the read-tool list in particular) hold against a live server. Azure Advisor side: the server starts and answers with the pinned credential (see section 3 and `docs/azure-advisor-mcp-server.md`); whether `--read-only` removes `advisor_recommendation_update`, and the agent's wiring against the Anthropic API, are not verified live.
- How the per-call terminal gate described in section 3 (`mcp_tools.py`) relates to the proposal-level approval flow in section 6 (`approval.py`).
- How exemptions are exposed by the Azure MCP server (Advisor suppressions vs. Azure Policy exemptions), and whether the Advisor API already omits them or the agent must filter.
- What to do with an existing Jira ticket whose finding later becomes exempted (leave alone, or propose closing).
- Single vs. multiple subscriptions in scope.
- Jira project, issue type and custom field mapping.
- Approval channel beyond the CLI.
- Where scheduled runs are hosted.
- Database: SQLite vs. PostgreSQL vs. Azure Cosmos DB for multi-instance deployments. Abstraction layer in Phase 1 allows swapping later.
- How to populate `fix_evaluation`: manual entry in the Jira ticket, an agent follow-up query to Advisor to check if the recommendation is resolved, or both?
- Retention policy: how long to keep closed mappings in the database?
