---
layout: docs
title: "Using the app"
description: "The consoles, how a person is matched and erased, the evidence, and sizing and limits."
permalink: /lodestar-privacy/docs/using-the-app/
sidebar: privacy_docs
product: "Lodestar Privacy"
prev_url: /lodestar-privacy/docs/getting-started/
prev_title: "Install and set up"
next_url: /lodestar-privacy/docs/sql-reference/
next_title: "SQL reference"
---
{% raw %}
# Using the app {#using-the-app}


Three consoles, one per role (Snowsight -> Catalog -> Apps -> the app): **Lodestar Privacy** (everyone),
**Lodestar Privacy - Officer** and **Lodestar Privacy - Admin**.

- **Overview**: open requests by deadline (overdue first), requests on hold, completions and rows erased in
  the last 30 days, recent runs - and a warning when approved requests have waited more than a day for a run
  that finished, when tables wait to be approved again, or when a table failed.
- **Requests** (Officer): *New request* - case reference (yours, never the person's name), received date,
  legal basis and the identifiers, one per line as `KIND: value`, e.g. `EMAIL: ann@example.com`.
  *Upload many* - a CSV or Excel file with the columns `reference, kind, value, received_on, legal_basis, note`,
  one identifier per row; rows with the same reference are one request; all or nothing. *Open requests* - the
  identifiers, the tables and their counts; count again, withdraw; Admins retry a partly failed request or
  close it as incomplete.
- **Approvals** (Officer): requests waiting for a second officer, with the identifiers and the rows each table
  holds; approve, or reject with a reason. Above them, approved requests with tables to approve again (added,
  or changed since the approval).
- **Evidence**: every request with its tables, rows, query ids, recovery dates and trail; download the report
  as PDF, CSV or JSON; export all evidence.
- **Legal holds**: active and released holds; Admins place one on a person (identified like a request), a
  table or everything, with a reason and reference, and release it with a note.
- **Targets** (Admin): add tables, register or change how a person is found and erased in each, retire one.
- **Settings & audit** (Admin): self-approval, the default deadline and reminder days, the daily run and
  **Run erasures now**, email notifications, and the audit log.

Notes are kept with the evidence after the identifiers are deleted, so never put personal data in a note,
reason or reference.

## How matching and erasure work {#how-matching-and-erasure-work}

- **Identifiers** have a kind and a value. Email addresses are compared in lower case, phone numbers by
  their digits only, everything else exactly (spaces, tabs and line breaks at either end are ignored) - in
  the request and in your columns alike, so `Ann@Example.com ` matches `ann@example.com`.
- **A row belongs to a person** when any of the target's identifying columns equals one of the person's
  identifiers of that column's kind. Identifiers of a kind no target uses match nothing (the request page
  warns you).
- **The run** - one at a time: the daily task and *Run now* share a lock, so a second run started meanwhile is
  refused. For each table it first claims the parts it will erase (a withdrawal or closure waits for it, or is
  refused while a table is being erased), then in one transaction counts each approved request's rows (one
  scan for all of them), runs one `DELETE` (or `UPDATE ... SET <columns> = NULL / '[erased]'`) for all of them,
  checks the statement changed exactly the rows counted, and records the evidence, an attempt row and the
  audit entries. If anything differs, or the table changed (a column renamed, the table dropped), nothing in
  that table is changed, that part of the request is marked failed with the reason, and the other tables still
  run. An Admin fixes the cause and retries - each retry is a new attempt; the evidence of earlier ones is
  kept - or closes the request as incomplete. A run that dies (e.g. the warehouse is stopped) leaves nothing
  half done: the next run takes over its lock after 30 minutes without a heartbeat and runs what it left.
- **What an approval covers**: each table as it was defined when the request was approved (its revision). If
  an Admin later changes the table's source, identifying columns or action - or registers a new table - that
  part waits as *needs approval* until an officer approves the request's new scope; the time travel days,
  table kind and description don't count as changes.
