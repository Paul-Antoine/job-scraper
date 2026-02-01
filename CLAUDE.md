# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Automated n8n workflow for daily job monitoring: scrapes alternance (apprenticeship) offers for "Product Builder No-Code & IA Generative" positions around Paris, summarizes them with AI, and delivers a formatted HTML email every morning at 8:00 AM.

This is a **configuration-and-documentation project** — there is no traditional source code, build system, or test suite. The deliverable is an n8n workflow deployed via the n8n MCP server.

## Architecture

Linear 6-node n8n workflow:

```
Schedule Trigger (8h00) → France Travail API → SerpAPI Google Jobs → Code (merge/dedup) → OpenAI (summary) → Gmail (send)
```

- **Schedule Trigger**: Daily at 8:00 Europe/Paris
- **France Travail API** (HTTP Request): OAuth2 Client Credentials, queries official government employment API for apprenticeship contracts (E2/FS) within 30km of Paris, last 7 days
- **SerpAPI Google Jobs** (HTTP Request): API key auth, aggregates Google Jobs results (covers WTTJ, Indeed, LinkedIn, Glassdoor)
- **Code Node**: JavaScript that normalizes both API response formats, deduplicates by title+company, sorts by date, outputs unified JSON
- **OpenAI** (gpt-4o-mini): Scores relevance 1-10, generates per-offer summaries, identifies top 3, outputs formatted HTML
- **Gmail**: Sends HTML email with date and offer count in subject line

Both HTTP nodes have `continueOnFail=true` so one source failing doesn't break the workflow.

## Key Files

- `PRD.md` — Complete specification: node configurations (JSON), JavaScript code for the Code node, OpenAI prompts, API parameters, implementation phases, edge case handling
- `WORKFLOW-DOC.md` — Concise implementation reference: Mermaid diagrams, data structure examples, credentials matrix, production checklist
- `.mcp.json` — MCP server configuration (n8n, Airtable, Context7)

## Working with the n8n Instance

The workflow is managed via the **n8n MCP tools** (not local files). Key operations:

- List workflows: `mcp__n8n-mcp__n8n_list_workflows`
- Create/update workflow: `mcp__n8n-mcp__n8n_create_workflow` / `mcp__n8n-mcp__n8n_update_full_workflow`
- Validate: `mcp__n8n-mcp__n8n_validate_workflow`
- Test execution: `mcp__n8n-mcp__n8n_test_workflow`
- Search node types: `mcp__n8n-mcp__search_nodes` / `mcp__n8n-mcp__get_node`

The n8n instance is hosted at `https://n8n.srv1263730.hstgr.cloud`.

## Required Credentials (configured in n8n)

| Credential | Type | Notes |
|---|---|---|
| France Travail | OAuth2 API (Client Credentials) | Token URL includes `?realm=/partenaire`, scope: `api_offresdemploiv2 o2dsoffre` |
| SerpAPI | API key in URL query param | 100 free searches/month |
| OpenAI | API Key | Uses gpt-4o-mini |
| Gmail | OAuth2 Google Account | |

## n8n Environment Variables

These must be set in n8n Settings > Variables (not in local `.env`):

- `FRANCE_TRAVAIL_CLIENT_ID` / `FRANCE_TRAVAIL_CLIENT_SECRET`
- `SERPAPI_KEY`
- `EMAIL_DESTINATAIRE`

## Conventions

- Node names are descriptive, short, in French: `Declencheur 8h00`, `France Travail API`, `Merge Dedup Format`, `Resume IA`, `Envoi Email`
- Use `$('NodeName')` references in the Code node to access upstream data (no Merge node needed)
- Always use latest stable typeVersions: scheduleTrigger 1.3, httpRequest 4.4, code 2, openAi 1.1, gmail 2.2
- Descriptions truncated to 500 chars to reduce OpenAI token usage
- All date calculations use `$now` (Luxon, timezone-aware)

## Future Scope (documented in PRD Section 10)

Airtable integration for offer history/dedup tracking is planned — the Airtable MCP server is already configured.
