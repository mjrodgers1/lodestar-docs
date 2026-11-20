---
layout: docs
title: "Install and set up"
description: "The privileges the app asks for, the 5-minute start and post-install configuration."
permalink: /docs/getting-started/
prev_url: /docs/
prev_title: "Overview"
next_url: /docs/using-the-app/
next_title: "Using the app"
---
{% raw %}
# Install and set up {#install-and-set-up}

## Privileges requested {#privileges-requested}

| Privilege | Why |
|---|---|
| `CREATE WAREHOUSE` | The app creates one X-Small warehouse named `<APP_NAME>_WH` (standard Gen1, query acceleration off, auto-suspend 60 s, initially suspended). The app's consoles and the procedures they call run on it, so this compute is isolated from your workloads and its cost is attributable to the app (see Cost). Roles with `viewer` get USAGE on it because the agent's and MCP server's tools run there with the caller's rights (see [AI agents and MCP](/docs/ai-and-mcp/#ai-agents-and-mcp)); those roles can also run their own queries on it. |
| `READ SESSION` | Recommended. Lets Snowflake tell the app who is signed in when someone uses its consoles, so changes, approvals and audit entries are recorded against that person by Snowflake itself. Without it the console passes the signed-in user's name to the app. Snowflake does not grant it at install: see step 2 below. Calls you make from SQL always record your own user, either way. |
| `EXECUTE TASK` | New in 1.1.0. Lets the app run its one task, `jobs.run_checks`, on its own warehouse: once a day it runs the scheduled checks that are due (step 11). The task is suspended while nothing is scheduled, so it costs nothing until an Admin schedules a check. Snowflake grants this privilege automatically at install, and on the upgrade to 1.1.0 for existing installs. |

Optional references (each can hold as many objects as you like):

| Reference | Object type | Privilege | Why |
|---|---|---|---|
| `import_tables` | TABLE | SELECT | Read-only sources for Bulk Import - your own tables or tables in a database shared with you (e.g. a Marketplace listing). Columns are mapped to the code, description, sort order, valid-from date, parent code and typed attributes in the app. |
| `import_views` | VIEW | SELECT | The same for views, including secure views and views in shared databases. |
| `managed_tables` | TABLE | SELECT, INSERT, UPDATE, DELETE | Your own tables that Stewards edit through the app (see [Managed tables](/docs/managed-tables/#managed-tables)). The app reads them and writes only approved changes, to the key and editable columns an Admin registers (plus any created/updated by/at columns you choose). Tables in shared databases are read-only, so they can't be added here. |

The app only reads the objects you add for import, and writes only approved changes to the tables you
add as managed tables; it does not request access to any other objects in your account.

Optional database role in the SNOWFLAKE database:

| Role | Why |
|---|---|
| `SNOWFLAKE.CORTEX_EMBED_USER` | Only for AI mapping suggestions, and only if **Test AI access** asks for it: lets the app call Cortex `AI_EMBED` (embeddings only - no other AI functions or models). Not granted at install; see step 5 below. Without AI access, suggestions use exact and alias matches only. |

## Get started in 5 minutes {#get-started-in-5-minutes}

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

## Post-install configuration {#post-install-configuration}

Run these as the role that installed the app (replace `<app>` with your application name,
e.g. `Lodestar`).

### 1. Grant application roles to your users {#1-grant-application-roles-to-your-users}

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

### 2. (Recommended) Let Snowflake identify console users {#2-recommended-let-snowflake-identify-console-users}

Grant the app `READ SESSION` (see Privileges requested). In the Admin console go to
**Audit & Settings -> App info** and click **Grant READ SESSION**, or use SQL (needs a role with
MANAGE GRANTS, e.g. ACCOUNTADMIN):

```sql
GRANT READ SESSION ON ACCOUNT TO APPLICATION <app>;
```

**Check identity** on the same tab then shows who the app records you as and how it knows.

### 3. (Optional) Add tables or views to import from {#3-optional-add-tables-or-views-to-import-from}

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

### 4. (Optional) Manage your own tables {#4-optional-manage-your-own-tables}

To let Stewards edit one of your tables with approvals and audit, add it to the `managed_tables`
reference and register its key and editable columns (see [Managed tables](/docs/managed-tables/#managed-tables)):

```sql
CALL <app>.config.register_reference('MANAGED_TABLES', 'ADD',
  SYSTEM$REFERENCE('TABLE', '<db>.<schema>.<table>', 'PERSISTENT', 'SELECT', 'INSERT', 'UPDATE', 'DELETE'));
```

The role running `SYSTEM$REFERENCE` needs all four privileges on the table.

### 5. (Optional) Turn on AI mapping suggestions {#5-optional-turn-on-ai-mapping-suggestions}

Mappings -> **Suggest** always offers exact and alias matches. To add AI similarity, turn on **Suggest
mappings with Snowflake Cortex AI** in **Audit & Settings -> Settings** (setting `ai_suggestions_enabled`,
off by default) and press **Test AI access** on the same tab (or `CALL <app>.core.check_ai();`). In most
accounts the app can call `AI_EMBED` straight away. If it can't, the test shows the GRANTs that give it
access (needs ACCOUNTADMIN): the application role for the one model it uses - needed where Cortex model
access is restricted by role (`CORTEX_MODELS_ALLOWLIST` other than `'All'`), because code inside an app
doesn't inherit the PUBLIC role's model access - and the embeddings-only database role. See
[Mapping suggestions](/docs/mappings/#mapping-suggestions) for what it costs.

```sql
CALL <app>.core.set_setting('ai_suggestions_enabled', 'true');
CALL <app>.core.check_ai();   -- available: true, or the GRANTs below
GRANT APPLICATION ROLE SNOWFLAKE."CORTEX-MODEL-ROLE-SNOWFLAKE-ARCTIC-EMBED-L-V2.0" TO APPLICATION <app>;
GRANT DATABASE ROLE SNOWFLAKE.CORTEX_EMBED_USER TO APPLICATION <app>;
```

### 6. (Optional) Connect AI tools {#6-optional-connect-ai-tools}

Nothing to install: the agent **Lodestar reference data** and the MCP server are created by the app, for
everyone whose role holds `viewer`. The account needs Cortex Agents and Snowflake CoWork enabled. If your
account has a Snowflake CoWork object (it is created the first time someone opens CoWork or its settings),
CoWork shows only the agents added to it or to its profile - add Lodestar's agent once, see
[Connecting](/docs/ai-and-mcp/#ai-agents-and-mcp). The **Connect** page (every console) shows the MCP endpoint and the
set-up for each kind of client. Review what the app created before you grant it:

```sql
DESC AGENT <app>.core.lodestar_agent;
DESCRIBE MCP SERVER <app>.core.lodestar_mcp;
SELECT * FROM <app>.core.agent_status;   -- whether setup could create them, and why not
```

If your account uses a feature policy that blocks apps from creating agents or MCP servers
(`BLOCKED_OBJECT_TYPES_FOR_CREATION = (AGENTS, MCP_SERVERS)`), the app installs without them and
`core.agent_status` says so; the semantic view and lookup functions still work. To add them later, lift the
policy for the app and upgrade it. See [AI agents and MCP](/docs/ai-and-mcp/#ai-agents-and-mcp).

### 7. Open the app {#7-open-the-app}

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

### 8. (Optional) Give business users a direct link {#8-optional-give-business-users-a-direct-link}

Stewards who only need the app can use its **app-viewer URL**, which opens a console without the
rest of Snowsight. In Snowsight open the console, select **Share -> Copy link -> For app viewers**
(the URL has the form `https://app.snowflake.com/streamlit/<org>/<account>/#/apps/<url_id>`; `url_id`
is returned by `DESCRIBE STREAMLIT <app>.core.<console>`). To keep such users in the app viewer only:

```sql
ALTER USER <user_name> SET ALLOWED_INTERFACES = (STREAMLIT);
```

From a direct link, Snowflake cannot show permission dialogs, so adding import sources or managed
tables (steps 3 and 4) and granting READ SESSION (step 2) must be done in Snowsight or with the SQL above.

### 9. (Optional) Choose when upgrades happen {#9-optional-choose-when-upgrades-happen}

New versions of the app install automatically. To have them applied only in a window that suits you,
create a maintenance policy and apply it to the app:

```sql
CREATE MAINTENANCE POLICY <db>.<schema>.lds_upgrades
  SCHEDULE = 'USING CRON 0 2 * * SAT UTC' COMMENT = 'Upgrade apps on Saturday nights';
ALTER APPLICATION <app> SET MAINTENANCE POLICY <db>.<schema>.lds_upgrades;
```

Your data is kept across upgrades.

### 10. (Optional) Email approvers and requesters {#10-optional-email-approvers-and-requesters}

The app can email Admins when changes wait for their approval, and the person who submitted changes when
they are approved, rejected or fail. Snowflake sends the emails, only to verified email addresses of users
in your account (each user verifies theirs in Snowsight, under their profile). Once, as ACCOUNTADMIN (or a
role with CREATE INTEGRATION), create an email notification integration and let the app use it:

```sql
CREATE NOTIFICATION INTEGRATION IF NOT EXISTS lodestar_email TYPE = EMAIL ENABLED = TRUE;
GRANT USAGE ON INTEGRATION lodestar_email TO APPLICATION <app>;
```

Then an Admin turns them on in **Audit & Settings -> Settings -> Email notifications**, which first sends
them a test email (or from SQL: `CALL <app>.core.set_email_notifications('LODESTAR_EMAIL', 'you@example.com');`).
An existing integration works too, as long as the app is granted USAGE on it.

Who gets what:

- While notifications are on, the Steward and Admin consoles record each person's Snowflake user name and
  verified email (as Snowflake gives it to the console) when they open them - see `core.email_contacts`.
  People who only use SQL aren't recorded until they open a console.
- A submission that needs approval (changes, imports, reverts, managed-table edits) emails the Admins who
  opened the Admin console in the last 90 days, except the person who submitted it: who, how many changes
  to which code set, mapping set, hierarchy or managed table, and the reason and ticket. Codes and values
  are not included.
- A review emails each requester the outcome (approved, rejected, failed) and the reviewer's comment, unless
  they reviewed their own changes. Auto-applied changes send nothing.
- An email that can't be sent never stops a change: the console shows a warning instead.

Turning notifications off (same page, or `CALL <app>.core.set_email_notifications(NULL);`) also deletes
every recorded email. Revoking USAGE on the integration stops the emails at once.

### 11. (Optional) Schedule column checks {#11-optional-schedule-column-checks}

Any **Check a column** (step-by-step in [Pattern 3](/docs/existing-tables/#pattern-3---check-data-quality)) can run on its own. After
running a check, an Admin opens **Schedule this check** under the result and picks **Daily** or **Weekly** (and
the day). From then on:

- Once a day, at the time set in **Audit & Settings -> Settings -> Scheduled checks** (default 06:00 UTC; any
  time zone), the app's task runs the checks that are due, on the app's warehouse.
- Each run is kept (`core.check_runs`, 400 days) and compared with the one before. Values that became
  problems since then are emailed - when email notifications are on (step 10) - to whoever scheduled the
  check and the code set's or mapping set's named stewards, if their email is recorded. The email lists up
  to 20 of the new values with their rows and reason. A check that starts failing (for example because its
  table was removed from the app) emails the error once.
- What the check finds when you schedule it is the starting point and isn't emailed.
- The latest result of each scheduled check is shown above **Check a column** on the Code Values and Mappings
  pages, and **My work** lists the checks that found new problems in the last 7 days. Admins can **Run now**
  (the same as a scheduled run, but you aren't emailed yourself) or **Remove** a check there.

Up to 50 checks can be scheduled; scheduling the same table and column for the same set again only changes
when it runs. The task needs the `EXECUTE TASK` privilege, which Snowflake grants automatically. If the
Settings page says the app doesn't hold it (for example after a feature policy in your account limited
apps' tasks), an account administrator can run `GRANT EXECUTE TASK ON ACCOUNT TO APPLICATION <app>;`.
From SQL:

```sql
CALL <app>.core.schedule_check('CODE_SET', 'COUNTRY', 'MY_DB.SALES.ORDERS', 'SHIP_COUNTRY', 'WEEKLY', 'MON');
CALL <app>.core.set_check_time('07:30', 'Europe/Dublin');
SELECT * FROM <app>.core.check_schedules;
```
{% endraw %}
