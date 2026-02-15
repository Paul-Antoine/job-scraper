# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Automated n8n workflow for daily job monitoring: an AI Agent autonomously searches alternance (apprenticeship) offers for "Product Builder No-Code & IA Generative" positions around Paris, scores relevance, and delivers a formatted HTML email every morning at 10:00 AM.

This is a **configuration-and-documentation project** — there is no traditional source code, build system, or test suite. The deliverable is an n8n workflow deployed via the n8n MCP server.

## Architecture

Agent-based workflow:

```
Schedule Trigger (10h00) → AI Agent → Gmail (send)
                              ├── LLM : OpenAI gpt-4o-mini
                              └── Tool : SerpAPI (Google Search)
```

- **Schedule Trigger**: Daily at 10:00 Europe/Paris
- **AI Agent** (`@n8n/n8n-nodes-langchain.agent` v3.1): Autonomous agent with up to 15 iterations. Receives a prompt with search instructions, uses SerpAPI to search with varied keywords, scores offers, and outputs formatted HTML.
- **OpenAI Chat Model** (`gpt-4o-mini`): Language model powering the agent. Temperature 0.3, max 4096 output tokens.
- **SerpAPI Tool**: Google Search (not Google Jobs) — the agent decides its own queries and can make multiple searches with different keywords.
- **Gmail** (regular node, not a tool): Receives the agent's HTML output (`$json.output`) and sends the email. Gmail Tool variant doesn't work well with dynamic fields — use the regular Gmail node after the agent instead.

## Key Files

- `PRD.md` — Original specification (v1 architecture, still useful for context on the job profile and requirements)
- `WORKFLOW-DOC.md` — Current implementation reference: Mermaid diagrams, node configs, credentials matrix
- `workflow.json` — Sanitized n8n workflow export (committed to repo)
- `.mcp.json` — MCP server configuration (n8n, Airtable, Context7)

## Sensitive Data Policy

**This is a public repository.** When updating `workflow.json`, always verify that it does NOT contain:
- Email addresses (use `YOUR_EMAIL@example.com` as placeholder)
- Personal names or identifying information in prompts
- API keys, OAuth tokens, or any credential values

Credential IDs (internal n8n references like `"id": "FarKrMH7V2phy9UE"`) are safe to keep — they are not exploitable without access to the n8n instance.

## Working with the n8n Instance

The workflow is managed via the **n8n MCP tools** (not local files). Key operations:

- List workflows: `mcp__n8n-mcp__n8n_list_workflows`
- Create/update workflow: `mcp__n8n-mcp__n8n_create_workflow` / `mcp__n8n-mcp__n8n_update_full_workflow`
- Validate: `mcp__n8n-mcp__n8n_validate_workflow`
- Test execution: `mcp__n8n-mcp__n8n_test_workflow`
- Search node types: `mcp__n8n-mcp__search_nodes` / `mcp__n8n-mcp__get_node`

**Workflow ID**: `6G7Ceg2OUZZ7ng5n`

## Required Credentials (configured in n8n)

| Credential | Type | Notes |
|---|---|---|
| OpenAI | API Key | gpt-4o-mini |
| SerpAPI | SerpApi native credential | 100 free searches/month — agent uses multiple per run |
| Gmail | OAuth2 Google Account (`Mam OAuth`) | |

## Conventions

- Node names are descriptive, short, in French: `Declencheur 10h00`, `Agent Veille`, `OpenAI Chat Model`, `SerpAPI Tool`, `Envoi Email`
- **Node typeVersions:** scheduleTrigger 1.3, agent 3.1, lmChatOpenAi 1.2, toolSerpApi 1, gmail 2.2
- **IMPORTANT — typeVersion mismatch risk:** The n8n MCP tool registry may report newer typeVersions than what the actual n8n instance supports. Always verify against the running n8n version. Using an unsupported typeVersion causes `"Install this node to use it"` errors in the UI.
- All date calculations use `$now` (Luxon, timezone-aware)

## Future Scope (documented in PRD Section 10)

Airtable integration for offer history/dedup tracking is planned — the Airtable MCP server is already configured.
