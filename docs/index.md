---
layout: docs
title: "Overview"
description: "What the app does, and its features."
permalink: /docs/
next_url: /docs/getting-started/
next_title: "Install and set up"
---
{% raw %}
# Lodestar Data Services documentation {#documentation}

Governed data changes inside your own Snowflake account. Lodestar is one simple, safe place for
business and data teams to maintain the data everything else depends on - customer segments, product
and cost-centre codes, financial hierarchies, country and currency codes, cross-system mappings, and
your own control tables such as price lists and thresholds. Business users work in a familiar grid;
every change gets maker/checker approval, effective dating and a complete audit trail, and is live in
your queries as soon as it is approved (or on its effective date). Your data stays in your account; only
error and warning logs reach us (see [Telemetry](/docs/plan-cost-support/#telemetry)).

## In this documentation {#in-this-documentation}

- [Install and set up](/docs/getting-started/) - The privileges the app asks for, the 5-minute start and post-install configuration.
- [Using the app](/docs/using-the-app/) - Every page of the consoles, and sizing and limits.
- [Mapping suggestions](/docs/mappings/) - Suggested target codes for source codes, from exact and alias matches to AI similarity.
- [Managed tables](/docs/managed-tables/) - Govern edits to your own tables where they live.
- [AI agents and MCP](/docs/ai-and-mcp/) - The Cortex Agent, the MCP server, the semantic view and the lookup functions.
- [Working with your existing tables](/docs/existing-tables/) - Lookups, translations, data-quality checks and taking existing tables under control.
- [SQL reference](/docs/sql-reference/) - The published views and functions, and every stored procedure.
- [Plan, cost and support](/docs/plan-cost-support/) - Telemetry, the trial and plan, running cost, uninstalling and support.

## Features {#features}

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
- **Check a column**: point Lodestar at a column of one of your tables or views and see which of its values
  aren't codes of a code set, or aren't mapped by a mapping set, today - and why: missing, valid only from
  a later date, no longer valid, waiting for approval, or blank - with the number of rows each affects.
  Changes nothing.
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
- **Email notifications** (optional): Admins are emailed when changes wait for their approval, and
  requesters when their changes are approved or rejected - sent by Snowflake, only to verified emails of
  users in your account. See step 10 below.
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
  user's own access - see [AI agents and MCP](/docs/ai-and-mcp/#ai-agents-and-mcp).
- **Audit log**: every submission, decision, change, correction and setting update is recorded.
- **Starter data**: sample ISO 3166 country and ISO 4217 currency code sets so you can try the app immediately.
{% endraw %}
