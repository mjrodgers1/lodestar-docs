---
layout: docs
title: "Working with your existing tables"
description: "Lookups, translations, data-quality checks and taking existing tables under control."
permalink: /docs/existing-tables/
sidebar: docs
prev_url: /docs/ai-and-mcp/
prev_title: "AI agents and MCP"
next_url: /docs/sql-reference/
next_title: "SQL reference"
---
{% raw %}
# Working with your existing tables {#working-with-your-existing-tables}


The app keeps the governed copy of your reference data in its own tables. Your databases use it
through the app's views, so there is a single source of truth and nothing to copy or sync. There are
four common patterns and one migration step.

### Pattern 1 - Look up and standardise {#pattern-1---look-up-and-standardise}

Join your tables to `core.code_values_current` to add governed descriptions and attributes. The
**Use in SQL** panel on the Code Values, Mappings and Hierarchies pages generates a starting view.

### Pattern 2 - Translate source-system codes {#pattern-2---translate-source-system-codes}

When a source system uses its own codes (e.g. numeric country keys, `"Eire"`, `"IRL"`), create a
mapping set from that system to the governed code set and join through `core.mappings_current`:

```sql
CREATE OR REPLACE VIEW my_db.my_schema.customer_std AS
SELECT c.customer_id, c.customer_name,
       c.nation_code                  AS source_nation_code,
       m.target_code                  AS country_code,
       cv.description                 AS country_name,
       cv.attributes:region::VARCHAR  AS region
FROM my_db.my_schema.customer c
LEFT JOIN <app>.core.mappings_current m
       ON m.mapping_set = 'CRM_COUNTRY' AND m.source_code = TRIM(c.nation_code::VARCHAR)
LEFT JOIN <app>.core.code_values_current cv
       ON cv.code_set = 'COUNTRY' AND cv.code = m.target_code;
```

`TRIM(...::VARCHAR)` compares the source value as text without surrounding spaces - exactly as **Check a
column** does - so a padded `'IE '` or a numeric key still finds its mapping. Case still matters.

### Pattern 3 - Check data quality {#pattern-3---check-data-quality}

Rows whose codes are not governed show up as NULLs in the joins above. List them in a view and use it
in your monitoring:

```sql
CREATE OR REPLACE VIEW my_db.my_schema.customer_dq AS
SELECT customer_id, 'NATION_CODE' AS field, source_nation_code AS value, 'No CRM_COUNTRY mapping' AS issue
FROM my_db.my_schema.customer_std
WHERE country_code IS NULL;
```

Once a Steward adds the missing mapping or code in the app, the rows disappear from the check.
No reload is needed.

**Before you build the joins**, or to see the gaps without writing SQL, add the table as an import source
(Bulk Import) and use **Check a column** on the Code Values or Mappings page: pick the table and the column,
and the app lists each value that isn't a code of the code set (or isn't mapped by the mapping set) today,
most rows first, with the reason:

| Status | Meaning |
|---|---|
| Not a code / Not mapped | The app has never had this value |
| Valid only from a later date | It has a version, but that starts after today |
| No longer valid | It had a version, but that has ended (e.g. a retired code) |
| Change waiting for approval | A change for it has been submitted and not yet approved |
| Blank | Empty or NULL |

Values are compared as text after trimming spaces, as the `TRIM(...::VARCHAR)` joins above do (`ie` is not
`IE`).
The check reads the column once on the app's warehouse and changes nothing; the first 1,000 values that
aren't OK are listed, and the counts cover all of them. On a mapping set, **Suggest** with the same table and
column proposes targets for the values that aren't mapped. From SQL:

```sql
CALL <app>.core.check_values('MAPPING_SET', 'CRM_COUNTRY', 'MY_DB.MY_SCHEMA.CUSTOMER', 'NATION_CODE');
```

