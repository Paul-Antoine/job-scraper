# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Automated n8n workflow for daily job monitoring: scrapes alternance (apprenticeship) offers for "Product Builder No-Code & IA Generative" positions around Paris, summarizes them with AI, and delivers a formatted HTML email every morning at 8:00 AM.

This is a **configuration-and-documentation project** — there is no traditional source code, build system, or test suite. The deliverable is an n8n workflow deployed via the n8n MCP server.

## Architecture

Linear 4-node n8n workflow:

```
Schedule Trigger (8h00) → SerpAPI Google Jobs → OpenAI (summary) → Gmail (send)
```

- **Schedule Trigger**: Daily at 8:00 Europe/Paris
- **SerpAPI Google Jobs** (HTTP Request): API key auth, aggregates Google Jobs results (covers WTTJ, Indeed, LinkedIn, Glassdoor). `continueOnFail=true`.
- **OpenAI** (gpt-4o-mini): Receives raw SerpAPI JSON (`jobs_results[]`), scores relevance 1-10, generates per-offer summaries, identifies top 3, outputs formatted HTML
- **Gmail**: Sends HTML email with date in subject line

## Key Files

- `PRD.md` — Complete specification: node configurations (JSON), OpenAI prompts, API parameters, implementation phases, edge case handling
- `WORKFLOW-DOC.md` — Concise implementation reference: Mermaid diagrams, data structure examples, credentials matrix, production checklist
- `.mcp.json` — MCP server configuration (n8n, Airtable, Context7)

## Working with the n8n Instance

The workflow is managed via the **n8n MCP tools** (not local files). Key operations:

- List workflows: `mcp__n8n-mcp__n8n_list_workflows`
- Create/update workflow: `mcp__n8n-mcp__n8n_create_workflow` / `mcp__n8n-mcp__n8n_update_full_workflow`
- Validate: `mcp__n8n-mcp__n8n_validate_workflow`
- Test execution: `mcp__n8n-mcp__n8n_test_workflow`
- Search node types: `mcp__n8n-mcp__search_nodes` / `mcp__n8n-mcp__get_node`

The n8n instance URL should be configured in `.mcp.json` (not committed).

## Required Credentials (configured in n8n)

| Credential | Type | Notes |
|---|---|---|
| SerpAPI | API key in URL query param | 100 free searches/month |
| OpenAI | API Key | Uses gpt-4o-mini |
| Gmail | OAuth2 Google Account | |

## n8n Environment Variables

These must be set in n8n Settings > Variables or as server-level env vars (community edition has no UI for Variables):

- `SERPAPI_KEY`
- `EMAIL_DESTINATAIRE`

## Conventions

- Node names are descriptive, short, in French: `Declencheur 8h00`, `SerpAPI Google Jobs`, `Resume IA`, `Envoi Email`
- **Node typeVersions (verified on n8n 1.83.2):** scheduleTrigger 1.3, httpRequest **4.2**, openAi 1.1, gmail 2.2
- **IMPORTANT — typeVersion mismatch risk:** The n8n MCP tool registry may report newer typeVersions (e.g. 4.4) than what the actual n8n instance supports. Always verify against the running n8n version. Using an unsupported typeVersion causes `"Install this node to use it"` errors in the UI. When in doubt, check official n8n templates at n8n.io or the instance's node list.
- Descriptions truncated to 500 chars to reduce OpenAI token usage
- All date calculations use `$now` (Luxon, timezone-aware)

## Future Scope (documented in PRD Section 10)

Airtable integration for offer history/dedup tracking is planned — the Airtable MCP server is already configured.
