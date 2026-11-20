---
layout: docs
title: "SQL reference"
description: "The published views and functions, and every stored procedure."
permalink: /docs/sql-reference/
prev_url: /docs/existing-tables/
prev_title: "Working with your existing tables"
next_url: /docs/plan-cost-support/
next_title: "Plan, cost and support"
---
{% raw %}
# SQL reference {#sql-reference}

## Querying reference data from SQL {#querying-reference-data-from-sql}

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

### Published objects {#published-objects}

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
| `core.lookup_code(code_set, code)`, `core.translate_code(mapping_set, source_code)`, `core.hierarchy_path(hierarchy, code)` | viewer | One code, one translation or one hierarchy position valid today, as JSON - see [AI agents and MCP](/docs/ai-and-mcp/#ai-agents-and-mcp) |
| `core.lodestar_sv` | viewer | Semantic view for Cortex Analyst over all of the above |
| `core.lodestar_agent`, `core.lodestar_mcp`, `core.agent_status` | viewer | Cortex Agent, MCP server, and whether setup could create them |
| `core.change_requests` | steward | Change requests, their reason / ticket and their outcome |
| `core.comments` | steward | Comments on a change request (`target_type` `REQUEST`) or a whole submission (`BATCH`), with the request's `batch_id`, container and key, author and time. Never edited. |
| `core.work_queue(user)` | steward | Table function: one row per submission and queue for a user - `TO_REVIEW`, `MY_OPEN`, `MY_DECIDED` (see My work above) - with counts, dates, reason, decision, review comment, error and comment count. Call it with `CURRENT_USER()` for your own queue. |
| `core.managed_tables` | steward | Registered managed tables: name, source table, key / editable / audit columns, whether rows may be added or deleted, the table's columns and types when registered, status, approval policy and stewards |
| `core.managed_field_history` | steward | Applied changes to managed tables, one row per column: `row_key` (the key, or a JSON array for a composite key), `key_values`, `action` (`INSERT`, `UPDATE`, `DELETE`), `column_name`, `old_value` / `new_value` (text, or JSON for JSON columns), who proposed and approved it, when, the reason and ticket |
| `core.audit_log` | admin | Full audit trail |
| `core.active_editors` | admin | Editors in the last 30 days (user, actions, first / last action) - what your plan counts, see [Trial and plan](/docs/plan-cost-support/#trial-and-plan) |
| `core.email_contacts` | admin | People recorded for email notifications: `user_name`, `email`, `last_seen`, and `gets_approval_requests` (opened the Admin console in the last 90 days). Empty while notifications are off - see step 10 of [Post-install configuration](/docs/getting-started/#post-install-configuration) |
| `core.check_schedules` | steward | Scheduled checks (step 11): `check_id`, the set (`target_type`, `target_name`), `source` and `column_name`, `frequency` (`DAILY`, `WEEKLY`) and `weekday`, who scheduled it, and its latest run: `last_run_at`, `last_status` (`OK`, `PROBLEMS`, `ERROR`), `last_run_type`, values and rows checked, % OK, `problem_values`, `new_problem_count`, `notified` (people emailed) and `error` |
| `core.check_runs` | steward | Every run of a scheduled check (400 days): `run_type` (`BASELINE` when scheduled, `MANUAL` = Run now, `SCHEDULE`), `run_by`, `status`, counts and % OK, `new_problem_count` and `new_problems` (up to 100 `{value, rows, status}`), `notified`, `error` |

### Stored procedures {#stored-procedures}

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
| `core.submit_row_changes(table_name, changes, reason, ticket_ref)` | steward | Submit inserts, updates and deletes for a managed table: `changes` is an array of `{"action": "INSERT" / "UPDATE" / "DELETE", "key": {column: value}, "values": {column: value}, "row_hash": ...}` (`row_hash` optional). Applied at once if the table's approval policy doesn't require approval. See [Managed tables](/docs/managed-tables/#managed-tables) |
| `core.suggest_mappings(mapping_set, sources)` | steward | Suggested target codes for source codes the mapping set doesn't map yet: `sources` is an array of codes or `{"code": ..., "description": ...}` objects (up to 1,000). Changes nothing. See [Mapping suggestions](/docs/mappings/#mapping-suggestions) |
| `core.suggest_mappings_from_source(mapping_set, source, code_column, description_column)` | steward | The same for the distinct values of a column of an import source (`source` = DATABASE.SCHEMA.NAME as listed by `core.import_sources`; `description_column` optional) |
| `core.check_values(target_type, target_name, source, column_name)` | steward | Which values of a column of an import source aren't codes of a code set (`target_type` `CODE_SET`) or aren't mapped by a mapping set (`MAPPING_SET`) today: per status (`OK`, `FUTURE`, `PENDING`, `ENDED`, `MISSING`, `BLANK`) the number of values and rows, the coverage in %, and the values that aren't OK, most rows first (up to 1,000). Changes nothing. See [Pattern 3](/docs/existing-tables/#pattern-3---check-data-quality) |
| `core.review_requests(request_ids, decision, comment)` | admin | `decision`: `APPROVE` or `REJECT` |
| `core.create_code_set(code_set, description, domain, owner_name)` | admin | Create a code set |
| `core.create_mapping_set(mapping_set, description, source_system, target_code_set)` | admin | Create a mapping set |
| `core.create_hierarchy(hierarchy, description, code_set)` | admin | Create a hierarchy |
| `core.set_container_status(container_type, name, status)` | admin | `container_type`: `CODE_SET`, `MAPPING_SET`, `HIERARCHY`, `MANAGED_TABLE`; `status`: `ACTIVE` or `RETIRED` |
| `core.set_setting(setting_key, setting_value)` | admin | `require_approval`, `allow_self_approval`, `require_change_reason`, `ai_suggestions_enabled` (`true`/`false`) |
| `core.set_settings(settings)` | admin | Several settings at once, all or nothing: `PARSE_JSON('{"require_approval": "true", "require_change_reason": "true"}')`. Every key is checked first; only changed values are saved and audited. Returns `settings` and `changed`. |
| `core.set_email_notifications(integration, test_recipient)` | admin | Turn email notifications on through an email notification integration the app may use (`test_recipient` optional: a test email is sent first, and nothing is saved unless Snowflake accepts it - a refusal returns Snowflake's `error` and the set-up SQL as `fix`), or off with `integration` NULL (also deletes every recorded email). Audited as `SET_EMAIL_NOTIFICATIONS` |
| `core.note_contact(email)`, `core.note_admin_contact(email)` | steward, admin | Used by the Steward and Admin consoles: record the caller's verified email for notifications, only while they are on (the admin one also makes the caller someone to ask for approval). Not audited |
| `core.schedule_check(target_type, target_name, source, column_name, frequency, weekday)` | admin | Schedule a `check_values` check: `frequency` `DAILY` or `WEEKLY` (`weekday` `MON` ... `SUN`). It runs once straight away (the baseline, not emailed). The same set, source and column again changes its schedule. Returns `check_id`, `baseline` and the task's state (`task`). Audited as `SCHEDULE_CHECK`. At most 50 |
| `core.run_check_now(check_id)` | admin | Run a scheduled check now, as the schedule would - recorded, and new problems emailed - except that you aren't emailed. Not audited (it changes nothing) |
| `core.unschedule_check(check_id)` | admin | Stop a scheduled check and delete its runs. Audited as `UNSCHEDULE_CHECK` |
| `core.run_due_checks()` | admin | What the app's task calls every day: run every scheduled check that is due (daily ones not run in the last 20 hours; weekly ones on their day, or once a week was missed). Returns `ran`, `failed` and each run |
| `core.set_check_time(run_time, time_zone)` | admin | When the task runs: `'HH:MM'` (24-hour clock) and a time zone such as `UTC` or `Europe/Dublin`. Audited as `SET_CHECK_TIME` |
| `core.check_task_status()` | admin | The scheduled checks task: `state` (`started` / `suspended`), `schedule`, `definition`, `warehouse`, `schedules`, `run_time` / `time_zone`, and its latest `runs` from Snowflake's task history (the next one has state `SCHEDULED`), or `error` |
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
{% endraw %}
