---
name: Run and drill into Dreamdata Analytics Hub reports
description: Connect to the Dreamdata MCP server and run saved Analytics Hub reports, re-run them over a custom date range, and drill into the accounts behind a metric.
api: mcp/dreamdata-mcp.yml
endpoint: https://mcp.dreamdata.io/mcp
grounding: https://developer.dreamdata.io/mcp/mcp-server/
operations:
  - list_my_accounts
  - list_saved_reports
  - run_saved_report
  - run_saved_report_for_date_range
  - list_report_metric_drilldown_entities
generated: '2026-08-13'
method: generated
source: https://developer.dreamdata.io/mcp/mcp-server/ + mcp/dreamdata-mcp.yml
---

# Run and drill into Dreamdata Analytics Hub reports

Grounded in the tool names Dreamdata publishes on its MCP server page. Every tool named here
appears verbatim in that catalog — nothing is invented. The live `tools/list` is auth-gated
(401 anonymously), so treat the input shapes as discovered at runtime, not assumed.

## Connect

Remote HTTP MCP server, one URL for every account:

```
https://mcp.dreamdata.io/mcp
```

- `claude mcp add --transport http dreamdata https://mcp.dreamdata.io/mcp`
- OAuth 2.1 authorization code with **SHA-256 PKCE is mandatory**. A client without S256 PKCE
  cannot connect.
- Two consents are presented: **Read Dreamdata Resources** (required) and **Save Dreamdata
  Reports** (optional, `reports:write`). This skill needs only the read consent.

## Steps

1. **Resolve the account.** If the user may belong to more than one Dreamdata account, call
   `list_my_accounts` first and confirm which one the question is about. With a single account
   you are connected automatically and can skip this.
2. **Find the report.** `list_saved_reports` browses the Analytics Hub catalog. Match on the
   user's words ("my SQL reports", "the pipeline one") rather than asking them for an id.
3. **Run it.** `run_saved_report` returns the saved report as configured. If the user asked for
   a different window ("but just for last month"), use
   `run_saved_report_for_date_range` instead — it does **not** modify the saved report, which
   is the whole reason it exists. Never mutate a saved report to change a date range.
4. **Drill down.** When the user asks who is behind a number ("which companies are in the SQL
   column?"), call `list_report_metric_drilldown_entities` against that metric value. It
   returns the contacts, companies, stages or deals behind it.
5. **Present it.** Report results are tabular; render them and state the date window you
   actually ran, especially after step 3's custom range.

## Failure modes

| Signal | Meaning | Do this |
|---|---|---|
| `401` | Session expired / token invalid | Tell the user to disconnect and reconnect the connector |
| `403` | Authenticated but not a member of the account queried | Run `list_my_accounts`; else the account owner must grant access |
| `502` | Dreamdata internal API unreachable | Server-side; stop retrying, point at mcp-feedback@dreamdata.io |
| "MCP server is not enabled for this user" | Account entitlement missing | The Dreamdata account owner must enable it |

## Caveats worth saying out loud

The server is labelled **Beta** by Dreamdata and its own docs warn that "tools may change".
There is no published changelog where such a change would be announced
(`changelog/dreamdata-changelog.yml`), so re-discover the tool list rather than caching it
across sessions.
