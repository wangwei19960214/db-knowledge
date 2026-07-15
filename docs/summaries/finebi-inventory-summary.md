# FineBI Inventory Summary

## 用途

库存、有货率、断货、可用、售罄、仓库、采购 SKU、预计到仓相关任务的轻量入口。

## 原文档

- 专项原文：`docs/finebi/FineBI库存数据梳理.md`
- 指标语义：`docs/business/指标口径字典.md`
- 表/数据集：`docs/business/数据表目录.md`
- 校验：`docs/business/校验规则.md`
- 坑点：`docs/business/已知坑点.md`
- 有货率流程：`docs/playbooks/有货率报表生成流程.md`

## 核心主题索引

- 实时库存与预计到仓：`warehouse_stock_最新`、`公司sku上下架表`、`预计到仓表`。
- 仓库库存可用预估与断货：`tb_kyxl_forecast_and_reality`、`各sku每月断货天数`、FineBI `zzdnb_断货sku数量` / 物理表 `断货sku数量`。
- 采购 SKU / Model 映射：`采购sku_model映射_2`。
- CG 仓库存：`wayfair后台cg可用`、`cg_part_sku映射`。
- 每天各平台各仓库可用：核心来源是 `库存拆分记录`。
- 售罄预估与库存库龄：`每月销量售罄预估表`。
- Wayfair 上架 SKU 与链接 SKU 有货率：`平台产品详情表`、`wayfair上架sku销售映射`、`warehouse_stock_在售`。

## 数据集索引

- `数据连接_tb_kyxl_forecast_and_reality`：`tb_kyxl_forecast_and_reality` 的 FineBI 入口。
- `各sku每月断货天数`：按 `年月 + sku` 聚合断货天数和有货率。
- `zzdnb_断货sku数量`：月度断货 KPI 表。
- `zzdnb_tb_kyxl_forecast_and_reality_合并sku`：合并 SKU 版事实表。
- `zzdnb_断货sku数量_合并`：合并 SKU 版断货 KPI 表。
- `采购sku_model映射_2`：采购 SKU 到 model 的公共映射层。
- `warehouse_stock_在售`：Wayfair 有货率报表库存历史来源。
- `仓库映射表`：按所在国家筛选仓库。
- `wf_supplier_parts_performance`：Wayfair 销量来源。
- `advertising_product_report_by_day`：有货率报表可选广告关联来源。

## 已确认核心口径

- 库存预测断货日：`可用 = 0` 或 `预估可用 = 0`。
- `各sku每月断货天数` 的 `断货天数` 是当月断货日数。
- 库存预测有货率：`不断货销量 / (不断货销量 + 断货数量)`。
- `考虑放弃sku清单` 中 SKU 被排除。
- Wayfair 上架 SKU 粒度：`店铺 + 上架 SKU`。
- 上架日计入，下架日不计入。
- Wayfair 采购 SKU 优先用 `wayfair上架sku销售映射`，缺失时用 `平台产品详情表.采购sku`。
- Wayfair 库存按采购 SKU、日期、店铺对应国家仓库汇总 `可用`。
- 组合产品任一采购 SKU 当天可用小于等于 0，则上架 SKU 当天断货。
- 上架 SKU 有货率 = 有货天数 / 实际在架天数。
- 链接 SKU 有货率按上架 SKU 销量权重加权；权重为最近 90 个有货日平均销量。

## Wayfair 国家仓映射

- `WF-TD`、`WF-FOP`：美国仓。
- `WF-TD-CA`、`WF-FOP-CA`：加拿大仓。
- `WF-TD-UK`：英国仓。

## 已知截止日期差异示例

- 2026 年 5-6 月 Wayfair 有货率报告：
- 库存数据截止：2026-06-15。
- 销量数据截止：2026-06-12。
- 广告数据截止：2026-06-14。
- 2026-06-25 CS0002 组合链接历史+预测有货率报告：
- 历史有货率：复用 2026-01 至 2026-06-15 有货率 JSON，其中本次主表只展示 2026-01 至 2026-05。
- 预测窗口：默认 `2026-06-25` 至 `2026-09-25`。
- 预测源：`tb_kyxl_forecast_and_reality`，当前按采购 SKU 全局预测使用，未拆国家仓 / 店铺仓，需标 `待核实`。
- 2026-07-01 只读复核：`tb_kyxl_forecast_and_reality` 物理主键为 `date + sku`，日期 `2025-06-30` 至 `2026-11-01`，字段未见 store / 国家 / 仓库；`断货sku数量` 是数据库物理表名，FineBI `zzdnb_断货sku数量` 是显示 / 数据集名。
- 2026-07-01 只读复核：`库存拆分记录` 最新日 `2026-07-01`，`sku + date` 粒度；`其他仓库可用库存 + cg仓可用库存` 是仓库维度拆分，AMZ / 垂直是渠道维度拆分，不要把所有拆分字段相加。
- 2026-07-01 性能提醒：`每月销量售罄预估表` 视图 `LIMIT 5` 和限时聚合均超时，智能取数优先查基础结果表或先确认视图生成 SQL。

## 待核实重点

- FineBI `采购sku_model映射_2` 的抽取缓存时间、组件二次过滤和 56 个真实 Model 冲突的展示优先级；其 SQL 与当前数据库复算已确认。
- `库存拆分记录` 重复键修复、补跑和 FineBI 恢复日期；数据库生成链已确认，但结果仍停在 2026-07-08。
- “每天各平台各仓库可用”组件计算字段公式。
- 加拿大、英国版本组件和最终仪表板引用路径。
- `每月销量售罄预估表` 28 个字段的完整生成 SQL。

## 已知坑点

- 采购 SKU 可能一对多。
- FineBI `采购sku_model映射_2` 会补入预测 / 公司 SKU 范围，空 Model 明显多于物理映射；普通采购 SKU -> Model 查询不能无条件替换为该数据集口径。
- 上架 SKU / 链接 SKU / 采购 SKU 可能缺失或冲突。
- 组合产品不能只看单个组件库存。
- 缺失库存日不能直接当有货或断货。
- 库存、销量、广告截止日期可能不同。
- FineBI 部分库存仪表板无权限，无法直接取得 activeTab URL。

## 推荐读取章节

- 断货预测：`docs/finebi/FineBI库存数据梳理.md` 的“仓库库存可用预估与断货”。
- 采购 SKU 映射：同文档“采购 SKU / Model 映射”。
- Wayfair 有货率：同文档“Wayfair 上架 SKU 与链接 SKU 有货率”。
- 待核实：同文档“待继续核实”。
