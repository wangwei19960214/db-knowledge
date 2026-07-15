# FineBI Ad Summary

## 用途

广告、广告组、ROI、ROAS、BID、WSC、广告销售额、自然销售额、OS 广告、WF 广告相关任务的轻量入口。

## 原文档

- 专项原文：`docs/finebi/FineBI广告数据梳理.md`
- 指标语义：`docs/business/指标口径字典.md`
- 表/数据集：`docs/business/数据表目录.md`
- 看板映射：`docs/business/看板到数据集映射.md`
- 校验：`docs/business/校验规则.md`
- 坑点：`docs/business/已知坑点.md`
- 广告组月报流程：`docs/playbooks/广告组月报生成流程.md`

## 看板索引

- `WF-WSP整体广告分析`：`zzd/广告`；筛选器 `年月区间`、`店铺`；主要数据集 `总广告数据`。
- `WF-WSP分类广告分析`：`zzd/广告`；筛选器 `年月区间`、`分类`、`店铺`；主要数据集 `分类目广告数据`。
- `WF-WSP广告组广告分析`：`zzd/广告`；筛选器 `日期区间`、`广告组`、`店铺`、`活动垂直标签`；核心组件 `WSP广告组每月汇总`。
- `wf广告`：`zzd/广告`；主要数据集 `公共数据/王威/增量更新/wf广告`。
- `wf新广告表现`：`zzd/广告`；涉及 Sponsored Display、Sponsored Video、Sponsored Shops、Google Shopping、Enhanced Attribution、New to Wayfair。
- `OS广告`：`zzd/广告`；主要数据集 `os广告重新计算`、`os广告2`。

## 数据集索引

- `WF广告汇总`：月度广告类型汇总，合并 WSP、Google Shopping、Sponsored Display、Sponsored Shops、Sponsored Video。
- `总广告数据`：`store + 年月` 粒度，支撑整体广告分析。
- `分类目广告数据`：`store + 年月 + 分类` 粒度，支撑分类广告分析。
- `WF广告数据/广告`：广告组月度分析核心自助表。
- `wf广告`：WF 日粒度广告明细 SQL，主表 `advertising_product_report_by_day`。
- `wf产品`：WF 产品维表增强层，补产品属性、采购 SKU、当前月 DDP、`base_costs_USD`、`promotional_base_cost`。
- `zzdnb_wf上架sku每日销售额`：`store + 日期 + supplier_part` 日销售额/销量，血缘进入 `广告` 自助表；2026-06-30 数据库复核 `supplier_part` 是销售源 Supplier Part Number / 销售 SKU 层，可为 `Kit`、`Standard`、`Sellable Component` 或少量 `Nonsellable Component`，不是 `总账单详情_extract.上架sku` 的订单子件层。
- 2026-06-19 数据库补充：物理表 `wf上架sku每日销售额` 由同名过程每日写入，来源 `wf_supplier_parts_performance`，补 `链接sku/wayfair_sku` 并对 `GBP/CAD` 做最新汇率折算。
- `zzdnb_wf_class中文分类` / 物理表 `wf_class中文分类`：广告 `class_name` 到中文 `分类` 的映射表，字段为 `Class Name`、`类目`、`分类`，支撑 WSP 分类广告和广告组分区。
- `wf_sku不断货销量`：广告链路库存可用 / 不断货销量辅助表，SQL 为 `select * from wf_sku不断货销量`。
- `zzdnb_detailed_listing_health`：Listing health 月度表，补 `评分` / `评论数`，源字段为英文 review/rating 字段。
- `zzdnb_advertising_product_report_by_day`：WF 广告日报抽取入口。
- `zzdnb_wf广告出价表`：`store + 日期 + Campaign name + wayfair SKU` 粒度 BID 数据。
- `zzdnb_广告组合单买详情`：广告组合/单卖辅助映射。
- `os广告重新计算`：OS 广告 SQL 数据集，补订单和采购 SKU。
- `os广告2`：OS 广告 SQL 数据集，补采购 SKU 和 DDP；当前使用状态待核实。

