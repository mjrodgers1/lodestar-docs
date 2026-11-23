---
layout: docs
title: "Install and set up"
description: "The privileges and the table reference the app asks for, the 10-minute start and post-install configuration."
permalink: /lodestar-privacy/docs/getting-started/
sidebar: privacy_docs
product: "Lodestar Privacy"
prev_url: /lodestar-privacy/docs/
prev_title: "Overview"
next_url: /lodestar-privacy/docs/using-the-app/
next_title: "Using the app"
---
{% raw %}
# Install and set up {#install-and-set-up}

## Privileges requested {#privileges-requested}

| Privilege | Why |
|---|---|
| `CREATE WAREHOUSE` | The app creates one X-Small warehouse named `<APP_NAME>_WH` (standard Gen1, query acceleration off, auto-suspend 60 s, initially suspended). Its consoles, procedures and nightly run use it, so this compute is isolated from your workloads and its cost is attributable to the app (see [Cost](/lodestar-privacy/docs/plan-cost-support/#cost)). |
| `READ SESSION` | Recommended. Lets Snowflake tell the app who is signed in when someone uses its consoles, so requests, approvals and holds are recorded against that person by Snowflake itself. Without it the console passes the signed-in user's name to the app. Not granted at install: see step 2 below. |
| `EXECUTE TASK` | Lets the app run its one task, `jobs.run_erasures`, on its own warehouse: once a day it carries out approved requests and sends deadline reminders. The task is suspended while no request is open. Snowflake grants this privilege automatically at install. |

One reference, holding as many tables as you like:

| Reference | Object type | Privileges | Why |
|---|---|---|---|
| `personal_data_tables` | TABLE | SELECT, UPDATE, DELETE | The tables the app erases people from. It reads them to count the rows matching a request, and deletes or updates only rows matching an approved one - in the columns an Admin registered. Nothing else is changed; no INSERT or TRUNCATE is requested. |

The app requests no access to any other object in your account.

## Get started in 10 minutes {#get-started-in-10-minutes}

1. **Grant roles** (SQL below): `admin` to your privacy lead, `officer` to the people who handle requests (at
   least two, so one can approve what another submits), `viewer` to auditors. Grant the app `READ SESSION`.
2. **Add your tables** (Admin console -> Targets -> *Add a table*, or the SQL in step 3 below), then **register**
   each: the columns that identify a person, delete or anonymise, and the table's Time Travel days.
3. **Submit a test request** for a test person (Officer console -> Requests) and look at the counts.
4. **Approve it** as a second officer (Approvals), then **Run erasures now** (Admin console -> Settings & audit
   -> Daily run) - or wait for the nightly run.
5. **Download the evidence** (Evidence -> the request -> Prepare PDF).

## Post-install configuration {#post-install-configuration}

Run these as the role that installed the app (replace `<app>` with your application name, e.g.
`LODESTAR_PRIVACY`).

### 1. Grant application roles {#1-grant-application-roles}

```sql
-- Admins: register tables, place and release legal holds, run erasures, settings
GRANT APPLICATION ROLE <app>.admin   TO ROLE <privacy_admin_role>;
-- Officers: submit, approve (not their own), withdraw requests; see identifiers of open requests
GRANT APPLICATION ROLE <app>.officer TO ROLE <privacy_officer_role>;
-- Viewers (auditors): requests, evidence and reports - never identifiers
GRANT APPLICATION ROLE <app>.viewer  TO ROLE <privacy_auditor_role>;
```

Roles are hierarchical: `admin` includes `officer`, which includes `viewer`.

### 2. Let Snowflake identify console users (recommended) {#2-let-snowflake-identify-console-users-recommended}

```sql
GRANT READ SESSION ON ACCOUNT TO APPLICATION <app>;
```

### 3. Add the tables that hold personal data {#3-add-the-tables-that-hold-personal-data}

In the Admin console -> **Targets** -> **Add or remove tables**, *Add a table* opens Snowflake's permission
dialog where Snowsight offers it; where it doesn't, the page shows this SQL instead. Run it as a role that has
the app's `admin` role (step 1) and owns the table, then press *Reload tables*:

```sql
CALL <app>.config.register_reference('PERSONAL_DATA_TABLES', 'ADD',
  SYSTEM$REFERENCE('TABLE', '<db>.<schema>.<table>', 'PERSISTENT', 'SELECT', 'UPDATE', 'DELETE'));
```

Then register each table on the Targets page, or:

```sql
CALL <app>.core.register_target(
  'CRM_CUSTOMERS', '<DB>.<SCHEMA>.<TABLE>',
  PARSE_JSON('[{"column": "EMAIL", "kind": "EMAIL"}, {"column": "CUSTOMER_ID", "kind": "CUSTOMER_ID"}]'),
  'DELETE', NULL, 1, 'PERMANENT', 'CRM customers');
-- Keep the rows, clear the person's columns:
CALL <app>.core.register_target(
  'ORDERS', '<DB>.<SCHEMA>.<TABLE>',
  PARSE_JSON('[{"column": "CUSTOMER_ID", "kind": "CUSTOMER_ID"}]'),
  'ANONYMISE', PARSE_JSON('[{"column": "CUSTOMER_ID"}, {"column": "SHIP_TO_NAME", "method": "REDACT"}]'),
  7, 'PERMANENT', 'Orders kept for accounting');
```

The Time Travel days are the table's effective retention: its `DATA_RETENTION_TIME_IN_DAYS` (`SHOW TABLES`
shows it as `retention_time`) or, if greater, your account's minimum (`SHOW PARAMETERS LIKE
'MIN_DATA_RETENTION_TIME_IN_DAYS' IN ACCOUNT`) - at most 1 for a transient table. The evidence uses them to say
until when erased rows can be recovered. Snowflake doesn't let an app read these settings through a reference,
so they are declared here - update the target if you change the table or the account minimum.

### 4. Email notifications (optional) {#4-email-notifications-optional}

```sql
-- Once, as ACCOUNTADMIN (or a role with CREATE INTEGRATION) - Lodestar Data Services can use the same one:
CREATE NOTIFICATION INTEGRATION IF NOT EXISTS LODESTAR_EMAIL TYPE = EMAIL ENABLED = TRUE;
GRANT USAGE ON INTEGRATION LODESTAR_EMAIL TO APPLICATION <app>;
```

Then turn them on in Settings & audit -> Email. Snowflake sends only to verified email addresses of users in
your account. Emails name case references, statuses and dates - never who a request is about.

### 5. When it runs {#5-when-it-runs}

The nightly run is at 02:00 UTC. Change it in Settings & audit -> Daily run, or:

```sql
CALL <app>.core.set_run_time('03:30', 'Europe/Dublin');
```
{% endraw %}
