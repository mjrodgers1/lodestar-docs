---
layout: docs
title: "Overview"
description: "What the app does, and its features."
permalink: /lodestar-privacy/docs/
sidebar: privacy_docs
product: "Lodestar Privacy"
next_url: /lodestar-privacy/docs/getting-started/
next_title: "Install and set up"
---
{% raw %}
# Lodestar Privacy documentation {#documentation}

Right-to-erasure requests carried out inside your own Snowflake account, with maker/checker approval, legal
holds and evidence. Lodestar Privacy is one controlled place to erase a person from the tables that hold
their data: an officer records the request, a second officer approves it after seeing what it will touch,
the app deletes or anonymises the matching rows, and every request ends with a report of what was erased,
when, by which Snowflake query, and until when it could still be recovered. Your data stays in your account;
only error and warning logs reach us, and they never contain a person's identifiers (see
[Telemetry](/lodestar-privacy/docs/plan-cost-support/#telemetry)).

Part of the Lodestar suite, with Lodestar Data Services.

## In this documentation {#in-this-documentation}

- [Install and set up](/lodestar-privacy/docs/getting-started/) - The privileges and the table reference the app asks for, the 10-minute start and post-install configuration.
- [Using the app](/lodestar-privacy/docs/using-the-app/) - The consoles, how a person is matched and erased, the evidence, and sizing and limits.
- [SQL reference](/lodestar-privacy/docs/sql-reference/) - The published views and every stored procedure.
- [Plan, cost and support](/lodestar-privacy/docs/plan-cost-support/) - Telemetry, the trial and plan, running cost, uninstalling and support.

## Features {#features}

- **Targets**: the tables that hold personal data, added once by an Admin. For each, say which columns
  identify a person (email, customer id, phone or your own kind) and whether a person's rows are **deleted**
  or **anonymised** - kept, with their identifying columns emptied or replaced by `[erased]`, e.g. orders you
  must keep for accounting.
- **Requests**: an officer enters the case reference, the date it was received and the person's identifiers -
  in the console, by uploading a CSV or Excel file of many requests, or from SQL (so your DSAR tool can feed
  it). The due date defaults to one month after receipt.
- **Preview**: as soon as a request is submitted the app counts the rows each table holds about the person.
  Counts only - no row is shown.
- **Maker/checker approval**: another officer approves or rejects every request after seeing its identifiers
  and the counts. Self-approval is off by default.
- **Legal holds**: stop erasure of one person, one table or everything - for litigation, a regulator's
  investigation or a retention duty - until the hold is released. What a hold covers waits and runs when it
  is released.
- **Carried out for you**: approved requests run every night on the app's own warehouse (or straight away
  with **Run now**). Each table is erased in one transaction that counts each request's rows, runs one
  DELETE or UPDATE, checks it changed exactly those rows and records the evidence - or changes nothing.
- **Evidence**: per request and table: the action, the rows affected, the time, the Snowflake query id, and
  the dates until which Time Travel and Fail-safe could still recover the rows. Download it as a PDF, CSV or
  JSON report, or query it in SQL.
- **Personal data minimised**: the app keeps a person's identifiers only while their request is open, and
  deletes them when it closes; afterwards it keeps only salted hashes, so holds and repeat requests still
  match. Identifiers are never written to logs, the audit trail, reports or emails, and the Viewer console
  (for auditors) never shows them.
- **Deadlines and reminders**: open requests are listed oldest deadline first; when email notifications are
  on, officers are emailed requests waiting for approval and a daily list of those due soon or overdue.
- **Audit log**: every request, decision, hold, registration, run and setting change is recorded.
{% endraw %}