To keep watching a column, an Admin schedules the check ([step 11](/docs/getting-started/#11-optional-schedule-column-checks)): it
runs daily or weekly, and only values that
become problems are emailed. With more than 1,000 problem values, "new" is judged on the 1,000 with the
most rows.

### Pattern 4 - Load reference data you subscribe to {#pattern-4---load-reference-data-you-subscribe-to}

Reference data bought or shared through the Marketplace (country, currency, industry or product
code lists, for example) arrives as a shared database. Add its table or view as an import source and
map the columns once; each later load brings the new values in as change requests, with your approval
rules, effective dates and audit trail, and your own attributes and mappings stay attached to the codes.
The shared data itself is only read.

### Taking an existing reference table under control {#taking-an-existing-reference-table-under-control}

If you already maintain a lookup table (e.g. `SALES_DB.REF.MARKET_SEGMENT`) that people edit
directly, you can move it into the app without changing the queries that read it. Don't swap anything until
the app holds exactly what the table holds - each step below has a check, and the old table stays until the
end so you can go back:

1. **Load the current rows** with Bulk Import: add the table as an import source (or paste it) and
   map its columns to the code, description, sort order and attributes - the names don't need to
   match, and the mapping is remembered for the next load. Use an effective date of today or earlier, and a
   reason and ticket, e.g. *"Initial load from SALES_DB.REF.MARKET_SEGMENT"*, `CHG0001234`.
2. **Wait until every imported change is applied.** An import creates change requests: with approval
   required they wait for an Admin, and a row that fails validation is marked `FAILED`. Carry on only when
   this returns one line, `APPROVED`:

   ```sql
   SELECT status, COUNT(*) FROM <app>.core.change_requests
   WHERE container = 'MARKET_SEGMENT' AND change_reason = 'Initial load from SALES_DB.REF.MARKET_SEGMENT'
   GROUP BY status;
   ```

3. **Reconcile both ways.** Both queries must return no rows - every row of the table is in the app, and the
   app has nothing the table doesn't (compare the columns as the new view will present them):

   ```sql
   SELECT segment_code, segment_name, sort_order FROM SALES_DB.REF.MARKET_SEGMENT
   EXCEPT
   SELECT code, description, sort_order FROM <app>.core.code_values_current WHERE code_set = 'MARKET_SEGMENT';

   SELECT code, description, sort_order FROM <app>.core.code_values_current WHERE code_set = 'MARKET_SEGMENT'
   EXCEPT
   SELECT segment_code, segment_name, sort_order FROM SALES_DB.REF.MARKET_SEGMENT;
   ```

4. **Build the view under a temporary name and test it as a reader.** Grants are not inherited from the old
   table, so grant SELECT to every role from the `SHOW GRANTS` output; the role that owns the view needs
   `viewer`. Then `USE ROLE <a_reader_role>` and query the view - it must return the same rows:

   ```sql
   SHOW GRANTS ON TABLE SALES_DB.REF.MARKET_SEGMENT;   -- note which roles read it

   CREATE VIEW SALES_DB.REF.MARKET_SEGMENT_NEW AS
   SELECT code        AS segment_code,   -- keep the original column names and order
          description AS segment_name,
          sort_order
   FROM <app>.core.code_values_current
   WHERE code_set = 'MARKET_SEGMENT';

   GRANT SELECT ON VIEW SALES_DB.REF.MARKET_SEGMENT_NEW TO ROLE <each_reader_role>;
   ```

   Check what depends on the table first (streams, tasks, materialized views, dynamic tables - see
   [When more work is needed](#when-more-work-is-needed)).
5. **Swap them** - two renames, in quick succession; queries that name the table read the view from then on:

   ```sql
   ALTER TABLE SALES_DB.REF.MARKET_SEGMENT RENAME TO SALES_DB.REF.MARKET_SEGMENT_LEGACY;
   ALTER VIEW SALES_DB.REF.MARKET_SEGMENT_NEW RENAME TO SALES_DB.REF.MARKET_SEGMENT;
   ```

   **To go back**, rename them the other way round (the table kept its own grants):

   ```sql
   ALTER VIEW SALES_DB.REF.MARKET_SEGMENT RENAME TO SALES_DB.REF.MARKET_SEGMENT_NEW;
   ALTER TABLE SALES_DB.REF.MARKET_SEGMENT_LEGACY RENAME TO SALES_DB.REF.MARKET_SEGMENT;
   ```

6. **Close the back door.** Revoke write access to the legacy table, or drop it after a grace period.
   From then on changes can only come through the app, with approval, effective dating and audit:

   ```sql
   REVOKE INSERT, UPDATE, DELETE, TRUNCATE ON TABLE SALES_DB.REF.MARKET_SEGMENT_LEGACY FROM ROLE <writer_role>;
   ```

### When more work is needed {#when-more-work-is-needed}

The swap above covers tables that are read by queries, reports and other views. A few situations need
work on your side. This is integration work that any reference data tool requires.

| Situation | What to do |
|---|---|
| **Jobs or applications write to the table** (ETL loads, an admin screen, scripts) | Repoint them to the app: `core.submit_changes` for individual changes, `core.import_rows` or `core.import_from_source` for batches. They then go through the same approval rules. Until they are changed, step 6 will make them fail - so find them first, e.g. in `ACCOUNT_USAGE.ACCESS_HISTORY`. Or leave the table where it is and make it a [managed table](/docs/managed-tables/#managed-tables): people's edits go through approval while jobs keep writing. |
| **Something needs a physical table** (a tool that cannot read views, clustering, Time Travel on the reference data itself) | Keep a table and refresh it from the app's view on a schedule, e.g. a task running `CREATE OR REPLACE TABLE ... AS SELECT ... FROM <app>.core.code_values_current WHERE code_set = '...'`. The app remains the place where changes are made; the table is a read-only copy. |
| **Streams, tasks, materialized views or dynamic tables depend on the table** | Check each one before the swap. They may need to be recreated against the new view or the refreshed copy. |
| **The table is not really reference data** (many-to-many relationships, wide transactional or fact-like tables, high-volume data) | Leave it where it is. The app is designed for code lists, mappings and hierarchies. |
{% endraw %}
