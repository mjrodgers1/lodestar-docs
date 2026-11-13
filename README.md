# Lodestar Data Services

Governed data changes inside your own Snowflake account. Lodestar is one simple, safe place for
business and data teams to maintain the data everything else depends on - customer segments, product
and cost-centre codes, financial hierarchies, country and currency codes, cross-system mappings, and
your own control tables such as price lists and thresholds. Business users work in a familiar grid;
every change gets maker/checker approval, effective dating and a complete audit trail, and is live in
your queries as soon as it is approved (or on its effective date). Your data never leaves your account.

## Features

- **Code sets**: named lists of codes with descriptions and sort order.
- **Typed attributes**: Admins define attributes per code set (text, number, true/false, date, or a
  reference to another code set, with required / max length / allowed values). They appear as typed
  columns in the editor and are validated on submit and on approval. Code sets without definitions
  accept free-form JSON attributes.
- **Effective dating**: every change has an effective date; full version history is kept and any
  point in time can be queried. Same-day edits correct the current version instead of versioning it.
- **Mappings**: crosswalks from source-system codes to a governed code set, validated against the target.
- **Mapping suggestions**: give Lodestar the source codes (pasted, or a column of one of your tables or
  views) and it suggests the best target codes for each one not yet mapped, with a score and the reason:
  same code or description, a matching attribute (e.g. ISO3 `IRL` -> `IE`) and - once an Admin turns it
  on - AI similarity with Snowflake Cortex (`Deutschland` -> `DE`). Stewards accept or correct them;
  accepted ones go through the normal approval rules.
- **Hierarchies**: parent/child trees over a code set, with cycle detection.
- **Maker/checker workflow**: Stewards propose changes, Admins approve or reject them. Self-approval
  is blocked by default (a requester can still reject - i.e. withdraw - their own request). Approval
  can be required or waived per code set, mapping set or hierarchy.
- **Named stewards**: optionally restrict a code set, mapping set or hierarchy to named users.
- **Change reasons and tickets**: every submission can carry a reason and a ticket reference (e.g. Jira,
  ServiceNow). Approvers see both, and both stay with the resulting versions in the history. Admins can
  make the reason mandatory.
- **Revert**: undo an applied change in one click; the undo goes through the same approval rules.
- **My work**: Stewards and Admins land on their own work queue - submissions waiting for their review,
  their own open submissions, and decisions on their changes in the last 14 days (with the reviewer's
  comment, e.g. why a change was rejected). The sidebar shows how many need them.
- **Comments**: discuss any change request or whole submission. Comments are kept with the request,
  audited, and never edited or deleted (add another to correct one).
- **Bulk import**: paste rows copied from Excel, upload a CSV or Excel (.xlsx) file, or load from any
  number of your own tables and views - or ones in databases shared with you, e.g. Marketplace reference
  data. Map the source's columns to codes, descriptions, typed attributes and hierarchy parents once; the
  mapping is remembered.
- **Managed tables**: put your own tables under governed editing where they are. An Admin chooses the
  key (composite keys allowed) and the editable columns; Stewards' inserts, updates and deletes go
  through the same approval rules, stewards, reasons, comments and audit trail, and are then written to
  your table. A change never overwrites a row that changed after it was submitted. Field-level history
  and one-click revert included. As many tables as you like, at no extra cost.
- **Filter and export**: filter any read-only grid (column / operator / value, with NOT) and download
  the filtered rows as CSV or Excel.
- **Use in SQL**: each code set, mapping set and hierarchy shows ready-made SQL to use it from your own
  database - a view over the app, always current, nothing to copy or sync.
- **Ask with AI**: a Cortex Agent (**Lodestar reference data**, in Snowflake CoWork), an MCP server for
  Cortex Code, Claude, Cursor and other MCP clients, a semantic view for Cortex Analyst, and lookup functions
  for one code, one translation or one hierarchy position. Read-only, and every request runs with the
  user's own access - see [AI agents and MCP](#ai-agents-and-mcp).
- **Audit log**: every submission, decision, change, correction and setting update is recorded.
- **Starter data**: sample ISO 3166 country and ISO 4217 currency code sets so you can try the app immediately.

## Privileges requested

