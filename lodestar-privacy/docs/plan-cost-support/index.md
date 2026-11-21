---
layout: docs
title: "Plan, cost and support"
description: "Telemetry, the trial and plan, running cost, uninstalling and support."
permalink: /lodestar-privacy/docs/plan-cost-support/
sidebar: privacy_docs
product: "Lodestar Privacy"
prev_url: /lodestar-privacy/docs/sql-reference/
prev_title: "SQL reference"
---
{% raw %}
# Plan, cost and support {#plan-cost-and-support}

## Telemetry {#telemetry}

The app shares its **error and warning logs** with us (required while it is installed), so we can find and fix
faults. They name the app procedure and the problem - for example a table that couldn't be erased and why -
and Snowflake adds your organisation and account names and the app version. They never contain a person's
identifiers, the values in your tables, emails or user names: the app's messages are written so they can't
(and a test checks it). **Usage logs** (timings and counts per call) are shared only if you turn them on.
Our privacy notice is at https://www.lodestardataservices.com/privacy/.

## Trial and plan {#trial-and-plan}

Every feature free for 30 days. Then USD 990 a month or USD 9,900 a year: unlimited requests, tables and users,
no implementation fee. With Lodestar Data Services as well: ask us for the suite price.

## Cost {#cost}

Apart from the plan, the app uses Snowflake compute and storage in your account: its X-Small warehouse while
someone uses the consoles and for the nightly run (typically a minute or two while requests are open; none
while none are), plus the scans of your tables described under [Sizing and limits](/lodestar-privacy/docs/using-the-app/#sizing-and-limits). Its
own storage is a few MB.

## Uninstall {#uninstall}

`DROP APPLICATION <app> CASCADE;` removes the app, its warehouse, its task and everything it stores -
including the evidence. Export the evidence first (Evidence -> Export all evidence) if you need to keep it.
Your tables are not touched.

## Support {#support}

E-mail support@lodestardataservices.com. We reply within one business day. Please never send us a person's
identifiers - the case reference and the error message are enough. Documentation:
https://www.lodestardataservices.com/lodestar-privacy/docs/
{% endraw %}
