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
  the last 30 days, recent runs.
- **Requests** (Officer): *New request* - case reference (yours, never the person's name), received date,
  legal basis and the identifiers, one per line as `KIND: value`, e.g. `EMAIL: ann@example.com`.
  *Upload many* - a CSV or Excel file with the columns `reference, kind, value, received_on, legal_basis, note`,
  one identifier per row; rows with the same reference are one request; all or nothing. *Open requests* - the
  identifiers, the tables and their counts; count again, withdraw; Admins retry a partly failed request or
  close it as incomplete.
- **Approvals** (Officer): requests waiting for a second officer, with the identifiers and the rows each table
  holds; approve, or reject with a reason.
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
- **The run** handles each table in one transaction: it counts each approved request's rows, runs one
  `DELETE` (or `UPDATE ... SET <columns> = NULL / '[erased]'`) for all of them, and checks the statement
  changed exactly the rows counted. If anything differs, or the table changed (a column renamed, the table
  dropped), nothing in that table is changed, that part of the request is marked failed with the reason, and
  the other tables still run. An Admin fixes the cause and retries, or closes the request as incomplete.
- **Holds** are checked first: a held part waits (ON_HOLD) and runs at the first run after its hold is
  released.
- **Status**: SUBMITTED -> APPROVED -> COMPLETED; or REJECTED / WITHDRAWN (before anything ran); ON_HOLD
  and PARTIAL while it can't finish; CLOSED_INCOMPLETE when an Admin closes it unfinished.
- **Identifiers are deleted** from the app when a request closes, whatever the outcome. The table that held
  them is transient with no Time Travel, so they can't be recovered from the app afterwards.

## Evidence {#evidence}

For each request and table the evidence records the action, the rows affected, when it ran, the Snowflake
query id of the `DELETE` / `UPDATE` (look it up in your query history), the table's declared Time Travel days
and kind, and:

- **recoverable until** - the end of Time Travel for the change: until then, anyone with access to the
  table can still read or restore the old rows (`AT` / `BEFORE`, `UNDROP`);
- **fail-safe until** - Time Travel plus 7 days for permanent tables (none for transient): until then only
  Snowflake Support can recover the data, for disaster recovery. After it, nobody can.

The report also lists the legal holds that applied and the request's trail (submitted, approved, erased,
completed - by whom and when), and states what is **not** covered: copies outside the registered tables -
clones, replicas, data shared to other accounts, stages and files, backups and exports, logs - and any table
not registered in the app. Lower a table's Time Travel days if your policy needs erased data to become
unrecoverable sooner.

## Sizing and limits {#sizing-and-limits}

- Up to 200 active targets, 10 identifying columns per target and 20 identifiers per request.
- Up to 500 requests per upload; each run erases up to 200 requests per table (the rest run the next night).
- Matching normalises your columns (lower case, digits only, trimmed), so every count and every run reads the
  whole target table - Snowflake can't skip parts of it. Very large tables take longer and use more of the
  app's warehouse time; there is no row limit.
{% endraw %}
