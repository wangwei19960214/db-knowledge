# 项目级 Agent 协作规则

本项目是围绕 `zzdnb` 数据库、FineBI、库存/销售/断货/报表的数据协作项目。

## 工作原则

- 优先按具体业务需求推进，不需要先完整学习全库。
- 默认采用“定向取数 -> 验证口径 -> 生成产物 -> 沉淀经验”的流程；当用户明确临时取数、快速迭代或不固化时，按“快速临时取数 / 迭代改表模式”执行。
- 数据库操作默认只读，除非用户明确要求写入、改表或执行会改变数据的过程。
- 不在回复、文档、日志、脚本输出中泄露数据库凭据或其他敏感信息。
- 遇到已有文档、脚本或报表时，优先复用和延续当前项目口径。

## 线程与数据库连接控制

- 当前项目尽量只使用 1 个主 Codex 线程处理数据库、FineBI、报表和业务学习需求。
- 如确有必要并行，最多保留 2 个活跃的数据库相关 Codex 线程。
- 新建线程前，应先检查当前项目是否已有活跃线程可以继续使用；能续用就不要新建。
- Codex 线程本身不等于 MySQL 连接，但并行线程容易同时发起数据库查询，应主动避免。
- 所有数据库脚本必须在查询完成后关闭连接池或连接，例如调用 `pool.end()` 或 `connection.end()`。
- 不要启动长期占用 MySQL 连接的后台进程；如确需长任务，应说明原因、预估时长和中断方式。
- 遇到数据库超时、连接数异常或 RDS 白名单问题时，优先暂停新增数据库任务，先排查当前连接状态。

## 需求完成后的沉淀要求

每个数据库、FineBI、报表或智能问数需求完成后，都必须按 `docs/CLOSURE_PROTOCOL.md` 判断是否需要沉淀。需要沉淀时至少记录：

- 涉及的表、视图、存储过程或 FineBI 数据集。
- 指标口径和关键过滤条件。
- SQL、脚本路径或可复用查询方式。
- 产物路径，例如 `reports/` 下的文件。
- 异常数据、权限限制、口径差异和踩坑点。

## 快速临时取数 / 迭代改表模式

当用户说“临时取数”“临时改一下”“刚刚生成的表改一下”“先导出看一下”“不要固化”“不要沉淀”“不用更新文档”，或明显是在同一张已生成报表上连续迭代时，默认进入本模式。

本模式目标是快和少扰动：

- 优先复用刚生成的 JSON / CSV / Excel / 中间结果；没有必要不要重连数据库或重跑全量 SQL。
- 只做用户要求的改动和最小必要校验，例如行数、关键过滤条件、日期范围、采购 SKU 后缀、文件可打开。
- 本模式生成的 Excel / CSV / JSON / Markdown 默认保存到 `reports/临时取数和报表/<YYYY-MM-DD_任务名>/`；默认只保存结果文件，SQL / 脚本不进入 `scripts/`、不写入索引；如确需复现，可把 `.sql` / `.js` 伴随临时产物放在同目录，但仍不算固定流程。
- 全量查询中间数据、构建缓存、`*_rows.json` 等不得作为手工补丁文件处理；优先写入 `/private/tmp`、`.tmp/` 或已忽略路径，并由生成脚本内部清理，避免在变更视图中出现几千 / 几万行新增或删减。
- 除非用户明确确认固定流程、正式版、固化、沉淀或加入索引，不写入 `reports/固定流程取数和报表/`，不把临时代码提升为 `scripts/` 下的可复用脚本。
- 默认不更新 `reports/INDEX.md`、`docs/business/业务流程需求沉淀.md`、`docs/knowledge/`、`docs/lineage/` 等沉淀文档；最终回复要说明“本次为临时 / 迭代输出，未更新文档”的原因。
- 普通数据 Excel 优先用静态轻量方式生成，例如 `openpyxl` 或项目既有轻量脚本；不得默认使用会生成 `inspect.ndjson`、`*.xlsx.inspect.ndjson` 或大型 sidecar 日志的重型 inspect / artifact 流程。
- 如果快速迭代中确认了新的可复用口径、字段含义、坑点、SQL 模板、可复用脚本或正式报表产物，必须在最终回复标注；用户确认正式版、固化或沉淀后，再按闭环协议更新相关文档。

## 报表 / Excel 校验与临时文件控制

