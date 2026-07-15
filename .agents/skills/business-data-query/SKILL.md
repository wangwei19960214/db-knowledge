---
name: business-data-query
description: Use this skill for concrete business data questions, quick read-only data retrieval, semantic pre-checks before SQL, SQL safety, and lightweight query closure in the zzdnb/FineBI project. Trigger when the user asks for specific inventory, availability, sales, advertising, profit, cost, complaints, reviews, returns, warehouse movement, logistics, purchasing, or FineBI-vs-SQL result checks. Do not use for business-domain learning, report development, FineBI lineage review, complex anomaly investigation, or documentation governance; use business-data-closed-loop for those.
---

# Business Data Query Skill / 智能问数 Skill

## Purpose

Use this skill to answer concrete business data questions with the smallest safe path: hit existing local knowledge first, use read-only SQL only when necessary and allowed, return the answer with the lightweight Closure Packet from `docs/CLOSURE_PROTOCOL.md`, and transfer to `business-data-closed-loop` when learning or documentation closure is needed.

This skill is not for learning the whole domain, building reports, or maintaining project knowledge assets.

## Required Reading

Default reading:

1. `AGENTS.md`
2. `docs/INDEX.md`
3. `docs/CLOSURE_PROTOCOL.md`
4. This `SKILL.md`

Read `docs/summaries/business-summary.md` when the question depends on broad business context, current high-risk assumptions, or cross-domain history; it is not mandatory for every quick query.

Then search only task-relevant sections of:

- `docs/knowledge/取数意图字典.md`
- `docs/knowledge/问法到指标映射.md`
- `docs/knowledge/数据知识沉淀表.md`
- `docs/knowledge/SQL模板库.md`
- `docs/business/指标口径字典.md`
- `docs/business/数据表目录.md`
- `docs/business/已知坑点.md`
- `docs/business/待核实问题清单.md`
- related `docs/finebi/` or `docs/playbooks/`

Do not read all business, FineBI, knowledge, or lineage documents by default.

## Query Pre-Checks

Before writing or running SQL, complete the minimum semantic checks below.

- Business theme: inventory, availability, out-of-stock, sales, advertising, profit, cost, complaints, reviews, returns, warehouse movement, logistics, purchasing, finance, FineBI reconciliation, or `待核实`.
- Input object type: Model, purchase SKU, platform SKU, Wayfair listed SKU, link SKU, store, platform, campaign, order, warehouse, ticket, review, or `待核实`.
- Metric intent: quantity, amount, ratio, status, days, count, occurrence, rank, anomaly reason, forecast result, or reconciliation difference.
- Time semantics: historical, current-day, future, or mixed; identify the relevant date field or mark `待核实`.
- Table type: source table, fact table, snapshot, forecast table, fact+forecast mixed table, intermediate table, result table, mapping table, dimension table, or FineBI aggregate.
- Field semantics: fact value, forecast value, status, flag, manual value, system-calculated value, or intermediate value.
- Null semantics: never treat NULL, blank, 0, missing row, not occurred, not applicable, and not generated as equivalent unless confirmed.
- Grain and unique key: confirm row grain, group-by dimensions, dedupe rules, and one-to-many join risk before aggregating.
- Mapping risk: check SKU, Model, platform SKU, listed SKU, link SKU, campaign, order, complaint, warehouse, store, and platform mappings before joining.

Do not default user input directly to a database field. For example, do not assume a Model is a purchase SKU, and do not assume `campaign_name` is a stable `campaign_id`.

## SQL Safety

Database access is read-only by default. Do not connect unless the user request requires it.

Forbidden without explicit authorization:

- `INSERT`
- `UPDATE`
- `DELETE`
- `DROP`
- `ALTER`
- `TRUNCATE`
- `CREATE`
- `REPLACE`

Do not read or output `.env`, account names, passwords, hosts, ports, tokens, connection strings, or other sensitive access material.

When querying large tables, first narrow the time range, object scope, and returned rows. Do not default to whole-database scans or unconditional `SELECT *` on large tables.

## Output Rules

Default output is business-facing:

- result or answer
- key metric definition and filters
- conclusion level from `docs/CLOSURE_PROTOCOL.md`
- exceptions, `待核实`, `推断`, or `权限不足`
- lightweight Closure Packet

Do not output full SQL, debug logs, connection steps, or intermediate details unless the user asks for SQL, process details, an export, or row-level detail.

## Transfer To Closed Loop

Stop treating the task as quick query work and transfer to `business-data-closed-loop` when any of these appear:

- new business theme without existing knowledge
- unclear metric definition
- unclear input object type
- missing mapping rule
- unconfirmed field semantics
- unclear NULL / 0 / missing-row meaning
- unclear table grain or one-to-many join risk
- need for a new SQL template or knowledge card
- new pitfall, FineBI lineage review, report output, script, or documentation closure

The transfer handoff should include the lightweight Closure Packet.

## Final Closure Packet

End each query task with the lightweight packet defined in `docs/CLOSURE_PROTOCOL.md`.
