---
layout: docs
title: "Using the app"
description: "Every page of the consoles, and sizing and limits."
permalink: /docs/using-the-app/
prev_url: /docs/getting-started/
prev_title: "Install and set up"
next_url: /docs/mappings/
next_title: "Mapping suggestions"
---
{% raw %}
# Using the app {#using-the-app}


The sidebar groups pages into **My work** (My work, Home), **Reference data** (Code Sets, Code Values,
Mappings, Hierarchies, Bulk Import), **Managed tables**, **Governance** (Approvals, Audit & Settings) and
**AI tools** (Connect). Each console shows only the pages its role can use. Stewards and Admins open on **My work**; the Viewer console on Home.

| Page | Console | What |
|---|---|---|
| My work | Steward / Admin | One row per submission: **Awaiting my review** (Admins: pending changes they may approve - not their own unless self-approval is on), **My open submissions**, and **Decided in the last 14 days** (rejected, failed or approved by a reviewer, with the review comment and any error; changes applied automatically are listed only if they failed). Pick a submission to read or add comments. The sidebar count is submissions awaiting your review plus those returned to you (rejected or failed); it refreshes after each action or with **Refresh**. |
| Home | all | Counts, and for Admins the set-up checklist and plan usage. |
| Code Sets | all / Admin | Browse code sets. Admins create, retire and reactivate them, set the approval policy and stewards, and define attributes. |
| Code Values | all / Steward | View codes as of any date. Stewards edit a grid (add, change, delete = retire) and submit with an effective date. **Check a column** (Steward): pick a column of an import source and see which of its values aren't codes of the code set today (see [Pattern 3](/docs/existing-tables/#pattern-3---check-data-quality)). |
| Mappings | all / Steward | Map source codes to target codes. Admins create mapping sets. **Suggest** (Steward): paste source codes or pick a column of an import source, and review the suggested targets - each with its score, method (exact, alias, AI similarity), reason and the next best alternatives; tick the ones to accept (change the target if needed) and submit them as mapping changes. **Check a column** (Steward): which values of a column the mapping set doesn't map today. |
| Hierarchies | all / Steward | View the tree. Stewards set each code's parent (empty = root). |
| Approvals | Steward / Admin | Stewards track pending requests; Admins approve or reject them, per request or a whole batch (one grid save, import or revert) at a time. A change that fails validation is marked `FAILED` with the reason; the others are applied. Under **Recent decisions**, select applied changes and **Revert** them. Every grid has a **comments** count; pick a request or submission under it to read its thread (comments plus review decisions) and add a comment. |
| Bulk Import | Steward | **Paste or upload rows**: paste cells copied from Excel or CSV text, or upload a CSV / TSV / TXT or Excel (.xlsx; pick the sheet) file of up to 25 MB - the first row is the header and must include `CODE`; check the preview (empty rows are skipped) and import. Up to 50,000 rows per run. Or import **from a table or view**: pick one of the sources an Admin added (your own or shared databases), check the preview, map its columns to the code, description, sort order, valid-from date, parent code (with a hierarchy) and typed attributes, and run it. The mapping is remembered per source and code set. Admins add and remove sources here. |
| Managed tables | Steward / Admin | **Edit rows**: pick a table, search, and edit its rows in a grid (editable columns are typed; key columns marked; other columns shown for reference). Add or delete rows where the table allows it, give a reason and ticket, and submit. Key values of existing rows can't be edited (delete the row and add a new one). **Pending changes**: each change as before -> after, with comments. **History**: every applied change per column, for the whole table or one row, with comments. **Set up** (Admin): add or remove tables, register key / editable / audit columns and what Stewards may do, approval policy, stewards, retire. See [Managed tables](/docs/managed-tables/#managed-tables). |
| Audit & Settings | Admin | Search the audit log, toggle approval requirements, mandatory change reasons and AI mapping suggestions (with **Test AI access**), turn email notifications on or off and see who is recorded for them (step 10), check how users are identified (READ SESSION) and event sharing. The Settings tab always shows the values as stored now, and **Save settings** changes only the switches you changed, so two Admins can't undo each other's changes. Other pages read the settings when the console opens - press **Refresh** in the sidebar after another Admin changes them. |
| Connect | all | Whether the Cortex Agent and MCP server are available, the MCP endpoint, the MCP tools, how to connect Snowflake CoWork, Cortex Code, other MCP clients and your own agents, and example questions. See [AI agents and MCP](/docs/ai-and-mcp/#ai-agents-and-mcp). |

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

## Sizing and limits {#sizing-and-limits}

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
{% endraw %}