- 普通 Excel、CSV 或数据表交付，默认只做最小必要校验：行数、主键或业务唯一键、日期 / 月份覆盖、关键空值 / 缺失、公式错误扫描、最终文件是否存在且可打开。
- 校验强度必须匹配任务风险，不允许为了“闭环”机械执行重型视觉 / 结构检查。
- 禁止默认执行大范围 `workbook.inspect({ kind: "table" | "region" | "computedStyle" })`、全表结构 inspect、整表 NDJSON 输出或类似会生成大型 sidecar 日志的操作。
- `inspect.ndjson`、`*.xlsx.inspect.ndjson`、大型预览日志、临时 `node_modules` 链接等非业务临时文件，不得作为交付产物，不得写入 `reports/INDEX.md`。
- 预览图只在布局、视觉质量或演示效果是任务核心时生成；普通业务数据表不默认生成预览图。
- 如工具自动生成大型 sidecar / inspect 文件，交付前必须删除，并在最终回复说明已清理。
- 对正式演示型报表、复杂财务模型或用户明确要求视觉验收的文件，可以做渲染和结构检查，但必须先说明必要性，并限制输出范围和文件体积。

## 文档分工

- `AGENTS.md`：项目级协作规则，供所有后续 agent 线程阅读。
- `docs/business/数据表目录.md`：长期数据库业务地图，记录主题、核心表、字段口径、上下游和更新逻辑。
- `docs/business/业务流程需求沉淀.md`：单次需求经验库，按需求追加目标、用表、口径、SQL、脚本、产物和坑点。
- `docs/finebi/FineBI库存数据梳理.md`：FineBI 库存专项资料，继续维护库存、仓库可用、预计到仓、断货、售罄等看板链路。
- `docs/knowledge/`：智能取数知识层，记录取数意图、知识卡片、SQL 模板、问法映射和安全边界。
- `docs/lineage/`：数据血缘层，记录结果表、过程、刷新任务和报表链路。
- `docs/CLOSURE_PROTOCOL.md`：`business-data-query` 和 `business-data-closed-loop` 共用的唯一闭环协议。
- `docs/CLOSED_LOOP_CHECKLIST.md`：每次任务结束前的闭环检查清单。
- `reports/INDEX.md`：报表产物索引，不替代实际报表文件。
- `reports/临时取数和报表/`：默认存放临时取数、一次性导出、迭代改表和未确认正式版产物，不默认进入 `reports/INDEX.md`。
- `reports/固定流程取数和报表/`：存放已经形成 REQ、可复用流程、脚本默认输出或正式 / 半正式报表的产物。
- `scripts/`：可复用查询、分析和报表生成脚本。

## 纯净文档仓库同步

- 每日给外部协作者使用的纯净版仓库由 `npm run docs:clean:sync` 生成；默认目标目录为 `/Users/niuniu/Documents/库存查询和整合-纯净版`，可通过 `CLEAN_DOCS_REPO_DIR` 覆盖。
- 同步白名单只包含 `AGENTS.md`、`docs/**/*.md`、`.agents/skills/**/SKILL.md`；不会同步 `.env*`、`config/`、`scripts/`、`reports/`、`node_modules/`、`.tmp/`、Excel / CSV / JSON / NDJSON / ZIP 或 `.DS_Store`。
- 同步前可运行 `npm run docs:clean:check` 做 dry-run 和敏感信息扫描；目标仓库没有 `origin` 时只提交本地，不推送。
- 远端仓库通过 `CLEAN_DOCS_REMOTE_URL` 或在目标仓库手动配置 `origin` 接入；目标分支默认 `main`，可通过 `CLEAN_DOCS_BRANCH` 覆盖。
- 本机每日定时可用 `npm run docs:clean:launchd:load` 写入并加载 LaunchAgent；默认每天 08:30 运行，日志写入 `.tmp/clean-docs-sync.log` 和 `.tmp/clean-docs-sync.err.log`。

## Skill 路由规则

- 如果任务是用户询问具体数据、快速取数、库存、销量、广告、利润、客诉、刷仓、退货、物流、采购、FineBI 对账结果，优先使用 `.agents/skills/business-data-query/SKILL.md`。
- 如果任务是学习业务、梳理数据库表、做报表、写脚本、核指标、FineBI 复核、复杂异常排查、更新文档、沉淀经验，使用 `.agents/skills/business-data-closed-loop/SKILL.md`。
- `business-data-query` 也要闭环，但使用 `docs/CLOSURE_PROTOCOL.md` 的轻量 Closure Packet。
- `business-data-closed-loop` 也要闭环，但使用 `docs/CLOSURE_PROTOCOL.md` 的完整沉淀 Closure Packet。
- 如果 `business-data-query` 发现口径不清、对象类型不明、映射规则缺失、字段语义未确认、NULL/0 含义不明、一对多映射风险未确认、新 SQL 模板、新坑点、报表或文档沉淀需求，应转为 `business-data-closed-loop` 的学习 / 沉淀任务。
- 两个 skill 都必须遵守只读数据库、安全凭据保护、按相关性读取文档、不编造口径和字段的规则。

## 阅读入口规则

后续 agent 在开始数据库、FineBI、SQL、报表、指标、排查、智能问数或文档沉淀任务时，默认只读：

