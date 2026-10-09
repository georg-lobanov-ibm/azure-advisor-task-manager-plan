# Azure Advisor Task Manager: Implementation Plan

## 1. Goal & scope

Build a Claude agent that reads Azure Advisor findings through an MCP server and, based on its decision, creates and syncs Jira tickets through a second MCP server.

- **Inputs:** Advisor recommendations across Cost, Security, Reliability, Performance and Operational Excellence.
- **Per-finding decision:** `create`, `update`, `skip` or `close`.
- **Autonomy:** human approval is required before any Jira write.
- **Non-goals:** remediating Azure resources, and any write access to Azure.

## 2. Architecture

A Python app built on the Claude Agent SDK (`claude_agent_sdk`) with two MCP servers attached:

| Server | Purpose | Access |
|---|---|---|
| Azure MCP (Advisor tools) | List recommendations | Read-only |
| Jira / Atlassian MCP | Search, create, update, transition issues | Writes gated by approval |

Flow:

```
fetch Advisor findings -> normalize -> dedupe against Jira -> decide
  -> structured proposal -> human approval -> create/sync in Jira -> audit log
```

## 3. MCP integration

- **Server choice is an open item.** Phase 0 verifies the tool names exposed by the current Azure MCP server (Advisor) and the Atlassian/Jira MCP server.
- Restrict the agent with `allowed_tools`: Azure tools are read-only, and Jira read/search tools are allowed.
- Jira write tools (`create_issue`, `update_issue`, transitions) are blocked until approval. The gate is a `can_use_tool` permission callback or a PreToolUse hook.
- **Auth:** managed identity or a service principal with the Reader role for Azure, and a Jira API token. Secrets come from env vars or a secret store, never from the repo.

## 4. Decision policy

Encoded in the system prompt (`prompts/system.md`) and in deterministic rules (`policy.py`).

- **Inputs per finding:** category, impact (High/Medium/Low), resource, subscription, recommendation ID, potential savings.
- **Create** when impact is at or above the configured threshold and no open ticket exists.
- **Skip** when the finding is suppressed or postponed in Advisor, below the threshold, or a duplicate.
- **Group** by recommendation type per subscription or resource group to avoid ticket floods.
- **Sync:** if a finding disappears from Advisor, propose closing its ticket. If severity changes, propose an update.
- **Output:** a structured proposal (JSON schema) with action, rationale, and Jira fields (project, issue type, summary, description, labels, priority).

## 5. Dedupe and idempotency

Store a stable key derived from the Advisor recommendation ID (`advisor-id:<hash>`) in a Jira label or custom field. Before proposing, search Jira by it with JQL. Re-running the agent must never produce a second ticket for the same finding.

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
prompts/system.md
config.yaml
tests/
```

## 9. Phased rollout

| Phase | Deliverable |
|---|---|
| 0 | Spike: connect to both MCP servers, list tools, fetch sample data |
| 1 | Read-only report and dry-run proposals |
| 2 | Approval-gated Jira create with dedupe |
| 3 | Sync: update and close tickets |
| 4 | Scheduled runs (cron or Azure Container Apps job) and notifications |

## 10. Testing and evaluation

- Recorded Advisor payload fixtures.
- A mocked Jira MCP server.
- A golden-set eval of decisions (create/skip/update/close).
- Dedupe and idempotency tests (double run produces no new tickets).
- A prompt-injection test using a malicious recommendation text.

## 11. Risks and open questions

- Maturity and tool naming of the Azure and Jira MCP servers.
- Single vs. multiple subscriptions in scope.
- Jira project, issue type and custom field mapping.
- Approval channel beyond the CLI.
- Where scheduled runs are hosted.