- **Holds** are checked first, and again before each table: a hold placed during a run applies to every table
  not yet erased. A hold on a person stops any request whose rows carry one of the hold's identifiers - in any
  identifying column - and, before each run, the app follows the person's other identifiers on those rows (e.g.
  the customer id on their CRM row) to other tables, up to three tables away. A held part waits (ON_HOLD) and
  runs at the first run after its hold is released. Limits: a hold can only follow identifiers that appear on
  registered tables; an identifier several people share (a family phone number) holds all of them - the safe
  side.
- **Status**: SUBMITTED -> APPROVED -> COMPLETED; or REJECTED / WITHDRAWN (before anything ran); ON_HOLD
  and PARTIAL while it can't finish; CLOSED_INCOMPLETE when an Admin closes it unfinished. A table's part is
  PENDING, RUNNING (being erased), HELD, NEEDS_REVIEW (table added or changed after the approval), DONE,
  FAILED or SKIPPED (the request was withdrawn or closed, or the table retired, before it ran).
- **Identifiers are deleted** from the app when a request closes, whatever the outcome. The table that held
  them is transient with no Time Travel, so they can't be recovered from the app afterwards - also when your
  account sets a minimum Time Travel (`MIN_DATA_RETENTION_TIME_IN_DAYS`): measured on 25 Nov 2026 with a
  minimum of 1 and 7 days, a transient table at 0 stays at 0 (a deleted row can't be read back) and the app's
  identifier table reports retention 0 and no Time Travel bytes. The app's own tables aren't granted to any
  role, so only the app could read them.

## Evidence {#evidence}

For each request and table the evidence records the action, the rows affected, when it ran, the Snowflake
query id of the `DELETE` / `UPDATE`, the table, action and definition (revision) it actually ran against, the
table's declared Time Travel days and kind, and:

- **recoverable until** - the end of Time Travel for the change: until then, anyone with access to the
  table can still read or restore the old rows (`AT` / `BEFORE`, `UNDROP`);
- **fail-safe until** - Time Travel plus 7 days for permanent tables (none for transient): until then only
  Snowflake Support can recover the data, for disaster recovery. After it, nobody can.

The report opens with a summary of how many tables were erased, skipped, failed, held or still waiting - only
tables marked DONE were erased for the request - and lists every attempt (a retry adds one; none is ever
changed), the legal holds that applied and the request's trail (submitted, approved, erased, completed - by
whom and when). It states what is **not** covered: copies outside the registered tables -
clones, replicas, data shared to other accounts, stages and files, backups and exports, logs - and any table
not registered in the app. Lower a table's Time Travel days if your policy needs erased data to become
unrecoverable sooner (an account minimum, `MIN_DATA_RETENTION_TIME_IN_DAYS`, overrides lower settings of
permanent tables; measured: not of a transient table set to 0).

**Checking a query id.** An account administrator can find each statement in Snowflake's query history, e.g.
`SELECT query_type, user_name, role_name, rows_deleted, rows_updated FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE query_id = '<query id>';` (up to 45 minutes' delay; `TABLE(<db>.INFORMATION_SCHEMA.QUERY_HISTORY())` has
none). It shows a DELETE or UPDATE with the rows it changed, run as the app's role. Snowflake hides the text of
an app's own statements, so no identifier shows there either.

## Sizing and limits {#sizing-and-limits}

- Up to 200 active targets, 10 identifying columns per target and 20 identifiers per request.
- Up to 500 requests per upload; each run erases up to 200 requests per table (the rest run the next night).
- Matching normalises your columns (lower case, digits only, trimmed), so every count and every run reads the
  whole target table - Snowflake can't skip parts of it. A run reads each table once for all the requests
  waiting (plus, while a legal hold on a person is active, a few more reads per table to follow the person's
  linked identifiers); there is no row limit.
- Measured (24 Nov 2026, the app's X-Small warehouse, two tables of 1,000,000 rows, 200 approved requests with
  overlapping identifiers): the run took **36 seconds** (63 statements) and **49 seconds** with a legal hold on
  a person active; counting one request in both tables (preview) about 12 seconds; each approval about 11
  seconds (mostly Snowflake's start-up time for a procedure call). Larger tables take proportionally longer.
{% endraw %}