## 核心字段索引

- 广告维度：`store`、`date_est`、`年月`、`campaign_id`、`campaign_name`、`campaign_status`、`isB2b`、`sku`、`first_10_part_numbers`、`class_name`、`product_status`。
- 广告指标：`clicks`、`impressions`、`spend_USD`、`cost_per_click_USD`、`click_through_rate`、`attributed_wholesale_cost_window_view_through_USD_Day_14`、`attributed_units_window_view_through_Day_14`。
- Retail 字段：`attributed_retail_sales_window_view_through_USD_Day_14`，不得与 WSC 广告销售额混用。
- 利润字段：`总成本`、`总利润`、`成本`、`利润`。
- BID 字段：`product_max_bid`、`Default bid(现在的bid)`、`Suggested Bid detail(具体金额)`、`最小建议bid`、`最大建议bid`。
- 商品辅助：`评论数`、`评分`、`是否有价格优势`、`广告组利润类型`、`断货百分比`。

## 已确认核心口径

- WF 广告销售额在广告利润口径中取 WSC：`attributed_wholesale_cost_window_view_through_USD_Day_14`。
- 广告销售额 FineBI 公式：`SUM_AGG(attributed_wholesale_cost_window_view_through_USD_Day_14)`，为 0 时显示空。
- 成本 FineBI 公式：`SUM_AGG(总成本)`，为 0 时显示空。
- 利润 FineBI 公式：`SUM_AGG(总利润)`，为 0 时显示空。
- 广告 ROI：`SUM_AGG(总利润) / SUM_AGG(spend_USD)`。
- 广告 ROAS：`SUM_AGG(WSC) / SUM_AGG(spend_USD)`。
- 广告组利润类型：按 `SUM_AGG(总利润)` 正负分为盈利、亏损、无效广告组。
- 断货百分比：`SUM_AGG(断货part个数) / SUM_AGG(活动的上架sku数量)`，分子为 0 时显示空。
- `断货part个数`、`活动的上架sku数量`、`上架sku可用库存` 未发现数据库同名物理列，是 FineBI 自助层从 `wf广告` 回补的活动维度字段；`wf_sku不断货销量` 的断货销量字段不能替代。
- 广告销售占比：`广告销售额 / AVG_AGG(总销售额1)`，公式条件还检查 `总销售额修`。
- `总销售额修`：`IF(AVG_AGG(总销售额1)=0,"",AVG_AGG(总销售额1))`，是 `总销售额1` 聚合后的空值展示包装，不是新的销售额来源。
- 广告花费占广告销售额的比：`SUM_AGG(spend_USD) / SUM_AGG(WSC)`。
- WSP 分类 / 广告组分区：按中文 `分类` 字段筛选，来源为 `advertising_product_report_by_day.class_name -> wf_class中文分类.\`Class Name\` -> 分类`；不要直接用 `category/first_class/second_class` 替代。映射表值含 `室内家具`、`户外家具`、`灯具`、`厨房卫浴`、`其他`、`家居装饰`；2026-01 至 2026-05 广告事实当前只落到 `室内家具/户外家具/灯具/家居装饰/其他`，`其他` 为 `Multichannel-Only (MCO) Fulfillment` / `多渠道专供商品`，该范围无 `厨房卫浴`。
- `wf广告` SQL 不直接计算 `总成本`、`总利润` 或利润率。
- 公共数据 `wf广告` 血缘已展开到 `WSP广告组每月汇总`、`广告组广告花费、销售额、利润与ROI分析`、`销售和利润`、`回报率` 等利润相关组件；该血缘不能反推 `wf广告` SQL 生成了 `总成本/总利润`。
- `zzdnb_wf上架sku每日销售额` 血缘已确认进入 `广告` 自助表；主题内已见按 `store + 月份 + sku/wayfair_sku` 回补 SKU 月销售额。
- `本竞品价格对比` 在 FineBI 只暴露数据源和店铺过滤；2026-06-23 数据源节点字段级确认可见 `本品最新最低价/竞品最低前台价/website/更新日期` 等展示字段。数据库无同名物理表，复刻源为 `产品近两天的最小前台价`，上游 `link_sku_price_qtj_fabric`，属于当前快照；SQL 字段为 `最近日期最小前台价`，不是 FineBI 展示字段名。FineBI `是否有价格优势` 原式在本品价等于竞品价时返回“是”；若报表定义为“本品必须低于竞品才有优势”，等价样本需改判“否”。2026-06-23 已抽等价、低于、高于三类样本，FineBI 导出逐行一致性仍待补。
- `活动的垂直标签` 未发现数据库同名物理列；2026-06-22 FineBI 上游复核确认直接来源为 `WF店铺绩效2.0/广告` 从 `zzdnb_公司sku状态表` 按 `UPPER(采购SKU)=UPPER(sku)` 补 `广告细分标签/垂直标签/model/公司标签` 最大值，生成 `广告标签` 后按 `[extend,isB2b]` 拼接。数据库补核确认 `zzdnb_公司sku状态表` 实际物理表为 `公司sku状态表`，由事件/过程 `公司sku状态表_2` 每天生成；FineBI 当前使用当前快照，历史月度标签需另接留存表。`上架sku利润表.垂直标签/垂直细分标签` 只作为数据库侧对照候选。
- 2026-06-22 内网 `WF店铺绩效2.0/广告` 表 `6d42a82f92604810b3abf468d7876bba` 继续复核末端时间辅助字段：`14天周期` 滚动 14 天区间、`是否满足14天`、`周`、`星期`、`星期日` 均由 `date_est` 计算；`星期六/日期范围` 本轮未再次点到，沿用原广告专项记录，需样本时再回 FineBI 点节点。
- `wf_sku不断货销量` 血缘已确认进入 `广告` 自助表，并继续到 `总广告数据`、`分类目广告数据`、`WSP广告组每月汇总`、广告组汇总和广告组组件；可作为广告看板可用/断货链路的重要入口。
- `wf产品` SQL 已确认整合平台产品、WF product management、鲸汇销售产品、公司 SKU 状态、当前月 DDP 和当天 WF price 成本字段；但存在 GROUP BY 风险。
- `zzdnb_detailed_listing_health` 血缘已确认进入 `广告` 自助表，并继续到整体 / 分类 / 广告组组件；2026-06-23 页面确认 `广告` 表从该表按最大值补 `评分`、`评论数`，匹配 `store + 年月 + sku=Wayfair SKU`；源字段为 `Lifetime Review Count`、`Lifetime Average Customer Review Rating`。
- `zzdnb_advertising_product_report_by_day` 2026-06-17 08:00 更新后 `442,448` 行；血缘样本已见 `广告花费明细表`、`广告每日变化情况`、`广告销售变化情况`。
- `公共数据/tsq/ddp利润表/ddp汇总` SQL 已确认由 `ddp汇总表_美国`、`ddp汇总表_加拿大`、`ddp汇总表_英国` union 补 `国家` 字段；血缘可展开到 `广告`、`zzdnb_advertising_product_report_by_day`、`利润广告比率`。
- `ddp_利润查询` 页面利润参考口径为 `总销售额 - 佣金 - ddp - 出货费用`，且排除 `分销`；该口径不能直接等同广告 `总利润`。

