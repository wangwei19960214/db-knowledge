# Report Playbooks Summary

## 用途

报表生成任务的轻量流程索引。后续做报表时先读本摘要，再读相关 playbook、summary 和专项文档。

## Playbook 位置

- 广告组月报：`docs/playbooks/广告组月报生成流程.md`
- 有货率报表：`docs/playbooks/有货率报表生成流程.md`

## 通用报表流程

1. 读取 `AGENTS.md`、`docs/INDEX.md`、闭环 skill；涉及报表、SQL、指标或沉淀时读取 `docs/CLOSURE_PROTOCOL.md`。
2. 读取本摘要和相关业务 / FineBI summary。
3. 搜索相关指标、表、看板、坑点，不全文读取所有文档。
4. 如信息不足且影响口径，最多问 5 个关键问题。
5. 确认时间范围、平台/店铺/国家、输出形式和核心指标。
6. 写 SQL 或脚本。
7. 检查日期覆盖、粒度、重复、映射和空值。
8. 比率指标按分子分母重算。
9. 输出异常清单。
10. 保存产物到 `reports/`，可复用脚本放 `scripts/`。
11. 按 `docs/CLOSURE_PROTOCOL.md` 判断并只更新相关业务流程、指标、数据表、校验、坑点、playbook 或 `reports/INDEX.md`。

## 广告组月报

- 适用：Wayfair / WF-WSP 广告组月度经营分析。
- 输入：月份、店铺、广告组、SKU 范围、是否需要利润/库存/评分评论。
- 核心数据：`advertising_product_report_by_day`、`wf广告`、`WF广告数据/广告`、`zzdnb_wf广告出价表`、`warehouse_stock_在售`、`wayfair上架sku销售映射`。
- 核心指标：广告销售额(WSC)、广告花费、点击、曝光、CPC、CTR、CVR、CPM、CPA、ROAS、ROI、利润、成本、BID、有货率、评论数、评分。
- 核心校验：FineBI 月/店铺对账、campaign_id/name 检查、WSC vs retail sales、SKU 映射、成本利润样本复算、比率重算。
- 常见坑：campaign_name 变更、利润非广告日报原生、总成本/总利润需复用 FineBI 上游公式链；数值 QA 不要直接扫无索引映射/成本表，BID 当前快照优先用 `campaignlayer_id` 而不是只靠名称。
- 产物建议：`reports/wayfair_ad_group_analysis_YYYY-MM/`。

## 有货率报表

- 适用：Wayfair 上架 SKU、链接 SKU、采购 SKU 的指定时间段有货率。
- 输入：时间范围、店铺/国家、SKU 范围、是否关联广告。
- 核心数据：`平台产品详情表`、`wayfair上架sku销售映射`、`warehouse_stock_在售`、`wf_supplier_parts_performance`、`advertising_product_report_by_day`、`仓库映射表`。
- 核心指标：上架 SKU 有货率、链接 SKU 有货率、断货天数、缺失库存天数、销量权重。
- 核心校验：日期覆盖、SKU 映射、一对多、组合产品、缺失库存、链接 SKU 权重。
- 常见坑：组合产品组件库存、链接 SKU 空值、库存/销量/广告截止日期不同。
- 已有脚本：`scripts/generate-wayfair-availability-data.js`、`scripts/build-wayfair-availability-workbook.mjs`。
- 已有产物：`reports/固定流程取数和报表/REQ-004_Wayfair有货率报表/wayfair_availability_2026-06-15/`。

## 报表交付前检查

- 是否读了 `AGENTS.md` 和 `docs/INDEX.md`。
- 是否只按需读取相关 summary / playbook / 专项章节。
- 是否确认指标口径和字段来源。
- 是否记录默认假设、待核实、权限不足。
- 是否检查时间范围。
- 是否检查表粒度。
- 是否检查重复和 join 放大。
- 是否检查 SKU 映射。
- 是否检查 campaign_id / campaign_name。
- 是否检查 WSC vs retail sales。
- 是否按分子分母重算比率指标。
- 是否输出异常清单。
- 是否保存产物到 `reports/`。
- 是否按 `docs/CLOSURE_PROTOCOL.md` 判断并更新相关闭环文档。

## 推荐短命令

- 学业务：`请使用 business-data-closed-loop，继续学习【业务主题】业务。先读 AGENTS、INDEX、CLOSURE_PROTOCOL 和相关 summary，只按关键词读原文档章节，完成后按协议闭环沉淀。`
- 做报表：`请使用 business-data-closed-loop，做【报表名称】。先查已有口径、表、看板、坑点和 playbook；如信息不足先问最多 5 个问题；产物放 reports，脚本放 scripts，并按协议更新相关闭环文档。`
