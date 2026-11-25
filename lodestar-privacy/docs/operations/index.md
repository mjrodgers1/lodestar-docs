---
layout: docs
title: "Operations and data flows"
description: "What leaves your account, and how to check the app works, recover a stopped run, upgrade and uninstall."
permalink: /lodestar-privacy/docs/operations/
sidebar: privacy_docs
product: "Lodestar Privacy"
prev_url: /lodestar-privacy/docs/sql-reference/
prev_title: "SQL reference"
next_url: /lodestar-privacy/docs/plan-cost-support/
next_title: "Plan, cost and support"
---
{% raw %}
# Operations and data flows {#operations-and-data-flows}

## What leaves your account {#what-leaves-your-account}

| Where | What | Who sees it |
|---|---|---|
| The app's tables, in your account | Requests (references, dates, statuses), identifiers while a request is open, salted hashes afterwards, evidence, holds, audit trail, settings | Your users, through the app's roles |
| Error and warning logs, shared with us | Fixed messages, counts, procedure names, error types and Snowflake's error codes, plus your organisation, account and app version (see [Telemetry](/lodestar-privacy/docs/plan-cost-support/#telemetry)) | Us (the provider) |
| Emails (optional, sent by Snowflake) | Case references, statuses, dates and the name of the person who acted | The verified users in your account they go to |
| Anything else | Nothing: the app calls no external service and no AI model | - |

## Operations {#operations}

- **Is it working?** The Overview warns when approved requests have waited more than a day for a run that
  finished, when tables wait to be approved again, and when a table failed; Settings & audit -> Daily run shows
  the task's last runs. A task run that succeeded can still have failed tables - the run's line in Overview
  and the request say so.
- **A run that stopped** (the warehouse was suspended, the task timed out after 4 hours): its tables either
  committed or changed nothing. The next run - the next night, or *Run now* once 30 minutes have passed
  without a heartbeat - marks it failed and carries on.
- **Upgrades** keep every request, its evidence, the task's state and schedule. Requests approved before an
  upgrade stay approved at their tables' definitions then. Settings you saved are kept. The deadline: an install
  that kept 1.0.x's default (30 days) moves to one calendar month on upgrade, as it never stored the 30; one that
  stored 30 (changed, then set back) is reset to one calendar month once, on upgrade to 1.1.1 (audited as
  SET_SETTING by SYSTEM, with the value it was). Set 30 again in Settings if you want it; later upgrades leave it.
- **When the app can't run** (e.g. Snowflake disables it when a trial ends without a purchase): nothing runs,
  and open requests wait - their deadlines don't. Once it runs again the next run carries on; check Overview.
- **Before uninstalling**: export the evidence (Evidence -> Export all evidence, and the PDF of any request you
  must keep) - uninstalling deletes it. Keeping it, and for how long, is up to your retention policy.
- **Retention**: a person's identifiers - while their request is open, then deleted (see
  [How matching and erasure work](/lodestar-privacy/docs/using-the-app/#how-matching-and-erasure-work)); salted hashes of them, the evidence, holds
  and the audit trail - for as long as the app is installed; nothing anywhere else. There is no retention
  setting: if your policy says evidence goes after N years, export what you keep and uninstall, or ask us.
- **Backup and recovery**: everything the app keeps is in your account, and we hold no copy, so we can't
  restore it. Snowflake's own durability applies to it; your backup is the evidence export, as often as your
  recovery-point objective needs (it can be scripted: `core.erasure_evidence`, `core.erasure_attempts`,
  `core.audit_log`). The erasures themselves are recoverable only as the evidence says (Time Travel, then
  Fail-safe through Snowflake Support).
{% endraw %}
