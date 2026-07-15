# 项目知识索引

最近整理日期：2026-06-29

## 使用原则

本索引用来快速定位文档，不承载完整业务口径和闭环协议。执行流程以入口 skill 和 `docs/CLOSURE_PROTOCOL.md` 为准。

默认启动只读：

1. `AGENTS.md`
2. `docs/INDEX.md`
3. 当前任务相关 skill
4. 涉及问数、SQL、FineBI、报表、指标、排查或沉淀时读取 `docs/CLOSURE_PROTOCOL.md`

随后按关键词读取相关章节，不要默认全文读取所有 business、finebi、knowledge、archive 文档。

## 主入口

| 需求 | 优先入口 | 读取方式 |
| --- | --- | --- |
| 项目协作规则、数据库安全、skill 路由 | `AGENTS.md` | 先读规则，具体闭环细节回到 `docs/CLOSURE_PROTOCOL.md`。 |
| 历史需求和报表经验 | `docs/business/需求沉淀索引.md`、`docs/business/业务流程需求沉淀.md` | 先定位 REQ 编号，再按需读摘要或归档原文。 |
| 历史执行原文 | `docs/business/需求沉淀索引.md` | 先看归档入口和 REQ 索引，再按主题读取对应原文。 |
| 业务总览 | `docs/summaries/business-summary.md` | 用来定位业务主线和高风险待核实项。 |
| 报表和产物状态 | `reports/INDEX.md` | 路径不存在时以索引状态和备注为准，不自行假设文件存在。 |

## Business 文档

| 文档 | 用途 | 什么时候读 |
| --- | --- | --- |
| `docs/business/业务流程需求沉淀.md` | 需求摘要入口和高风险入口 | 查历史需求、复用脚本 / 报表经验。 |
| `docs/business/需求沉淀索引.md` | REQ 级轻量目录 | 查主题、脚本、产物或需求状态。 |
| `docs/business/数据表目录.md` | 表、字段、粒度、关联键、上下游 | 写 SQL、确认字段或 join 关系。 |
| `docs/business/数据库对象学习记录.md` | 数据库对象学习过程 | 追溯对象分类、过程、事件或旧学习记录。 |
| `docs/business/指标口径字典.md` | 指标公式、字段来源、汇总规则 | 计算指标、核公式、做报表字段。 |
| `docs/business/看板到数据集映射.md` | FineBI 看板、组件、数据集映射 | 对账 FineBI、追组件字段来源。 |
| `docs/business/校验规则.md` | 对账、覆盖、重复、映射、异常检查 | 做报表 QA 或异常排查。 |
| `docs/business/已知坑点.md` | 字段混用、权限、口径冲突、映射异常 | 写 SQL、做报表或记录新坑。 |
| `docs/business/待核实问题清单.md` | 高价值待核实项 | 遇到未确认字段、利润来源、刷新失败或对账差异。 |

Archive 原文统一从 `docs/business/需求沉淀索引.md` 的“归档入口”进入；本索引不逐个列 archive 文件，避免形成第二套归档目录。

## FineBI 文档

| 文档 | 什么时候读 |
| --- | --- |
| `docs/finebi/FineBI广告数据梳理.md` | 广告、广告组、ROI、ROAS、BID、WSC、广告销售额、自然销售额、OS 广告、WF 广告。 |
| `docs/finebi/FineBI库存数据梳理.md` | 库存、有货率、断货、可用、售罄、仓库、采购 SKU。 |
| `docs/finebi/FineBI公共数据目录地图.md` | 按公共数据负责人、目录名或数据集名定位 FineBI 数据集。 |
| `docs/summaries/finebi-ad-summary.md` | 广告 FineBI 轻量入口。 |
| `docs/summaries/finebi-inventory-summary.md` | 库存 FineBI 轻量入口。 |

## Knowledge 文档

