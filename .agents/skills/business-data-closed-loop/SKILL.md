---
name: business-data-closed-loop
description: Use this skill for business-domain learning, FineBI lineage or dashboard review, report or reusable script development, metric verification, complex anomaly investigation, pitfall recording, and documentation or knowledge governance in the zzdnb/FineBI project. Use when the task requires learning, validation, reporting, reconciliation, or durable project knowledge closure. Do not use for quick concrete data questions or simple data retrieval; use business-data-query for those.
---

# Business Data Closed Loop Skill

## Purpose

Use this skill when business data work needs durable project knowledge rather than a one-off answer. It coordinates learning, reporting, review, investigation, documentation updates, and closure through `docs/CLOSURE_PROTOCOL.md`.

Concrete quick data questions belong to `business-data-query`.

## Required Reading

Default reading:

1. `AGENTS.md`
2. `docs/INDEX.md`
3. `docs/CLOSURE_PROTOCOL.md`
4. This `SKILL.md`

Then search for task-specific documents by business theme, platform, shop, metric name, table name, FineBI dashboard name, report name, or known pitfall. Read only relevant sections.

## Task Types

Classify the user request as one or more of:

- New business domain learning
- Existing business supplement learning
- FineBI dashboard or lineage analysis
- Metric definition or verification
- Report generation
- Reusable script generation
- Data validation
- Complex data issue investigation
- Pitfall recording
- Playbook creation or update
- Documentation closure
- Knowledge governance

## Division From `business-data-query`

- Use `business-data-query` for concrete business data questions, quick read-only retrieval, inventory, sales, advertising, profit, complaints, warehouse movement, returns, logistics, purchasing, and FineBI-vs-SQL result checks.
- Use this skill for learning business processes, database table mapping, report or script work, metric verification, FineBI lineage review, complex anomaly investigation, documentation updates, and durable knowledge closure.
- If `business-data-query` finds unclear metric definitions, unknown object types, missing mappings, unconfirmed field semantics, unclear NULL / 0 meaning, one-to-many join risk, new SQL templates, new pitfalls, or report/documentation needs, this skill handles the follow-up closure.
- Document updates and Closure Packet shape follow `docs/CLOSURE_PROTOCOL.md`; update only relevant locations.

## Clarification Rules

If key information is missing and will affect correctness, ask the user up to 5 questions.

Prioritize:

1. Business goal
2. Time range
3. Platform/shop/business scope
4. Output format
5. Core metrics or metric definition

If the task can proceed using reasonable assumptions, proceed and mark:

- 默认假设
- 待核实
- 推断
- 权限不足
- 需要用户确认

Do not repeatedly interrupt the user.

## Document Routing

Use `docs/CLOSURE_PROTOCOL.md` as the only shared closure protocol and document routing source.

Do not update every document mechanically. Update only documents related to the confirmed learning, metric, table, FineBI mapping, validation rule, pitfall, knowledge card, lineage finding, report output, or reusable workflow.

If new findings conflict with old documentation, append a conflict note instead of silently replacing history.

## Core Rules

- Do not invent tables, fields, SQL, metric definitions, or FineBI lineage.
- Mark unverified information as `待核实`.
- Mark inferred information as `推断`.
- Mark missing permission as `权限不足`.
- Database operations are read-only by default.
- Prefer one main Codex thread for this project; keep at most two active database-related threads.
- Reuse an existing project thread before creating a new one.
- Close MySQL connections and pools after each task; do not leave long-running database processes without explicit reason.
- Do not expose sensitive access material.
- Do not overwrite historical documentation. Append or add update records.
- For Excel / CSV / data-table reports, validation must match task risk. Do not run heavy workbook structure/content inspect just to satisfy the closed-loop process.
- Do not deliver or index non-business temporary artifacts such as `inspect.ndjson`, `*.xlsx.inspect.ndjson`, large preview logs, or temporary `node_modules` links.
- If the user says 临时取数, 临时改一下, 刚刚生成的表改一下, 先导出看一下, 不要固化, 不要沉淀, or 不用更新文档, follow the fast temporary query / iterative workbook mode in `docs/CLOSURE_PROTOCOL.md`: save result files under `reports/临时取数和报表/`, keep any one-off SQL / JS beside the temporary output only when needed for reproduction, do not promote temporary code into `scripts/`, reuse existing intermediate outputs, avoid unnecessary DB reruns, use minimal validation, and skip documentation updates unless the user later confirms a formal / durable version.

## Metric Rules

- WSC / wholesale cost and retail sales must not be mixed.
- ROI, ROAS, CTR, CVR, CPA, CPM, and CPC must be recomputed from numerator and denominator at aggregate level.
- Do not average row-level ratio metrics.
- Campaign analysis should prefer `campaign_id`.
- If only `campaign_name` is available, mention name-change risk.
- Profit and cost metrics must state their upstream source.
- If cost or profit upstream is not confirmed, mark it as `待核实`.

## Workflow

For every task:

1. Complete the Required Reading.
2. Classify the request and search only task-relevant documents.
3. Ask up to 5 clarification questions only when missing information affects correctness.
4. Execute the learning, report, verification, investigation, or closure task.
5. Validate only the dimensions relevant to the task, such as date coverage, grain, duplicates, mapping, nulls, aggregate reconciliation, ratio recomputation, or spot checks.
6. Run the relevant parts of `docs/CLOSED_LOOP_CHECKLIST.md` and update only relevant documents; in fast temporary query / iterative workbook mode, record the skip reason instead of mechanically updating report indexes or knowledge docs.
7. Final response includes the Closure Packet from `docs/CLOSURE_PROTOCOL.md`; use the full packet for complex reports, documentation governance, or explicit user requests, and compressed display only when the protocol allows.

## Shortcut Commands

When the user says `继续学习 xxx 业务`:

1. Read lightweight rules and index
2. Search related docs
3. Ask key questions if needed
4. Learn the business theme
5. Update related tables, metrics, dashboards, validation rules, pitfalls, and FineBI docs when new knowledge appears
6. Add a learning record and suggest the next task

When the user says `做 xxx 报表`:

1. Read lightweight rules and index
2. Search related docs and playbooks
3. Ask key questions if needed
4. Confirm metric definitions
5. Identify tables and dashboards
6. Generate SQL, scripts, and report only as needed
7. Validate with the minimum necessary checks
8. Save confirmed reusable or formal report output under `reports/固定流程取数和报表/` and reusable scripts under `scripts/`; in fast temporary mode, save result files under `reports/临时取数和报表/` and do not add scripts unless the user later confirms a durable version
9. Update process notes, metrics, tables, pitfalls, and playbooks if new knowledge appears

When the user says `记录这个坑`:

1. Classify the pitfall
2. Update `docs/business/已知坑点.md`
3. Update related metric, table, dashboard, and playbook docs if relevant

When the user says `闭环沉淀`:

Run documentation closure for the just-completed task using `docs/CLOSURE_PROTOCOL.md`.