1. `AGENTS.md`
2. `docs/INDEX.md`
3. 当前任务相关 skill
4. 涉及问数、学习、报表、SQL、FineBI、指标、排查或沉淀闭环时读取 `docs/CLOSURE_PROTOCOL.md`

随后按 `docs/INDEX.md`、任务关键词和入口 skill 指引读取相关 summary、索引、专项文档或知识卡片。`docs/summaries/business-summary.md` 是复杂业务学习、报表或治理任务的推荐轻量入口，不作为所有任务的强制必读。

如果相关文档不存在，先判断是否能归入现有入口；只有用户明确需要新文档，或现有文档确实无法承载时，才创建基础模板。已存在相关文档时，应按当前需求追加、压缩或归档，避免覆盖已有经验。

## 知识更新规则

知识更新按 `docs/CLOSURE_PROTOCOL.md` 的文档路由执行，`AGENTS.md` 不维护另一套完整路由。这里只保留项目级边界：

- 不把详细业务表结构塞进 `AGENTS.md`；这里只保留项目协作方法。
- 默认不为文档治理新增索引、README、总览或临时说明文档；优先复用 `docs/INDEX.md`、`docs/business/需求沉淀索引.md`、`docs/summaries/` 和对应专项文档。
- 新发现如果和旧文档冲突，追加冲突说明，不静默覆盖历史结论。

## 业务学习、报表执行与文档闭环规则

本项目所有数据库、FineBI、SQL、报表、指标、业务分析和文档治理任务，都必须按入口 skill 与 `docs/CLOSURE_PROTOCOL.md` 执行闭环；`AGENTS.md` 不复制完整流程。若本次完全复用已有知识、没有新增口径 / 字段 / 坑点 / 模板 / 报表 / 待核实项，可不更新文档，但最终回复要说明原因。

## 上下文节省规则

为了避免每个新 agent 占用过多上下文，本项目采用“索引优先、摘要优先、按需读取”的规则。默认阅读入口见上方“阅读入口规则”。

### 不要默认全文读取

不要在任务开始时全文读取以下所有文档：

- `docs/business/业务流程需求沉淀.md`
- `docs/business/指标口径字典.md`
- `docs/business/数据表目录.md`
- `docs/business/看板到数据集映射.md`
- `docs/business/校验规则.md`
- `docs/business/已知坑点.md`
- `docs/finebi/FineBI广告数据梳理.md`
- `docs/finebi/FineBI库存数据梳理.md`
- `docs/playbooks/` 下所有文档

这些文档只能按任务主题、关键词、表名、指标名、看板名、平台名、时间范围搜索后读取相关章节。

## 共享闭环协议

后续面向智能问数、业务学习、报表、SQL、FineBI、指标、排查和沉淀的任务，统一遵守 `docs/CLOSURE_PROTOCOL.md`。该协议是唯一共享闭环协议，负责定义结论等级、待核实规则、文档路由、沉淀触发条件和 Closure Packet。

`AGENTS.md`、`docs/INDEX.md`、`docs/CLOSED_LOOP_CHECKLIST.md` 和各 skill 只引用该协议，不复制另一套完整协议。

新增 Markdown 文件前必须先确认现有文档无法承载；能通过压缩、归档、合并入口或更新已有索引解决的，不新增文件。

## 核心口径红线

- 不允许编造表、字段、SQL、指标口径、FineBI 血缘。
- 未确认的信息必须标注“待核实”。
- 推断内容必须标注“推断”。
- 权限不足必须记录为“权限不足”。
- 广告销售额必须区分 WSC / wholesale cost 与 retail sales，不能混用。
- ROI、ROAS、CTR、CVR、CPA、CPM、CPC 等比率指标，汇总层必须按分子分母重算，不能直接平均明细比率。
- Campaign 分析优先使用 `campaign_id`；如果只能使用 `campaign_name`，必须提示名称变更风险。
- 涉及利润、成本、总利润、总成本时，必须说明字段来源；如果上游未确认，标注“待核实”。
- 数据库默认只读，除非我明确要求写入、改表或执行会改变数据的过程。

## 需求澄清提问机制

当用户提出新业务学习、报表生成、SQL 查询、FineBI 看板梳理、指标口径确认、异常排查或坑点记录需求时，agent 必须先判断信息是否足够。

如果信息不足，并且会影响 SQL、口径、报表结构或结论准确性，必须先问用户。

每次最多问 5 个关键问题，优先问：

1. 业务目标
2. 时间范围
3. 平台、店铺、国家、业务范围
4. 输出形式
5. 核心指标或口径

如果信息虽然不完整，但可以根据已有文档合理推进，不要卡住任务。可以先基于默认假设继续做，并标注：

- 默认假设
- 待核实
- 权限不足
- 推断
- 需要用户确认

提问格式：

我需要先确认几个关键点，避免后面口径或报表做错：

1. ...
2. ...
3. ...

你也可以只回答你知道的部分；不知道的我会标注为“待核实”继续推进。
