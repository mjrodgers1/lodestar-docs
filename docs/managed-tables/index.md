---
layout: docs
title: "Managed tables"
description: "Govern edits to your own tables where they live."
permalink: /docs/managed-tables/
sidebar: docs
prev_url: /docs/mappings/
prev_title: "Mapping suggestions"
next_url: /docs/ai-and-mcp/
next_title: "AI agents and MCP"
---
{% raw %}
# Managed tables {#managed-tables}


Some tables should stay where they are - read by many jobs, with their own columns and types - but be
changed only with approval and audit: price lists, thresholds, product or customer attributes, ETL
control tables. Make them **managed tables**: Stewards' edits become change requests under the same
rules as reference data, and approved changes are written to your table.

**Set up (Admin).** On **Managed tables -> Set up**, choose **Add a table** (or use the SQL in
[step 4](/docs/getting-started/#4-optional-manage-your-own-tables)), then register it with a name, its **key columns** (one or more - together they must
identify every row) and its **editable columns**. Optionally name the columns the app fills when it writes a change
(`created_by`, `created_at`, `updated_by`, `updated_at` - by = the Steward who proposed it), and allow
Stewards to add or delete rows. Approval policy, named stewards and retirement work as for code sets,
with container type `MANAGED_TABLE`. Registration checks the table as it is: the key must have a
value in every row and no duplicates, and if new rows are allowed every NOT NULL column without a
default must be a key or editable column.

```sql
CALL <app>.core.register_managed_table(
  'PRICE_LIST', 'SALES_DB.REF.PRICE_LIST',
  ARRAY_CONSTRUCT('REGION', 'SKU'),                    -- key
  ARRAY_CONSTRUCT('PRICE', 'CURRENCY', 'VALID_UNTIL'), -- editable
  PARSE_JSON('{"updated_by": "UPDATED_BY", "updated_at": "UPDATED_AT"}'),
  TRUE,    -- Stewards may add rows
  FALSE,   -- ... but not delete them
  'List prices by region');
CALL <app>.core.set_approval_policy('MANAGED_TABLE', 'PRICE_LIST', 'REQUIRED');
```

**Edit (Steward).** On **Managed tables -> Edit rows**, search, edit the grid and submit; the grid loads up
to 1,000 rows at a time in key order. From SQL, read rows with `core.managed_rows` (key order, up to
5,000 at a time, optional search) and submit changes with `core.submit_row_changes`:

```sql
CALL <app>.core.managed_rows('PRICE_LIST', 'emea');   -- search key and editable columns
CALL <app>.core.submit_row_changes('PRICE_LIST', PARSE_JSON('[
  {"action": "UPDATE", "key": {"REGION": "EMEA", "SKU": 1001}, "values": {"PRICE": "12.50"},
   "row_hash": "<_LODESTAR_ROW_HASH from managed_rows>"},
  {"action": "INSERT", "key": {"REGION": "EMEA", "SKU": 1002}, "values": {"PRICE": "9.99", "CURRENCY": "EUR"}},
  {"action": "DELETE", "key": {"REGION": "APAC", "SKU": 1001}}]'), 'Q3 price review', 'CHG0042');
```

Values are given as text (or JSON for VARIANT, OBJECT and ARRAY columns) and checked against the
column's type, length and NOT NULL rule on submit; dates, times and timestamps keep full precision.
Columns not given on an insert get the table's default. Each change records the row's values before
the change, so reviewers see exactly what changes.

**No lost updates.** A change is applied only if the row's key and editable columns are still as they
were when the change was submitted (and, with `row_hash`, as they were when the Steward loaded it).
If anything - another approved change, a job, someone in a worksheet - changed the row in between,
the change fails with a clear message instead of overwriting it; columns the app doesn't manage may
change freely. A row can have only one pending change at a time. In the console, the grid remembers each
row as you loaded it, so **Reload rows** before editing again if others may have changed the table.
Times, timestamps and JSON are edited as text and kept to the nanosecond; TIMESTAMP_TZ / LTZ values are
shown in the session's time zone (the same instant), and a changed value keeps the offset you type.

**History and revert.** `core.managed_field_history` lists every applied change, one row per column,
with old and new values, who proposed and approved it, the reason and the ticket. **Revert** (or
`core.revert_change`) undoes an applied change - deleting an added row, restoring changed values or
re-adding a deleted row - provided the row hasn't changed since; the revert goes through approval like
any change. A deleted row comes back whole: every column as it was when it was deleted, including columns
Lodestar doesn't manage and the original created_* stamps (updated_* record who undid it and when). Rows
deleted before Lodestar 1.2.0 come back with their key and editable columns only, and a table with columns
of a type Lodestar can't write (e.g. BINARY, GEOGRAPHY) can't have a row restored - use Time Travel. Undoing
an added row is refused if any of its columns changed since, not only the ones Lodestar manages. Reverting an insert or delete works even if Stewards may not add or delete rows.

**Security.** The app writes only approved changes, only to the registered key, editable and audit
columns, and only to tables you added; it never reads or writes other tables. Anyone with the
`steward` role can read the rows of every managed table through the app (as they can read the change
requests), so register only tables those users may see. Changes made directly to the table by roles
that can still write to it are not governed: revoke INSERT, UPDATE and DELETE from those roles if all
changes must go through the app. Remove a table with `register_reference('MANAGED_TABLES', 'REMOVE',
'<alias>')` (the alias is listed by `CALL <app>.core.managed_sources()`); its registration and history
stay, and changes to it are refused until it is added again.

**Limits.** Up to 50,000 row changes per submission. Supported column types: text, numbers, boolean,
date, time, timestamps, VARIANT, OBJECT and ARRAY (key columns: not VARIANT, OBJECT or ARRAY); other
columns (binary, geography, vector, ...) are shown but can't be keys or edited. Column names containing
`'` or `?` aren't supported. Tables in shared databases are read-only, so they can't be managed. With a
20,000-row table on an X-Small warehouse: reading 1,000 rows (`managed_rows`) about 2 s; submitting a
change that applies at once about 12 s, 1,000 changes about 14 s; registering a table about 9 s.
{% endraw %}
