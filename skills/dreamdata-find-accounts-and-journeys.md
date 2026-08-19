---
name: Find in-market accounts and read their Dreamdata journey
description: Use the Dreamdata MCP server to turn a plain-language ICP question into an audience query, then read the full customer journey and pipeline stages of the companies it returns.
api: mcp/dreamdata-mcp.yml
endpoint: https://mcp.dreamdata.io/mcp
grounding: https://developer.dreamdata.io/mcp/mcp-server/
operations:
  - list_my_accounts
  - list_audience_filter_properties
  - list_audience_filter_values
  - describe_audience_config_schema
  - run_audience_query
  - list_audiences
  - get_audience_definition
  - search_companies
  - get_company_journey
  - list_company_journey_stages
generated: '2026-08-13'
method: generated
source: >-
  https://developer.dreamdata.io/mcp/mcp-server/ + mcp/dreamdata-mcp.yml +
  data-model/dreamdata-data-model.yml
---

# Find in-market accounts and read their Dreamdata journey

Grounded in the published Dreamdata MCP tool catalog. Connect first — see
`skills/dreamdata-run-and-drill-reports.md` for the OAuth 2.1 + S256 PKCE connection steps.
Read consent is sufficient for everything below.

## The shape of the job

Two halves, in order: **locate** the companies, then **explain** them.

### Locate

1. If the user may have several Dreamdata accounts, `list_my_accounts` and confirm which one.
2. Check for an existing audience before building one. `list_audiences` browses saved
   audiences; `get_audience_definition` reuses one ("who's in my 'ICP – Engaged' audience
   right now?"). Reusing the customer's own definition beats reconstructing their ICP.
3. Otherwise build the query. The discovery tools exist because filters are account-specific:
   - `list_audience_filter_properties` — what this account can filter on (company, contact and
     event properties, plus pipeline stages per revenue model).
   - `list_audience_filter_values` — the exact values a filter accepts. Use it; do not guess a
     value string.
   - `describe_audience_config_schema` — the full structure for more complex definitions.
4. `run_audience_query` returns the matching companies or contacts. Audience criteria combine
   firmographics, behaviour (events and signals) and pipeline stages with a time window — so a
   question like "companies that reached SQL in the last 90 days in Europe" or "companies that
   visited our pricing page at least 3 times this month" is one query, not three.

### Explain

5. `search_companies` looks a company up by name (use it when the user names one directly).
6. `get_company_journey` returns the full journey — summary, contacts, sessions and
   touchpoints.
7. `list_company_journey_stages` returns which pipeline stages the company reached, when, and
   at what value. Use it for "when did they become an SQL and what was it worth".

Comparing several companies is a loop over 6 and 7, not a special tool.

## Reading the answers correctly

The underlying warehouse model (`data-model/dreamdata-data-model.yml`) has two properties that
change how you should phrase results:

- **A contact can belong to several companies**, and an event is written once per company copy.
  Do not add up per-company numbers into a total occurrence count.
- **Exposures are not visits.** Impression-style touchpoints (e.g. LinkedIn ad impressions)
  carry a quantity greater than 1 and are deliberately excluded from session counts. If you
  report "sessions", say whether ad exposure is in or out.

Stage values are revenue-bearing (`stages.value`) — quote the currency the account is
configured in, and never present a stage value as closed revenue unless the stage says so.

## Failure modes

`401` reconnect · `403` not a member of that account — run `list_my_accounts` · `502`
Dreamdata-side, stop retrying · "MCP server is not enabled for this user" — account owner must
enable it. Full table in `errors/dreamdata-error-codes.yml`.

## Boundaries

This is a read surface. There is no tool to create, modify or delete an audience, a company or
a stage, and the only write tool on the whole server is `report_builder_save_report`, which
needs the separate optional consent. If the user asks you to change data in Dreamdata, say it
cannot be done through this server.