## 待核实重点

- `WF店铺绩效2.0/广告` 中 `总成本/总利润` 的公式链和关键右侧来源已确认；剩余是 FineBI 已计算样本或预聚合链路的数值 QA，不再按权限不足或上游不明处理。2026-06-23 已通过 FineBI 预览行序 + `extend` 回查广告事实补 5 条 2023-09 带业务键样本，复算差异为 0 或 0.01 的显示舍入差异；再次尝试 2026 同期时，预览第 `50/50` 页仍为 2024-04，未取得 2026-01 至 2026-05 已计算成本利润样本，不能用数据库 2026 广告事实样本冒充。
- 2026-06-23 数据库通后补核：库内没有广告链路可直接抽取的精确 `总成本/总利润/每个sku平均成本` 物理列；`ddp汇总.预计和实际ddp汇总`、`产品发货费.出货费用`、`广告组合单买详情.单卖` 可作为公式复算输入。`wf每月产品和广告数据汇总 -> 总广告数据 -> WF-WSP整体广告分析`，`wf每月产品和广告数据 -> 分类目广告数据 -> WF-WSP分类广告分析`；这两张月度表不含 `总成本/总利润`，也不是 `广告组活动 -> WSP广告组每月汇总` 的总销售额主源。当前最终表和 FineBI 主题预览仍停在 `2026-04`，但源表/中间表已有 2026-05，只读复算最终 SELECT 可产出 5 月，说明 5 月缺口在最终落表或 FineBI 主题刷新。
- `WF-WSP广告组广告分析 -> 广告组活动 -> WSP广告组每月汇总` 和最终 WSP 广告组数据表的 `总销售额` 相关字段，应走 `WF广告数据/广告.总销售额1 -> zzdnb_wf上架sku每日销售额 -> wf上架sku每日销售额.total_revenue`。匹配键为 `store + 月份 + sku=wayfair_sku`；该值是 SKU 月销售额，可对到广告组和月份，但在 bid / 日期拆行后不能直接求和。
- 2026-06-23 Chrome 复核：组件展示字段 `总销售额` 实际来源字段为 `广告.总销售额修`；`广告` 表中 `销售额` 添加列来自 `zzdnb_wf上架sku每日销售额` 求和，匹配 `store`、月份、`sku=wayfair_sku`；血缘视图确认 `zzdnb_wf上架sku每日销售额 -> 广告 -> WSP广告组每月汇总 -> WSP广告组广告分析`。
- 2026-06-19 从 `WF广告数据/广告` 前置血缘进入 `WF店铺绩效2.0/zzdnb_advertising_product_report_by_day` 后，当前节点显示另一套 DDP/发货费成本分支：`ddp平均成本` 基于 `每个上架skuddp`、分组 `[campaign_name, store, sku, date_est, extend]`，`每个sku平均成本 = ddp平均成本 + 发货费平均成本`；两次去重后收回到广告事实 + 成本结果字段；该前置节点 `总成本 = 每个sku平均成本 * attributed_units_window_view_through_Day_14 + spend_USD`，没有 `/ 单买数量`。继续回主 `WF店铺绩效2.0/广告` 表确认：主表 `单买数量 = IF(1=1,1,NVL(单卖,1))` 当前恒为 1，主表 `总成本` 公式仍带 `/ 单买数量`，主表 `总利润 = WSC * 店铺系数 - 总成本`。同时出现单 campaign、单 SKU、固定日期等疑似样例过滤，不能直接复刻进全量报表。
- 公共数据搜索按 `总利润`、`上架sku利润表`、`ddp` 未找到同名公共数据集节点；利润源不能按这些名称直接假设。
- `ddp汇总 -> 广告 -> 利润广告比率` 只能证明 DDP 进入广告利润相关链路，不能证明公共 `wf广告` SQL 直接生成 `总成本/总利润`。
- `总广告数据` 和 `分类目广告数据` 的完整 SQL。
- `wf_sku不断货销量` 的物理字段已确认；其 `断货销量/不断货销量/断货销量占比` 不能替代广告组 `断货part个数/活动的上架sku数量`，后续只需继续核同名表生成过程。
- `wf产品` 中 CA/UK 产品 ID 映射和 GROUP BY 重复取值稳定性。
- `总成本/总利润` 2026-01 至 2026-05 FineBI 已计算样本；需要导出或过滤到 2026 的自助表 / 组件样本。
- `广告花费占比` 分母。
- `zzdnb_wf广告出价表` 与广告日报 `campaign_id` 的稳定映射。
- `WSP广告组每月汇总` 组件维度 `bid` 已确认来源字段名为 `product_max_bid`、来源表为 `广告`，不是 `Default bid(现在的bid)`；`广告组bid趋势` 已确认使用 `Default bid(现在的bid)`、`Suggested Bid detail(具体金额)`、`最小建议bid`、`最大建议bid`、B2B bid adjustment 等当前/建议 bid 快照字段。
- 2026-06-22 组件槽位复核：`WSP广告组每月汇总` 维度槽已见 `store`、`日期(年月)`、`model`、`campaign_name`、`标签`、`sku`、`isB2b`、`活动是否有ddp`、`class_name`；结果过滤为空。分类页签已见 `室内家具/户外家具/灯具/家具装饰/其他` 等。2026-06-23 补核：仪表盘组件页面只作为展示/过滤核对入口，不作为底层数据逻辑主线；`灯具广告组汇总数据` 组件弹窗确认 `分类 属于 灯具`；`家具装饰广告组汇总数据` 展示值与数据库 `分类='家居装饰'` 聚合对齐；`其他广告组汇总数据` 弹窗未成功打开，但看板页签显示值与数据库 `分类='其他'`、`Class Name=Multichannel-Only (MCO) Fulfillment` 聚合一致，`spend=34.64/WSC=0/clicks=115/impressions=3332`，当前范围无 `厨房卫浴`，可按 `其他` 字段值落地，页面配置层仍待补。
- `wf新广告表现` 中 Enhanced Attribution、NTW/NTB 的数据集来源。
- `OS广告` 组件实际使用 `os广告重新计算`、`os广告2` 还是另有自助数据集。