| Privilege | Why |
|---|---|
| `CREATE WAREHOUSE` | The app creates one X-Small warehouse named `<APP_NAME>_WH` (standard Gen1, query acceleration off, auto-suspend 60 s, initially suspended). The app's consoles and the procedures they call run on it, so this compute is isolated from your workloads and its cost is attributable to the app (see Cost). Roles with `viewer` get USAGE on it because the agent's and MCP server's tools run there with the caller's rights (see [AI agents and MCP](#ai-agents-and-mcp)); those roles can also run their own queries on it. |
| `READ SESSION` | Recommended. Lets Snowflake tell the app who is signed in when someone uses its consoles, so changes, approvals and audit entries are recorded against that person by Snowflake itself. Without it the console passes the signed-in user's name to the app. Snowflake does not grant it at install: see step 2 below. Calls you make from SQL always record your own user, either way. |

Optional references (each can hold as many objects as you like):

| Reference | Object type | Privilege | Why |
|---|---|---|---|
| `import_tables` | TABLE | SELECT | Read-only sources for Bulk Import - your own tables or tables in a database shared with you (e.g. a Marketplace listing). Columns are mapped to the code, description, sort order, valid-from date, parent code and typed attributes in the app. |
| `import_views` | VIEW | SELECT | The same for views, including secure views and views in shared databases. |
| `managed_tables` | TABLE | SELECT, INSERT, UPDATE, DELETE | Your own tables that Stewards edit through the app (see [Managed tables](#managed-tables)). The app reads them and writes only approved changes, to the key and editable columns an Admin registers (plus any created/updated by/at columns you choose). Tables in shared databases are read-only, so they can't be added here. |

The app only reads the objects you add for import, and writes only approved changes to the tables you
add as managed tables; it does not request access to any other objects in your account.

Optional database role in the SNOWFLAKE database:

| Role | Why |
|---|---|
| `SNOWFLAKE.CORTEX_EMBED_USER` | Only for AI mapping suggestions: lets the app call Cortex `AI_EMBED` (embeddings only - no other AI functions or models). Not granted at install; see step 5 below. Without it, suggestions use exact and alias matches only. |

## Get started in 5 minutes

1. **Grant roles** (SQL below): `admin` to yourself, `steward` to whoever proposes changes,
   `viewer` to readers. Approvals need two people: one proposes, another approves. Also grant the
   app `READ SESSION` (step 2) so Snowflake identifies console users.
2. **Open the Admin console** (Snowsight -> Catalog -> Apps -> the app -> *Lodestar - Admin*).
   Its Home page shows a **set-up checklist** that ticks itself off as you go.
3. **Create a code set** (Code Sets) or try the COUNTRY / CURRENCY samples.
4. **Review the approval rules** (Audit & Settings -> Settings).
5. **Load codes**: paste rows from Excel or upload a CSV / Excel file in Bulk Import, import from your
   own (or shared) tables and views, or edit the grid in Code Values.
6. **Use them from SQL**: each code set shows ready-made SQL under **Use in SQL**.

## Post-install configuration

Run these as the role that installed the app (replace `<app>` with your application name,
e.g. `Lodestar`).

### 1. Grant application roles to your users

```sql
-- Admins: create code sets, approve/reject changes, change settings
GRANT APPLICATION ROLE <app>.admin   TO ROLE <lodestar_admin_role>;
-- Stewards: propose changes and run bulk imports
GRANT APPLICATION ROLE <app>.steward TO ROLE <lodestar_steward_role>;
-- Viewers: use the UI read-only and query the published views
GRANT APPLICATION ROLE <app>.viewer  TO ROLE <lodestar_reader_role>;
```

Roles are hierarchical: `admin` includes `steward`, which includes `viewer`.
Maker/checker approval needs at least two different users: one with `steward` (or admin)
to submit and another with `admin` to approve.

### 2. (Recommended) Let Snowflake identify console users

Grant the app `READ SESSION` (see Privileges requested). In the Admin console go to
**Audit & Settings -> App info** and click **Grant READ SESSION**, or use SQL (needs a role with
MANAGE GRANTS, e.g. ACCOUNTADMIN):

```sql
GRANT READ SESSION ON ACCOUNT TO APPLICATION <app>;
```

**Check identity** on the same tab then shows who the app records you as and how it knows.

### 3. (Optional) Add tables or views to import from

In the Admin console go to **Bulk Import -> From a table or view -> Add or remove sources** and choose
**Add a table** or **Add a view**. Tables and views in databases shared with you (Marketplace listings,
direct shares) work too, as long as your role can read them. Or use SQL:

```sql
CALL <app>.config.register_reference('IMPORT_TABLES', 'ADD',
  SYSTEM$REFERENCE('TABLE', '<db>.<schema>.<table>', 'PERSISTENT', 'SELECT'));
CALL <app>.config.register_reference('IMPORT_VIEWS', 'ADD',
  SYSTEM$REFERENCE('VIEW', '<db>.<schema>.<view>', 'PERSISTENT', 'SELECT'));
```

The role running `SYSTEM$REFERENCE` needs SELECT on the object. A source is removed with
**Remove this source** on the same page (or `register_reference('IMPORT_TABLES', 'REMOVE', '<alias>')`,
the alias being listed by `CALL <app>.core.import_sources()`).

### 4. (Optional) Manage your own tables

To let Stewards edit one of your tables with approvals and audit, add it to the `managed_tables`
reference and register its key and editable columns (see [Managed tables](#managed-tables)):

```sql
CALL <app>.config.register_reference('MANAGED_TABLES', 'ADD',
  SYSTEM$REFERENCE('TABLE', '<db>.<schema>.<table>', 'PERSISTENT', 'SELECT', 'INSERT', 'UPDATE', 'DELETE'));
```

The role running `SYSTEM$REFERENCE` needs all four privileges on the table.

### 5. (Optional) Turn on AI mapping suggestions

Mappings -> **Suggest** always offers exact and alias matches. To add AI similarity, grant the app the
embedding role (needs ACCOUNTADMIN) and turn on **Suggest mappings with Snowflake Cortex AI** in
**Audit & Settings -> Settings** (setting `ai_suggestions_enabled`, off by default). **Test AI access** on
the same tab (or `CALL <app>.core.check_ai();`) confirms the app can call it. See
[Mapping suggestions](#mapping-suggestions) for what it costs.

```sql
GRANT DATABASE ROLE SNOWFLAKE.CORTEX_EMBED_USER TO APPLICATION <app>;
CALL <app>.core.set_setting('ai_suggestions_enabled', 'true');
```

### 6. (Optional) Connect AI tools

Nothing to install: the agent **Lodestar reference data** shows up in Snowflake CoWork, and the MCP server
accepts MCP clients, for everyone whose role holds `viewer`. The account needs Cortex Agents and Snowflake
CoWork enabled. The **Connect** page (every console) shows the MCP endpoint and the set-up for each kind of
client. Review what the app created before you grant it:

```sql
DESC AGENT <app>.core.lodestar_agent;
DESCRIBE MCP SERVER <app>.core.lodestar_mcp;
SELECT * FROM <app>.core.agent_status;   -- whether setup could create them, and why not
```

If your account uses a feature policy that blocks apps from creating agents or MCP servers
(`BLOCKED_OBJECT_TYPES_FOR_CREATION = (AGENTS, MCP_SERVERS)`), the app installs without them and
`core.agent_status` says so; the semantic view and lookup functions still work. To add them later, lift the
policy for the app and upgrade it. See [AI agents and MCP](#ai-agents-and-mcp).

### 7. Open the app

**Snowsight -> Catalog -> Apps -> <app>**. On first launch the starter COUNTRY and CURRENCY code
sets are created.

The app has **three consoles**. Each user sees the consoles their application role allows:

| Console (Streamlit) | Who can open it | Purpose |
|---|---|---|
| **Lodestar Data Services** (default) | viewer, steward, admin | Read-only browsing of code sets, values, mappings, hierarchies |
| **Lodestar - Steward** | steward, admin | Propose changes, track approvals, bulk import |
| **Lodestar - Admin** | admin | Everything above plus create sets, approve/reject, audit log, settings |

Access is enforced by Snowflake grants on each console and on every stored procedure, so a user
cannot reach a function above their role from the UI or from SQL.

### 8. (Optional) Give business users a direct link

Stewards who only need the app can use its **app-viewer URL**, which opens a console without the
rest of Snowsight. In Snowsight open the console, select **Share -> Copy link -> For app viewers**
(the URL has the form `https://app.snowflake.com/streamlit/<org>/<account>/#/apps/<url_id>`; `url_id`
is returned by `DESCRIBE STREAMLIT <app>.core.<console>`). To keep such users in the app viewer only:

```sql
ALTER USER <user_name> SET ALLOWED_INTERFACES = (STREAMLIT);
```

From a direct link, Snowflake cannot show permission dialogs, so adding import sources or managed
tables (steps 3 and 4) and granting READ SESSION (step 2) must be done in Snowsight or with the SQL above.

### 9. (Optional) Choose when upgrades happen

New versions of the app install automatically. To have them applied only in a window that suits you,
create a maintenance policy and apply it to the app:

```sql
CREATE MAINTENANCE POLICY <db>.<schema>.lds_upgrades
  SCHEDULE = 'USING CRON 0 2 * * SAT UTC' COMMENT = 'Upgrade apps on Saturday nights';
ALTER APPLICATION <app> SET MAINTENANCE POLICY <db>.<schema>.lds_upgrades;
```

Your data is kept across upgrades.

## Using the app

The sidebar groups pages into **My work** (My work, Home), **Reference data** (Code Sets, Code Values,
Mappings, Hierarchies, Bulk Import), **Managed tables**, **Governance** (Approvals, Audit & Settings) and
**AI tools** (Connect). Each console shows only the pages its role can use. Stewards and Admins open on **My work**; the Viewer console on Home.

| Page | Console | What |
|---|---|---|
| My work | Steward / Admin | One row per submission: **Awaiting my review** (Admins: pending changes they may approve - not their own unless self-approval is on), **My open submissions**, and **Decided in the last 14 days** (rejected, failed or approved by a reviewer, with the review comment and any error; changes applied automatically are listed only if they failed). Pick a submission to read or add comments. The sidebar count is submissions awaiting your review plus those returned to you (rejected or failed); it refreshes after each action or with **Refresh**. |
| Home | all | Counts, and for Admins the set-up checklist and plan usage. |
| Code Sets | all / Admin | Browse code sets. Admins create, retire and reactivate them, set the approval policy and stewards, and define attributes. |
| Code Values | all / Steward | View codes as of any date. Stewards edit a grid (add, change, delete = retire) and submit with an effective date. |
| Mappings | all / Steward | Map source codes to target codes. Admins create mapping sets. **Suggest** (Steward): paste source codes or pick a column of an import source, and review the suggested targets - each with its score, method (exact, alias, AI similarity), reason and the next best alternatives; tick the ones to accept (change the target if needed) and submit them as mapping changes. |
| Hierarchies | all / Steward | View the tree. Stewards set each code's parent (empty = root). |
| Approvals | Steward / Admin | Stewards track pending requests; Admins approve or reject them, per request or a whole batch (one grid save, import or revert) at a time. A change that fails validation is marked `FAILED` with the reason; the others are applied. Under **Recent decisions**, select applied changes and **Revert** them. Every grid has a **comments** count; pick a request or submission under it to read its thread (comments plus review decisions) and add a comment. |
| Bulk Import | Steward | **Paste or upload rows**: paste cells copied from Excel or CSV text, or upload a CSV / TSV / TXT or Excel (.xlsx; pick the sheet) file of up to 25 MB - the first row is the header and must include `CODE`; check the preview (empty rows are skipped) and import. Up to 50,000 rows per run. Or import **from a table or view**: pick one of the sources an Admin added (your own or shared databases), check the preview, map its columns to the code, description, sort order, valid-from date, parent code (with a hierarchy) and typed attributes, and run it. The mapping is remembered per source and code set. Admins add and remove sources here. |
| Managed tables | Steward / Admin | **Edit rows**: pick a table, search, and edit its rows in a grid (editable columns are typed; key columns marked; other columns shown for reference). Add or delete rows where the table allows it, give a reason and ticket, and submit. Key values of existing rows can't be edited (delete the row and add a new one). **Pending changes**: each change as before -> after, with comments. **History**: every applied change per column, for the whole table or one row, with comments. **Set up** (Admin): add or remove tables, register key / editable / audit columns and what Stewards may do, approval policy, stewards, retire. See [Managed tables](#managed-tables). |
| Audit & Settings | Admin | Search the audit log, toggle approval requirements, mandatory change reasons and AI mapping suggestions (with **Test AI access**), check how users are identified (READ SESSION) and event sharing. The Settings tab always shows the values as stored now, and **Save settings** changes only the switches you changed, so two Admins can't undo each other's changes. Other pages read the settings when the console opens - press **Refresh** in the sidebar after another Admin changes them. |
| Connect | all | Whether the Cortex Agent and MCP server are available, the MCP endpoint, the MCP tools, how to connect Snowflake CoWork, Cortex Code, other MCP clients and your own agents, and example questions. See [AI agents and MCP](#ai-agents-and-mcp). |

Creating or retiring sets and saving settings takes a second or two; saving changes, approving or
importing takes several seconds. A "Working" indicator shows while it runs and the
button can't be pressed again meanwhile. Afterwards the page refreshes with the outcome shown under the
button (and briefly in the corner); after a successful submit the grid and reason fields start fresh,
so the same changes can't be sent twice.

**Reason and ticket.** Every submission (grid edits, bulk imports and reverts) has a **Reason for change**
and a **Ticket reference** field. Use the reason for the narrative (why the change is needed) and the
ticket for the id of the related Jira issue, ServiceNow change or incident (several ids can be comma
separated). Approvers see both on the Approvals page, and both appear on every version in the History
tabs and in the `*_history` views, so you can always answer "why does this value look like this?".
With **Require a reason for every change** switched on (Audit & Settings), submissions without a
reason are refused. A revert without its own ticket reuses the ticket of the change it undoes.

**Revert.** Reverting submits the change that undoes an applied request, effective on the same date:
a new version gets the previous values back, a newly added code is retired so it never took effect,
a correction gets the overwritten values back, and a retirement re-opens the code. If the value has
changed again since, revert the later change first.

**Stewards.** If a code set, mapping set or hierarchy lists stewards, only those Snowflake users can
submit changes to it (including imports and reverts). Empty = anyone with `steward`.

**Effective dating rules.** Versions use half-open windows `[valid_from, valid_to)`.

- A change effective **after** the current version's start date creates a new version.
- A change effective **on** the current version's start date is a **correction**: it updates that
  version in place, and the previous values are kept in the audit log (action `CORRECT`).
- Earlier effective dates are rejected.
- Retiring a code ends its current version on the effective date.
- Approved changes with a future effective date appear in `*_current` views from that date.

**Approval policy.** Each code set, mapping set and hierarchy has a policy: *App default* (follows
the **Require approval** setting), *Always require approval*, or *Apply immediately*. Admins set it
on the Code Sets, Mappings or Hierarchies page.

## Sizing and limits

Tested with code sets of 50,000 codes, 10,000-row mapping sets and hierarchies, and hierarchies
1,000 levels deep (X-Small warehouse):

| Operation | Typical time |
|---|---|
| Import 10,000 / 50,000 rows | about 20 s / 30 s |
| Approve 10,000 pending changes in one go | about 15 s |
| A single change, approval or revert | about 7-14 s (mostly fixed start-up time per action) |

Limits:

- **Imports**: up to 50,000 rows per run (split larger loads).
- **Mapping suggestions**: up to 1,000 unmapped source codes per run (from an import source: the first
  1,000 by code - submit them and run again for the rest); about 10-15 s per run.
- **Hierarchies**: up to 1,000 levels deep.
- **Grids**: read-only grids show up to 5,000 rows; use **Filter** to narrow them down, or
  **Export** to download every row. Editing grids hold up to 1,000 rows: for larger sets, search for
  the codes you want to change (new rows can be added without searching), or use **Bulk Import**.
- **Approvals**: the per-request list shows the oldest 1,000 pending requests; review **by batch**
  to approve or reject whole submissions of any size.
- **Procedure results** list at most 1,000 request ids and 100 error messages (flagged with
  `request_ids_truncated` / `errors_truncated`); counts are always complete, and every request of
  a submission can be found in `core.change_requests` by its `batch_id`.

## Mapping suggestions

On **Mappings -> Suggest**, give the source codes - pasted with a header row (`CODE`, optionally
`DESCRIPTION`), or a code column (and optionally a description column) of one of the tables or views
added for import. For each code the mapping set doesn't map yet, Lodestar suggests up to three target
codes of its code set, best first. Codes already mapped, or with a change waiting for approval, are
skipped and counted.

| Method | Score | When |
|---|---|---|
| Exact | 1.00 / 0.99 | Same code / same description (ignoring case) |
| Alias | 0.97 / 0.96 | The source code or description equals a text attribute of the target (e.g. `iso3`) / the source code is the target's description or the other way round |
| AI similarity | the cosine similarity, typically 0.4-0.9 | `ai_suggestions_enabled` on: the meaning of "code - description" is compared, so `Deutschland`, `Holland` or `Nippon` find `DE`, `NL` and `JP` |

A target found more than one way keeps its best score. Suggestions change nothing: the Steward ticks the
ones to accept (exact and alias matches, and AI ones above an adjustable score, are ticked to start with),
can pick another target, and submits them as ordinary mapping changes - approval, stewards, reasons and
history apply as usual. Each run is audited as `SUGGEST_MAPPINGS` (it doesn't count as editing for the plan).

**AI similarity** uses Snowflake Cortex `AI_EMBED` with the model `snowflake-arctic-embed-l-v2.0`
inside your account - no data leaves Snowflake, and there is no LLM prompt. The app needs
`SNOWFLAKE.CORTEX_EMBED_USER` (step 5 of the configuration). If it can't call `AI_EMBED` (role not
granted, model not available in your region and cross-region inference not allowed by
`CORTEX_ENABLED_CROSS_REGION`, or a Snowflake trial account - Snowflake doesn't offer `AI_EMBED` on
trial accounts), suggestions fall back to exact and alias matches and say why.

**Cost.** Embeddings are billed as Cortex AI usage on the app's warehouse: about 10 tokens per code.
Each target code is embedded once and kept, and embedded again only when its code or description
changes; source codes are embedded on every run. Matching itself (`VECTOR_COSINE_SIMILARITY`) costs no
tokens. To see the app's AI usage:

```sql
SELECT usage_time, function_name, model_name, tokens, token_credits
FROM SNOWFLAKE.ACCOUNT_USAGE.CORTEX_AISQL_USAGE_HISTORY
WHERE function_name = 'AI_EMBED' AND warehouse_id = (
  SELECT warehouse_id FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_EVENTS_HISTORY
  WHERE warehouse_name = '<APP_NAME>_WH' LIMIT 1)
ORDER BY usage_time DESC;
```

From SQL:

```sql
CALL <app>.core.suggest_mappings('CRM_COUNTRY',
  PARSE_JSON('["IRL", {"code": "D1", "description": "Deutschland"}]'));
CALL <app>.core.suggest_mappings_from_source('CRM_COUNTRY', 'CRM_DB.PUBLIC.CUSTOMERS', 'COUNTRY_CODE', 'COUNTRY_NAME');
```

The result lists `suggestions` (`source_code`, `source_description`, `candidates`: `target_code`,
`target_description`, `score`, `method` `EXACT` / `ALIAS` / `SEMANTIC`, `reason`), the counts `unmapped`,
`already_mapped`, `pending`, `truncated`, and `ai` (`enabled`, `used`, `model`, `embedded_targets`, and
`error` / `fix` when AI couldn't be used, or `trial` = true and no `fix` on a Snowflake trial account,
where no grant helps).

## Managed tables

Some tables should stay where they are - read by many jobs, with their own columns and types - but be
changed only with approval and audit: price lists, thresholds, product or customer attributes, ETL
control tables. Make them **managed tables**: Stewards' edits become change requests under the same
rules as reference data, and approved changes are written to your table.

**Set up (Admin).** On **Managed tables -> Set up**, choose **Add a table** (or use the SQL in step 4 of
the configuration), then register it with a name, its **key columns** (one or more - together they must
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
any change. Reverting an insert or delete works even if Stewards may not add or delete rows.

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

## AI agents and MCP

Lodestar makes your reference data available to AI tools, read-only. Every request runs with the rights
of the person asking (restricted caller's rights): it reaches only the app's own objects - never your
other data - and only if that person's role holds `viewer`. Everything is the version valid today; the
agent can't see history, pending changes, approvals or the rows of managed tables, and can't change
anything. The **Connect** page shows all of this for your account, with copy-paste set-up.

| Object | Role | What |
|---|---|---|
| `core.lodestar_agent` | viewer | Cortex Agent **Lodestar reference data** (orchestration model `auto`). Shows up in Snowflake CoWork; also callable from the Cortex Agents REST API and `SNOWFLAKE.CORTEX.DATA_AGENT_RUN`. Answers with the semantic view (Cortex Analyst) and the three lookup functions. |
| `core.lodestar_mcp` | viewer | Snowflake-managed MCP server at `https://<account_url>/api/v2/databases/<app>/schemas/CORE/mcp-servers/LODESTAR_MCP`. Tools: `ask-lodestar` (the agent - best for most questions), `query-reference-data` (Cortex Analyst: returns SQL for the client to run, because an app's MCP server can't run SQL), `lookup-code`, `translate-code` and `hierarchy-path` (the functions below). |
| `core.lodestar_sv` | viewer | Semantic view over code sets, codes, attribute values, mapping sets and mappings, hierarchies and their nodes, and managed tables - registration details only (name, description, key and editable columns, whether rows may be added or deleted), never their rows or where the table lives. Query it yourself with `SEMANTIC_VIEW(...)` or point your own agents at it. |
| `core.lookup_code(code_set, code)` | viewer | One code valid today as JSON: `found`, `code_set`, `code`, `description`, `attributes`, `sort_order`, `valid_from`, `valid_to`, `version_no` |
| `core.translate_code(mapping_set, source_code)` | viewer | The mapping valid today for one source code: `found`, `source_system`, `source_code`, `source_description`, `target_code_set`, `target_code`, `target_description`, validity |
| `core.hierarchy_path(hierarchy, code)` | viewer | Where one code sits in a hierarchy today: `found`, `parent_code` (null for a root), `depth` (root = 1), `path` (`EU > IE`), `path_codes`, `root_code`, `children` (first 100, by code) and `child_count` |
| `core.agent_status` | viewer | Whether setup could create the agent and the MCP server (`available`), and Snowflake's `message` if it couldn't |

The lookup functions match codes ignoring case (an exact-case match wins) and trim spaces; names of code
sets, mapping sets and hierarchies are upper-cased. A miss returns `found: false` with a `message` saying
whether the code or the set itself is missing. They can be called row by row:

```sql
SELECT <app>.core.lookup_code('COUNTRY', 'ie');            -- {"found": true, "code": "IE", "description": "Ireland", ...}
SELECT s.*, <app>.core.translate_code('CRM_COUNTRY', s.country):target_code::VARCHAR AS country_code
FROM my_db.my_schema.crm_customers s;
SELECT * FROM SEMANTIC_VIEW(<app>.core.lodestar_sv METRICS codes.code_count DIMENSIONS code_sets.code_set_name);
```

For large tables a join to `core.mappings_current` (see [Querying reference data from SQL](#querying-reference-data-from-sql))
is faster than calling a function per row.

**Who can use them.** Any role with `viewer` that can use Cortex: `SNOWFLAKE.CORTEX_USER` (granted to PUBLIC
by default) or `SNOWFLAKE.CORTEX_AGENT_USER`. The account needs Cortex Agents and Snowflake CoWork enabled.
On a Snowflake **trial account** Snowflake refuses agent runs ("Access denied for trial accounts"), and with
them the MCP server's `ask-lodestar` and lookup tools; the semantic view, the `query-reference-data` tool
and the lookup functions in SQL still work.
The tools run on the app's warehouse `<APP_NAME>_WH`; that is why `viewer` has USAGE on it (see
[Privileges requested](#privileges-requested)).

**Connecting.**

- **Snowflake CoWork**: nothing to do - the agent appears for users whose role holds `viewer`.
- **Cortex Code**: add the server to `~/.snowflake/cortex/mcp.json`; Cortex Code (1.1.65 or later) signs in
  with a connection from your `connections.toml`:

  ```json
  {"mcpServers": {"lodestar": {"type": "http",
    "url": "https://<account_url>/api/v2/databases/<app>/schemas/CORE/mcp-servers/LODESTAR_MCP",
    "snowflake_connection": "<your_connection_name>"}}}
  ```

- **Claude, Cursor, ChatGPT and other MCP clients**: sign in with Snowflake OAuth. An account administrator
  creates one security integration for the account, and each user enters the client id and secret in the
  client. MCP sessions use the user's default role and default warehouse, so set both:

  ```sql
  CREATE SECURITY INTEGRATION lodestar_mcp_oauth
    TYPE = OAUTH OAUTH_CLIENT = CUSTOM ENABLED = TRUE
    OAUTH_CLIENT_TYPE = 'CONFIDENTIAL'                 -- 'PUBLIC' for PKCE clients such as Claude Desktop
    OAUTH_REDIRECT_URI = '<redirect URI shown by your MCP client>'
    OAUTH_USE_SECONDARY_ROLES = NONE
    ALLOWED_ROLES_LIST = ('<role_with_lodestar_viewer>');
  SELECT SYSTEM$SHOW_OAUTH_CLIENT_SECRETS('LODESTAR_MCP_OAUTH');
  ALTER USER <user> SET DEFAULT_ROLE = '<role_with_lodestar_viewer>' DEFAULT_WAREHOUSE = '<warehouse>';
  ```

  With a network policy, allow the client provider's outbound IP addresses.
- **Your own Cortex Agents**: add the semantic view as a Cortex Analyst tool and the functions as custom
  tools (the Connect page has the specification to paste).

**Review and control.** `DESC AGENT <app>.core.lodestar_agent` and `DESCRIBE MCP SERVER <app>.core.lodestar_mcp`
show exactly what they do (instructions, tools and the objects behind them). To stop apps creating agents
or MCP servers, use a feature policy (`CREATE FEATURE POLICY ... BLOCKED_OBJECT_TYPES_FOR_CREATION =
(AGENTS, MCP_SERVERS)`); Lodestar then installs and upgrades without them and reports it in
`core.agent_status`. Revoking `viewer` removes access to all of it. The app doesn't ask for agent telemetry
(see [Telemetry](#telemetry)), so questions and answers are not shared with the provider.

**Cost.** An agent question or a `query-reference-data` call uses Cortex AI credits (Cortex Agents and
Cortex Analyst pricing), plus a few seconds on `<APP_NAME>_WH`; an agent answer takes about 15 s. The
lookup functions are ordinary queries. All of it is billed to your account - see [Cost](#cost).

## Working with your existing tables

The app keeps the governed copy of your reference data in its own tables. Your databases use it
through the app's views, so there is a single source of truth and nothing to copy or sync. There are
four common patterns and one migration step.

### Pattern 1 - Look up and standardise

Join your tables to `core.code_values_current` to add governed descriptions and attributes. The
**Use in SQL** panel on the Code Values, Mappings and Hierarchies pages generates a starting view.

### Pattern 2 - Translate source-system codes

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
       ON m.mapping_set = 'CRM_COUNTRY' AND m.source_code = c.nation_code
LEFT JOIN <app>.core.code_values_current cv
       ON cv.code_set = 'COUNTRY' AND cv.code = m.target_code;
```

### Pattern 3 - Check data quality

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

### Pattern 4 - Load reference data you subscribe to

Reference data bought or shared through the Marketplace (country, currency, industry or product
code lists, for example) arrives as a shared database. Add its table or view as an import source and
map the columns once; each later load brings the new values in as change requests, with your approval
rules, effective dates and audit trail, and your own attributes and mappings stay attached to the codes.
The shared data itself is only read.

### Taking an existing reference table under control

If you already maintain a lookup table (e.g. `SALES_DB.REF.MARKET_SEGMENT`) that people edit
directly, you can move it into the app without changing the queries that read it:

1. **Load the current rows** with Bulk Import: add the table as an import source (or paste it) and
   map its columns to the code, description, sort order and attributes - the names don't need to
   match, and the mapping is remembered for the next load.
2. **Put a view with the same name and columns in place of the table.** Keep the old table for rollback:

   ```sql
   SHOW GRANTS ON TABLE SALES_DB.REF.MARKET_SEGMENT;   -- note which roles read it

   ALTER TABLE SALES_DB.REF.MARKET_SEGMENT RENAME TO SALES_DB.REF.MARKET_SEGMENT_LEGACY;

   CREATE VIEW SALES_DB.REF.MARKET_SEGMENT AS
   SELECT code        AS segment_code,   -- keep the original column names and order
          description AS segment_name,
          sort_order
   FROM <app>.core.code_values_current
   WHERE code_set = 'MARKET_SEGMENT';

   GRANT SELECT ON VIEW SALES_DB.REF.MARKET_SEGMENT TO ROLE <each_reader_role>;
   ```

   Grants are not inherited from the old table, so re-grant SELECT to every role from the
   `SHOW GRANTS` output. The role that owns the view needs `viewer`.
3. **Close the back door.** Revoke write access to the legacy table, or drop it after a grace period.
   From then on changes can only come through the app, with approval, effective dating and audit:

   ```sql
   REVOKE INSERT, UPDATE, DELETE, TRUNCATE ON TABLE SALES_DB.REF.MARKET_SEGMENT_LEGACY FROM ROLE <writer_role>;
   ```

It is worth recording the cut-over itself with a reason and ticket on the initial import, e.g.
*"Initial load from SALES_DB.REF.MARKET_SEGMENT"*, `CHG0001234`.

### When more work is needed

The swap above covers tables that are read by queries, reports and other views. A few situations need
work on your side. This is integration work that any reference data tool requires.

| Situation | What to do |
|---|---|
| **Jobs or applications write to the table** (ETL loads, an admin screen, scripts) | Repoint them to the app: `core.submit_changes` for individual changes, `core.import_rows` or `core.import_from_source` for batches. They then go through the same approval rules. Until they are changed, step 3 will make them fail - so find them first, e.g. in `ACCOUNT_USAGE.ACCESS_HISTORY`. Or leave the table where it is and make it a [managed table](#managed-tables): people's edits go through approval while jobs keep writing. |
| **Something needs a physical table** (a tool that cannot read views, clustering, Time Travel on the reference data itself) | Keep a table and refresh it from the app's view on a schedule, e.g. a task running `CREATE OR REPLACE TABLE ... AS SELECT ... FROM <app>.core.code_values_current WHERE code_set = '...'`. The app remains the place where changes are made; the table is a read-only copy. |
| **Streams, tasks, materialized views or dynamic tables depend on the table** | Check each one before the swap. They may need to be recreated against the new view or the refreshed copy. |
| **The table is not really reference data** (many-to-many relationships, wide transactional or fact-like tables, high-volume data) | Leave it where it is. The app is designed for code lists, mappings and hierarchies. |

## Querying reference data from SQL

Grant `viewer` to any role your pipelines use, then:

```sql
-- Current codes
SELECT code, description, attributes
FROM <app>.core.code_values_current
WHERE code_set = 'COUNTRY';

-- Point in time
SELECT * FROM TABLE(<app>.core.code_values_as_of('COUNTRY', '2025-01-01'::DATE));

-- Translate source-system codes
SELECT s.*, m.target_code
FROM my_db.my_schema.crm_customers s
LEFT JOIN <app>.core.mappings_current m
  ON m.mapping_set = 'CRM_COUNTRY' AND m.source_code = s.country;

-- Point-in-time mappings
SELECT * FROM TABLE(<app>.core.mappings_as_of('CRM_COUNTRY', CURRENT_DATE()));

-- Flattened hierarchy (depth, path from root)
SELECT * FROM <app>.core.hierarchy_current WHERE hierarchy = 'COUNTRY_REGION' ORDER BY path;
```

### Published objects

| Object | Role | Description |
|---|---|---|
| `core.code_sets` | viewer | Code set catalogue |
| `core.code_values_current` | viewer | Codes valid today (active code sets only) |
| `core.code_values_history` | viewer | All versions of all codes, with the change reason and ticket that produced each |
| `core.code_values_as_of(code_set, as_of)` | viewer | Table function: codes valid on a date |
| `core.attribute_defs` | viewer | Typed attribute definitions per code set |
| `core.code_value_attributes_current` | viewer | Current attribute values, one row per code and attribute, with the referenced code's description |
| `core.mapping_sets` | viewer | Mapping set catalogue |
| `core.mappings_current` | viewer | Mappings valid today, with target description |
| `core.mappings_history` | viewer | All mapping versions, with change reason and ticket |
| `core.mappings_as_of(mapping_set, as_of)` | viewer | Table function: mappings valid on a date |
| `core.hierarchies` | viewer | Hierarchy catalogue |
| `core.hierarchy_current` | viewer | Current hierarchy with depth (root = 1), path and root |
| `core.hierarchy_edges_history` | viewer | All parent/child versions, with change reason and ticket |
| `core.lookup_code(code_set, code)`, `core.translate_code(mapping_set, source_code)`, `core.hierarchy_path(hierarchy, code)` | viewer | One code, one translation or one hierarchy position valid today, as JSON - see [AI agents and MCP](#ai-agents-and-mcp) |
| `core.lodestar_sv` | viewer | Semantic view for Cortex Analyst over all of the above |
| `core.lodestar_agent`, `core.lodestar_mcp`, `core.agent_status` | viewer | Cortex Agent, MCP server, and whether setup could create them |
| `core.change_requests` | steward | Change requests, their reason / ticket and their outcome |
| `core.comments` | steward | Comments on a change request (`target_type` `REQUEST`) or a whole submission (`BATCH`), with the request's `batch_id`, container and key, author and time. Never edited. |
| `core.work_queue(user)` | steward | Table function: one row per submission and queue for a user - `TO_REVIEW`, `MY_OPEN`, `MY_DECIDED` (see My work above) - with counts, dates, reason, decision, review comment, error and comment count. Call it with `CURRENT_USER()` for your own queue. |
| `core.managed_tables` | steward | Registered managed tables: name, source table, key / editable / audit columns, whether rows may be added or deleted, the table's columns and types when registered, status, approval policy and stewards |
| `core.managed_field_history` | steward | Applied changes to managed tables, one row per column: `row_key` (the key, or a JSON array for a composite key), `key_values`, `action` (`INSERT`, `UPDATE`, `DELETE`), `column_name`, `old_value` / `new_value` (text, or JSON for JSON columns), who proposed and approved it, when, the reason and ticket |
| `core.audit_log` | admin | Full audit trail |
| `core.active_editors` | admin | Editors in the last 30 days (user, actions, first / last action) - what your plan counts, see [Trial and plan](#trial-and-plan) |

### Stored procedures

All procedures return a JSON object `{"ok": true|false, ...}`; on failure `error` holds the reason.
The procedures that submit, review, revert or import changes (and `define_attribute`,
`seed_starter_data`, `plan_usage`) also include `_perf` (`ms` spent inside the procedure and `sql`
statements run), which helps when diagnosing slow calls.
Every procedure also accepts an optional trailing `actor` argument used only by the app's own UI;
when called from SQL it is ignored and the caller's `CURRENT_USER()` is recorded.
Procedures that submit changes also take optional `reason` and `ticket_ref` arguments (before `actor`).

Approving (or auto-applying) many changes at once is done set-based in a single transaction, so a
1,000-row import runs the same number of statements as a 1-row one (about 10 seconds on an X-Small
warehouse). Each change is still validated on its own: a change that fails (for example an overlapping
date or an unknown code) is marked `FAILED` with the reason, and the others are applied.

| Procedure | Role | Purpose |
|---|---|---|
| `core.whoami()` | viewer | Current user, how they were identified (`user_source`: `SESSION` = by Snowflake, `UI` = name passed by the console), the application, its warehouse and the settings |
| `core.seed_starter_data()` | viewer | Creates the starter code sets once (runs automatically) |
| `core.submit_changes(entity_type, container, changes, effective_date, reason, ticket_ref)` | steward | Submit change requests. `entity_type`: `CODE_VALUE`, `MAPPING` or `HIERARCHY_EDGE`. `reason` / `ticket_ref` optional |
| `core.import_sources()` | steward | The tables and views added for import: `source` (DATABASE.SCHEMA.NAME), `kind`, `columns`, the first 10 rows (`preview`), `alias`, and the column mapping last used per code set (`mappings`) |
| `core.import_from_source(source, code_set, effective_date, hierarchy, column_map, reason, ticket_ref)` | steward | Import from one of those sources. `column_map`: `PARSE_JSON('{"CODE": "<column>", "DESCRIPTION": "<column>", "ISO3": "<column>"}')` - fields `CODE` (required), `DESCRIPTION`, `SORT_ORDER`, `VALID_FROM`, `PARENT_CODE` and attribute names; NULL = the mapping last used for this source and code set, else columns named like the fields. The mapping used is saved and returned as `column_map` |
| `core.import_rows(code_set, import_data, effective_date, hierarchy, reason, ticket_ref)` | steward | Bulk import from a JSON array of row objects (same columns as the source table) |
| `core.revert_change(request_id, comment, ticket_ref)` | steward | Submit the change that undoes an applied request; `comment` becomes the reason |
| `core.add_comment(target_type, target_id, body)` | steward | Comment on a change request (`REQUEST`, its `request_id`) or a whole submission (`BATCH`, its `batch_id`). Up to 4,000 characters, trimmed; audited as `COMMENT`. Comments can't be edited or deleted. |
| `core.managed_rows(table_name, search, max_rows)` | steward | Rows of a managed table in key order, with `_LODESTAR_ROW_HASH` to pass back with a change. `search` (optional) matches any key or editable column, ignoring case; `max_rows` default 1,000, at most 5,000. A problem (not registered, no longer added) comes back as one row with a `_LODESTAR_ERROR` column. |
| `core.submit_row_changes(table_name, changes, reason, ticket_ref)` | steward | Submit inserts, updates and deletes for a managed table: `changes` is an array of `{"action": "INSERT" / "UPDATE" / "DELETE", "key": {column: value}, "values": {column: value}, "row_hash": ...}` (`row_hash` optional). Applied at once if the table's approval policy doesn't require approval. See [Managed tables](#managed-tables) |
| `core.suggest_mappings(mapping_set, sources)` | steward | Suggested target codes for source codes the mapping set doesn't map yet: `sources` is an array of codes or `{"code": ..., "description": ...}` objects (up to 1,000). Changes nothing. See [Mapping suggestions](#mapping-suggestions) |
| `core.suggest_mappings_from_source(mapping_set, source, code_column, description_column)` | steward | The same for the distinct values of a column of an import source (`source` = DATABASE.SCHEMA.NAME as listed by `core.import_sources`; `description_column` optional) |
| `core.review_requests(request_ids, decision, comment)` | admin | `decision`: `APPROVE` or `REJECT` |
| `core.create_code_set(code_set, description, domain, owner_name)` | admin | Create a code set |
| `core.create_mapping_set(mapping_set, description, source_system, target_code_set)` | admin | Create a mapping set |
| `core.create_hierarchy(hierarchy, description, code_set)` | admin | Create a hierarchy |
| `core.set_container_status(container_type, name, status)` | admin | `container_type`: `CODE_SET`, `MAPPING_SET`, `HIERARCHY`, `MANAGED_TABLE`; `status`: `ACTIVE` or `RETIRED` |
| `core.set_setting(setting_key, setting_value)` | admin | `require_approval`, `allow_self_approval`, `require_change_reason`, `ai_suggestions_enabled` (`true`/`false`) |
| `core.set_settings(settings)` | admin | Several settings at once, all or nothing: `PARSE_JSON('{"require_approval": "true", "require_change_reason": "true"}')`. Every key is checked first; only changed values are saved and audited. Returns `settings` and `changed`. |
| `core.plan_usage()` | admin | Editors active in the last 30 days against the editors your plan includes: `active_editors`, `included_editors`, `over` |
| `core.check_ai()` | admin | Whether the app can call Cortex `AI_EMBED` (one embedding of a few tokens): `available`, `enabled` (the `ai_suggestions_enabled` setting), `model`, `region`; when not available, Snowflake's `error` and the GRANT that usually fixes it (`fix`) - or, on a Snowflake trial account, `trial` = true and no `fix` (Snowflake doesn't offer AI functions on trials) |
| `core.set_approval_policy(container_type, name, policy)` | admin | `policy`: `DEFAULT`, `REQUIRED` or `NOT_REQUIRED` |
| `core.set_stewards(container_type, name, stewards)` | admin | Array of Snowflake user names allowed to change it; empty array = any Steward |
| `core.define_attribute(code_set, attr_name, label, data_type, is_required, max_length, allowed_values, ref_code_set, sort_order)` | admin | Create or update a typed attribute. `data_type`: `TEXT`, `NUMBER`, `BOOLEAN`, `DATE`, `REFERENCE` |
| `core.set_attribute_status(code_set, attr_name, status)` | admin | Retire or reactivate an attribute definition |
| `core.managed_sources()` | admin | Tables added to the `managed_tables` reference: `source` (DATABASE.SCHEMA.NAME), `alias`, the name it is registered under (`table_name`, NULL if not yet) and its `columns` (name, type, nullable, has_default, supported) |
| `core.register_managed_table(table_name, source, key_columns, editable_columns, audit_columns, allow_insert, allow_delete, description)` | admin | Register or update a managed table: key and editable columns (arrays of column names, any case), `audit_columns` `PARSE_JSON('{"updated_by": "<column>", ...}')` (keys `created_by`, `created_at`, `updated_by`, `updated_at`; `{}` for none), whether Stewards may add / delete rows. The key columns can't change while the table has pending changes |

Example - propose two new codes from SQL:

```sql
CALL <app>.core.submit_changes(
  'CODE_VALUE', 'CURRENCY',
  PARSE_JSON('[{"key":"MXN","payload":{"description":"Mexican Peso"}},
               {"key":"BRL","payload":{"description":"Brazilian Real"}}]'),
  CURRENT_DATE(),
  'LATAM expansion - new billing currencies', 'JIRA-1234');
```

Example - your work queue, and a comment on a submission waiting for you:

```sql
SELECT queue, container, requests, requested_by, review_comment, comments, batch_id
FROM TABLE(<app>.core.work_queue(CURRENT_USER()));

CALL <app>.core.add_comment('BATCH', '<batch_id>', 'Please add the ISO numeric codes too.');
```

Example - define typed attributes on COUNTRY:

```sql
CALL <app>.core.define_attribute('COUNTRY', 'iso3', 'ISO alpha-3', 'TEXT', TRUE, 3, NULL, NULL, 1);
CALL <app>.core.define_attribute('COUNTRY', 'currency', 'Currency', 'REFERENCE', FALSE, NULL, NULL, 'CURRENCY', 2);
CALL <app>.core.define_attribute('COUNTRY', 'region', 'Region', 'TEXT', FALSE, NULL,
                                 ARRAY_CONSTRUCT('EMEA','AMER','APAC'), NULL, 3);

SELECT code, attr_name, value_text, ref_description
FROM <app>.core.code_value_attributes_current WHERE code_set = 'COUNTRY';
```

`changes` items: `{"key": "<code>", "action": "UPSERT"|"RETIRE", "payload": {...}, "effective_date": "YYYY-MM-DD"}`.
Payload fields: CODE_VALUE `description`, `sort_order`, `attributes`; MAPPING `source_description`,
`target_code`; HIERARCHY_EDGE `parent_code`. Fields you omit keep their current value.

## Telemetry

The app shares **error and warning logs** with the provider (mandatory) so we can detect and fix
failures. **INFO usage logs** are optional and off by default. Logs contain procedure names,
counts and error messages; validation errors can include the code or code-set identifier that
failed (for example "Target code XX is not valid in COUNTRY"). Descriptions, attributes and
bulk-import contents are not logged. Event sharing requires an active event table in your account.
The app doesn't request traces or agent telemetry, so questions asked of its Cortex Agent or MCP server
and their answers stay in your account.

## Trial and plan

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

## Cost

Apart from any listing price, the app only uses Snowflake compute and storage in your account:

| What | Runs on | Billed as |
|---|---|---|
| The consoles (Streamlit) and the procedures they call | The app's warehouse `<APP_NAME>_WH` (X-Small Gen1, auto-suspend 60 s) | Credits of that warehouse, while someone is using a console |
| Procedures you `CALL` from SQL (jobs, scripts) | The warehouse of your session | Your warehouse |
| Queries on the published views / functions (pipelines, BI) | The warehouse of your session | Your warehouse |
| The app's tables (codes, history, audit log) | - | Storage, typically a few MB |
| AI mapping suggestions (only if an Admin turns them on) | Cortex `AI_EMBED` on the app's warehouse | Cortex AI usage: about 10 tokens per code, each target code embedded once - see [Mapping suggestions](#mapping-suggestions) |
| Questions to the agent or the MCP server's Cortex Analyst tool | Cortex Agents / Cortex Analyst, with their queries on the app's warehouse | Cortex AI usage per question, plus that warehouse - see [AI agents and MCP](#ai-agents-and-mcp) |

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

## Uninstall

`DROP APPLICATION <app> CASCADE;` removes the app, all of its data and the account-level objects
it owns (its warehouse). Export anything you need first, e.g.
`CREATE TABLE my_db.my_schema.lds_codes AS SELECT * FROM <app>.core.code_values_history;`

## Support

E-mail `support@<your-domain>` - we reply within 1 business day. Please include the app's version
(`SHOW APPLICATIONS LIKE '<app>';` - the `version`, `patch` and `label` columns) and, for an error, the
message the console showed. Documentation: https://github.com/mjrodgers1/lodestar-docs
