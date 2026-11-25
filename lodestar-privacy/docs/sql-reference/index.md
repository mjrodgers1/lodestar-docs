---
layout: docs
title: "SQL reference"
description: "The published views and every stored procedure."
permalink: /lodestar-privacy/docs/sql-reference/
sidebar: privacy_docs
product: "Lodestar Privacy"
prev_url: /lodestar-privacy/docs/using-the-app/
prev_title: "Using the app"
next_url: /lodestar-privacy/docs/operations/
next_title: "Operations and data flows"
---
{% raw %}
# SQL reference {#sql-reference}


Published views (schema `core`):

| View | Role | What |
|---|---|---|
| `requests` | viewer | One row per request: reference, status, dates, days left, identifier kinds and count (never the values), progress, rows affected. |
| `erasure_evidence` | viewer | One row per request and table: action, status, preview and affected rows, query id, recovery dates, hold, error. |
| `targets` | viewer | The registered tables, with their revision (`config_rev`: up by one at each change an approval must cover). |
| `legal_holds` | viewer | Holds, active and released (identifier kinds only). |
| `erasure_runs` | viewer | Each run: when, how, totals. |
| `erasure_attempts` | viewer | Every attempt to erase a request's part of a table: status, rows, query id, the table, action and revision it ran against, error. Never changed; a retry adds a row. |
| `audit_log` | viewer | Every change. |
| `open_identifiers` | officer | The identifiers of open requests. |
| `email_contacts` | admin | Who gets emails. |

Procedures (schema `core`). Every procedure returns a JSON object: `{"ok": true, ...}` with the fields below,
or `{"ok": false, "error": "<why>"}`. A refused call changes nothing; an unexpected error rolls back the change it
was making (a run keeps the tables it finished - each is its own transaction). Each also takes an optional
trailing `actor` argument, used only by the consoles. Arguments with a default can be left out.

| Procedure | Role | Returns (besides `ok`) |
|---|---|---|
| `submit_request(case_reference VARCHAR, identifiers VARIANT, received_on DATE DEFAULT NULL, legal_basis VARCHAR DEFAULT NULL, note VARCHAR DEFAULT NULL)` - identifiers `[{"kind", "value"}]`; received_on defaults to today | officer | `request_id`, `reference`, `due_on`, `preview` `[{target, source, rows}]`, `rows`, `repeat_of` (earlier references about the same person), `unmatched_kinds` |
| `submit_requests(requests VARIANT)` - `[{"reference", "identifiers", "received_on", "legal_basis", "note"}]`, at most 500, all or nothing | officer | `batch_id`, `submitted`, `requests` `[{request_id, reference, due_on}]`, `repeat_of` |
| `preview_request(request_id VARCHAR)` | officer | `preview`, `rows` |
| `review_request(request_id VARCHAR, decision VARCHAR, review_comment VARCHAR DEFAULT NULL)` - `APPROVE` or `REJECT` (a comment is required to reject); not the submitter's own | officer | `status`, `task`. Refused if another officer decided first. |
| `approve_scope(request_id VARCHAR, review_comment VARCHAR DEFAULT NULL)` - approves the request's tables that wait for approval again; not the submitter's own | officer | `approved_parts`, `statuses` |
| `withdraw_request(request_id VARCHAR, reason VARCHAR)` - before anything was erased | officer | `status` |
| `register_target(target_name VARCHAR, source VARCHAR, subject_columns VARIANT, erase_action VARCHAR, anonymise_columns VARIANT DEFAULT NULL, retention_days NUMBER DEFAULT 1, table_kind VARCHAR DEFAULT 'PERMANENT', description VARCHAR DEFAULT NULL)` - subject_columns `[{"column", "kind"}]`; erase_action `DELETE` or `ANONYMISE`; anonymise_columns `[{"column", "method": "NULL" \| "REDACT"}]` | admin | `target_name`, `created`, the definition, `config_rev`, `parts_to_review` |
| `retire_target(target_name VARCHAR)`, `bound_tables()` | admin | `skipped_parts`, `statuses` / `tables` |
| `place_hold(scope VARCHAR, identifiers VARIANT DEFAULT NULL, target_name VARCHAR DEFAULT NULL, reason VARCHAR DEFAULT NULL, case_reference VARCHAR DEFAULT NULL)` - scope `SUBJECT` (identifiers), `TARGET` (target_name) or `ALL`; a reason is required | admin | `hold_id`, `covers_open_parts`, `rows_checked_each_run` |
| `release_hold(hold_id VARCHAR, note VARCHAR)` | admin | `hold_id`, `task` |
| `run_erasures()` - refused while another run is in progress | admin | `run_id`, `requests`, `targets`, `rows`, `failed`, `held`, `review`, `statuses`, `reminded`, `task` |
| `retry_request(request_id VARCHAR)`, `close_request(request_id VARCHAR, note VARCHAR)` | admin | `status` |
| `set_settings(settings VARIANT)` - `{"allow_self_approval": true/false, "due_days": 0-365 (0 = one calendar month), "reminder_days": 0-90}`, all or nothing | admin | `settings`, `changed` |
| `set_run_time(run_time VARCHAR, time_zone VARCHAR DEFAULT 'UTC')` - `HH:MM` (or `HH.MM`), an IANA zone | admin | `run_time`, `time_zone`, `schedule`, `task` |
| `set_email_notifications(integration VARCHAR, test_recipient VARCHAR DEFAULT NULL)` - empty integration turns them off | admin | `enabled`, `integration`, `tested` |
| `task_status()` | admin | `state`, `schedule`, `run_time`, `time_zone`, `open_requests`, `runs` |
| `evidence_report(request_id VARCHAR, report_format VARCHAR DEFAULT 'JSON')` - `JSON`, `CSV` or `PDF` | viewer | `report` / `csv` / `pdf_base64` |
| `whoami()` | viewer | `user`, `user_source`, `application`, `warehouse`, `settings` |

Calls are safe to repeat: a second identical decision is refused ("already decided") rather than applied twice,
and `submit_request` refuses a reference that is already open.

Example - feed a request from your DSAR tool:

```sql
CALL <app>.core.submit_request('DSR-2026-041',
  PARSE_JSON('[{"kind": "EMAIL", "value": "ann@example.com"}, {"kind": "CUSTOMER_ID", "value": "C-1001"}]'),
  '2026-11-18'::DATE, 'GDPR Art. 17', NULL);
```
{% endraw %}
