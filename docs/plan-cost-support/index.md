---
layout: docs
title: "Plan, cost and support"
description: "Telemetry, the trial and plan, running cost, uninstalling and support."
permalink: /docs/plan-cost-support/
sidebar: docs
prev_url: /docs/sql-reference/
prev_title: "SQL reference"
---
{% raw %}
# Plan, cost and support {#plan-cost-and-support}

## Telemetry {#telemetry}

The app shares **error and warning logs** with the provider (mandatory) so we can detect and fix
failures. Snowflake adds your organisation and account name, account locator, region and the app
version to each event. A message names the procedure and the problem; a rejected change usually names
what it was about - the code set, mapping set, hierarchy or managed table, the code or row key, the
attribute or column, and sometimes the value that was refused or the allowed values (for example
"currency: GBP is not a valid CURRENCY code"). An unexpected error includes Snowflake's error message,
which can quote a value involved, and a stack trace of the app's code. The app never logs whole rows,
pasted rows or uploaded files, comments, email addresses, or your users' names (a message that names stewards in the
console is logged with only their number). If your codes or row keys are themselves personal data (for
example employee numbers), they can appear in these messages. **INFO usage logs** are optional and off
by default: per call, the time taken and the number of statements, and per review the number of changes
approved, rejected, failed and skipped. Event sharing
requires an active event table in your account.
The app doesn't request traces or agent telemetry, so questions asked of its Cortex Agent or MCP server
and their answers stay in your account. Privacy notice:
https://www.lodestardataservices.com/privacy/

## Trial and plan {#trial-and-plan}

The free trial is the full app for 30 days. Snowflake emails you before it ends and suspends the
app at the end unless you buy a plan; when you buy, the same install continues with all your data.
If the app is suspended, your data is kept as long as you don't uninstall it.

The standard plan includes **10 editors**. An editor is anyone who proposed, approved, rejected,
reverted or imported a change, or changed the app's set-up, in the last 30 days
(`core.active_editors`). People who only read the published views or browse the Viewer console
don't count, so viewers and SQL users are unlimited. Commenting and asking for mapping suggestions don't
count either.

The Admin console's home page shows editors used against the plan. Going over doesn't block
anyone: the page shows a notice, and the app tells us through its warning logs (the number of
editors only, never names). We'll contact you about a larger plan; you can also ask for one through
the listing.

## Cost {#cost}

Apart from any listing price, the app only uses Snowflake compute and storage in your account:

| What | Runs on | Billed as |
|---|---|---|
| The consoles (Streamlit) and the procedures they call | The app's warehouse `<APP_NAME>_WH` (X-Small Gen1, auto-suspend 60 s) | Credits of that warehouse, while someone is using a console |
| Procedures you `CALL` from SQL (jobs, scripts) | The warehouse of your session | Your warehouse |
| Queries on the published views / functions (pipelines, BI) | The warehouse of your session | Your warehouse |
| The app's tables (codes, history, audit log) | - | Storage, typically a few MB |
| AI mapping suggestions (only if an Admin turns them on) | Cortex `AI_EMBED` on the app's warehouse | Cortex AI usage: about 10 tokens per code, each target code embedded once - see [Mapping suggestions](/docs/mappings/#mapping-suggestions) |
| Questions to the agent or the MCP server's Cortex Analyst tool | Cortex Agents / Cortex Analyst, with their queries on the app's warehouse | Cortex AI usage per question, plus that warehouse - see [AI agents and MCP](/docs/ai-and-mcp/#ai-agents-and-mcp) |
| Email notifications (only if an Admin turns them on) | Sent within the submission or review that triggers them, on the same warehouse | That warehouse's time for the extra statements |
| Scheduled checks (only once an Admin schedules one) | The app's task, once a day, on the app's warehouse | That warehouse: it resumes for the run (billed for at least 60 s, about 0.017 credits on X-Small), plus a few seconds per check - about 0.5 credits a month for a handful of checks. Suspended while nothing is scheduled |

Each action in a console takes a few seconds of warehouse time; a code set of 50,000 codes is
imported in about 30 s. To see what the app itself costs:

```sql
-- Daily credits and storage of the app (includes the warehouse it owns; up to 1 day latency)
SELECT usage_date, credits_used, credits_used_breakdown, storage_bytes
FROM SNOWFLAKE.ACCOUNT_USAGE.APPLICATION_DAILY_USAGE_HISTORY
WHERE application_name = '<APP>'
ORDER BY usage_date DESC;
```

To be alerted if its spending changes, add the app to a custom budget (Snowsight -> Admin -> Cost
management -> Budgets).

## Uninstall {#uninstall}

`DROP APPLICATION <app> CASCADE;` removes the app, all of its data and the account-level objects
it owns (its warehouse). Export anything you need first, e.g.
`CREATE TABLE my_db.my_schema.lds_codes AS SELECT * FROM <app>.core.code_values_history;`

## Support {#support}

E-mail `support@lodestardataservices.com` - we reply within 1 business day. Please include the app's version
(`SHOW APPLICATIONS LIKE '<app>';` - the `version`, `patch` and `label` columns) and, for an error, the
message the console showed. Documentation: https://www.lodestardataservices.com/docs/
{% endraw %}
