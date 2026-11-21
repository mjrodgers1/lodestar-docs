---
layout: docs
title: "AI agents and MCP"
description: "The Cortex Agent, the MCP server, the semantic view and the lookup functions."
permalink: /docs/ai-and-mcp/
sidebar: docs
prev_url: /docs/managed-tables/
prev_title: "Managed tables"
next_url: /docs/existing-tables/
next_title: "Working with your existing tables"
---
{% raw %}
# AI agents and MCP {#ai-agents-and-mcp}


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

For large tables a join to `core.mappings_current` (see [Querying reference data from SQL](/docs/sql-reference/#querying-reference-data-from-sql))
is faster than calling a function per row.

**Who can use them.** Any role with `viewer` that can use Cortex: `SNOWFLAKE.CORTEX_USER` (granted to PUBLIC
by default) or `SNOWFLAKE.CORTEX_AGENT_USER`. The account needs Cortex Agents and Snowflake CoWork enabled.
On a Snowflake **trial account** Snowflake refuses agent runs ("Access denied for trial accounts"), and with
them the MCP server's `ask-lodestar` and lookup tools; the semantic view, the `query-reference-data` tool
and the lookup functions in SQL still work.
The tools run on the app's warehouse `<APP_NAME>_WH`; that is why `viewer` has USAGE on it (see
[Privileges requested](/docs/getting-started/#privileges-requested)).

**Connecting.**

- **Snowflake CoWork**: users whose role holds `viewer` can use the agent at https://ai.snowflake.com (the
  **CoCo** tab in Snowsight is Cortex Code, a different assistant that doesn't use the agent). Whether it
  appears by itself depends on how CoWork is set up in your account (`SHOW SNOWFLAKE INTELLIGENCES;` and
  `SHOW AGENTS IN SNOWFLAKE INTELLIGENCE <si_name>;` tell you):
  - **No CoWork object**: nothing to do - it appears for those users.
  - **A CoWork object without profiles**: an account administrator adds it once:
    `ALTER SNOWFLAKE INTELLIGENCE <si_name> ADD AGENT <app>.CORE.LODESTAR_AGENT;`
  - **CoWork profiles** (the object lists an agent such as `SNOWFLAKE_INTELLIGENCE.PROFILES.DEFAULT_PROFILE`):
    a profile routes each question to the agents it names, so add `<app>.CORE.LODESTAR_AGENT` to the
    profile's subagents - in **AI & ML -> CoWork Admin -> Agents -> (profile) -> Edit profile**, or with
    `ALTER AGENT ... MODIFY LIVE VERSION SET SPECIFICATION` (keep the rest of its specification).
    Listing it gives no one access by itself: users still need `viewer`.
- **Cortex Code**: add the server to `~/.snowflake/cortex/mcp.json`; Cortex Code (1.1.65 or later) signs in
  with a connection from your `connections.toml`:

  ```json
  {"mcpServers": {"lodestar": {"type": "http",
    "url": "https://<account_url>/api/v2/databases/<app>/schemas/CORE/mcp-servers/LODESTAR_MCP",
    "snowflake_connection": "<your_connection_name>"}}}
  ```

  In **Cortex Code Desktop** that connection must be the **active** connection: otherwise the server is
  refused (401) and its tools don't appear. Start a new session after switching. In the **Cortex Code CLI**,
  check under `/mcp` that `lodestar` is switched on and connected (select it and press T to turn it on).
  The connection's role needs `viewer`.

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
(see [Telemetry](/docs/plan-cost-support/#telemetry)), so questions and answers are not shared with the provider.

**Cost.** An agent question or a `query-reference-data` call uses Cortex AI credits (Cortex Agents and
Cortex Analyst pricing), plus a few seconds on `<APP_NAME>_WH`; an agent answer takes about 15 s. The
lookup functions are ordinary queries. All of it is billed to your account - see [Cost](/docs/plan-cost-support/#cost).
{% endraw %}