## 已知坑点

- WSC / wholesale cost 与 retail sales 不能混用。
- `campaign_name` 可能变更，广告组分析优先用 `campaign_id`。
- 利润不是广告日报原生字段。
- `总成本/总利润` 上游公式链已确认；数据库即席数值复算受 `cg_part_sku映射时间范围`、DDP、发货费等无索引表影响，后续应使用预聚合/带索引小表或 FineBI 已计算样本做 QA，不再按权限不足或上游不明处理。
- FineBI 成本前置节点存在分支差异和局部过滤；遇到 `campaign_name in (...)`、`sku=...`、固定历史日期或最新 N 过滤时，先标待核实，不能默认作为全量广告报表条件。
- 主 `WF店铺绩效2.0/广告` 的 `单买数量` 当前由 `IF(1=1,1,NVL(单卖,1))` 写死为 1；做 SQL 复刻时既不能忽略字段来源，也不能把历史 `单卖` 当作当前一定生效的除数。
- DDP 利润查询排除 `分销`，与销售看板可能有差距，不能直接套到广告 ROI。
- `wf产品` 内层按 `产品id` 分组但选择多个非聚合字段，历史更新曾因 GROUP BY 规则失败。
- Listing Health 源字段为英文，广告主题中文 `评分/评论数` 是后续命名；该表存在重复键，补广告报表时应取最大值或先去重，不能明细求和。
- 比率指标不能直接平均明细值。
- BID 表补当前 bid 快照时优先用 `campaignlayer_id` 对广告日报 `campaign_id`；仅使用 Campaign name 有名称变更风险。
- OS 广告 `os广告2` 有更新失败记录，当前使用状态待核实。

## 推荐读取章节

- 看板入口：`docs/finebi/FineBI广告数据梳理.md` 的“广告看板入口与已见内容”。
- 月度汇总：同文档“王威 / 广告：月度广告汇总链路”。
- 广告利润：同文档“WF广告数据 分析主题 / 自助数据表”中的“利润相关口径”。
- WF 明细：同文档“王威 / 增量更新：WF 广告明细链路”。
- BID：同文档“tsq / WF店铺曝光：广告出价链路”。
- OS：同文档“王威 / 增量更新：OS 广告链路”。
