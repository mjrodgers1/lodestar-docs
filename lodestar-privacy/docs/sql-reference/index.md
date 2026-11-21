---
layout: docs
title: "SQL reference"
description: "The published views and every stored procedure."
permalink: /lodestar-privacy/docs/sql-reference/
sidebar: privacy_docs
product: "Lodestar Privacy"
prev_url: /lodestar-privacy/docs/using-the-app/
prev_title: "Using the app"
next_url: /lodestar-privacy/docs/plan-cost-support/
next_title: "Plan, cost and support"
---
{% raw %}
# SQL reference {#sql-reference}


Published views (schema `core`):

| View | Role | What |
|---|---|---|
| `requests` | viewer | One row per request: reference, status, dates, days left, identifier kinds and count (never the values), progress, rows affected. |
| `erasure_evidence` | viewer | One row per request and table: action, status, preview and affected rows, query id, recovery dates, hold, error. |
| `targets` | viewer | The registered tables. |
| `legal_holds` | viewer | Holds, active and released (identifier kinds only). |
| `erasure_runs` | viewer | Each run: when, how, totals. |
| `audit_log` | viewer | Every change. |
| `open_identifiers` | officer | The identifiers of open requests. |
| `email_contacts` | admin | Who gets emails. |

Procedures (schema `core`; each takes an optional trailing `actor` argument used only by the consoles):

| Procedure | Role |
|---|---|
| `submit_request(case_reference, identifiers VARIANT, received_on DATE, legal_basis, note)` | officer |
| `submit_requests(requests VARIANT)` - `[{"reference", "identifiers", "received_on", "legal_basis", "note"}]` | officer |
| `preview_request(request_id)`, `review_request(request_id, 'APPROVE' \| 'REJECT', comment)`, `withdraw_request(request_id, reason)` | officer |
| `register_target(...)`, `retire_target(name)`, `bound_tables()` | admin |
| `place_hold('SUBJECT' \| 'TARGET' \| 'ALL', identifiers, target_name, reason, case_reference)`, `release_hold(hold_id, note)` | admin |
| `run_erasures()`, `retry_request(request_id)`, `close_request(request_id, note)` | admin |
| `set_settings(VARIANT)`, `set_run_time(time, time_zone)`, `set_email_notifications(integration, test_recipient)`, `task_status()` | admin |
| `evidence_report(request_id, 'JSON' \| 'CSV' \| 'PDF')`, `whoami()` | viewer |

Example - feed a request from your DSAR tool:

```sql
CALL <app>.core.submit_request('DSR-2026-041',
  PARSE_JSON('[{"kind": "EMAIL", "value": "ann@example.com"}, {"kind": "CUSTOMER_ID", "value": "C-1001"}]'),
  '2026-11-18'::DATE, 'GDPR Art. 17', NULL);
```
{% endraw %}