| 文档 | 什么时候读 |
| --- | --- |
| `docs/knowledge/README.md` | 理解智能取数知识层分工。 |
| `docs/knowledge/取数意图字典.md` | 将自然语言问题归类为取数、报表或排查意图。 |
| `docs/knowledge/问法到指标映射.md` | 把用户问法映射到指标、维度和风险提示。 |
| `docs/knowledge/数据知识沉淀表.md` | 复用已沉淀业务卡片。 |
| `docs/knowledge/SQL模板库.md` | 复用 SQL 骨架；不确定字段保留 `TODO: 待核实`。 |
| `docs/knowledge/业务实体关系图.md` | 理解平台、店铺、SKU、Model、Campaign、库存、销售和报表关系。 |
| `docs/knowledge/智能取数安全边界.md` | 涉及数据库连接、SQL 执行、敏感信息或大范围取数。 |

## Lineage 和刷新

| 文档 | 什么时候读 |
| --- | --- |
| `docs/lineage/数据血缘总览.md` | 从上游表、脚本、FineBI 数据集追到报表产物。 |
| `docs/lineage/核心结果表血缘.md` | 查结果表、汇总表、利润表、库存表、预测表来源。 |
| `docs/lineage/存储过程到结果表映射.md` | 查过程、函数、触发器或结果表生成线索。 |
| `docs/lineage/事件任务刷新时间表.md` | 排查刷新延迟、自动化失败或任务执行时间。 |

## Playbook

| 文档 | 什么时候读 |
| --- | --- |
| `docs/playbooks/广告组月报生成流程.md` | 做广告组月报、广告组利润、有货率结合广告分析。 |
| `docs/playbooks/有货率报表生成流程.md` | 做上架 SKU、链接 SKU、有货率、断货天数报表。 |
| `docs/playbooks/FineBI业务学习与血缘字段复核流程.md` | 学习 FineBI 看板、公共数据、SQL 数据集、字段来源和血缘。 |
| `docs/governance/文档治理瘦身与闭环收敛执行工单.md` | 做项目文档瘦身、归档、入口收敛和规则去重。 |
| `docs/summaries/report-playbooks-summary.md` | 快速选择报表流程。 |

## Skill 和协议

| 入口 | 什么时候读 |
| --- | --- |
| `.agents/skills/business-data-query/SKILL.md` | 具体业务问数、快速取数、库存、销量、广告、利润、客诉、刷仓、退货、物流、采购、FineBI 对账结果。 |
| `.agents/skills/business-data-closed-loop/SKILL.md` | 学习业务、梳理表字段、做报表、写脚本、核指标、FineBI 复核、复杂异常排查、记录坑点、更新文档、沉淀经验。 |
| `docs/CLOSURE_PROTOCOL.md` | 唯一共享闭环协议；定义结论等级、待核实规则、文档路由、沉淀触发和 Closure Packet。 |
| `docs/CLOSED_LOOP_CHECKLIST.md` | 任务结束前的轻量检查清单，不替代闭环协议。 |
| `docs/TASK_STARTER.md` | 给新 agent 的极简启动卡片。 |

## 任务路由速查

| 用户要做什么 | 入口 |
| --- | --- |
| 快速查一个业务数字或对账结果 | `business-data-query` skill + 相关 business / finebi / knowledge 章节 |
| 学习新业务、核口径、复核 FineBI、沉淀文档 | `business-data-closed-loop` skill + `docs/CLOSURE_PROTOCOL.md` |
| 做广告组月报 | `docs/playbooks/广告组月报生成流程.md` |
| 做有货率 / 断货报表 | `docs/playbooks/有货率报表生成流程.md` |
| 查历史需求、脚本或产物 | `docs/business/需求沉淀索引.md` + `reports/INDEX.md` |
| 查历史原文 | `docs/business/需求沉淀索引.md` 的归档入口 |
| 治理文档臃肿、重复或冲突 | `docs/governance/文档治理瘦身与闭环收敛执行工单.md` |

## 不要做

- 不要把 `docs/INDEX.md` 当完整业务口径。
- 不要把 archive 原文当当前口径。
- 不要在多个入口复制完整闭环协议。
- 不要为了“全面了解”默认全文读取所有大文档。
- 不要为了整理文档再新增索引、README、总览或临时说明；优先压缩和复用现有入口。
