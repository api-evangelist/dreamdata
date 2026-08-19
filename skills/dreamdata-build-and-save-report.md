---
name: Build a Dreamdata report from scratch and save it
description: Drive the Dreamdata MCP report builder — discover the analysis types, metrics, filters and time window an account supports, run the assembled config, and save the result to the Analytics Hub under explicit user consent.
api: mcp/dreamdata-mcp.yml
endpoint: https://mcp.dreamdata.io/mcp
grounding: https://developer.dreamdata.io/mcp/mcp-server/
operations:
  - report_builder_start
  - report_builder_list_analysis_types
  - report_builder_list_subjects
  - report_builder_get_components
  - report_builder_resolve_calendar_period
  - report_builder_resolve_rolling_window
  - report_builder_list_property_values
  - report_builder_search_property_values
  - report_builder_run_config
  - report_builder_save_report
generated: '2026-08-13'
method: generated
source: https://developer.dreamdata.io/mcp/mcp-server/ + mcp/dreamdata-mcp.yml
---

# Build a Dreamdata report from scratch and save it

Grounded in the ten `report_builder_*` tools Dreamdata publishes. This is the only flow on the
server that can write anything, so the consent rules below are load-bearing.

## Consent

`report_builder_save_report` requires the **optional** `reports:write` consent ("Save Dreamdata
Reports") granted on the OAuth authorization screen. If the user only granted read, every other
tool here still works — build and run the report, present it, and tell them saving needs the
extra permission and a reconnect. The permission cannot modify or delete existing reports; it
only creates.

## Flow

1. **`report_builder_start`** — opens the guided flow and returns the analysis types this
   account can build on. Always start here; capability is account-specific.
2. **Discover the shape** — `report_builder_list_analysis_types`,
   `report_builder_list_subjects`, `report_builder_get_components` return the analysis types,
   subjects, metrics, filters and segmentation available. Assemble the config from what these
   return, never from what a different Dreamdata account supported.
3. **Resolve the time window** — do not hand-compute dates.
   - `report_builder_resolve_calendar_period` for named periods ("Q1 2025", "last September").
   - `report_builder_resolve_rolling_window` for relative ones ("the last 45 days").
4. **Resolve filter values** — `report_builder_list_property_values` to enumerate,
   `report_builder_search_property_values` to find a specific one. Filter values are strings
   the account actually holds; a guessed value silently returns an empty report rather than an
   error.
5. **Run it** — `report_builder_run_config` validates and executes the assembled config and
   returns results. Show the user the results *and* the config you built.
6. **Save only on confirmation** — `report_builder_save_report`. Dreamdata's own guidance is
   that the agent always asks the user to confirm before anything is saved. Ask, name the
   report, and repeat the config back before calling it.

## Rules of thumb

- Prefer `run_saved_report_for_date_range` over building a new report when the user just wants
  an existing report over a different window (see
  `skills/dreamdata-run-and-drill-reports.md`).
- Do not save a report the user did not ask to keep. An unsaved run costs nothing; a saved
  report clutters someone else's Analytics Hub.
- The builder tools are documented as "behind the scenes" — do not narrate every discovery call
  to the user. Narrate the config you assembled and the window you ran.

## Failure modes

`401` reconnect · `403` not a member of the account queried · `502` Dreamdata-side ·
"MCP server is not enabled for this user" — account owner must enable it. A save attempt
without `reports:write` will fail on consent, not on the config; do not retry it as if it were
a transient error. See `errors/dreamdata-error-codes.yml`.

## Stability

The server is Beta and Dreamdata warns that tools may change, with no changelog. Re-run
`report_builder_start` each session instead of caching an analysis-type list.
