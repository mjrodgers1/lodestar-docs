---
layout: docs
title: "Mapping suggestions"
description: "Suggested target codes for source codes, from exact and alias matches to AI similarity."
permalink: /docs/mappings/
sidebar: docs
prev_url: /docs/using-the-app/
prev_title: "Using the app"
next_url: /docs/managed-tables/
next_title: "Managed tables"
---
{% raw %}
# Mapping suggestions {#mapping-suggestions}


On **Mappings -> Suggest**, give the source codes - pasted with a header row (`CODE`, optionally
`DESCRIPTION`), or a code column (and optionally a description column) of one of the tables or views
added for import. For each code the mapping set doesn't map yet, Lodestar suggests up to three target
codes of its code set, best first. Codes already mapped, or with a change waiting for approval, are
skipped and counted.

| Method | Score | When |
|---|---|---|
| Exact | 1.00 / 0.99 | Same code / same description (ignoring case) |
| Alias | 0.97 / 0.96 | The source code or description equals a text attribute of the target (e.g. `iso3`) / the source code is the target's description or the other way round |
| AI similarity | the cosine similarity, typically 0.4-0.9 | `ai_suggestions_enabled` on: the meaning of "code - description" is compared, so `Deutschland`, `Holland` or `Nippon` find `DE`, `NL` and `JP` |

A target found more than one way keeps its best score. Suggestions change nothing: the Steward ticks the
ones to accept (exact and alias matches, and AI ones above an adjustable score, are ticked to start with),
can pick another target, and submits them as ordinary mapping changes - approval, stewards, reasons and
history apply as usual. Each run is audited as `SUGGEST_MAPPINGS` (it doesn't count as editing for the plan).

**AI similarity** uses Snowflake Cortex `AI_EMBED` with the model `snowflake-arctic-embed-l-v2.0`
inside your account - no data leaves Snowflake, and there is no LLM prompt. If the app can't call
`AI_EMBED` (Cortex model access restricted by role without the grants in step 5, model not available in
your region and cross-region inference not allowed by
`CORTEX_ENABLED_CROSS_REGION`, or a Snowflake trial account - Snowflake doesn't offer `AI_EMBED` on
trial accounts), suggestions fall back to exact and alias matches and say why.

**Cost.** Embeddings are billed as Cortex AI usage on the app's warehouse: about 10 tokens per code.
Each target code is embedded once and kept, and embedded again only when its code or description
changes; source codes are embedded on every run. Matching itself (`VECTOR_COSINE_SIMILARITY`) costs no
tokens. To see the app's AI usage:

```sql
SELECT usage_time, function_name, model_name, tokens, token_credits
FROM SNOWFLAKE.ACCOUNT_USAGE.CORTEX_AISQL_USAGE_HISTORY
WHERE function_name = 'AI_EMBED' AND warehouse_id = (
  SELECT warehouse_id FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_EVENTS_HISTORY
  WHERE warehouse_name = '<APP_NAME>_WH' LIMIT 1)
ORDER BY usage_time DESC;
```

From SQL:

```sql
CALL <app>.core.suggest_mappings('CRM_COUNTRY',
  PARSE_JSON('["IRL", {"code": "D1", "description": "Deutschland"}]'));
CALL <app>.core.suggest_mappings_from_source('CRM_COUNTRY', 'CRM_DB.PUBLIC.CUSTOMERS', 'COUNTRY_CODE', 'COUNTRY_NAME');
```

The result lists `suggestions` (`source_code`, `source_description`, `candidates`: `target_code`,
`target_description`, `score`, `method` `EXACT` / `ALIAS` / `SEMANTIC`, `reason`), the counts `unmapped`,
`already_mapped`, `pending`, `truncated`, and `ai` (`enabled`, `used`, `model`, `embedded_targets`, and
`error` / `fix` when AI couldn't be used, or `trial` = true and no `fix` on a Snowflake trial account,
where no grant helps).
{% endraw %}
