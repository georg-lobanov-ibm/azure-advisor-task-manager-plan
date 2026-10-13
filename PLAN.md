# Azure Advisor Task Manager: Implementation Plan

## 1. Goal & scope

Build a Claude agent that reads Azure Advisor findings through an MCP server and, based on its decision, creates and syncs Jira tickets through a second MCP server.

- **Inputs:** Advisor recommendations across Cost, Security, Reliability, Performance and Operational Excellence.
- **Per-finding decision:** `create`, `update`, `skip` or `close`.
- **Autonomy:** human approval is required before any Jira write.
- **Non-goals:** remediating Azure resources, and any write access to Azure.

## 2. Architecture

A Python app built on the Claude Agent SDK (`claude_agent_sdk`) with two MCP servers attached and a database to track Advisor-to-Jira mappings:

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

- **The agent connects to both MCP servers directly as an MCP client** — via the Claude Agent SDK's `mcp_servers` option, not by shelling out to a CLI. Each server is a locally-spawned stdio subprocess the SDK launches and talks MCP to directly: `npx -y @azure/mcp@latest server start --namespace advisor --read-only` for Azure Advisor; the equivalent launch command for the Jira/Atlassian MCP server is still TBD (Phase 0).
- MCP tools are named `mcp__<server_name>__<tool_name>` (e.g. `mcp__azure_advisor__recommendation_list`). `allowed_tools` wildcards only match after a literal `mcp__<server>__` prefix — a bare `mcp__*` or `*` is ignored, so each server's allowed tools must be listed or scoped per-server explicitly.
- Restrict the agent with `allowed_tools`: Azure Advisor tools are read-only, and Jira read/search tools are allowed.
- Jira write tools (`create_issue`, `update_issue`, transitions) are blocked until approval. The gate must be a **PreToolUse hook** (matching `^mcp__jira__`, returning an `ask`/`defer` permission decision) rather than a `can_use_tool` callback alone: hooks run before `allowed_tools` is evaluated, so the gate can't be silently bypassed by an allow-list entry; `can_use_tool` is skipped entirely for any tool `allowed_tools` already approves.
- **Auth:** managed identity or a service principal with the Reader role for Azure — resolved via the standard Azure credential chain in the environment the subprocess inherits — and a Jira API token. Secrets come from env vars or a secret store, never from the repo.

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
  agent.py      # Agent SDK setup, MCP config, permission gate
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

- Maturity and tool naming of the Jira MCP server (the Azure Advisor side is resolved — see section 3 and `docs/azure-advisor-mcp-server.md`).
- How exemptions are exposed by the Azure MCP server (Advisor suppressions vs. Azure Policy exemptions), and whether the Advisor API already omits them or the agent must filter.
- What to do with an existing Jira ticket whose finding later becomes exempted (leave alone, or propose closing).
- Single vs. multiple subscriptions in scope.
- Jira project, issue type and custom field mapping.
- Approval channel beyond the CLI.
- Where scheduled runs are hosted.
- Database: SQLite vs. PostgreSQL vs. Azure Cosmos DB for multi-instance deployments. Abstraction layer in Phase 1 allows swapping later.
- How to populate `fix_evaluation`: manual entry in the Jira ticket, an agent follow-up query to Advisor to check if the recommendation is resolved, or both?
- Retention policy: how long to keep closed mappings in the database?
