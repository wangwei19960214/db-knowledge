# FineBI 广告数据梳理

最近追加日期：2026-06-26

## 范围与原则

- FineBI 地址：`http://192.168.110.18:37799/webroot/decision`
- 梳理对象：FineBI 目录 `zzd/广告` 下的广告看板，以及这些看板实际引用或疑似引用的公共数据、自助数据集、字段、血缘、更新信息、SQL 和公式。
- 数据来源限制：只记录 FineBI 页面、数据集详情页、血缘/关联页、编辑页可见 SQL 或公式；不通过直连数据库反推 FineBI 生成逻辑。
- 库存、断货、可用、售罄等底层链路如已在 `FineBI库存数据梳理.md` 记录，广告专项只记录“广告看板是否引用/如何引用”，不重复展开库存生成链路。

## 1. 核心结论

1. `zzd/广告` 下已确认 6 个广告看板入口：`WF-WSP整体广告分析`、`WF-WSP分类广告分析`、`WF-WSP广告组广告分析`、`wf广告`、`wf新广告表现`、`OS广告`。
2. WSP 三个看板的层级是“整体 -> 分类 -> 广告组”：整体按店铺和年月看广告/自然销售及花费占比；分类增加 `分类` 筛选；广告组下钻到日期、广告组、店铺、SKU/Campaign 明细。
3. `公共数据/王威/广告/WF广告汇总` 是月度广告类型汇总 SQL 数据集，已确认合并 WSP、Google Shopping、Sponsored Display、Sponsored Shops、Sponsored Video 五类广告。
4. `公共数据/王威/广告/总广告数据` 和 `分类目广告数据` 属于 FineBI `广告更新任务`，每天约 `07:27` 更新，分别支撑整体和分类广告分析。
5. `公共数据/王威/增量更新/wf广告` 是 WF 日粒度广告明细 SQL 数据集，主表为 `advertising_product_report_by_day`，补充店铺 SKU、采购 SKU、数量和 DDP。
6. `公共数据/tsq/WF店铺曝光/zzdnb_wf广告出价表` 是广告出价快照数据集，按 `store + 日期 + Campaign name + wayfair SKU` 粒度补 Bid 相关字段，每天约 `09:00` 更新。
7. `公共数据/王威/店铺绩效/WF/zzdnb_广告组合单买详情` 是广告组/组合链接辅助映射，字段包括 `sku`、`campaign_name`、`单卖`。
8. `公共数据/王威/增量更新/os广告重新计算` 和 `os广告2` 是 OS 广告链路关键 SQL 数据集，均基于 `sku_level_daily_advertising_rep`，再补订单、SKU 映射或 DDP。
9. `公共数据/章森淼/滞销sku列表/滞销品列表` 可作为广告看板中滞销/清仓标签的候选来源；其底层逻辑来自售罄预估、库龄和计划发货。
10. `仓库销量可用预估2` 下的可用/断货数据集已在库存专项记录，广告专项后续只需确认具体广告看板是否引用这些数据集。
11. `WF广告数据/广告` 自助表的基础数据源显示为 `wangwei的分析/店铺绩效/WF店铺绩效2.0/广告`；当前广告主题是在该上游 `广告` 表基础上继续补自然/总销售、库存可用、评分、价格优势等字段。
12. 2026-06-18 定向复核确认：`WSP广告组每月汇总` 组件维度槽中的短字段 `bid` 来源字段名为 `product_max_bid`、来源表为 `广告`，即广告底表自带 bid；不是 `Default bid(现在的bid)`，也不是 `Suggested Bid detail(具体金额)`、`最小建议bid`、`最大建议bid`。
13. `WSP广告组明细表` 在 `广告组活动` 分区直接展示出价表补充字段，包括 `Bid adjustment applied on B2B bids`、`Default bid(现在的bid)`、`Suggested Bid detail(具体金额)`、`最小建议bid`、`最大建议bid`；它和月汇总组件里的短字段 `bid` 不是同一个口径。

## 2. 广告看板入口与已见内容

### 看板总览

| 看板 | FineBI 目录 | 筛选器/粒度 | 已见组件或字段 | 主要数据集线索 |
| --- | --- | --- | --- | --- |
| `WF-WSP整体广告分析` | `zzd/广告` | `年月区间`、`店铺`；月度店铺粒度 | `广告vs自然销售额`、`广告花费占比` | `总广告数据`，上游含 `WF广告汇总` |
| `WF-WSP分类广告分析` | `zzd/广告` | `年月区间`、`分类`、`店铺`；月度店铺分类粒度 | `室内家具分类广告vs自然销售额`、`室内家具分类广告花费占比` | `分类目广告数据` |
| `WF-WSP广告组广告分析` | `zzd/广告` | `日期区间`、`广告组`、`店铺`、`活动垂直标签` | `广告组汇总数据`、分类广告组汇总、`广告组活动明细表`、`全部广告组广告分析组合图1` | `总广告数据_广告组数据`、`分类目广告数据_广告组数据`、WF 广告日报/辅助标签数据集待核实 |
| `wf广告` | `zzd/广告` | `广告日期区间`、`活动名称`、`产品状态`、`广告店铺`、`first_10_part_numbers`、`class_name`、`分类`、`wayfair_sku` | `广告`、`产品展示和点击`、`销售和利润` | `公共数据/王威/增量更新/wf广告` |
| `wf新广告表现` | `zzd/广告` | `年月1`、`ordered sku` | Sponsored Display、Sponsored Video、Sponsored Shops、Google Shopping、Enhanced Attribution、New to Wayfair | `WF广告汇总` 已确认部分新广告表；其他数据集待核实 |
| `OS广告` | `zzd/广告` | `广告日期区间`、`链接sku`、`广告店铺`、`采购sku1`、`上架sku`、`广告活动名称` | `广告`、`销售与利润`、`产品广告销售情况` | `os广告重新计算`、`os广告2` |

### WF-WSP广告组广告分析已见字段

- 筛选器：`日期区间`、`店铺`、`广告组`、`isB2B`、`wayfair_sku`、`活动垂直标签`
- 页面分区：`广告组汇总数据`、`室内家具广告组汇总数据`、`户外家具广告组汇总数据`、`灯具广告组汇总数据`、`家具装饰广告组汇总数据`、`其他广告组汇总数据`、`广告组活动`
- 明细字段：`campaign_name`、`isB2b`、`sku`、`class_name`、`活动垂直标签`、`分类`
- 广告指标：`广告点击`、`广告曝光`、`广告花费`、`广告销售额`、`广告件数`、`广告订单数`
- 经营指标：`总成本`、`总利润`
- 后续重点：`总成本`、`总利润` 的公式链和基础字段来源已确认，剩余数值 QA 需用预聚合/带索引小表或 FineBI 已计算样本；`WSP广告组每月汇总` 中 `bid` 已确认来自 `product_max_bid`，`WSP广告组明细表` 的 `Default bid(现在的bid)` 等出价字段另走出价表补充链路。

### wf广告已见字段

- 维度字段：`sku`、`campaign_name`、`isB2b`、`class_name`、`first_10_part_numbers`、`product_status`、`活动是否有ddp`
- 指标字段：`clicks`、`impressions`、`spend_USD`、`attributed_wholesale_cost_window_view_through_USD_Day_14`、`attributed_units_window_view_through_Day_14`、`总成本`、`总利润`
- 样例值：`OARI1301`、`CS0007-chaise+armless+armchair`、`Sectionals`、`有ddp`

### wf新广告表现页面文字口径

- Direct attribution：Google Shopping 是购买前最后触点时计入直接归因。
- Assisted attribution：Google Shopping 作为早期触点影响购买，但未在 Last Meaningful Touch 下拿到最终归因。
- NTW：首次 Wayfair 客户，无历史 Wayfair 订单。
- NTB：两年回看期内未购买过该品牌 SKU。
- Google Shopping 活动通常需要 `4-6` 周学习期。
- 取消订单可能有销量但 WSC 为 `0`。
- 客户可能访问一个 SKU，但最终购买另一个 SKU。

## 3. 数据集与 SQL 来源矩阵

### 3.1 王威 / 广告：月度广告汇总链路

#### `WF广告汇总`

- FineBI 路径：`公共数据/王威/广告/WF广告汇总`
- 类型：SQL 数据集
- 粒度：`store + 年月 + 广告类型`
- 数据量：FineBI 预览显示 `32` 行
- 已见字段：`store`、`年月`、`impressions`、`clicks`、`spend`、`orders`、`WSC`、`广告类型`
- 下游：血缘图显示继续流向自助数据集/看板节点；具体下游节点名称待继续核实。

```sql
select
  store,
  left(date_est,7) 年月,
  sum(impressions) impressions,
  sum(clicks) clicks,
  ROUND(sum(spend_USD),2) spend,
  sum(attributed_orders_window_view_through_Day_14) orders,
  ROUND(sum(attributed_wholesale_cost_window_view_through_USD_Day_14),2) WSC,
  "WSP" 广告类型
from advertising_product_report_by_day
WHERE left(date_est,7)>="2026-01"
group by store,left(date_est,7)

UNION ALL
SELECT
  store, 年月,
  sum(Impressions) Impressions,
  sum(Clicks) Clicks,
  sum(Orders) Orders,
  sum(Spend) Spend,
  sum(`WSC`) WSC,
  "WGS" 广告类型
FROM google_shopping
group by store, 年月

UNION ALL
SELECT
  store, 年月,
  sum(Impressions) Impressions,
  sum(Clicks) Clicks,
  sum(Orders) Orders,
  sum(Spend) Spend,
  sum(`WSC Revenue`) WSC,
  "WSD" 广告类型
FROM (
  SELECT store,年月,Impressions,Clicks,Orders,Spend,`WSC Revenue`
  FROM sponsored_display_class_performance
  UNION ALL
  SELECT store,年月,Impressions,Clicks,Orders,Spend,`WSC Revenue`
  FROM sponsored_display_new_sku_class_performance
) T
group by store, 年月

UNION ALL
SELECT
  store, 年月,
  sum(Impressions) Impressions,
  sum(Clicks) Clicks,
  sum(Orders) Orders,
  sum(Spend) Spend,
  sum(`WSC Revenue`) WSC,
  "WSS" 广告类型
FROM sponsored_shops_shop_performance
group by store, 年月

UNION ALL
SELECT
  store, 年月,
  sum(Impressions) Impressions,
  sum(Clicks) Clicks,
  sum(Orders) Orders,
  sum(Spend) Spend,
  sum(`WSC Revenue`) WSC,
  "WSV" 广告类型
FROM sponsored_video_class_performance
group by store, 年月
```

逻辑结论：

- `WSP` 来自 `advertising_product_report_by_day`，只取 `2026-01` 及以后。
- `WGS` 来自 `google_shopping`。
- `WSD` 合并 `sponsored_display_class_performance` 与 `sponsored_display_new_sku_class_performance`。
- `WSS` 来自 `sponsored_shops_shop_performance`。
- `WSV` 来自 `sponsored_video_class_performance`。
- WSP 的 WSC 取 `attributed_wholesale_cost_window_view_through_USD_Day_14`；其他新广告表取 `WSC` 或 `WSC Revenue`。

2026-06-18 复核补充：

- 更新信息：最新更新为 `2026/06/18 07:29:37` 到 `07:30:29`，更新成功，`增加0条数据, 共32条数据`；隶属于 `system` 定时触发的 `广告更新任务`，任务开始于 `2026/06/18 07:27:06`。
- 血缘：血缘图显示 `WF广告汇总 -> WF广告汇总` 下游节点，右侧数字为 `10`，表示后续仍有 10 个引用；本轮未继续展开，避免偏离公共数据目录主线。
- 预览样例：`WF-FOP / 2026-04 / WSP` 对应 `impressions=1,516,877`、`clicks=24,832`、`spend=6,432`、`orders=541`、`WSC=73,985.45`；`WF-TD / 2026-04 / WSP` 对应 `WSC=886,383.98`。

### 3.2 王威 / 新广告：新广告明细与 attribution 数据集

#### `zzdnb_sponsored_shops_sku_level_attribution`

- FineBI 路径：`公共数据/王威/新广告/zzdnb_sponsored_shops_sku_level_attribution`
- 类型：表型 / 抽取数据集，编辑页显示表名同数据集名，未见可复制 SQL。
- 粒度：预期 `store + 年月 + Shop + SKU`。
- 数据量：2026-06-17 09:30 更新后 `95` 行；数据占用空间 `13.34 KB`。
- 已见字段：`store`、`年月`、`Shop`、`SKU`、`Impressions`、`Clicks`、`Orders`、`Spend`、`Retail Revenue`、`WSC Revenue`、`create_time`。其中预览页只展示到 `Spend`，编辑字段列表补充显示收入和生成时间字段。
- 更新任务：`system` 定时触发的 `新广告更新任务`，2026-06-17 09:30:07 更新成功，增加 `0` 条，共 `95` 条；2026-06-05 曾有 `wangwei` 手动触发的新广告文件夹更新。
- 血缘：初始后续数字为 `10`；点击后展开到同名自助层，并可见 `zzdnb_sponsored_shops_shop_performance`、`SKU_Level_Attribution`。`SKU_Level_Attribution` 后续数字为 `2`，具体组件 / 仪表板名称待核实。
- 口径提示：Sponsored Shops 销售额字段同时存在 `Retail Revenue` 和 `WSC Revenue`，在广告销售额 / ROAS 场景必须明确使用哪个字段；与 WF WSP 的 wholesale cost 字段不能混用。
- 状态：部分学习，待继续确认 SQL、下游组件名称和 `WSS` 汇总时是否直接使用该 SKU attribution 表或经 `shop_performance` 汇总。

#### `zzdnb_sponsored_shops_shop_performance`

- FineBI 路径：`公共数据/王威/新广告/zzdnb_sponsored_shops_shop_performance`
- 类型：表型 / 抽取数据集，编辑页显示表名同数据集名，未见可复制 SQL。
- 粒度：预期 `store + 年月 + Sponsored Shop`。
- 数据量：2026-06-17 09:30 更新后 `10` 行；数据占用空间 `3.10 KB`。
- 已见字段：`store`、`年月`、`Sponsored Shop`、`Impressions`、`Clicks`、`Orders`、`Spend`、`ROAS`、`Retail Revenue`、`WSC Revenue`、`CTR`、`create_time`。
- 更新任务：`system` 定时触发的 `新广告更新任务`，2026-06-17 09:30:23 更新成功，增加 `0` 条，共 `10` 条；2026-06-05 曾有 `wangwei` 手动触发的新广告文件夹更新。
- 血缘：初始后续数字为 `6`；点击后展开到同名自助层，并可见 `shop_performance`、`SKU_Level_Attribution`，两者后续数字均为 `2`。具体组件 / 仪表板名称待核实。
- 口径提示：该表自带 `ROAS`、`CTR`，汇总层不能直接平均；应使用 `WSC Revenue / Spend`、`Clicks / Impressions` 等分子分母重算。`WF广告汇总` 中 `WSS` 子查询曾显示直接 `FROM sponsored_shops_shop_performance`，但该库表与 FineBI 数据集 `zzdnb_sponsored_shops_shop_performance` 的数据库层关系待核实。
- 状态：部分学习，待继续确认 SQL、`WF广告汇总` 字段级来源和下游组件名称。

#### `zzdnb_google_shopping_sku_level_attribution`

- FineBI 路径：`公共数据/王威/新广告/zzdnb_google_shopping_sku_level_attribution`
- 类型：表型 / 抽取数据集，编辑页显示表名同数据集名，未见可复制 SQL。
- 粒度：预期 `store + 年月 + Ordered SKU`。
- 数据量：2026-06-17 09:30 更新后 `270` 行；数据占用空间 `30.52 KB`。
- 已见字段：`store`、`年月`、`Ordered SKU`、`Orders`、`Retail Revenue`、`Wholesale Revenue`、`create_time`。
- 更新任务：`system` 定时触发的 `新广告更新任务`，2026-06-17 09:30:07 更新成功，增加 `0` 条，共 `270` 条；2026-06-05 曾有 `wangwei` 手动触发的新广告文件夹更新。
- 血缘：初始后续数字为 `7`；点击后展开到同名自助层，并可见 `zzdnb_google_shopping`、`SKU-Level Attribution`，后续数字分别可见 `3`、`2`。具体组件 / 仪表板名称待核实。
- 口径提示：该表同时包含 `Retail Revenue` 和 `Wholesale Revenue`，其中 `Wholesale Revenue` 与 `WF广告汇总` / 其他新广告表中的 `WSC`、`WSC Revenue` 是否完全等价待核实；广告销售额场景不能与 `Retail Revenue` 混用。
- 状态：部分学习，待继续确认 SQL、`WGS` 汇总字段级来源和下游组件名称。

#### `zzdnb_google_shopping_enhanced_attribution`

- FineBI 路径：`公共数据/王威/新广告/zzdnb_google_shopping_enhanced_attribution`
- 类型：表型 / 抽取数据集，编辑页显示表名同数据集名，未见可复制 SQL。
- 粒度：预期 `store + 年月 + Enhanced Attribution`。
- 数据量：2026-06-17 09:30 更新后 `4` 行；数据占用空间 `1.26 KB`。
- 已见字段：`store`、`年月`、`Enhanced Attribution`、`Orders`、`Sales`、`WSC`、`create_time`。预览样例已见 `Assisted Attribution`、`Direct Attribution` 两类。
- 更新任务：`system` 定时触发的 `新广告更新任务`，2026-06-17 09:30:22 到 09:30:23 更新成功，增加 `0` 条，共 `4` 条；2026-06-05 曾有 `wangwei` 手动触发的新广告文件夹更新。
- 血缘：初始后续数字为 `3`；点击后展开到同名自助层，并可见 `Enhanced Attribution`，该节点后续数字为 `2`。具体组件 / 仪表板名称待核实。
- 口径提示：页面文字口径已说明 Direct attribution 表示 Google Shopping 是购买前最后触点，Assisted attribution 表示 Google Shopping 是早期触点但不是最终归因；该表收入字段为 `Sales` 与 `WSC`，广告销售额口径需优先确认使用 `WSC` 还是展示层另有命名。
- 状态：部分学习，待继续确认 SQL、Enhanced Attribution 组件和 `WF广告汇总` 中对应关系。

#### `zzdnb_google_shopping_attribution_new_to_wayfair`

- FineBI 路径：`公共数据/王威/新广告/zzdnb_google_shopping_attribution_new_to_wayfair`
- 类型：表型 / 抽取数据集，编辑页显示表名同数据集名，未见可复制 SQL。
- 粒度：预期 `store + 年月`，并携带 New to Wayfair 相关归因指标。
- 数据量：2026-06-17 09:30 更新后 `2` 行；数据占用空间 `0.97 KB`。
- 已见字段：`store`、`年月`、`Orders`、`NTW Orders`、`Sales`、`WSC`、`create_time`。
- 更新任务：`system` 定时触发的 `新广告更新任务`，2026-06-17 09:30:22 更新成功，增加 `0` 条，共 `2` 条；2026-06-05 曾有 `wangwei` 手动触发的新广告文件夹更新。
- 血缘：初始后续数字为 `3`；点击后展开到同名自助层，并可见 `Attribution - New to Wayfair`，该节点后续数字为 `2`。具体组件 / 仪表板名称待核实。
- 口径提示：`NTW Orders` 从样例看是比例或小数值字段，不能直接当订单数求和使用；其业务含义、分子分母和展示层命名待核实。该表收入字段为 `Sales` 与 `WSC`，广告销售额口径需优先确认展示层是否取 `WSC`。
- 状态：部分学习，待继续确认 SQL、NTW Orders 口径和下游组件名称。

#### `新广告`目录叶子数据集清单

- `zzdnb_sponsored_shops_sku_level_attribution`：2026-06-18 已学习 `WF店铺绩效2.0` 右侧链路；拆分 `Shop` 取 `Class`，字段含 `store`、`年月`、`Shop`、`SKU`、`Impressions`、`Clicks`、`Orders`、`Spend`、`Retail Revenue`、`WSC Revenue`、`Class`。从 `上架sku利润` 补 `出单价`、`利润`、`平均利润率`、`最小利润率`、`最大利润率`，匹配 `store = store`、`SKU = SKU`、`年月 = LEFT(日期,7)`；生成 `平均利润 = WSC Revenue * 平均利润率 - Spend`、`最大利润 = WSC Revenue * 最大利润率 - Spend`、`最小利润 = WSC Revenue * 最小利润率 - Spend`、`关联id = CONCATENATE(store, Class, 年月)`。
- `zzdnb_sponsored_shops_shop_performance`：2026-06-18 已学习右侧链路；拆分 `Sponsored Shop` 取 `Class`，保留 `store`、`年月`、`Sponsored Shop`、`Impressions`、`Clicks`、`Orders`、`Spend`、`ROAS`、`Retail Revenue`、`WSC Revenue`、`CTR`、`Class`；从 `zzdnb_sponsored_shops_sku_level_attribution` 补 `平均利润`、`最大利润`、`最小利润`，匹配 `store = store`、`年月 = 年月`、`Class = Class`；生成 `关联id = CONCATENATE(store, Class, 年月)`。
- `zzdnb_google_shopping_sku_level_attribution`：2026-06-18 已学习右侧链路；字段含 `store`、`年月`、`Ordered SKU`、`Orders`、`Retail Revenue`、`Wholesale Revenue`。从 `上架sku利润` 补 `出单价`、`利润`、`平均利润率`、`最小利润率`、`最大利润率`，匹配 `store = store`、`Ordered SKU = SKU`、`年月 = LEFT(日期,7)`；生成 `平均利润 = Wholesale Revenue * 平均利润率`、`最大利润 = Wholesale Revenue * 最大利润率`、`最小利润 = Wholesale Revenue * 最小利润率`、`关联id = CONCATENATE(store, 年月)`；再从 `zzdnb_detailed_listing_health` 补 `Supplier Part Number`，匹配 `store = store`、`Ordered SKU = Wayfair SKU`、`LEFT(年月,7) = 年月`。
- `zzdnb_google_shopping_enhanced_attribution`：2026-06-18 已学习右侧链路；字段设置保留 `store`、`年月`、`Enhanced Attribution`、`Orders`、`Sales`、`WSC`；生成 `关联id = CONCATENATE(store, 年月)`。该表仍需注意 `Sales` 与 `WSC` 口径不能混用。
- `zzdnb_google_shopping_attribution_new_to_wayfair`：2026-06-18 已学习 `WF店铺绩效2.0` 右侧链路；字段设置保留 `store`、`年月`、`Orders`、`NTW Orders`、`Sales`、`WSC`；生成 `关联id = CONCATENATE(store, 年月)`。`NTW Orders` 样例为小数，仍需核实其分子分母和展示口径。
- `zzdnb_google_shopping`：2026-06-18 已学习右侧链路；字段含 `store`、`年月`、`Impressions`、`Clicks`、`Orders`、`Spend`、`CTR`、`Retail`、`WSC`、`Retail ROAS`、`WSC ROAS`、`SKUS w/Imps`。从 `zzdnb_google_shopping_sku_level_attribution` 补 `平均利润`、`最大利润`、`最小利润`，匹配 `store = store`、`年月 = 年月`；生成 `关联id = CONCATENATE(store, 年月)`。
- `zzdnb_sponsored_video_sku_level_attribution`：2026-06-18 已学习右侧链路；字段含 `store`、`年月`、`Class`、`SKU`、`Orders`、`Retail Revenue`、`WSC Revenue`。从 `上架sku利润` 补 `出单价`、`利润`、`平均利润率`、`最小利润率`、`最大利润率`，匹配 `store = store`、`SKU = SKU`、`年月 = LEFT(日期,7)`；生成 `平均利润 = WSC Revenue * 平均利润率`、`最小利润 = WSC Revenue * 最小利润率`、`最大利润 = WSC Revenue * 最大利润率`、`关联id = CONCATENATE(store, Class, 年月)`。
- `zzdnb_sponsored_video_class_performance`：2026-06-18 已学习右侧链路；字段含 `store`、`年月`、`Class`、`Impressions`、`Clicks`、`Orders`、`Spend`、`ROAS`、`Retail Revenue`、`WSC Revenue`。从 `zzdnb_sponsored_video_sku_level_attribution` 补 `平均利润`、`最小利润`、`最大利润`，匹配 `store = store`、`年月 = 年月`、`Class = Class`；生成 `平均利润1 = 平均利润 - Spend`、`最小利润1 = 最小利润 - Spend`、`最大利润1 = 最大利润 - Spend`、`关联id = CONCATENATE(store, Class, 年月)`。
- `zzdnb_sponsored_display_sku_level_attribution`：2026-06-18 已学习 `WF店铺绩效2.0` 右侧链路；字段含 `store`、`年月`、`Class`、`SKU`、`Orders`、`Retail Revenue`、`WSC Revenue`、`Impressions`、`Clicks`、`CTR`、`Spend`。右侧从 `上架sku利润` 补 `出单价`、`利润`、`平均利润率`、`最大利润率`、`最小利润率`，匹配 `store = store`、`SKU = SKU`、`LEFT(年月,7) = LEFT(日期,7)`；生成 `平均利润 = WSC Revenue * 平均利润率 - Spend`、`最大利润 = WSC Revenue * 最大利润率 - Spend`、`最小利润 = WSC Revenue * 最小利润率 - Spend`、`关联id = CONCATENATE(store, Class, 年月)`。该利润是 Sponsored Display SKU attribution 辅助利润，不等同 `WF店铺绩效2.0/广告` 的 `总利润` 公式。
- `zzdnb_sponsored_display_new_sku_level_attribution`：2026-06-18 已学习右侧链路；字段含 `store`、`年月`、`Class`、`SKU`、`Impressions`、`Clicks`、`Orders`、`CTR`、`Spend`、`Retail Revenue`、`WSC Revenue`。从 `上架sku利润` 补 `出单价`、`利润`、`平均利润率`、`最大利润率`、`最小利润率`，匹配 `store = store`、`SKU = SKU`、`年月 = LEFT(日期,7)`；过滤条件显示 `年月` 日期最晚 N=3；生成 `平均利润 = WSC Revenue * 平均利润率 - Spend`、`最大利润 = WSC Revenue * 最大利润率 - Spend`、`最小利润 = WSC Revenue * 最小利润率 - Spend`、`关联id = CONCATENATE(store, Class, 年月)`。
- `zzdnb_sponsored_display_new_sku_class_performance`：2026-06-18 已学习右侧链路；字段含 `store`、`年月`、`Class`、`Impressions`、`Clicks`、`Orders`、`Spend`、`ROAS`、`Retail Revenue`、`WSC Revenue`。从 `sponsored_display_new_sku_level_attribution` 补 `平均利润`、`最大利润`、`最小利润`，匹配 `store = store`、`Class = Class`、`LEFT(年月,7) = LEFT(年月,7)`；生成 `平均利润1 = IF(Orders>0, 平均利润, -Spend)`、`最小利润1 = IF(Orders>0, 最小利润, -Spend)`、`最大利润1 = IF(Orders>0, 最大利润, -Spend)`、`关联id = CONCATENATE(store, Class, 年月)`。
- `zzdnb_sponsored_display_class_performance`：2026-06-18 已在 `WF店铺绩效2.0` 中学习到部分右侧链路；字段保留 `store`、`年月`、`Class`、`Impressions`、`Clicks`、`Orders`、`Spend`、`Retail Revenue`、`WSC Revenue`、`ROAS`，并从 `sponsored_display_sku_level_attribution` 补 `平均利润`、`最大利润`、`最小利润`，匹配 `年月 = 年月`、`Class = Class`、`store = store`。后续公式为 `平均利润1 = IF(Orders>0, 平均利润, -Spend)`，`最大利润1 = IF(Orders>0, 最大利润, -Spend)`，`最小利润1 = IF(Orders>0, 最小利润, -Spend)`。待继续确认下游组件名称。

#### 2026-07-08 数据库物理表复核补充

- FineBI `zzdnb_*` 新广告数据集在当前数据库中通常对应无前缀物理表，例如 `google_shopping_sku_level_attribution`、`sponsored_shops_shop_performance`、`sponsored_display_class_performance`、`sponsored_video_class_performance`。
- 物理字段名多为下划线版：`Ordered_SKU`、`Attribution_Type`、`Sponsored_Shop`、`Retail_Revenue`、`WSC_Revenue`、`Wholesale_Revenue`；不能直接照抄 FineBI 展示名写 SQL。
- `google_shopping_sku_level_attribution.Wholesale_Revenue` 在 `2026-02` 至 `2026-05` 样本中与 `google_shopping.WSC` 汇总一致；`Retail_Revenue` 与 `google_shopping.Retail` 汇总近似一致，Google Shopping 广告销售额场景优先使用 `Wholesale_Revenue / WSC`，不要混用 Retail。
- `google_shopping_attribution_new_to_wayfair` 物理字段为 `pct_NTW_Orders`，不是整数订单数字段；展示层 `NTW Orders` 的分子分母仍待核实。
- Sponsored 系列表多批 `create_time` 共存，默认数据库复刻时应先取最新批次；`sponsored_display_sku_level_attribution` 最新批次内仍有同 `store + 年月 + Class + SKU` 多行，必须按目标粒度聚合。
- 本轮只做数据库侧物理表和字段复核，未进入 FineBI 组件确认是否已过滤最新批次；组件侧仍为待核实。

#### `WF店铺绩效2.0` Listing / SKU 维度补充

- `zzdnb_wf_class中文分类`：简单映射表，字段样例为英文 Class、中文分类、分类；`WF店铺绩效2.0/广告` 中 `分类` 字段通过 `class_name = Class Name` 从该表补入。
- `zzdnb_detailed_listing_health`：字段设置后生成 `关联id = CONCATENATE(store, 年月, Supplier Part Number, Wayfair SKU, Wayfair Brand)`；后续被 `sku维度表2` 用于补 `Impression Percentile (by Class)(曝光百分位)`、`Impression Rank (by Class)(曝光排名)`、`Supplier Part Number1`。广告组件中文 `评分/评论数` 的最终重命名位置仍待继续确认。
- `总览表`：来源为 `account_overview_conversion`，字段含店铺月度收入、订单、销量和 Sponsored Product 汇总指标；生成 `关联id2 = CONCATENATE(store, LEFT(年月,7))`。
- `sku维度表`：来源为 `zzdnb_option_drill_down`，对指标做列转行，生成 `关联id = CONCATENATE(store, 年月, 维度)` 和 `关联id2 = CONCATENATE(store, 年月)`。
- `sku维度表2`：来源为 `zzdnb_option_drill_down`，过滤 `Supplier Part Number` 非空，补 Listing Health、采购 SKU、model、分类和 `zzdnb_wfproduct_management_latest1.SKU`；生成 `最大时间`、`四月收入`、`四月曝光排名`、`上月总收入`、`上架sku`、`wayfairsku最大活跃日期` 等字段。`四月收入` / `四月曝光排名` 虽然命名写“四月”，公式实际依赖 `MONTHDELTA(日期,1)=最大时间`，解释时需结合当前最大时间。

#### `总广告数据`

- FineBI 路径：`公共数据/王威/广告/总广告数据`
- 类型：抽取数据集
- 粒度：`store + 年月`
- 数据量：2026-06-18 更新后 `76` 行
- 数据占用空间：`76.60 KB`
- 更新任务：`system` 定时触发的 `广告更新任务`，最新记录 `2026/06/18 07:28:48` 到 `07:28:49` 更新成功，`增加0条数据, 共76条数据`，任务开始于 `2026/06/18 07:27:06`。
- 已见字段：`store`、`年月`、`销售额`、`订单量`、`件数`、`Sessions`、`自然曝光量`、`自然访问量`、`广告点击`、`广告曝光`、`广告花费`、`广告销售额`、`广告订单量`、`广告件数`、`总销售额`、`总订单量`、`总销量`、`自然销售额`、`自然订单量`、`自然销量`、`广告CPC`、`广告CTR`、`广告CVR`、`广告CR`、`广告CPM`、`广告CPA`、`广告花费占比`、`自然CVR`、`自然CR`、`create_time`
- 用途：支撑 `WF-WSP整体广告分析` 的月度店铺整体广告/自然/总量分析。
- 编辑页复核：进入 `编辑` 后左侧只见 `选字段` 单一步骤；未见 FineBI 自助层的 `新增公式列`、`分组汇总`、`关联` 或可复制 SQL。当前判断这些比率和销售/自然字段已在来源表中存在，`总广告数据` 这里只做字段选择；底层生成 SQL/公式仍待核实。
- 本轮操作风险：2026-06-18 复核时误触 `保存`，未改字段但页面随后提示 `当前表未更新`；未点击 `更新数据`。2026-06-18 继续复核时，数据预览仍显示 `当前表未更新！`、`共0条数据`、`计算结果为空`，未手动触发更新。后续复核该表时需先确认是否已由定时任务恢复为可预览状态。
- 血缘/关联：
  - `总广告数据 -> 总广告数据_广告组数据`，右侧数字入口初始为 `109`；点击数字后展开到下游 `广告` 节点，剩余数字显示为 `108`。
  - `总广告数据 -> 总广告数据`，右侧数字入口为 `14`；点击数字后展开到整体广告分析组件。
- `总广告数据_广告组数据` 节点信息：位置 `wangwei的分析/广告/WF广告数据`，创建者 `wangwei`，创建时间 `2026-04-20 14:32:44`，血缘层级 `2`。
- `总广告数据` 已见后续组件：
  - `广告CR与自然CR百分比对比（*100）`
  - `广告vs自然销售额`
  - `广告曝光点击`
  - `广告花费占比`
  - `广告花费、销售额、利润与ROI分析`
  - `广告点击转化`
  - `广告增长VS销售增长`
- 已确认组件终点：2026-06-18 继续点击各组件右侧数字后确认，上述 7 个整体组件均继续到 `WSP广告分析`。
- 当前判断：`总广告数据` 直接支撑 `WF-WSP整体广告分析`/`WSP广告分析` 的整体组件；同时也通过 `总广告数据_广告组数据` 进入广告组相关自助数据集链路。具体组件公式仍需继续进入分析主题或自助数据集编辑页核实。

#### `分类目广告数据`

- FineBI 路径：`公共数据/王威/广告/分类目广告数据`
- 类型：抽取数据集
- 粒度：`store + 年月 + 分类`
- 数据量：2026-06-18 更新后 `402` 行
- 数据占用空间：`246.04 KB`
- 更新任务：`system` 定时触发的 `广告更新任务`，最新记录 `2026/06/18 07:28:49` 更新成功，`增加0条数据, 共402条数据`，任务开始于 `2026/06/18 07:27:06`。
- 用途：支撑 `WF-WSP分类广告分析` 的分类维度广告/自然/总量分析。
- 已见字段：`store`、`分类`、`年月`、`销售额`、`订单量`、`件数`、`Sessions`、`自然曝光量`、`自然访问量`、`广告点击`；右侧后续字段与 `总广告数据` 同套广告/自然/总量指标待继续横向滚动确认。
- 分类样例：`室内家具`、`户外家具`、`厨房卫浴`、`灯具`、`其他`。
- 血缘/关联：
  - `分类目广告数据 -> 分类目广告数据_广告组数据`，右侧数字入口展开后显示 `109` 个后续引用。
  - `分类目广告数据 -> 分类目广告数据`，右侧数字入口显示 `20` 个后续引用。
- `分类目广告数据` 同名节点 `20` 分支已确认后续组件，点击各组件右侧数字 `1` 后均落到 `WSP类目广告分析`：
  - `点击与转化`
  - `分类广告vs自然销售额`
  - `分类广告趋势分析组合图`
  - `分类广告分析组合图3`
  - `分类广告CR对比*100`
  - `分类广告花费、销售额、利润与ROI分析`
  - `分类广告点击转化`
  - `分类广告分析组合图4`
  - `分类广告曝光点击`
  - `分类广告花费占比`
- `分类目广告数据_广告组数据` 已见后续节点：
  - 广告组汇总类：`WSP广告组每月汇总`、`广告组汇总数据_在售`、`广告组汇总数据_不能筛选`、`广告组汇总数据_新品3个月`、`广告组汇总数据_新品6个月`、`广告组汇总数据_下架+没有标签的`、`广告组汇总数据_清仓+滞销`
  - 分类广告组汇总类：`室内家具广告组汇总数据`、`户外家具广告组汇总数据`、`灯具广告组汇总数据`、`家具装饰广告组汇总数据`、`其他广告组汇总数据`
  - 标签拆分类：`室内家具广告组汇总数据_滞销+清仓`、`户外家具广告组汇总数据_滞销+清仓`、`灯具广告组汇总数据_滞销+清仓`、`室内家具广告组汇总数据_在售`、`户外家具广告组汇总数据_在售`、`灯具广告组汇总数据_在售`
  - 新品拆分类：`室内家具广告组汇总数据_新品3个月`、`户外家具广告组汇总数据_新品3个月`、`灯具广告组汇总数据_新品3个月`、`室内家具广告组汇总数据_新品6个月`、`户外家具广告组汇总数据_新品6个月`、`灯具广告组汇总数据_新品6个月`
  - 下架/无标签类：`室内家具广告组汇总数据_下架+没有标签`、`户外家具广告组汇总数据_下架+没有标签`、`灯具广告组汇总数据_下架+没有标签`
  - 看板组件类：`WSP广告组明细表`、`广告组活动明细表`、`广告组广告分析组合图1`、`广告组广告分析组合图3`、`广告组广告分析组合图4`、`广告组广告趋势分析组合图`、`广告组广告花费、销售额、利润与ROI分析`、`广告组广告点击与转化`、`广告组bid趋势`
  - 分类看板组件类：`分类广告vs自然销售额`、`分类广告花费占比`、`分类广告趋势分析组合图`、`分类广告分析组合图3`、`分类广告分析组合图4`、`分类广告CR对比*100`、`分类广告花费、销售额、利润与ROI分析`、`分类广告点击转化`、`分类广告曝光点击`、`点击与转化`
- 2026-06-18 继续展开确认：`分类目广告数据_广告组数据 -> 广告` 后，右侧剩余数字为 `108`；继续展开后可见 `分类目广告数据` 数字 `20`、`总广告数据` 数字 `14`，以及广告组汇总/标签拆分/看板组件节点。
- 已确认广告组关键组件终点：
  - `WSP广告组每月汇总 -> WSP广告组广告分析`
  - `广告组广告花费、销售额、利润与ROI分析 -> WSP广告组广告分析`
  - `广告组bid趋势 -> WSP广告组广告分析`
  - `广告组广告点击与转化 -> WSP广告组广告分析`
  - `广告组广告趋势分析组合图 -> WSP广告组广告分析`
  - `广告组活动明细表 -> WSP广告组广告分析`
- 2026-06-18 补充逐点确认：`WSP广告组明细表`、`广告组广告分析组合图1`、`广告组广告分析组合图3`、`广告组广告分析组合图4`、`户外家具广告组汇总数据_在售`、`灯具广告组汇总数据_滞销+清仓`、`广告组汇总数据_下架+没有标签的`、`其他广告组汇总数据`、`户外家具广告组汇总数据_新品6个月`、`广告组汇总数据_清仓+滞销`、`室内家具广告组汇总数据_下架+没有标签`、`广告组汇总数据_新品6个月`、`室内家具广告组汇总数据_新品6个月`、`灯具广告组汇总数据_在售`、`户外家具广告组汇总数据_下架+没有标签`、`灯具广告组汇总数据_新品3个月`、`室内家具广告组汇总数据_在售` 均可展开到 `WSP广告组广告分析`。
- 2026-06-18 本轮可见但未完全逐点闭环的同构节点：`广告组汇总数据_新品3个月`、`广告组汇总数据_在售`、`广告组汇总数据_不能筛选`、`家具装饰广告组汇总数据`、`室内家具广告组汇总数据`、`户外家具广告组汇总数据`、`室内家具广告组汇总数据_滞销+清仓`、`户外家具广告组汇总数据_滞销+清仓`、`室内家具广告组汇总数据_新品3个月`、`户外家具广告组汇总数据_新品3个月`、`灯具广告组汇总数据_下架+没有标签`、`灯具广告组汇总数据`。这些节点右侧均显示可继续展开的 `1`，但因画布自动位移，本轮只作为待下轮逐点复核项。
- `WSP广告组每月汇总` 悬浮信息：组件名 `WSP广告组每月汇总`，位置 `wangwei的分析/广告/WF广告数据`，创建者 `wangwei`，创建时间 `2026/04/22 15:11:42`，最近编辑时间 `2026/06/16 20:47:26`。
- 当前判断：`分类目广告数据_广告组数据` 是 `WF-WSP广告组广告分析` 的核心自助数据集之一，且已经承载滞销/清仓、在售、新品、下架/无标签等广告组拆分逻辑。
- 2026-06-18 数据库补样本确认：
  - `WSP广告组每月汇总` 的 `bid` 是 `advertising_product_report_by_day.product_max_bid`，作为维度参与分组；不是 `Default bid(现在的bid)` 或建议 bid。
  - 单一映射样本：`WF-TD / 2026-01 / campaign_id=342884 / campaign_name=41907-LED / sku=OARI1049 / bid=0.3 / first_10_part_numbers=TD-A-41907LED`，经 `cg_part_sku映射时间范围` 匹配到 `上架SKU=TD-A-41907LED / 链接SKU=W110467359 / 采购SKU=41907LED x 1`。
  - 多映射样本：`WF-TD / 2026-03 / campaign_id=347274 / campaign_name=CS0002 / sku=OARI1014 / bid=0.3`，`first_10_part_numbers` 下 `7` 个上架 SKU 映射到同一 `链接SKU=W110073209`，并拆出 `14` 个采购 SKU 组件；广告指标不能在采购 SKU 展开后直接求和。
  - 多 bid 样本：`WF-FOP / 2026-05 / campaign_id=458816 / campaign_name=金属屏风-OS0014 / sku=UXIF3679` 存在 `product_max_bid=0.4/0.6/0.8` 三组，按 bid 拆行后分别汇总对应曝光、点击、花费、WSC、订单；三组均映射到 `上架SKU=FOP-OS0014-BLACK/BROWN/WHITE / 链接SKU=UXIF3679 / 采购SKU=OS0014B-BLACK/BROWN/WHITE x 1`。
- 2026-06-18 继续补核成本样本：`OARI1049` 样本已取到广告事实 `units14=2`、`spend_USD=13.8`、`WSC=48.6`，并补到 `采购SKU=41907LED`、`DDP=15.222222`、`出货费用=1.2`；但 `广告组合单买详情` 无 `OARI1049 + 41907-LED` 的 `单卖`，无法完成 `总成本` 公式。继续找有 `单卖` 的样本时，`cg_part_sku映射时间范围`、`ddp汇总`、`cg仓发货费`、`产品发货费` 均无索引，数据库即席精确复算超时。
- 待核实：`分类目广告数据_广告组数据` 本身当前只见字段设置；标签归类公式已在 `WF广告数据/广告` 自助表中确认。`总利润/总成本` 和 `ROI` 的公式链已确认，剩余数值 QA 需用预聚合/带索引小表或 FineBI 已计算样本，不应再按权限不足或字段未追清处理。

#### `wf产品`

- FineBI 路径：`公共数据/王威/WF产品/wf产品`
- 类型：SQL / 抽取数据集
- 数据量：2026-06-17 10:41 更新后 `23,719` 行；数据占用空间 `4.13 MB`。
- 已见字段：`产品id`、`platform`、`store`、`上架sku`、`Product_Type`、`Collection`、`Launch_Date`、`产品状态`、`wayfair_sku`、`链接sku`、`category`、`first_class`、`second_class`、`model`、`采购sku`、`国家`、`base_costs_USD`、`promotional_base_cost`、`采购SKU`、`数量`、`垂直细分标签`、`采购状态`、`预计和实际ddp汇总`。
- 业务作用：Wayfair 产品维表 / 商品属性增强层，把平台产品、WF product management、鲸汇销售产品、公司 SKU 状态、当前月 DDP、当天 WF price 成本字段串在一起，可用于广告商品维度、listing health、supplier performance、产品详情等下游。
- 更新记录：`system` 定时触发的 `WF产品更新任务`，约 `10:40` 更新；2026-06-17 成功，增加 `30` 条，共 `23,719` 条。2026-05-31 曾更新失败，报错涉及非聚合字段不在 `GROUP BY` 中。
- 血缘：`wf产品` 初始后续数字为 `11`；点击后展开到 `zzdnb_detailed_listing_health`、`zzdnb_wf_supplier_parts_performance`、`wf产品详情`，更深节点本轮未继续展开。

```sql
select a.*, e.base_costs_USD, e.promotional_base_cost, b.`采购SKU`, b.数量,
       c.`垂直细分标签`, c.`采购状态`, d.预计和实际ddp汇总
from (
  SELECT
    a.产品id,
    platform,
    store,
    上架sku,
    Product_Type,
    Collection,
    Launch_Date,
    产品状态,
    平台sku wayfair_sku,
    链接sku,
    category,
    first_class,
    second_class,
    model,
    采购sku,
    case
      when store in ("WF-TD-CA","WF-FOP-CA") then "加拿大"
      when store in ("WF-TD-UK","WF-FOP-UK") then "英国"
      else "美国"
    end 国家
  FROM `平台产品详情表` a
  left join (
    select concat(店铺,"_",Supplier_Part_Number) 产品id, Product_Type, Collection, `Launch_Date`
    from wfproduct_management
    where left(insert_date,10) = (select left(max(insert_date),10) from wfproduct_management)
      and length(Product_Status)>0
  ) b on a.产品id = b.产品id
  where platform = "wayfair"
    and b.产品id is not null
  group by a.产品id
) a
left join (
  select * from `鲸汇店铺销售产品最新`
  where 店铺 in ("WF FOP","WF TD")
) b
  on case
       when a.store in ("WF-TD-CA","WF-FOP-CA","WF-TD-UK","WF-FOP-UK")
       THEN concat(SUBSTRING_INDEX(a.store,"-",2),"_",A.上架sku)
       ELSE a.产品id
     END = b.产品id
left join 公司sku状态表 c
  on b.`采购SKU`= c.sku
left join (
  select *,"美国" 国家 from `ddp汇总表_美国` where 年月 = left(now(),7)
  union all
  select * ,"加拿大" 国家 from `ddp汇总表_加拿大` where 年月 = left(now(),7)
  union all
  select *,"英国" 国家 from `ddp汇总表_英国` where 年月 = left(now(),7)
) d
  on b.采购SKU = d.SKU and a.国家 = d.国家
left join (
  select * from wfprice where date = left(now(),10)
) as e
  on a.产品id = e.产品id
order by 产品id
```

- 逻辑要点：
  - 主体来自 `平台产品详情表`，限制 `platform = "wayfair"`，并要求最新日 `wfproduct_management` 有 `Product_Status`。
  - 国家按店铺后缀映射：CA 为加拿大，UK 为英国，其余为美国。
  - 采购 SKU 和数量来自 `鲸汇店铺销售产品最新`，该子查询当前只取 `WF FOP`、`WF TD` 两类店铺；CA/UK 通过 `SUBSTRING_INDEX(store,"-",2) + "_" + 上架sku` 匹配。
  - DDP 只取当前月美国/加拿大/英国 `ddp汇总表_*`，按 `采购SKU + 国家` 关联。
  - 成本字段来自当天 `wfprice` 的 `base_costs_USD`、`promotional_base_cost`。
- 风险 / 待核实：SQL 在内层 `group by a.产品id` 的同时选择多个非聚合字段，且已出现过 GROUP BY 更新失败；若 MySQL SQL mode 或重复产品记录变化，可能影响字段取值稳定性。

#### `zzdnb_wf上架sku每日销售额`

- FineBI 路径：`公共数据/王威/广告/zzdnb_wf上架sku每日销售额`
- 类型：抽取数据集；页面未展示 `修改SQL`，本轮只能确认字段、更新和血缘。
- 数据量：2026-06-17 07:27 更新后 `83,005` 行；数据占用空间 `14.10 MB`。
- 已见字段：`store`、`日期`、`supplier_part`、`currency`、`total_revenue`、`units_sold`、`create_time`、`update_time`。
- 更新记录：`system` 定时触发的 `广告更新任务`，约 `07:27` 更新；2026-06-17 增加 `488` 条，共 `83,005` 条。2026-06-15 曾有多次手动单表更新记录。
- 血缘：初始后续数字 `109`，点击后展开到 `广告` 节点，剩余后续数字 `108`。结合 `WF广告数据/广告` 自助表字段设置，该表用于按 `store + 月份 + sku/wayfair_sku` 回补 SKU 月销售额。
- 与广告看板关系：`广告` 自助表中已见“补 SKU 月销售额”步骤，从该表取销售额求和，匹配 `store = store`、`LEFT(date_est,7) = LEFT(日期,7)`、`sku = wayfair_sku`；`总销售额1` 再按店铺系数折算。
- 2026-06-30 数据库补充：物理表 `wf上架sku每日销售额` 当前字段含 `链接sku`、`wayfair_sku`，无主键 / 索引、无直接触发器；同名事件每天调用同名过程，最近执行 `2026-06-30 08:27:00`。过程删除近 15 天 `create_time` 窗口后，从 `wf_supplier_parts_performance` 写入，补 `链接sku/wayfair_sku`，并对 `CAD/GBP` 按最新 `货币汇率` 折算。2026-06-30 查询时目标表和源表最新业务日期均为 `2026-06-27`。本表 `supplier_part` 是销售源 Supplier Part Number / 销售 SKU 层，按产品管理分类可为 `Kit`、`Standard`、`Sellable Component` 或少量 `Nonsellable Component`，不是 `总账单详情_extract.上架sku` 的订单子件层。

#### `WF广告数据` 分析主题 / 自助数据表

- FineBI 位置：`wangwei的分析/广告/WF广告数据`
- 已确认入口：从 `总广告数据_广告组数据` 血缘节点右侧“跳转”进入，Chrome 新开标签标题为 `WF广告数据`。
- 2026-06-18 复核入口：直接进入 Chrome 已打开的 `WF广告数据` 分析主题，不再切 `WF店铺绩效2.0`。当前选中自助表 `广告`，数据预览显示“前 5,000 条数据，共 `372,422` 条数据”。
- `广告` 数据预览前段字段：`14天周期`、`store`、`date_est`、`campaign_id`、`campaign_name`、`campaign_status`、`campaign_is_active`、`campaign_daily_cap_USD`、`campaign_lifetime_budget_USD`。
- 主题内数据集清单：
  - `zzdnb_wf上架sku每日销售额`
  - `zzdnb_detailed_listing_health`
  - `wf广告`
  - `wf_sku不断货销量`
  - `本竞品价格对比`
  - `zzdnb_wfproduct_management_latest`
  - `zzdnb_product_fabric`
  - `广告`
  - `WF广告汇总`
  - `分类目广告数据`
  - `分类目广告数据_广告组数据`
  - `总广告数据`
  - `总广告数据_广告组数据`
- 主题内页面/组件页签：`WSP广告分析`、`WSP类目广告分析`、`WSP广告组广告分析`、`WSP广告组每月汇总`、`整体广告分析`。

##### `广告` 自助表已见处理步骤

- 基础数据源：`广告`，编辑页显示位置为 `wangwei的分析/店铺绩效/WF店铺绩效2.0`。
- 当前判断：`WF广告数据/广告` 不是直接以 `公共数据/王威/增量更新/wf广告` SQL 作为唯一基础源，而是引用上游主题 `WF店铺绩效2.0/广告`，再在广告主题内继续做“其他表添加列”和公式列。
- 上游 `WF店铺绩效2.0/广告` 血缘节点信息：曾在 FineBI 血缘节点看到 `当前表无权限` 提示；数据集名 `广告`，创建者 `wangwei`，创建时间 `2024/11/26 14:31:10`，最近更新时间 `2026/06/16 20:13:09`，血缘层级 `3`。2026-06-17 用户确认当前权限足够，后续已进入上游并确认 `总成本/总利润` 公式链，因此该项不再按“权限不足”或“上游不明”处理。
- 补评分/评论：
  - 从 `zzdnb_product_fabric` 取 `评分` 最大值、`评论数量` 最大值。
  - 匹配：`sku = SKU`，且 `IF(store="WF-TD_CA","Wayfair-CA",IF(store="WF-TD_UK","Wayfair-UK","Wayfair")) = 网站`。
- 补 Display SKU：
  - 从 `zzdnb_wfproduct_management_latest` 取 `Display_SKUs` 最大值。
  - 匹配：`sku = SKU`。
- 补本品/竞品前台价：
  - 从 `本竞品价格对比` 取 `本品最低前台价` 最小值、`竞品最低前台价` 最小值。
  - 匹配：`Display_SKUs = link_sku`，`store = 店铺`。
  - 2026-06-19 外网 FineBI 复核：进入左侧 `本竞品价格对比` 表后，右侧流程仅暴露 `数据来源 本竞品价格对比 -> 过滤`，过滤条件为 `店铺 属于 WF-TD,WF-TD-CA,WF-FOP,WF-TD-UK`；预览字段可见 `产品id`、`model`、`link_sku`、`店铺`、`产品类型`、`本品售卖方式`、`总评论数`、`评分` 等，样例 `产品类型=本品`。页面未暴露 SQL 或物理底表。
  - 同页提示 `来源字段变动，新增字段：平台sku,上架sku,产品id-21`，后续对账时需先确认该节点计算状态。
  - 2026-06-23 继续 FineBI 页面核实：`WF广告数据 / 本竞品价格对比` 右侧数据源节点预览为 `5,000` 行 / `50` 页，过滤节点为目标 WF 店铺子集；数据源节点可见字段 `最新最高前台价`、`更新日期`、`竞品最低前台价`、`website`、`本品最新最低价`、`国家`、`采购sku`、`first_class`、`上下架状态`，可见样例 `更新日期=2026-06-23`、`website=LOWES/Overstock/Bedbathandbeyond/Homedepot`。该发现只确认 FineBI 数据源字段存在；由于页面横向滚动后行字段易错位，未作为逐行样本对账结论。
  - 2026-06-19 数据库复核：库内没有同名物理表 `本竞品价格对比`，可复刻源是 `产品近两天的最小前台价`。该表由同名事件每日 06:40 截断重建，核心来源 `link_sku_price_qtj_fabric`，取每个 `产品id` 最近两天价格，按 `model + link_sku + 产品类型 + rk` 取最小 price，并用 `link_sku_table_2`、`平台产品详情表` 补空 model。竞品行的 `store` 为 `Wayfair`，复刻时竞品最低价应按 `platform + model` 汇总，不能按广告店铺直接匹配。
  - 2026-07-02 数据库补充复核：`产品近两天的最小前台价`、`temp_link_sku_price_qtj_fabric`、`temp_latest_link_sku_price_scj` 当前均未发现可直接用于 `竞品最低前台价` 的竞品价格行；`竞品对标表` 只有关系映射，`本竞品失效sku` 只有失效状态。当前 SQL 侧不能强算 `是否有价格优势`，需返回 `竞品价缺失 / 待核实`。
  - 2026-07-02 过程补充：`link_sku_price_qtj_fabric` 过程仍有竞品分支，最新竞品映射样本在 `product_fabric` 近 90 天能命中部分竞品价格，但没有落入当前结果表。因此若 FineBI 当前仍显示竞品价，优先怀疑历史抽取 / 缓存 / 另一路数据源，或数据库过程写入分支异常。
  - 2026-07-02 用户确认：本次竞品价未落入当前结果表的原因是相关函数/例程执行失败，现已修复；恢复后的 `竞品最低前台价` 覆盖率和 FineBI 样本一致性待只读复核。
  - 2026-07-02 修复后只读复核：数据库快照 `产品近两天的最小前台价` 已恢复 `竞品 4,649` 行，竞品平台覆盖 `Wayfair 4,312`、`Wayfair-CA 266`、`Wayfair-UK 70`；目标 WF 店铺可重新按当前快照复刻价格优势。FineBI 当前样本逐行一致性仍待补。
  - 2026-07-07 上游恢复复核：上游 `link_sku_price_qtj_fabric` 已恢复大量 `竞品` 行，`Wayfair 1,465,373`、`Wayfair-CA 73,323`、`Wayfair-UK 17,161`、`Overstock 688`，`max_create_time=2026-07-07 01:35:48`；下游快照 `产品近两天的最小前台价` 当前 `create_time=2026-07-06 06:40:00` 仍有竞品行。数据库侧当前快照口径可复刻价格优势，但 FineBI `本竞品价格对比` 当前逐行样本一致性仍待补。
- 公式 `是否有价格优势`：

- FineBI 页面原式：

```text
IF(ISNULL(竞品最低前台价), null,
  IF(竞品最低前台价 < 本品最低前台价, "否", "是")
)
```

- 当前报表严格业务口径：如果业务定义为“本品最低前台价必须低于竞品最低前台价才算有价格优势”，应使用：

```text
IF(ISNULL(竞品最低前台价), null,
  IF(本品最低前台价 < 竞品最低前台价, "是", "否")
)
```

- 2026-06-19 外网 FineBI 页面确认：该公式位于 `其他表添加列` 补完本品/竞品前台价之后；FineBI 原式在本品价等于竞品价时返回“是”，若报表按严格业务口径执行，等价样本应改判“否”。
- 补整体自然/总销售：
  - 从 `总广告数据_广告组数据` 取 `自然销售额` 求和、`总销售额` 求和。
  - 匹配：`store = store`，`年月 = LEFT(年月,7)`。
- 补分类自然/总销售：
  - 从 `分类目广告数据_广告组数据` 取 `分类自然销售额` 求和、`分类总销售额` 求和。
  - 匹配：`store = store`，`分类 = 分类`，`年月 = LEFT(年月,7)`。
- 公式 `广告标签修`：根据 `活动的垂直标签` 归类，优先级为 `清仓` -> `滞销` -> `在售` -> `新品6个月` -> `新品3个月` -> `下架` -> 空。

```text
IF(FIND("清仓", 活动的垂直标签) > 0, "清仓",
  IF(FIND("滞销", 活动的垂直标签) > 0, "滞销",
    IF(FIND("在售", 活动的垂直标签) > 0, "在售",
      IF(FIND("新品6个月", 活动的垂直标签) > 0, "新品6个月",
        IF(FIND("新品3个月", 活动的垂直标签) > 0, "新品3个月",
          IF(FIND("下架", 活动的垂直标签) > 0, "下架", "")
        )
      )
    )
  )
)
```

- 2026-06-18 复核：`广告标签修` 公式节点显示后续字段包括 `断货销量`、`不断货销量`、`断货销量占比`、`自然销售额`、`总销售额`、`分类自然销售额`、`分类总销售额`，说明它位于价格优势、断货/销售回补之后的标签修正层。
- 2026-06-19 外网 FineBI 复核：当前 `WF广告数据/广告` 基础源 `广告` 的预览中已带 `广告细分标签`、`广告标签`、`垂直标签`、`平均ddp修`、`活动是否有ddp`、`数量` 等字段；`广告标签修` 的输入 `活动的垂直标签` 来自上游 `WF店铺绩效2.0/广告`。
- 2026-06-22 回到上游 `WF店铺绩效2.0/广告` 右侧流程确认：`活动的垂直标签` 不是数据库同名物理列，也不是直接由 `上架sku利润表` 回补。主 `广告` 表先从 `zzdnb_公司sku状态表` 按 `UPPER(采购SKU) = UPPER(sku)` 补 `公司标签`、`model`、`广告细分标签`、`垂直标签`，聚合方式均为最大值；再生成 `广告标签`；最后 `活动的垂直标签 = 按 [extend, isB2b] 组内字符串拼接 [广告标签]`。
- 2026-06-19 数据库复核：数据库未发现物理列 `活动的垂直标签` 或 `广告标签修`。历史候选链路 `advertising_product_report_by_day.sku -> wfproduct_management.SKU/Supplier_Part_Number -> 上架sku利润表.上架sku -> 垂直标签/垂直细分标签` 可作为数据库侧标签补充/对照入口，但不再作为当前 FineBI 页面直接来源；样本 `WF-TD / 2026-05 / campaign_id=395570 / sku=OARI1170` 可桥到 16 个上架 SKU、8 个采购 SKU，聚合标签含 `下架,在售,新品,滞销`。
- 补可用/断货：
  - 从 `wf广告` 取 `可用` 求和、`断货part个数` 求和、`活动的上架sku数量` 求和、`上架sku可用库存` 求和。
  - 匹配：`isB2b = isB2b`，`extend = extend`。
  - 2026-06-19 外网 FineBI 复核：在 `WF广告数据/广告` 右侧添加列节点中可见上述 4 个字段均按求和回补，页面只暴露 `isB2b`、`extend` 两个匹配键；这说明广告主题层是从已加工好的 `wf广告` 回补库存断货结果，不应把该节点误读为底层库存明细 Join。
  - 当前判断：广告看板里的“可用/断货”指标在广告主题内由 `wf广告` 回补；底层库存生成链路仍沿用已学习的 `WF店铺绩效2.0/wf广告 -> 各sku每月断货天数 -> 可用 -> 上架sku可用库存/断货part个数/活动的上架sku数量`。
  - 2026-06-19 数据库复核：未发现物理列 `断货part个数`、`活动的上架sku数量`、`上架sku可用库存`；这些字段仍应视为 FineBI 自助层计算结果。`wf_sku不断货销量` 物理表含 `断货销量/不断货销量/断货销量占比`，是销量辅助字段，不等同于广告组断货 part 口径。
- 再补评分/评论：
  - 从 `zzdnb_detailed_listing_health` 取 `评分` 最大值、`评论数` 最大值。
  - 匹配：`store = store`，`年月 = 年月`，`sku = Wayfair SKU`。
- 补 SKU 月销售额：
  - 从 `zzdnb_wf上架sku每日销售额` 取 `销售额` 求和。
  - 匹配：`store = store`，`LEFT(date_est,7) = LEFT(日期,7)`，`sku = wayfair_sku`。
  - 2026-06-19 数据库复核：物理表 `wf上架sku每日销售额` 由同名存储过程每日写入，来源为 `wf_supplier_parts_performance` 左连 `平台产品详情表` 补 `链接sku`、`平台sku as wayfair_sku`，并对 `GBP/CAD` 按最新 `货币汇率` 折算；因此 CA/UK 金额与原始 `wf_supplier_parts_performance.total_revenue` 不完全一致。
- 公式 `总销售额1`：

```text
if(ISNULL(销售额), "", 销售额 / IF(store="WF-FOP", 0.93, 0.94))
```

- 2026-06-18 复核：`总销售额1` 公式节点在右侧流程末段，位于多次“其他表添加列”和 `广告标签修` 之后；该字段用于后续广告销售占比类组件，不是 SQL 源字段。

##### `WF广告汇总` / `总广告数据` / `分类目广告数据` 主题内汇总层

- `WF广告汇总`：
  - 数据预览：`32` 条。
  - 处理方式：右侧流程仅见 `数据来源 = WF广告汇总` -> `字段设置`，未见公式列或 SQL 编辑入口。
  - 已见字段：`store`、`年月`、`clicks`、`impressions`、`spend`、`orders`、`WSC`、`广告类型`。
  - 样例：广告类型为 `WSP`，时间字段为月初日期格式。
- `总广告数据`：
  - 数据预览：`76` 条。
  - 处理方式：`数据来源 = 总广告数据` -> `字段设置` -> `其他表添加列` -> `广告ROI`。
  - 字段设置已见字段：`广告花费`、`广告CR`、`广告CPM`、`广告CPA`、`广告花费占比`、`自然CVR`、`自然CR`、`create_time` 等。
  - 添加列：从主题内 `广告` 表取 `广告利润` 求和；匹配依据为 `LEFT(年月,7) = LEFT(date_est,7)`、`store = store`。
  - 公式：`广告ROI = 广告利润 / 广告花费`。
  - 注意：该 ROI 是自助表字段级公式；组件汇总层仍需按利润 / 花费重算，不能平均明细 ROI。
- `分类目广告数据`：
  - 数据预览：`402` 条。
  - 处理方式：`数据来源 = 分类目广告数据` -> `字段设置` -> `其他表添加列` -> `广告ROI`。
  - 字段设置已见字段：`store`、`分类`、`年月`、`广告花费`、`广告CR`、`广告CPM`、`广告CPA`、`广告花费占比`、`自然CVR`、`自然CR`、`create_time` 等。
  - 添加列：从主题内 `广告` 表取 `广告利润` 求和；匹配依据为 `LEFT(年月,7) = LEFT(date_est,7)`、`store = store`、`分类 = 分类`。
  - 公式：`广告ROI = 广告利润 / 广告花费`。
- `分类目广告数据_广告组数据`：
  - 当前处理方式：`数据来源 = 分类目广告数据` -> `字段设置`。
  - 数据预览仍为 `402` 条；已见字段为 `store`、`分类`、`年月`、`销售额`、`订单量`、`件数`、`Sessions`、自然/广告相关指标。
  - 当前未见新增公式或 SQL；名称里的“广告组数据”在该自助表层没有直接展开成 campaign / SKU 粒度，真实广告组组件仍需回到 `广告` 自助表和组件字段继续核对。
- `总广告数据_广告组数据`：
  - 当前处理方式：`数据来源 = 总广告数据` -> `字段设置`。
  - 数据预览为 `76` 条；已见字段为 `store`、`年月`、`销售额`、`订单量`、`件数`、`Sessions`、自然曝光/访问、广告点击/曝光/花费/销售额/订单量/件数、总销售额/总订单量、自然销售额/自然订单量/自然销量、广告 CPC/CTR/CVR/CR/CPM/CPA、广告花费占比、自然 CVR/CR。
  - 当前未见新增公式或 SQL；更像是 `总广告数据` 的字段选择版，供广告组血缘/组件复用。

##### 利润相关口径

- 本轮口径：利润部分重点按 `广告利润`、`总成本`、`广告销售额`、`自然销售额`、`采购SKU` 维度的利润和利润率来梳理。
- `wf广告` 看板的 `销售和利润` 区域已见字段：
  - 维度：`sku`、`campaign_name`、`isB2b`、`class_name`、`first_10_part_numbers`、`product_status`、`活动是否有ddp`
  - 指标：`clicks`、`impressions`、`spend_USD`、`attributed_wholesale_cost_window_view_through_USD_Day_14`、`attributed_units_window_view_through_Day_14`、`总成本`、`总利润`
- `wf广告` 看板继续滚动后，在 `产品广告详情` 明细区还可见维度/字段：`store`、`sku`、`date_est`、`campaign_id`、`isB2b`、`campaign_name`、`first_10_part_numbers`；同时可见指标 `cost_per_click_USD`、`attributed_retail_sales_window_view_through_USD_Day_14`。
- `公共数据/王威/增量更新/wf广告` SQL 已确认不直接计算 `总成本`、`总利润` 或利润率；该 SQL 主要补 `店铺SKU`、`采购SKU`、`数量`、`预计和实际ddp汇总`，为采购 SKU 维度利润分析提供 SKU/DDP 映射基础。
- 2026-06-19 数据库复核：`advertising_product_report_by_day` 和广告辅助表未发现可直接抽取的 `总成本/总利润` 物理列；同名命中位于 `model弃养状态`、`垂直采购sku销量断货情况表`、`钉钉利润提醒表` 等非当前广告精确利润链路表。精确数值仍需按 `WF店铺绩效2.0/广告` 公式链预聚合复算或取 FineBI 已计算样本对账。
- 2026-06-17 公共数据血缘复核：
  - 在 `公共数据/王威/增量更新/wf广告` 的 `血缘分析` 页，点击后续节点数字后可展开到广告看板利润相关组件，包括 `WSP广告组每月汇总`、`广告组广告花费、销售额、利润与ROI分析`、`销售和利润`、`回报率`、`销售额`、`产品展示和点击`、`产品广告详情` 等。
  - 该血缘只能证明 `wf广告` 进入广告看板和利润展示区域，不能证明 `总成本/总利润` 由 `wf广告` SQL 生成；当前已确认 `wf广告` SQL 仍不直接计算 `总成本/总利润`。
  - 在公共数据搜索中按 `总利润`、`上架sku利润表`、`ddp` 未找到同名公共数据集节点；因此不能把这些名称当作 FineBI 公共数据集事实，仍需从 `WF店铺绩效2.0/广告` 或数据库 SQL 继续追。
  - 在 `WF广告数据/广告` 数据准备页点击右侧 `数据来源` 的上游 `广告` 节点，页面提示位置为 `wangwei的分析/店铺绩效/WF店铺绩效2.0`；当前可见该上游片段字段包括 `每个采购skuddp`、`model`、`model1`、`广告细分标签`、`广告标签`、`垂直标签`、`平均ddp修`、`活动是否有ddp`、`数量`。
  - 2026-06-17 三线关联复核已能进入 `WF店铺绩效2.0/广告` 数据准备页，确认 `总成本`、`总利润`、`每个sku平均成本`、`平均ddp修`、`发货费平均成本`、`ddp平均成本`、`每个采购skuddp` 的公式，原“上游权限不足”不再适用。2026-06-18 数据库字段搜索未发现当前广告链路可用的 `每个sku每月平均成本/最终成本` 字段；唯一命中的 `store_bujian_sal.最终成本` 属于补件/物流成本表，不属于广告成本链路。
- `采购SKU` 当前已确认存在于两个层面：
  - 公共 SQL 数据集 `wf广告` 通过 `cg_part_sku映射时间范围` 补 `采购SKU`、`数量`。
  - `WF广告数据/广告` 自助表字段列表中也可见 `采购SKU`，因此 WSP 广告组明细理论上可以按采购 SKU 下钻/聚合；但当前已见 `WSP广告组每月汇总` 组件维度没有放出 `采购SKU`，组件层是否隐藏或未使用仍待核实。
- `WF广告数据/WSP广告组每月汇总` 中点击 `利润` 指标卡片，提示 `来源表：广告`；因此 WSP 广告组月汇总里的 `利润` 来自主题内 `广告` 表字段，不是组件现场公式。
- `WSP广告组每月汇总` 已见利润相关指标：`利润`、`成本`、`ROI`、`ROAS`、`广告销售额`、`总销售额`、`广告销售占比`。
- 2026-06-18 定向复核 `bid` 字段：
  - `WSP广告组每月汇总` 维度槽包含短字段 `bid`；点击字段后 FineBI 字段信息显示 `来源字段名：product_max_bid`、`来源表：广告`。
  - `WF广告数据/广告` 最终自助表字段设置中搜索 `bid` 可见 `product_max_bid`、`Bid adjustment applied on B2B bids`、`Default bid(现在的bid)`、`最小建议bid`、`最大建议bid`、`Suggested Bid detail(具体金额)`，未见独立原始字段名 `bid`；因此组件中的 `bid` 是 `product_max_bid` 的展示名/重命名。
  - `WSP广告组明细表` 位于 `广告组活动` 分区，表头直接展示 `Bid adjustment applied on B2B bids`、`Default bid(现在的bid)`、`Suggested Bid detail(具体金额)`、`最小建议bid` 等字段；这些是出价表补充字段，不等同于月汇总维度 `bid`。
  - 只读数据库抽样核对 `advertising_product_report_by_day.product_max_bid`：同一 `store + 月份 + campaign_id + campaign_name + sku` 下可能出现多个 bid。样例 `WF-TD / 2026-05 / campaign_id=301368 / campaign_name=SF0003 / sku=UXIF3070` 有 `0.25` 与 `0.3` 两个 `product_max_bid`；按 bid 分组后 `0.25` 为 `18` 行、`clicks=3616`、`impressions=157047`、`spend_USD=1252.86`，`0.3` 为 `44` 行、`clicks=5827`、`impressions=266451`、`spend_USD=1602.20`。该样例支持“组件把 bid 作为维度时，同月同 campaign/SKU 的不同 bid 会分成多行展示不同广告效果”。
  - 只读数据库字段复核 `wf广告出价表`：无字段名 `campaign_id`，但存在 `campaignlayer_id`。2026-06-18 早期统计中，按 `store + 日期 + wayfair SKU + campaignlayer_id = advertising_product_report_by_day.campaign_id` 可匹配 `124,503 / 149,650` 行，按 `Campaign name` 匹配为 `122,243 / 148,983` 行；同一 `campaignlayer_id` 存在多个名称样本（如 `449653: DINCH / DINCH-2SET/4SET`），因此当前 bid 快照优先使用 `campaignlayer_id`，名称只作为展示和兜底。2026-07-07 已做广告事实行覆盖率收口，后续以 3.3 出价链路补充结论为准。
  - 继续抽样最新 `200` 条出价快照时，在找到 `10` 条 `campaignlayer_id` 未匹配样本前已匹配 `12` 条；未匹配样本集中在 `2026-05-31 / WF-FOP`，包括 `413812/UXIF3507`、`479268/UXIF3697`、`594040/UXIF4299`、`594037/UXIF3910`、`594042/UXIF4283`、`328165/UXIF3103/UXIF3105/UXIF3474`、`434519/UXIF3769`。当前 bid 快照仍优先 `campaignlayer_id`，但正式报表必须输出未匹配异常清单。
  - 结论：广告组月汇总里的 `bid` 应理解为广告底表 `product_max_bid`，用于按历史广告底表 bid 分组分析广告效果；不要把它解释成出价快照表的“当前 bid”。
- 2026-06-18 定向复核 `广告订单数`：
  - `WF广告汇总` 的 WSP SQL 明确写为 `sum(attributed_orders_window_view_through_Day_14) orders`。
  - 因此 WSP 中文 `广告订单数` / `广告订单量` 在报表实现中应取 `advertising_product_report_by_day.attributed_orders_window_view_through_Day_14` 的汇总；在 `WF广告汇总` 层对应字段名为 `orders`。
  - CPA、CVR 应在输出粒度重新计算：`CPA = SUM(spend_USD) / SUM(attributed_orders_window_view_through_Day_14)`，`CVR = SUM(attributed_orders_window_view_through_Day_14) / SUM(clicks)`，不能平均 FineBI 展示比率。
- 2026-06-18 定向复核 SKU 映射链：
  - WSP 组件 / `advertising_product_report_by_day` 中的 `sku` 是 Wayfair 广告平台 SKU，不是店铺 SKU、上架 SKU 或采购 SKU。
  - `first_10_part_numbers` 在 `WF店铺绩效2.0/zzdnb_advertising_product_report_by_day` 中会拆分成行，并生成 `上架sku = LOWER(TRIM(first_10_part_numbers-拆分行结果))`。
  - 公共 `wf广告` SQL 记录为以 `advertising_product_report_by_day` 为主表，左连 `cg_part_sku映射时间范围` 补 `店铺SKU`、`采购SKU`、`数量`，再补 DDP；该层存在 `first_10_part_numbers` / 上架 SKU / 采购 SKU 一对多风险。
  - 链接 SKU 不属于广告底表原生字段；报表补链接 SKU 时优先用 `wayfair上架sku销售映射`，缺失时回退 `平台产品详情表.链接sku` / `平台产品详情表.采购sku` 拆分逻辑。已有有货率脚本样本显示 `wayfair上架sku销售映射` 可提供上架 SKU 到采购 SKU 的组件数量。
- 抽样补充：
  - 广告底表 bid 样例：`WF-TD / 2026-05 / campaign_id=301368 / campaign_name=SF0003 / sku=UXIF3070` 下有 `product_max_bid=0.25` 与 `0.3`，按 bid 分组后分别汇总不同点击、曝光和花费。
  - 单一采购 SKU 映射样例：`WF-FOP / linkSku=EVYN4068 / platformSkus=EVYN4068 / listingSku=67920-BARST-2SET-BLACK / components=BARST-2SET-BLACK x 1`，映射源 `wayfair上架sku销售映射`。
  - 多采购 SKU 映射样例：`WF-FOP / linkSku=W116630066 / platformSkus=UXIF4208 / listingSku=FOP-OD0002-BEIGE-1 / components=OD0002-BEIGE-CHAIR-2ST x 1; OD0002-SIDE TABLE x 1`，映射源 `wayfair上架sku销售映射`。
  - 无链接 SKU / 无组件映射样例：`WF-TD-CA / platformSkus=UXIE9715 / listingSku=51945BN / linkSku=无 / mappingSource=无映射`，需进异常清单。
- 报表实现建议：
  - 广告指标先按 `store + 年月 + campaign_id + campaign_name + sku + product_max_bid` 聚合；如果报表不需要 bid 拆分，可在确认业务口径后去掉 `product_max_bid` 并把 bid 作为变化分析另表输出。
  - 在广告指标聚合完成后，再左连上架 SKU、链接 SKU、有货率、链接利润率等维度；不能先展开采购 SKU 后直接汇总广告点击、曝光、花费、WSC、订单数、件数，否则会放大广告指标。
  - 多上架 SKU / 多采购 SKU 只用于库存、有货率、采购 SKU 利润率和异常解释；广告指标需要保留广告聚合口径，必要时按权重或规则分摊，并单独输出分摊说明。
- `WF广告数据/广告` 自助表已确认从 `总广告数据_广告组数据` 回补 `自然销售额`、`总销售额`，从 `分类目广告数据_广告组数据` 回补 `分类自然销售额`、`分类总销售额`。
- `WF广告数据/广告` 自助表利润相关字段与公式已确认：

```text
广告销售额 = IF(
  SUM_AGG(attributed_wholesale_cost_window_view_through_USD_Day_14) = 0,
  "",
  SUM_AGG(attributed_wholesale_cost_window_view_through_USD_Day_14)
)

成本 = IF(SUM_AGG(总成本) = 0, "", SUM_AGG(总成本))

利润 = IF(SUM_AGG(总利润) = 0, "", SUM_AGG(总利润))

广告ROI = IF(
  OR(SUM_AGG(spend_USD) = 0, SUM_AGG(总利润) = 0),
  "",
  SUM_AGG(总利润) / SUM_AGG(spend_USD)
)

广告ROAS = IF(
  OR(
    SUM_AGG(attributed_wholesale_cost_window_view_through_USD_Day_14) = 0,
    SUM_AGG(spend_USD) = 0
  ),
  "",
  SUM_AGG(attributed_wholesale_cost_window_view_through_USD_Day_14) / SUM_AGG(spend_USD)
)

广告组利润类型 = IF(
  SUM_AGG(总利润) > 0,
  "盈利广告组",
  IF(SUM_AGG(总利润) < 0, "亏损广告组", "无效广告组")
)

断货part百分比 = IF(
  SUM_AGG(断货part个数) = 0,
  "",
  SUM_AGG(断货part个数) / SUM_AGG(活动的上架sku数量)
)

广告销售占比 = IF(
  OR(广告销售额 = 0, 总销售额修 = 0),
  "",
  广告销售额 / AVG_AGG(总销售额1)
)

总销售额修 = IF(
  AVG_AGG(总销售额1) = 0,
  "",
  AVG_AGG(总销售额1)
)

广告花费占广告销售额的比 =
  SUM_AGG(spend_USD) / SUM_AGG(attributed_wholesale_cost_window_view_through_USD_Day_14)
```

- 口径解释：
  - `广告销售额` 在该主题里取广告归因 WSC 字段 `attributed_wholesale_cost_window_view_through_USD_Day_14`，不是零售额 `attributed_retail_sales_window_view_through_USD_Day_14`。
  - `广告利润/利润` 当前展示口径等于 `总利润` 聚合值；`总利润` 的上游生成在 `WF店铺绩效2.0/广告` 基础源中，当前广告主题内只做展示聚合。
  - `总成本/成本` 当前展示口径等于 `总成本` 聚合值；`总成本` 的上游生成同样需要继续追 `WF店铺绩效2.0/广告`。
  - `广告ROI` 是 `总利润 / 广告花费(spend_USD)`，可作为当前看板里最明确的广告利润率/投入产出口径。
  - `广告ROAS` 是 `广告销售额(WSC) / 广告花费(spend_USD)`，不是利润率。
  - `断货百分比` 组件字段对应 `广告` 表计算字段 `断货part百分比`，分子是 `断货part个数`，分母是 `活动的上架sku数量`。
  - `总销售额修` 已在 Chrome / FineBI 计算字段弹窗确认：它是 `AVG_AGG(总销售额1)` 的展示包装，平均值为 0 时返回空；不是另一个销售额来源。
  - `广告销售占比` 分母使用 `AVG_AGG(总销售额1)`；公式条件检查 `总销售额修 = 0`，本质是同一分母的空值/零值保护。
  - `广告花费占广告销售额的比` 是 `广告花费 / 广告销售额(WSC)`，不是 `广告花费 / 总销售额`。
- `ddp_利润查询` 页面可见通用利润参考口径：`利润 = 总销售额 - 佣金 - ddp - 出货费用`，并提示当前利润计算排除 `分销`；该口径是利润字段的参考链路，是否完全等同广告看板 `总利润/利润` 仍需继续核实。
- 2026-06-17 继续从公共数据 DDP 节点验证：`公共数据/tsq/ddp利润表/ddp汇总` SQL 为 `ddp汇总表_美国`、`ddp汇总表_加拿大`、`ddp汇总表_英国` union，并补 `国家` 字段；预览字段包括 `sku`、`年月`、`预计ddp均值`、`实际ddp均值`、`实际和预计ddp整合`、`预计和实际ddp汇总`，FineBI 预览显示 `50,056` 行。
- `ddp汇总` 血缘点击节点数字后已展开到 `广告`、`zzdnb_advertising_product_report_by_day`、`利润广告比率`。该事实能证明 DDP 汇总进入广告利润相关链路，但仍不能证明广告看板 `总成本/总利润` 直接由 `ddp汇总` 生成；`WF店铺绩效2.0/广告` 仍是下一步必须追的上游。

##### `WF店铺绩效2.0/广告` 成本与利润上游公式

- FineBI 入口：从公共数据 `wf广告` 血缘节点悬停后点击“跳转”，进入 `wangwei的分析/店铺绩效/WF店铺绩效2.0` 的 `广告` 表。
- 节点信息：公共数据血缘悬停提示 `数据集名：wf广告`，位置为 `wangwei的分析/店铺绩效/WF店铺绩效2.0`，创建者 `wangwei`，创建时间 `2024/11/26 14:31:09`，最近更新时间 `2026/06/17 16:01:49`，血缘层级 `3`。
- 当前编辑画布显示 `广告` 表数据来源为公共 SQL `wf广告`，处理链包含 `ddp汇总`、`发货费`、`上架sku利润`、`zzdnb_advertising_product_report_by_day`、`zzdnb_wf广告出价表`、新广告 attribution 表等上游/辅助表。
- 已确认公式：

```text
最新ddp =
  IF(
    是否有ddp = "无ddp",
    DEF(AVG_AGG(新ddp), [extend, isB2b, 采购SKU]),
    新ddp
  )

总ddp = 最新ddp * 数量

每个上架skuddp =
  DEF(SUM_AGG(总ddp), [store, campaign_id, sku, date_est, 店铺SKU, isB2b, extend])

每个上架sku采购sku数量 =
  DEF(SUM_AGG(数量), [store, campaign_id, sku, date_est, 店铺SKU, isB2b, extend])

每个采购skuddp = 每个上架skuddp / 每个上架sku采购sku数量

ddp平均成本 =
  DEF(
    AVG_AGG(每个采购skuddp),
    [campaign_name, store, sku, date_est, isB2b],
    [每个采购skuddp > 0]
  )

平均ddp修 =
  IF(
    ISNULL(ddp平均成本),
    DEF(AVG_AGG(ddp平均成本), [campaign_id, sku, store]),
    ddp平均成本
  )

发货费平均成本 =
  按 [store, date_est, sku, campaign_id, extend, isB2b] 组内平均 [出货费用]

每个sku平均成本 = 平均ddp修 + 发货费平均成本

总成本 =
  (每个sku平均成本 * attributed_units_window_view_through_Day_14 / 单买数量)
  + spend_USD

总利润 =
  (attributed_wholesale_cost_window_view_through_USD_Day_14 * IF(store="WF-FOP", 0.93, 0.94))
  - 总成本
```

- 当前确认：广告 `总成本` 由单件平均成本、广告归因件数、单买数量和广告花费组成；广告 `总利润` 使用广告归因 WSC 按店铺系数折算后减 `总成本`。
- 2026-06-18 补充确认：
  - 左侧辅助表 `ddp汇总` 是 DDP 成本入口之一，处理链显示先过滤 `国家 属于 美国`；字段包括 `sku`、`年月`、`预计到港月份`、`预计ddp均值`、`入库月份`、`实际ddp均值`、`预计和实际ddp汇总`、`预计平台`、`实际平台`。
  - `广告` 表右侧处理链显示：`ddp汇总` 通过“其他表添加列”补 `预计和实际ddp汇总`，匹配依据为 `采购SKU = sku`、`年月 = 年月`，后续字段设置中作为 `新ddp` 参与 `是否有ddp`、`最新ddp` 和 `总ddp` 计算。
  - `是否有ddp = IF(ISNULL(新ddp), "无ddp", "有ddp")`；`是否有ddp新 = DEF(MAX_AGG(是否有ddp), [extend, isB2b])`。
  - `出货费用` 由左侧辅助表 `发货费` 补入，右侧步骤显示从 `发货费` 选择 `出货费用` 按求和，匹配依据为 `采购SKU = SKU`。
  - `单买数量` 由左侧辅助表 `zzdnb_广告组合单买详情` 补入，右侧步骤显示从该表选择 `单卖` 按最大值，匹配依据为 `sku = sku`、`campaign_name = campaign_name`，字段设置后作为 `单买数量` 参与 `总成本`。
  - `zzdnb_wf广告出价表` 通过“其他表添加列”补 `Bid adjustment applied on B2B bids`、`Default bid(现在的bid)`、`最小建议bid`、`最大建议bid`、`Suggested Bid detail(具体金额)`，均按求和；匹配依据为 `store = store`、`date_est = 日期`、`campaign_name = Campaign name`、`sku = wayfair SKU`。
  - `总ddp` 由 `最新ddp * 数量` 生成；`每个上架skuddp` 是按 `store + campaign_id + sku + date_est + 店铺SKU + isB2b + extend` 对 `总ddp` 求和；`每个上架sku采购sku数量` 是同粒度对 `数量` 求和。
  - `最终成本` 节点悬浮提示位置为 `公共数据/王威/增量更新/wf广告`，当前可见字段为 `店铺SKU`、`采购SKU`、`数量`、`预计和实际ddp汇总`、`isB2b`、`extend`；这与公共 SQL `wf广告` 补 SKU 映射和 DDP 的逻辑一致。
  - `wf广告` 左侧基础表本身的右侧处理链已确认：数据来源 `wf广告` -> 按 `店铺SKU + 采购SKU + isB2b + extend` 删除重复行 -> 字段设置保留广告原始字段和 `店铺SKU/采购SKU/数量/预计和实际ddp汇总/isB2b/extend` -> `国家` 按 `store` 赋值为加拿大/英国/美国 -> 从 `各sku每月断货天数` 补 `可用`，匹配 `UPPER(采购SKU) = UPPER(sku)`、`date_est = date`、`国家 = 国家` -> `上架sku可用库存 = 按 [extend, isB2b, 店铺SKU] 组内最小值 [可用]` -> `断货part个数 = NVL(DEF(COUNTD_AGG(店铺SKU), [isB2b, extend], [上架sku可用库存=0]), 0)` -> `活动的上架sku数量 = 按 [isB2b, extend] 组内去重计数 [店铺SKU]`。
  - 左侧辅助表 `上架sku利润` 当前可见过滤条件为 `platform 属于 Wayfair`；字段包括 `扣除平均仓库仓储费的利润率`、`first_class`、`second_class`、`产品id`、`利润率上限`、`利润率异常`、`model`、`不断货日均销量`、`利润率差异`、`SKU`。
  - `上架sku利润` 的“其他表添加列”显示从 `zzdnb_wfproduct_management` 取 `SKU` 最大值，匹配依据为 `store = 店铺`、`上架sku = Supplier_Part_Number`、`LEFT(日期,7) = LEFT(insert_date,7)`；当前预览显示 `共0条数据`、`当前部分数据计算结果为空`，且出现“来源字段变动，新增字段：model,不断货日均销量,利润率差异”提示。2026-06-19 使用外网 FineBI 链接复核，进入该节点后仍弹出同样字段变动提示，右侧流程确认为 `数据来源 -> 过滤 -> 其他表添加列`，过滤条件为 `platform 属于 Wayfair`，过滤后仍显示 `共0条数据`。结合数据库 `platform='Wayfair'` 非空，页面 0 行更可能是 FineBI 自助节点字段变动/计算状态异常，不是过滤条件本身导致底表无数据。
  - 2026-06-18 数据库复核确认：物理表 `上架sku利润表` 非空，`platform='Wayfair'` 有 `863,432` 行，覆盖 `WF-TD`、`WF-FOP`、`WF-TD-CA`、`WF-TD-UK`；字段为 `上架sku`、`采购sku`、`出单价`、`平台佣金`、`发货费`、`ddp`、`成本`、`利润率上限`、`model`、`不断货日均销量` 等，数据库原生没有 `平均利润率/最大利润率/最小利润率` 字段，这些属于 FineBI 后续公式/字段设置。
  - 同日抽样验证 `上架sku利润表 -> wfproduct_management` 添加列不是完全无匹配：`WF-FOP / BARST-SILVER-2SET-* / 2026-06` 可按 `店铺 + Supplier_Part_Number + insert_date 月份` 匹配到 `18` 条管理表记录，并补到 `SKU=UXIF3403`。因此 FineBI 节点预览 0 行更可能是自助节点过滤、字段变动或计算状态问题，不是底表为空或数据库层 join 全失效。
  - `zzdnb_平台产品详情表` 是广告链路里采购 SKU / 上架 SKU / 分类 / model / 本品备注 / 产品状态的补充表：右侧步骤拆分 `采购sku`，生成 `上架sku新 = LOWER(上架sku)`，并按 `上架sku、采购sku_first、采购sku-拆分行结果、采购sku、本品备注、产品状态、model、first_class、second_class` 去重。
  - `zzdnb_鲸汇店铺销售产品` 是数量补充表：先过滤 `店铺 属于 WF TD, WF FOP`，生成 `店铺新 = REPLACE(店铺, " ", "-")`，保留 `店铺/店铺SKU/采购SKU/数量/insert_date/店铺新/产品id`，再按 `insert_date` 最晚 N=1 取最新，并生成 `上架sku新 = LOWER(店铺SKU)`。
  - `zzdnb_advertising_product_report_by_day` 在 `WF店铺绩效2.0` 内也有一条广告处理链：生成 `年月`、`关联id`、`零售回报率`、`批发回报率`，拆分 `first_10_part_numbers` 得 `上架sku`，再从 `zzdnb_平台产品详情表` 补 `采购sku`，从 `ddp汇总` 补 `预计和实际ddp汇总`，从 `发货费` 补 `出货费用`，从 `zzdnb_鲸汇店铺销售产品` 补 `数量`，并继续生成 `总ddp`、`发货费平均成本`、`每个sku平均成本` 等成本字段。
  - 2026-06-18 单步复核补充：`zzdnb_advertising_product_report_by_day` 的 DDP/成本中间公式当前页面显示为：`每个上架skuddp = DEF(SUM_AGG(总ddp), [store, campaign_id, sku, date_est, 店铺SKU, isB2b, extend])`；`每个上架sku采购sku数量 = DEF(SUM_AGG(数量), [store, campaign_id, sku, date_est, 店铺SKU, isB2b, extend])`；`每个采购skuddp = 每个上架skuddp / 每个上架sku采购sku数量`；`ddp平均成本 = DEF(AVG_AGG(每个采购skuddp), [campaign_name, store, sku, date_est, isB2b], [每个采购skuddp > 0])`；`平均ddp修 = IF(ISNULL(ddp平均成本), DEF(AVG_AGG(ddp平均成本), [campaign_id, sku, store]), ddp平均成本)`；`发货费平均成本 = 按 [store, date_est, sku, campaign_id, extend, isB2b] 组内平均 [出货费用]`；`每个sku平均成本 = 平均ddp修 + 发货费平均成本`。该记录与后续外网页面当前节点显示存在分支/版本差异，最终报表复刻前需以主 `广告` 表的 `总成本/总利润` 引用链再次核对。
  - 2026-06-19 小批次闭环：用户提供外网页面 `WF广告数据/广告` 前置血缘后，跳转进入 `WF店铺绩效2.0/zzdnb_advertising_product_report_by_day` 当前打开节点，继续按右侧流程复核。当前节点显示：从 `ddp汇总` 取 `预计和实际ddp汇总` 求和，匹配 `采购sku修 = sku`、`年月 = 年月`，字段设置后作为 `新ddp`；`是否有ddp = IF(ISNULL(#ddp), "无ddp", "有ddp")`；`是否有ddp新 = DEF(MAX_AGG(是否有ddp), [campaign_id, sku, date_est, extend])`；从 `发货费` 取 `出货费用` 求和，匹配 `采购sku修 = SKU`；`最终成本 = IF(每个sku每月平均成本 >= 0, 每个sku每月平均成本, 出货费用)`，但该节点预览提示字段丢失，`每个sku每月平均成本` 的上游仍待继续确认；`最新ddp = IF(是否有ddp="无ddp", DEF(AVG_AGG(新ddp), [campaign_id, sku, 采购sku修, extend]), 新ddp)`；从 `zzdnb_鲸汇店铺销售产品` 取 `数量` 求和，匹配 `上架sku = 上架sku新`、`采购sku修 = 采购SKU`；`总ddp = 数量 * 最新ddp`；`每个上架skuddp = DEF(SUM_AGG(总ddp), [store, campaign_id, sku, date_est, 上架sku, extend])`；`ddp平均成本 = DEF(AVG_AGG(每个上架skuddp), [campaign_name, store, sku, date_est, extend], [每个上架skuddp > 0])`；`发货费平均成本 = 按 [store, date_est, sku, campaign_id, extend] 组内平均 [出货费用]`；`每个sku平均成本 = ddp平均成本 + 发货费平均成本`。
  - 同批次继续确认：后续字段设置保留原广告事实字段、`年月/关联id`、`click_through_rate`、`是否有ddp新`、`每个sku平均成本`、`ddp平均成本`、`每个上架skuddp`、`isB2b`、`extend`；第一次删除重复行的去重键为 `store,date_est,campaign_id,sku,extend`；第二次删除重复行页面显示“部分选择 34”，下拉可见原广告事实字段及 `年月`、`关联id`、`click_through_rate`、`extend`、`是否有ddp新`、`每个sku平均成本`、`ddp平均成本`、`每个上架skuddp`、`isB2b` 等字段，已不包含 `上架sku/采购sku/数量/新ddp/最新ddp/总ddp/出货费用` 等展开中间字段。该节点的 `总成本 = 每个sku平均成本 * attributed_units_window_view_through_Day_14 + spend_USD`，没有除以 `单买数量`；`总利润 = attributed_wholesale_cost_window_view_through_USD_Day_14 * IF(store="WF-FOP",0.93,0.94) - 总成本`；利润后排序为 `date_est` 降序；`是否满足14天 = IF(DATEDIF(date_est,NOW(),"D")>14,"已满足","未满足")`；`14天周期` 按 `date_est` 相对 `NOW()` 切 14 天滚动区间。末端字段设置保留原广告事实字段及 `总利润`、`14天周期`、`总成本`、`是否有ddp新`、`每个sku平均成本`、`ddp平均成本`、`每个上架skuddp`、`isB2b`、`extend`。
  - 同批次回到主 `WF店铺绩效2.0/广告` 表继续确认：主表右侧 `单买数量` 公式为 `IF(1=1,1,NVL(单卖,1))`，因此当前实际恒为 `1`，虽仍保留 `单卖` 字段来源但被 `IF(1=1)` 绕过；主表 `总成本 = (每个sku平均成本 * attributed_units_window_view_through_Day_14 / 单买数量) + spend_USD`；主表 `总利润 = (attributed_wholesale_cost_window_view_through_USD_Day_14 * IF(store="WF-FOP",0.93,0.94)) - 总成本`。因此前置节点没有 `/ 单买数量` 与主表公式带 `/ 单买数量` 在当前配置下数值上暂时等价，但这是由 `单买数量` 恒为 1 导致；如果后续 `单买数量` 公式改回 `NVL(单卖,1)` 或其他组合件数量口径，`总成本/总利润` 会随之变化。
  - 同批次发现：当前 `zzdnb_advertising_product_report_by_day` 节点中间和末端出现若干疑似样例/临时过滤，包括 `campaign_name in (FOP-CS0002 set, SF0003-2SET)` 且 `date_est=2023-10-01~2023-12-01`、`sku=UXIF3388` 且 `date_est` 取最晚 1 个、`campaign_name=CH0023 AD`、末端 `年月=2024-10` 且 `campaign_name=CS0002`。这些过滤导致部分预览为空或只返回局部样本，暂不应复刻到全量广告报表；需继续确认它们是否仅属于成本样本分支、历史调试残留或最终数据链真实过滤。当前排序节点已见：`campaign_id` 降序；后续排序为 `sku` 降序、`date_est` 降序、`campaign_id` 升序。
  - `zzdnb_wf广告出价表` 单步复核确认：源字段包含 `store`、`日期`、`Campaign name`、`wayfair SKU`、`Suggested Bid detail(具体金额)`、`Suggested Bid（金额范围)`、`Default bid(现在的bid)`、`Daily Cap`、`Bid adjustment applied on B2B bids`；右侧将 `Suggested Bid（金额范围)` 按 `-` 拆分为 2 列，字段设置中形成 `最小建议bid`、`最大建议bid`。
  - `广告` 表单步复核确认：来源为 `wf广告`，继续生成 `是否满足14天 = IF(DATEDIF(date_est,NOW(),"D")>14,"已满足","未满足")`，从 `zzdnb_wf广告出价表` 补 BID 字段后生成 `周`、`星期`、`星期日`、`星期六`、`日期范围`，并从 `zzdnb_wf_class中文分类` 按 `class_name = Class Name` 补 `分类`。
  - `zzdnb_wfproduct_management_latest` 单步复核确认：来源 `wfproduct_management`，生成 `日期 = LEFT(insert_date,10)`，保留产品管理字段，过滤 `日期` 最晚 N=1、`Product_Status` 非空、`Product_Class` 不属于 `Fabric Samples, Multichannel-Only (MCO) Fulfillment`。
  - `zzdnb_wfproduct_management_latest1` 是 `zzdnb_wfproduct_management_latest` 的派生入口，样例字段可见店铺 + `Supplier_Part_Number` 组合、店铺、`Supplier_Part_Number`、`SKU`；`sku维度表2` 已确认从它补 `SKU`。
  - `各sku每月断货天数` 单步复核当前只确认到源 `zzdnb_warehouse_stock`、字段设置和 `国家` 映射；实际“每月断货天数”聚合公式未在当前可见步骤展开，仍待继续确认。
  - 周期字段：`14天周期 = CONCATENATE(DATEDELTA(NOW(), -(TRUNC((DATEDIF(DATEDELTA(date_est,1), NOW(), "D")/14))+1)*14), "~", DATEDELTA(DATEDELTA(NOW(), -1), -(TRUNC((DATEDIF(DATEDELTA(date_est,1), NOW(), "D")/14)))*14))`；`是否满足14天 = IF(DATEDIF(date_est, NOW(), "D") > 14, "已满足", "未满足")`；周字段包括 `周 = WEEK(date_est)`、`星期 = WEEKDAY(date_est)`、`星期日 = DEF(MIN_AGG(IF(星期=0, date_est, null)), [YEAR(date_est), 周])`、`星期六 = DEF(MIN_AGG(IF(星期=6, date_est, null)), [YEAR(date_est), 周])`、`日期范围 = CONCATENATE(LEFT(星期日,10), "~", LEFT(星期六,10))`。
  - `分类` 由 `zzdnb_wf_class中文分类` 补入，匹配依据为 `class_name = Class Name`，取 `分类` 最大值。
  - 2026-06-22 回到用户指定内网页面 `WF店铺绩效2.0/广告`（表 ID `6d42a82f92604810b3abf468d7876bba`）继续按右侧流程逐节点确认：`zzdnb_公司sku状态表` 添加列从该表选择 `公司标签`、`model`、`广告细分标签`、`垂直标签`，均按最大值，匹配依据为 `UPPER(采购SKU)=UPPER(sku)`；`zzdnb_采购sku_model映射表` 另有一条 model 补充线，从该表选择 `model` 最大值，匹配 `UPPER(采购SKU)=UPPER(采购sku)`。
  - 同页公式确认：`广告标签 = IF(广告细分标签="新品1-3","新品3个月",IF(OR(广告细分标签="新品4",广告细分标签="新品5",广告细分标签="新品6"),"新品6个月",IF(AND(广告细分标签="新品7"),"在售",垂直标签)))`；`活动的垂直标签 = 按 [extend,isB2b] 组内字符串拼接 [广告标签]`；`model1 = 按 [extend,isB2b] 组内字符串拼接 [model]`。
  - 同页断货/活动数量确认：`活动的上架sku数量 = 按 [isB2b,extend] 组内去重计数 [店铺SKU]`；当前节点显示 `断货part个数 = DEF(COUNTD_AGG(店铺SKU), [isB2b,extend], [上架sku可用库存=0])`，页面没有显示此前记录里的 `NVL(...)` 包裹，并提示“字段丢失，暂无法预览数据”，因此该字段需按当前版本重新做样本对账。
  - 同页最终利润公式再次确认：主 `广告` 表 `总成本 = (每个sku平均成本 * attributed_units_window_view_through_Day_14 / 单买数量) + spend_USD`；`总利润 = attributed_wholesale_cost_window_view_through_USD_Day_14 * IF(store="WF-FOP",0.93,0.94) - 总成本`。
  - 2026-06-22 继续在同一内网页面复核末端时间辅助字段：`14天周期 = CONCATENATE(DATEDELTA(NOW(), -(TRUNC((DATEDIF(DATEDELTA(date_est,1), NOW(), "D")/14))+1)*14), "~", DATEDELTA(DATEDELTA(NOW(), -1), -(TRUNC((DATEDIF(DATEDELTA(date_est,1), NOW(), "D")/14)))*14))`；`是否满足14天 = IF(DATEDIF(date_est, NOW(), "D") > 14, "已满足", "未满足")`；`周 = WEEK(date_est)`；`星期 = WEEKDAY(date_est)`；`星期日 = DEF(MIN_AGG(IF(星期=0, date_est, null)), [YEAR(date_est), 周])`。本轮点 `星期六/日期范围` 时误入底部空白组件页，未再次复核；沿用既有记录，后续需要时从右侧流程重新点回节点确认。
  - 2026-07-07 数据库样本补充：`advertising_product_report_by_day` 最新业务日 `2026-07-06` 有广告行和花费，但 Day_14 WSC / orders / units 为 `0`；`2026-07-05` 已有非零归因。近日报表应结合 `是否满足14天` 标注或过滤成熟度，不应把最新日 WSC 为 0 直接判断为整表未刷新。
- 待核实：`最终成本` 旧公式中的 `每个sku每月平均成本` 字段仍需追是否存在未显示的月度成本辅助表或历史字段；`上架sku利润` 已确认数据库层非空且可作为 SKU 利润率 / 异常 / model / 预聚合成本辅助入口，但不能直接当作广告 `总利润` 的生成源。`单买数量` 的添加列曾显示来自 `单卖` 最大值，但主表最终公式当前写死为 1，字段名重命名位置和是否仍有组合件业务意图需继续留意。

##### `wf_sku不断货销量`

- FineBI 源路径：`公共数据/王威/广告/wf_sku不断货销量`
- FineBI 主题位置：`WF广告数据` 主题内数据表。
- 类型：SQL / 抽取数据集。
- 数据量：2026-06-17 07:27 更新后 `304,111` 行；数据占用空间 `24.30 MB`。
- 已见字段：`产品id`、`店铺`、`Supplier_Part_Number`、`wayfair_sku`、`采购sku`、`可用`、`库存情况`、`近30天不断货销量`、`断货销量`、`不断货销量`、`断货销量占比`、`create_time`。
- SQL：

```sql
select * from wf_sku不断货销量
```

- 更新记录：`system` 定时触发的 `广告更新任务`，约 `07:27` 更新；2026-06-17 增加 `7,074` 条，共 `304,111` 条。2026-06-05、2026-05-26 有 `wangwei` 手动触发的广告文件夹更新记录。
- 血缘：
  - 初始后续数字 `109`。
  - 点击后展开到 `广告` 自助表，剩余后续数字 `108`。
  - 再点击 `广告` 后续数字，展开到 `分类目广告数据`、`总广告数据`、`WSP广告组每月汇总`、多组广告组汇总和看板组件。
  - `总广告数据` 节点数字 `14` 可继续展开到整体组件：`广告CR与自然CR百分比对比（*100）`、`广告vs自然销售额`、`广告曝光点击`、`广告花费占比`、`广告花费、销售额、利润与ROI分析`、`广告点击转化`、`广告增长VS销售增长`。
  - `分类目广告数据` 节点数字 `20` 本轮点击后未新增可读下游，保留“待继续展开”。
- 与广告看板关系：该表的血缘已确认直接进入 `广告` 自助表，并继续到整体、分类、广告组组件；这解释了广告看板中“可用/断货/滞销/清仓/在售”等商品状态和库存相关拆分为什么出现在广告链路里。`广告` 自助表后续仍显示从 `wf广告` 回补 `可用`、`断货part个数`、`活动的上架sku数量`、`上架sku可用库存`，两者字段级关系还需继续在 `广告` 自助表编辑步骤中核实。

### 3.2 王威 / 增量更新：WF 广告明细链路

#### `wf广告`

- FineBI 路径：`公共数据/王威/增量更新/wf广告`
- 类型：SQL 数据集
- 主要服务看板：`zzd/广告/wf广告`，并可能被 `WF-WSP广告组广告分析` 明细组件复用。
- 已见字段：`store`、`date_est`、`campaign_id`、`campaign_name`、`campaign_status` 等；横向字段需继续滚动补全。

```sql
select
  a.*,
  b.`上架SKU` 店铺SKU,
  b.`采购SKU`,
  b.`数量`,
  c.预计和实际ddp汇总
from advertising_product_report_by_day a
left join cg_part_sku映射时间范围 b
  on concat(a.first_10_part_numbers,',') like concat('%',b.`上架SKU`,',%')
  and a.store = b.store
  and left(a.date_est,10) >= left(b.起始时间,10)
  and left(a.date_est,10) <= left(b.end_date,10)
left join (
  select *, "美国" 国家 from `ddp汇总表_美国`
  union all
  select *, "英国" 国家 from `ddp汇总表_英国`
) c
  on b.采购sku = c.sku
  and left(a.date_est,7) = c.年月
  and c.实预平台 = '垂直'
  and case
    when a.store in ("WF-TD","WF-FOP") then "美国"
    when a.store in ("WF-TD-UK","WF-FOP-UK") then "英国"
  end = c.国家
where left(a.date_est,7) >= left(date_sub(CURRENT_DATE, interval 12 month), 7)
```

逻辑结论：

- 广告事实主表是 `advertising_product_report_by_day`。
- 通过 `first_10_part_numbers` 模糊匹配 `cg_part_sku映射时间范围.上架SKU`，补 `店铺SKU`、`采购SKU`、`数量`。
- 通过 `store` 判断国家：`WF-TD/WF-FOP` 为美国，`WF-TD-UK/WF-FOP-UK` 为英国。
- DDP 来源为 `ddp汇总表_美国` 与 `ddp汇总表_英国` union 后按 `采购sku + 年月 + 国家` 关联。
- 时间范围为当前日期往前 12 个月的月份起。

#### `zzdnb_advertising_product_report_by_day`

- FineBI 路径：`公共数据/王威/店铺绩效/WF/zzdnb_advertising_product_report_by_day`
- 类型：抽取数据集
- 数据量：2026-06-17 08:00 更新后 `442,448` 行
- 数据占用空间：`118.29 MB`
- 更新任务：`system` 定时触发的 `WF更新任务`，约 `08:00` 更新。
- 2026-06-17 更新记录：`08:00:10` 至 `08:00:19`，更新成功，增加 `1,105` 条，共 `442,448` 条。
- 已见字段：
  - 维度：`store`、`date_est`、`campaign_id`、`isB2b`、`sku`、`extend`
  - Campaign：`campaign_name`、`campaign_status`、`campaign_is_active`、`campaign_daily_cap_USD`、`campaign_lifetime_budget_USD`
  - 商品：`product_status`、`first_10_part_numbers`、`class_name`
  - 广告指标：`clicks`、`impressions`、`spend_USD`、`cost_per_click_USD`、`click_through_rate`
  - 14 日归因：`attributed_wholesale_cost_window_view_through_USD_Day_14`、`attributed_units_window_view_through_Day_14`
- 用途：WF 广告日报基础事实表；被 `wf广告` SQL 直接引用。
- 2026-06-17 血缘复核：
  - 血缘页可见多个同名分支，右侧数字包括 `6`、`5`、`1` 等。
  - 展开第一个 `6` 分支后，已见直接下游组件 / 节点：`广告花费明细表`、`广告每日变化情况`、`广告销售变化情况`。
  - 因该页存在多个同名分支，本轮只记录已展开分支事实，其他分支待继续展开。

#### `zzdnb_detailed_listing_health`

- FineBI 路径：`公共数据/王威/店铺绩效/WF/zzdnb_detailed_listing_health`
- 类型：抽取数据集
- 数据量：2026-06-17 08:00 更新后 `258,363` 行
- 数据占用空间：`47.23 MB`
- 更新任务：`system` 定时触发的 `WF更新任务`，约 `08:00` 更新。
- 2026-06-17 更新记录：`08:00:08` 至 `08:00:13`，更新成功，增加 `0` 条，共 `258,363` 条；2026-06-13 曾增加 `6,318` 条，2026-06-12 曾减少 `6,318` 条。
- 已见字段：
  - 基础：`id`、`store`、`年月`、`Reporting Month`
  - 商品：`Supplier Part Number`、`Composite Part Number`、`Component Part Number`、`Manufacturer Part Number`、`Wayfair SKU`
  - 评论 / 评分：`Lifetime Review Count`、`Lifetime Average Customer Review Rating`、`Review Count Health`、`Review Rating Health`
- 用途：Listing health 月度数据；`WF广告数据/广告` 自助表中已见从该表取 `评分` 最大值、`评论数` 最大值，匹配 `store = store`、`年月 = 年月`、`sku = Wayfair SKU`。
- 字段命名注意：源数据集字段为英文 `Lifetime Review Count` / `Lifetime Average Customer Review Rating`，广告主题或组件里的中文 `评论数` / `评分` 是后续取数 / 展示命名；直接在源表搜中文 `评分` 为空。
- 2026-06-18 数据库复核：物理表名为 `detailed_listing_health`，不是带 `zzdnb_` 前缀的表名；共 `258,363` 行，`年月` 覆盖 `2023-11` 到 `2026-05`，`Lifetime Review Count` 非空 `258,363` 行，`Lifetime Average Customer Review Rating` 非空 `197,499` 行，覆盖 `4` 个店铺、`2,217` 个 Wayfair SKU。
- 重复键注意：按 `store + 年月 + Wayfair SKU + Supplier Part Number` 分组存在重复键，数据库抽样统计重复键约 `80,775` 组，单组最多样例为 `4` 行；因此 FineBI 从该表补 `评论数/评分` 时使用最大值聚合是合理的去重方式，报表侧也不要直接明细 join 后求和。
- 血缘：
  - 初始血缘可见多个同名分支，数字包括 `27`、`2`、`61`、`109`、`5` 等。
  - 展开 `109` 分支后进入 `广告` 自助表，继续展开 `108` 后可见 `分类目广告数据`、`总广告数据`、`WSP广告组每月汇总`、广告组汇总、`广告组广告花费、销售额、利润与ROI分析`、`广告组bid趋势`、`广告组活动明细表` 等。
- 当前判断：该表是广告主题评分 / 评论数补充来源之一，不是广告花费或利润生成源。

#### `zzdnb_广告组合单买详情`

- FineBI 路径：`公共数据/王威/店铺绩效/WF/zzdnb_广告组合单买详情`
- 类型：抽取数据集
- 数据量：2026-06-16 更新后 `296` 行
- 数据占用空间：`21.10 KB`
- 更新任务：`system` 定时触发的 `WF更新任务`，约 `08:00` 更新；近期多次记录均为增加 `0` 条，共 `296` 条。
- 已见字段：`sku`、`campaign_name`、`单卖`
- 预览样例：`CS0007-RH` 的 `单卖` 为 `组合链接`，`CS0013-组合链接` 等。
- 用途：广告组/组合 SKU 与单卖 SKU 关系补充。
- 待核实：是否直接进入 `WF-WSP广告组广告分析` 或 `wf广告` 自助数据集。

### 3.3 tsq / WF店铺曝光：广告出价链路

#### `zzdnb_wf广告出价表`

- FineBI 路径：`公共数据/tsq/WF店铺曝光/zzdnb_wf广告出价表`
- 类型：抽取数据集
- 粒度：`store + 日期 + Campaign name + wayfair SKU`
- 数据量：2026-06-16 更新后 `224,817` 行
- 数据占用空间：`48.18 MB`
- 更新任务：`system` 定时触发的 `WF店铺曝光更新任务`，约 `09:00` 更新。
- 2026-06-16 更新记录：增加 `716` 条数据，共 `224,817` 条数据。
- 已见字段：`store`、`日期`、`Campaign name`、`wayfair SKU`、`Suggested Bid detail(具体金额)`；血缘/页面文本同时可见 `Bid adjustment applied on B2B bids`、`Default bid(现在的bid)`、`最小建议bid`、`最大建议bid` 等展示字段。
- 血缘：
  - 页面可见该数据集有两个同名下游节点，右侧数量标记分别为 `27` 和 `134`。
  - 点击 `27` 后，展开为 `zzdnb_wf广告出价表 -> zzdnb_wf广告出价表`，剩余数量 `18`，并显示后续节点：`广告`、`每天bid变化明细`、`现在bid和建议bid日变化`、`最新日的现在bid和建议出价bid`、`建议出价_可联动下面图表`。
  - 点击 `134` 后，展开为 `zzdnb_wf广告出价表 -> zzdnb_wf广告出价表`，剩余数量 `133`，并显示后续节点 `广告`。
- 用途：广告组/SKU 分析中补充当前 BID、建议 BID、B2B bid adjustment、最小/最大建议 bid 等出价字段；已确认与 `广告组bid趋势`、bid 明细/日变化组件存在血缘关系。2026-06-18 复核补充：`WSP广告组每月汇总` 维度槽里的短字段 `bid` 已确认不是该出价表的 `Default bid(现在的bid)`，而是 `广告.product_max_bid` 的展示名。
- 2026-07-07 数据库侧复核补充：物理表名为 `wf广告出价表`，未见独立物理表 `zzdnb_wf广告出价表`；`zzdnb_wf广告出价表` 更像 FineBI 抽取数据集名称。2026-01 至 2026-05 广告事实按 `campaignlayer_id = campaign_id` 匹配出价快照 `124,503` 行，高于按 `Campaign name` 匹配的 `121,437` 行，仅名称匹配为 `0` 行。该范围内出价表只见 `WF-FOP`、`WF-TD`，未见 `WF-TD-CA`、`WF-TD-UK` 店铺快照；FineBI 是否有 CA / UK 业务兜底仍待核实。

### 3.4 王威 / 增量更新：OS 广告链路

#### `os广告重新计算`

- FineBI 路径：`公共数据/王威/增量更新/os广告重新计算`
- 类型：SQL 数据集
- 主要服务看板：`zzd/广告/OS广告`
- 数据量：2026-06-16 更新后 `155,885` 行
- 数据占用空间：`20.60 MB`
- 更新任务：`system` 定时触发的 `os广告重新计算更新任务`，约 `08:24` 更新；也存在 `wangwei` 手动触发记录。
- 已见字段：`store`、`Partner ID`、`Partner Name`、`Short SKU` 等。

```sql
select
  a.*,
  b.`SOFS Order Number`,
  b.`Supplier SKU`,
  b.`Retailer First Cost`,
  b.Quantity,
  c.采购SKU,
  数量
from (
  select * from sku_level_daily_advertising_rep
) a
left join (
  select *
  from overstock_orders_allinfo
  where left(`Order Date`,10) >= '2024-04-01'
    and status <> 'CANCELLED'
) b
  on a.Date = left(b.`Order Date`,10)
  and a.`Short SKU` = left(b.`SOFS SKU`,8)
  and a.store = b.store
left join (
  select distinct
    replace(店铺,' ' ,'-') 店铺,
    店铺SKU,
    采购SKU,
    数量,
    case when `更新时间` > insert_date then insert_date else `更新时间` end 更新时间,
    case when update_date is null then '9999-99-99 23:59:59' else update_date end update_date
  from 鲸汇店铺销售产品
  where left(店铺,2)='os'
) c
  on b.`Supplier SKU` = c.店铺SKU
  and b.store = c.店铺
  and (
    left(a.date,10) >= left(c.`更新时间`,10)
    and left(a.date,10) < left(c.update_date,10)
  )
where a.date >= '2025-01-01'
```

逻辑结论：

- 广告基础事实来自 `sku_level_daily_advertising_rep`。
- 订单补充来自 `overstock_orders_allinfo`，排除 `CANCELLED`，订单日期从 `2024-04-01` 起。
- 广告 SKU 与订单按 `a.Date = Order Date`、`Short SKU = left(SOFS SKU,8)`、`store` 关联。
- 采购 SKU 映射来自 `鲸汇店铺销售产品`，仅取 `left(店铺,2)='os'`，并按映射生效时间范围匹配。
- 最终广告数据只保留 `a.date >= '2025-01-01'`。

2026-07-10 数据库复核补充：

- `sku_level_daily_advertising_rep` 物理主键为 `store + Campaign ID + Date + Short SKU`，当前 `261,983` 行，覆盖 `2024-03-22` 至 `2026-07-08` 和 `OS-SP/OS-FOP/OS-LJD` 三店铺。
- 广告指标字段均为文本；抽样确认 `ROAS = First Cost Sales / Spend`、`ACoS = Spend / First Cost Sales`、`CVR = Units Sold / Clicks`，汇总层必须重算。
- 2026-06-01 至 2026-07-09，广告事实 `11,843` 行按 SQL 中的订单关联键投影后为 `14,060` 行，其中 `926` 条广告事实会匹配多个订单行，单个关联键最多 `14` 个订单行。
- 如果直接按关联后明细汇总，花费会从 `15,085.85` 投影为 `24,102.84`，`First Cost Sales` 会从 `163,492.74` 投影为 `335,242.39`。因此 `os广告重新计算` 不能被当作保持广告原粒度的一对一增强表；组件汇总前需先锁定广告主键或明确订单 / 采购 SKU 分摊规则。
- 2026-07-01 至 2026-07-09 的 `1,501` 条非取消订单行均命中生效期映射，但 `126` 条命中多个采购 SKU，最多 `5` 条映射。这类展开可能是组合产品业务关系，不应直接判为数据重复；但复制后的广告花费和销售额不能跨采购 SKU 直接求和。

#### `os广告2`

- FineBI 路径：`公共数据/王威/增量更新/os广告2`
- 类型：SQL 数据集
- 主要服务看板：`zzd/广告/OS广告`
- 数据占用空间：`215.78 MB`
- 更新信息页已见记录：2026-05-27 09:49 `wangwei` 手动触发 `增量更新文件夹更新`，结果为更新失败，原因是“更新任务被中断，不再执行”；页面当前未展示 2026-06 的常规更新记录，需继续核实是否仍在看板使用或已被 `os广告重新计算` 替代。
- 已见字段：`store`、`Partner ID`、`Partner Name`、`Short SKU` 等。

```sql
select
  a.*,
  b.`店铺SKU`,
  b.`采购SKU`,
  b.`数量`,
  c.预计和实际ddp汇总
from (
  select
    a.*,
    A.`Partner SKU` AS 上架sku
  from sku_level_daily_advertising_rep A
  where a.date >= '2024-03-22'
) a
left join (
  select *,
    max(rn) over(PARTITION by 店铺,`店铺SKU`) 采购sku数量
  from (
    select *,
      REPLACE(店铺,' ','-') 店铺修,
      RANK() over(PARTITION by 店铺,`店铺SKU` order by left(`更新时间`,10) desc) rn
    from `鲸汇店铺销售产品`
  ) t
) b
  on (
    a.上架sku like concat('%',b.`店铺SKU`,';%')
    and 采购sku数量 = 1
    and a.store = b.店铺修
  )
  or (
    采购sku数量 > 1
    and a.store = b.店铺修
    and a.上架sku like concat('%',b.`店铺SKU`,';%')
    and left(a.date,10) >= left(b.`更新时间`,10)
    and left(a.date,10) < left(b.update_date,10)
  )
left join `ddp汇总` c
  on b.采购sku = c.sku
  and left(a.date,7) = c.年月
  and c.实预平台 = '垂直'
```

逻辑结论：

- 基础广告事实同样来自 `sku_level_daily_advertising_rep`。
- `Partner SKU` 被重命名为 `上架sku`，用于与 `鲸汇店铺销售产品.店铺SKU` 匹配。
- 如果同一店铺 SKU 只有一个采购 SKU 映射，按 `上架sku like 店铺SKU` 和店铺匹配。
- 如果存在多个采购 SKU 映射，则额外按 `更新时间/update_date` 的生效区间匹配。
- DDP 来源为 `ddp汇总`，按 `采购sku + 年月` 且 `实预平台='垂直'` 关联。

2026-07-10 数据库复核补充：`ddp汇总` 当前为 11 字段视图，候选键是 `sku + 年月 + 实预平台`，但本轮轻量聚合未返回有效结果，唯一性仍为 `待核实`。OS 映射表还出现 `OS-EZS` 历史 / 映射店铺，而广告事实仅见 `OS-SP/OS-FOP/OS-LJD`；`OS-EZS` 是否仍属有效广告范围待业务确认。

### 3.5 王威 / 店铺绩效 / HD：HD 广告链路

#### `zzdnb_hd广告`

- FineBI 路径：`公共数据/王威/店铺绩效/HD/zzdnb_hd广告`
- 类型：抽取数据集
- FineBI 数据预览显示：`8,805` 行
- 数据占用空间：`2.13 MB`
- 更新信息页：未展示更新历史，需继续核实数据同步方式。
- 已见字段：`store`、`DATE`、`VENDOR_NAME`、`CAMPAIGN_ID`、`CAMPAIGN_NAME`、`PLACEMENT_TYPE`、`IMPRESSIONS`、`CLICKS`、`SPEND`、`CPC`、`CPM`、`CTR`、`UNITS_SOLD`、`TOTAL_SALES`、`ROAS`
- 当前用途：HD 广告相关数据集；是否实际进入 `zzd/广告` 下的看板仍需核实。

### 3.6 章森淼：广告辅助标签/可用引用

#### `滞销品列表`

- FineBI 路径：`公共数据/章森淼/滞销sku列表/滞销品列表`
- 类型：SQL 数据集
- 数据量：2026-06-16 更新后 `443` 行
- 数据占用空间：`96.27 KB`
- 更新任务：`system` 定时触发的 `滞销品列表更新任务`，约 `00:10` 更新。
- 已见字段：`sku`、`标签`、`分类`、`是否新品`、`可用` 等。
- 更新坑点：2026-05-28 更新失败，FineBI 报错 MySQL 临时表 full：`The table 'D:\azml\mysql-8.0.26-winx64\mysql_tmp\#sql196c_78a92_5cd' is full`。

```sql
SELECT
  a.*,
  b.`是否存在库龄超60天批次`,
  ifnull(c.计划发货量,0) as 后续计划发货量
from (
  select
    sku,
    标签,
    分类,
    是否新品,
    可用,
    在途,
    本月预估销量,
    case
      when 本月预估销量!=0 then ifnull(可用/本月预估销量,0)
      else "∞"
    end as 可用可销售月数,
    case
      when 本月预估销量!=0 then ifnull(在途/`本月预估销量`,0)
      else "∞"
    end as 在途可销售月数,
    case
      when 本月预估销量!=0 then ifnull((可用+在途)/本月预估销量,0)
      else "∞"
    end as 总可销售月数
  FROM `每月销量售罄预估表`
  where 本月预估销量 is not null
) as a
left join (
  SELECT sku,"是" as `是否存在库龄超60天批次`
  FROM `入库单库龄明细_2`
  where 库龄>60
  GROUP BY sku
) as b on a.sku=b.sku
left join (
  select sku,sum(计划发货量) as 计划发货量
  from `入库单物流明细all`
  where 状态="计划发货"
  GROUP BY sku
) as c on a.sku=c.sku
where (a.总可销售月数>=3 or a.总可销售月数="∞")
  and b.sku is not null
  and 分类!="零件"
```

逻辑结论：

- 滞销/清仓判断基于 `每月销量售罄预估表` 的可用、在途、本月预估销量计算可销售月数。
- 只保留存在 `库龄 > 60` 批次的 SKU。
- 排除 `分类 = 零件`。
- 补充 `入库单物流明细all` 中状态为 `计划发货` 的后续计划发货量。
- 待核实：具体哪个广告看板、自助数据集或组件引用该数据集。

#### `仓库销量可用预估2`

- FineBI 路径：`公共数据/章森淼/仓库销量可用预估2`
- 状态：库存专项已梳理，不在广告专项重复展开 SQL/字段。
- 已有资料：见 `FineBI库存数据梳理.md` 中 `公共数据/章森淼/仓库销量可用预估2` 章节。
- 广告看板可能引用的数据集：
  - `zzdnb_tb_kyxl_forecast_and_reality_合并sku`
  - `各sku每月断货天数`
  - `zzdnb_断货sku数量_合并`
- 广告专项后续任务：只确认具体广告看板/组件/自助数据集是否引用这些已知库存数据，不重复学习库存生成链路。

### 3.7 Wayfair WSP 广告组剩余字段来源核实（2026-06-19）

#### `活动的垂直标签` / `广告标签修`

- 已确认展示公式：`广告标签修` 输入为 `活动的垂直标签`，优先级为 `清仓 -> 滞销 -> 在售 -> 新品6个月 -> 新品3个月 -> 下架 -> 空`。
- 数据库核实：未发现物理列 `活动的垂直标签` 或 `广告标签修`。
- FineBI 上游已确认：`活动的垂直标签` 在 `WF店铺绩效2.0/广告` 中生成，不是物理同名列。主 `广告` 表从 `zzdnb_公司sku状态表` 按 `UPPER(采购SKU) = UPPER(sku)` 补 `公司标签`、`model`、`广告细分标签`、`垂直标签`，聚合方式均为最大值；然后生成 `广告标签`，再按 `[extend, isB2b]` 对 `广告标签` 做组内字符串拼接得到 `活动的垂直标签`。
- 2026-06-22 补充：同一主表还从 `zzdnb_采购sku_model映射表` 按 `UPPER(采购SKU)=UPPER(采购sku)` 补 `model` 最大值，之后再生成 `model1 = 按 [extend,isB2b] 组内字符串拼接 [model]`。因此标签来源仍以 `zzdnb_公司sku状态表` 的 `广告细分标签/垂直标签` 为准，model 展示存在公司状态表和采购 SKU model 映射表两条补充线。
- `广告标签` 公式：

```text
IF(广告细分标签="新品1-3", "新品3个月",
  IF(OR(广告细分标签="新品4", 广告细分标签="新品5", 广告细分标签="新品6"), "新品6个月",
    IF(AND(广告细分标签="新品7"), "在售", 垂直标签)
  )
)
```

- 历史物理候选/对照入口：`上架sku利润表.垂直标签/垂直细分标签` 经 `wfproduct_management.Supplier_Part_Number -> SKU` 桥接到广告 SKU，可用于数据库侧复核标签覆盖，但不是本次 FineBI 页面确认的直接来源。
- 桥接逻辑候选：
  - `上架sku利润表.store = wfproduct_management.店铺`
  - `上架sku利润表.上架sku = wfproduct_management.Supplier_Part_Number`
  - `LEFT(上架sku利润表.日期,7) = LEFT(wfproduct_management.insert_date,7)`
  - 再与广告事实按 `store + sku + LEFT(date_est,7)` 对齐。
- 抽样事实：`WF-TD / 2026-05 / campaign_id=395570 / OARI1170` 可桥到 16 个上架 SKU、8 个采购 SKU，聚合 `垂直标签` 含 `下架,在售,新品,滞销`，`垂直细分标签` 含 `下架,在售,新品1-3,滞销1`。`WF-FOP / EVYN4068` 也存在同类桥接样本。
- 2026-06-19 再核实：`INFORMATION_SCHEMA` 精确搜索未发现 `活动的垂直标签`、`广告标签修` 物理列；`上架sku利润表` 2026-01 至 2026-05 Wayfair 行覆盖 `WF-FOP/WF-TD/WF-TD-CA/WF-TD-UK`，存在 `垂直标签/垂直细分标签` 且按日期变化。若用它做数据库侧对照，应先按广告月 + SKU 映射预聚合，不能直接把一对多明细展开后汇总广告指标。
- 2026-06-22 FineBI + 数据库复核：FineBI 左侧 `zzdnb_公司sku状态表` 实际对应数据库物理表 `公司sku状态表`，不是 SQL 抽取表；更新信息显示“当前表无需抽取，自动获取最新数据”。血缘为 `DB 公司sku状态表 -> zzdnb_公司sku状态表 -> 广告`，下游数字 `133`。预览 `3,535` 行，主键 `sku`，核心字段包括 `category`、`model`、`公司标签`、`是否新品`、`采购状态`、`垂直标签`、`垂直细分标签`、`垂直标签改动原因`、`垂直标签改动时间`、`amz标签`、`广告细分标签`、`来源`、`create_time`、`update_time`。
- 数据库生成逻辑：事件 `公司sku状态表_2` 每天执行，最近一次本地时间约 `2026-06-22 06:45`，调用过程 `公司sku状态表_2()`；过程先从 `warehouse_stock_最新`、`公司sku上下架表`、`工厂订货表` 补新 SKU，再用 `采购sku_model映射表` 补 `category/model`，结合入库、订单、库存、特殊 SKU 等生成 `公司sku状态表_temp_2`，再回写 `公司sku状态表` 的 `垂直标签/垂直细分标签/广告细分标签/采购状态/公司标签`。
- `广告细分标签` 生成规则：基于 `公司sku状态表.垂直标签` 和 `公司sku状态表_temp_2` 的 `首次到仓时间/首次出单时间/最近一次垂直标签进入滞销时间/填报进入滞销时间`。新品按 0-90 / 90-120 / 120-150 / 150-180 / 180-210 天拆为 `新品1-3/新品4/新品5/新品6/新品7`；滞销按进入滞销时间 2 个月内 / 2 个月以上拆为 `滞销1/滞销2`；其他默认取 `垂直标签`。
- 是否按月变化：FineBI 当前广告链路使用的是 `公司sku状态表` 当前快照表，join 只见 `UPPER(采购SKU)=UPPER(sku)`，未见月份、store、campaign_id 条件；因此当前页面的 `活动的垂直标签` 不是按广告月份回溯取历史标签。若报表需要历史月度标签，应另接 `公司sku状态表_记录留存` 或 `公司sku状态表_temp_2_记录留存` 的 `留存date` 版本，并按月选择对应留存快照。
- 2026-06-23 留存表覆盖核实：`公司sku状态表_记录留存` 覆盖约 `2026-03-10` 至 `2026-06-22`，`公司sku状态表_temp_2_记录留存` 覆盖约 `2026-03-10` 至 `2026-06-22`；2026-01、2026-02 无月末留存快照，2026-03 可取 3 月末附近快照，2026-04/05 可取月末快照。推荐口径：若目标是严格复刻当前 FineBI，用当前 `公司sku状态表` 快照；若目标是历史业务分析，可对 2026-03 至 2026-05 用留存月末快照，对 2026-01/02 标注“历史快照缺失”或单独用当前快照兜底。留存复刻 join 仍以 `UPPER(采购SKU)=UPPER(sku)` 为核心，再按广告月份选择 `留存date <= 月末` 的最近快照；留存表字段是否完全等同当前 `广告细分标签` 仍需抽样核对。

#### `本竞品价格对比` / `是否有价格优势`

- FineBI 已见：`WF广告数据/广告` 从 `本竞品价格对比` 回补 `本品最低前台价`、`竞品最低前台价`，匹配 `Display_SKUs = link_sku`、`store = 店铺`。
- 数据库核实：未发现同名物理表 `本竞品价格对比`；可用物理表 `产品近两天的最小前台价` 复刻字段。该物理表由同名事件每日 06:40 截断重建，核心源为 `link_sku_price_qtj_fabric`，并用 `link_sku_table_2`、`平台产品详情表` 补空 model。
- 物理表字段：`store`、`platform`、`model`、`link_sku`、`产品类型`、`最近日期最小前台价`、`上一日期最小前台价`、`create_time`。
- 复刻逻辑：取 `产品类型="本品"` 的 `store + link_sku + model` 作为本品组合；竞品行的 `store` 为 `Wayfair`，同表应按 `platform + model + 产品类型="竞品"` 聚合最小 `最近日期最小前台价` 为 `竞品最低前台价`，本品侧最小 `最近日期最小前台价` 为 `本品最低前台价`。
- 抽样覆盖：2026-06-19 表内本品 `9,163` 行、竞品 `4,640` 行；FineBI 目标店铺样本可取出本品最低价和竞品最低价。
- 时间属性：当前快照；字段为最近/上一日期价格，不是 2026-01 至 2026-05 月度历史。
- FineBI 页面原式：`IF(ISNULL(竞品最低前台价), null, IF(竞品最低前台价 < 本品最低前台价, "否", "是"))`，该写法在 `本品最低前台价 = 竞品最低前台价` 时返回“是”。
- 当前报表严格业务口径：若定义为“本品最低前台价低于竞品最低前台价才算有优势”，应使用 `IF(ISNULL(竞品最低前台价), null, IF(本品最低前台价 < 竞品最低前台价, "是", "否"))`，等价时返回“否”。
- 2026-06-19 再核实：同名事件 `产品近两天的最小前台价` 状态启用，最近执行时间为 `2026-06-19 06:40`；过程会 `TRUNCATE` 后写入该表，源表为 `link_sku_price_qtj_fabric`，过程内先按 `产品id` 取最近两天，再以 `min(price) over(partition by model, link_sku, 产品类型, rk)` 生成最近/上一日期最小前台价，并用 `link_sku_table_2`、`平台产品详情表` 补空 `model`。
- 2026-06-19 样本：目标 WF 店铺当前快照 `1442` 个本品组合，`1368` 个有竞品价；本品低于竞品 `420`、等于竞品 `13`、高于竞品 `935`。等价样本会导致 FineBI 原式与严格业务口径结果不一致。
- 2026-06-23 数据库复刻样本：当前快照可按 `platform + model` 汇总竞品最低价。本品等于竞品价样例 `WF-TD / model=12113 / link_sku=W010902757 / 本品=39.99 / 竞品=39.99`、`WF-TD / model=14561 / link_sku=W002010929 / 本品=46.99 / 竞品=46.99`，FineBI 原式会返回“是”，严格业务口径返回“否”；本品低于竞品价样例 `WF-FOP / model=OD0001 / W118334495 / 149.99 < 169.00`、`WF-FOP / model=OD0001 / W114034018 / 98.99 < 169.00` 返回“是”；本品高于竞品价样例 `WF-FOP / model=OD0001 / W118364453 / 183.00 > 169.00`、`WF-FOP / model=OD0001 / W117628967 / 286.99 > 169.00` 返回“否”。FineBI `本竞品价格对比` 页面仍未暴露 SQL，未完成组件导出样本对账，因此“物理复刻源”和“等价处理”已确认，“FineBI 导出样本逐行一致”仍待核实。
- 2026-06-23 二次数据库样本：`产品近两天的最小前台价` 当前 `14,358` 行，物理字段仍只有 `store/platform/model/link_sku/产品类型/最近日期最小前台价/上一日期最小前台价/create_time`；未发现 FineBI 页面字段 `本品最新最低价/竞品最低前台价/更新日期/website` 的同名物理列在该表内。目标 WF 本品行数：`WF-FOP 298`、`WF-TD 1106`、`WF-TD-CA 269`、`WF-TD-UK 379`；竞品行数：`Wayfair 4301`、`Wayfair-CA 267`、`Wayfair-UK 71`。补充三类样本：等价 `WF-TD / BARST-2SET / EVYN4068 / 113.99=113.99`、`WF-FOP / BARST-2SET / EVYN4068 / 113.99=113.99`、`WF-TD / 12113 / W010902757 / 39.99=39.99`；本品低于竞品 `WF-TD-UK / SF0006 / U110613509 / 115.99<289.99`、`WF-FOP / OT0002 / W010831993 / 105.99<109.00`、`WF-FOP / OT0003 / W010832448 / 78.99<144.99`；本品高于竞品 `WF-TD-CA / BARST-2SET / C009785300 / 420.00>113.99`、`WF-TD / BARST06 / OARI2480 / 286.99>237.99`、`WF-TD-UK / BARST-2SET / U110612946 / 299.98>113.99`。
- 2026-07-01 当前数据库复核：`产品近两天的最小前台价` 当前 `8,077` 行且 `产品类型` 仅见 `本品`；上游 `link_sku_price_qtj_fabric` 当前也只见 `本品`，但 `link_sku_table_2` 仍保留 Wayfair / CA / UK 竞品映射。结论：历史样本证明该复刻逻辑曾能生成竞品价，但当前快照不能直接产出 `竞品最低前台价 / 是否有价格优势`；FineBI 当前页面是否仍有竞品价及其来源仍待核实。
- 2026-07-02 迁移探查：已检查 `temp_link_sku_price_qtj_fabric`、`temp_latest_link_sku_price_scj`、`最新价格表`、`竞品对标表`、`本竞品失效sku`。前两张 temp 表回连 `link_sku_table_2` 后也只见 `本品` 或未映射；`最新价格表` 无竞品字段且记录日期偏旧；`竞品对标表` 只存竞品关系；`本竞品失效sku` 只存状态。因此未发现可替代当前竞品最低价的数据库入口，FineBI 若当前仍显示竞品价，来源可能是缓存、旧抽取或另一路未识别数据源，待核实。
- 2026-07-02 过程补充：小样本确认 `product_fabric` 近 90 天仍可命中部分最新竞品映射价格，但 `link_sku_price_qtj_fabric` 和当前快照仍无竞品行；后续应优先追 `link_sku_price_qtj_fabric` 竞品分支写入 / 覆盖原因，而不是继续盲找替代表。
- 2026-07-02 用户确认：缺竞品根因是相关函数/例程执行失败，现已修复；恢复后的当前快照和 FineBI 当前样本仍需只读复核后再作为智能取数默认入口。
- 2026-07-02 修复后只读复核：`产品近两天的最小前台价` 当前 `本品 10,485` 行、`竞品 4,649` 行，`create_time=2026-07-02 06:40:00`；目标 WF 店铺严格业务口径优势覆盖已可计算。后续只剩 FineBI 当前组件样本对账，而不是继续判断数据库是否缺竞品。

#### `总成本` / `总利润`

- 数据库字段搜索结果：未发现广告链路可直接抽取的精确 `总成本/总利润` 物理列；同名字段只命中 `钉钉利润提醒表`、`model弃养状态`、`垂直采购sku销量断货情况表` 等非当前广告精确利润链路。
- 2026-06-23 数据库通后复核：`INFORMATION_SCHEMA` 精确搜索 `总成本/总利润/每个sku平均成本/ddp平均成本/平均ddp修/发货费平均成本/单买数量/每个上架skuddp/每个采购skuddp/总ddp/新ddp/最新ddp/出货费用/预计和实际ddp汇总`，仍未发现广告事实或广告自助结果可直接抽取的精确 `总成本/总利润/每个sku平均成本` 落地列；命中的 `总成本` 仍在 `model弃养状态`、`temp_垂直采购sku销量表`、`垂直采购sku销量断货情况表`，`总利润` 在 `钉钉利润提醒表`，均不属于当前 WSP 广告精确利润链路。可用物理字段包括 `ddp汇总.预计和实际ddp汇总`、`产品发货费.出货费用`、`广告组合单买详情.单卖`，需要按 FineBI 公式预聚合复算。
- 当前可信来源仍是 FineBI 自助表公式链：
  - `总成本 = (每个sku平均成本 * attributed_units_window_view_through_Day_14 / 单买数量) + spend_USD`
  - `总利润 = attributed_wholesale_cost_window_view_through_USD_Day_14 * IF(store="WF-FOP",0.93,0.94) - 总成本`
- 2026-06-22 主 `广告` 表再次确认上述最终公式；前置节点 `zzdnb_advertising_product_report_by_day` 曾显示不除以 `单买数量`，但主表最终字段仍带 `/ 单买数量`。当前 `单买数量` 公式写死为 `1`，所以两者在当前配置下数值暂时等价，后续若单买数量恢复为组合件数量，必须以主表最终公式重算。
- 可落地预聚合方案：
  1. 广告事实层：`advertising_product_report_by_day` 先按广告日 / campaign / SKU / `product_max_bid` 粒度保留 WSC、件数、花费。
  2. SKU 映射层：按公共 `wf广告` 同款逻辑，用 `first_10_part_numbers` / `cg_part_sku映射时间范围` 拆到 `店铺SKU + 采购SKU + 数量 + extend + isB2b`。
  3. DDP 月度层：从 `ddp汇总` / `ddp汇总表_美国/加拿大/英国` 预聚合 `采购SKU + 年月 + 国家` 的 `预计和实际ddp汇总`。
  4. 发货费层：从 `发货费` / `产品发货费` / `cg仓发货费` 预聚合 `采购SKU` 的 `出货费用`。
  5. 单买数量层：从 `广告组合单买详情` 按 `sku + campaign_name` 取 `单卖` 最大值。
  6. 成本公式层：按 FineBI 已确认分组重算 `每个上架skuddp`、`每个上架sku采购sku数量`、`每个采购skuddp`、`ddp平均成本`、`平均ddp修`、`发货费平均成本`、`每个sku平均成本`、`总成本/总利润`。
- 结论：不能用 `上架sku利润表` 冒充广告精确 `总成本/总利润`；该表只可作为利润率 / model / 成本辅助和异常排查。
- 月度预聚合补充：`wf每月产品和广告数据汇总` 对应 FineBI 公共数据 `总广告数据`，主要支撑 `zzd/广告/WF-WSP整体广告分析`、`WSP广告分析` 的整体广告 vs 自然销售、广告花费占比、广告花费/销售额/利润/ROI 等月度组件；`wf每月产品和广告数据` 对应 FineBI 公共数据 `分类目广告数据`，主要支撑 `zzd/广告/WF-WSP分类广告分析`、`WSP类目广告分析` 的分类广告 vs 自然销售、分类广告趋势、分类广告花费/销售额/利润/ROI 等组件。两张表只含月度 `广告花费/广告销售额/总销售额/自然销售额` 等经营指标，没有 `总成本/总利润`。
- 2026-06-23 修正核实：数据库事件/过程 `wf每月产品和广告数据` 每天仍在执行，最近执行到 `2026-06-22 23:17`；上游 `option_drill_down`、`detailed_listing_health`、`advertising_product_report_by_day`、`order_details_tb` 均已有 2026-05，且中间表 `temp_wf每月产品数据`、`temp_wf每月产品总销售数据2` 也有 2026-05。但最终表 `wf每月产品和广告数据`、`wf每月产品和广告数据汇总` 当前仍停在 `2026-04`，`create_time` 约 `2026-05-30 07:18`。Chrome 进入 `WF广告数据/总广告数据` 后预览也显示 `共 76 条`、`create_time=2026-05-30 07:18:02`，与物理最终表一致，未见 2026-05 已落入该主题预览。
- 只读复算最终 SELECT 逻辑可产出 2026-05：汇总层样例 `WF-FOP spend=5730.39/WSC=73427.24/总销售额=280738.03`、`WF-TD spend=61535.06/WSC=986596.30/总销售额=1785119.40`、`WF-TD-CA spend=1312.76/WSC=17121.04/总销售额=64493.22`、`WF-TD-UK spend=1910.95/WSC=11970.98/总销售额=64727.22`。因此“5 月应该可更新”成立；当前异常是最终落表 / FineBI 主题预览未包含 2026-05，而不是底层源数据没有 5 月。
- 2026-06-23 FineBI 可见样本对账补齐：先在 `WF店铺绩效2.0/广告` 预览横向定位到 `每个sku平均成本/总成本/总利润/单买数量/model1`，再用同一行序右侧 `extend` 回查 `advertising_product_report_by_day`，已形成 5 条带业务键样本。复算公式为 `总成本 = 每个sku平均成本 * units14 / 单买数量 + spend_USD`，`总利润 = WSC * 0.93 - 总成本`（样本均为 `WF-FOP`）。样本对账：`2023-09-27 / campaign_id=307358 / sku=UXIF3086 / extend=f66b... / WSC=0 / units14=0 / spend=0.44 / FineBI 总成本=0.44 / 总利润=-0.44 / 复算一致`；`2023-09-28 / 307359 / UXIF3246 / 7199... / WSC=0 / units14=0 / spend=0.84 / FineBI 0.84/-0.84 / 复算一致`；`2023-09-28 / 307358 / UXIF3086 / 8669... / WSC=399.99 / units14=2 / spend=0.91 / FineBI 7.06/364.93 / 复算成本按显示均值为 7.07、差异 0.01，利润按 FineBI 成本可对上`；`2023-09-29 / 307358 / UXIF3086 / 5e1... / WSC=192 / units14=1 / spend=1.65 / FineBI 4.73/173.84 / 复算利润差异 0.01`；`2023-09-30 / 307362 / UXIF3348 / ca2... / WSC=258 / units14=2 / spend=10 / FineBI 17.50/222.44 / 复算一致`。差异均为 `每个sku平均成本` 显示两位小数导致的 0.01 级舍入差异。该批样本能证明公式可脚本复刻；但 FineBI 预览第 50 页仍停在 2024-04，未拿到 2026-01 至 2026-05 同屏样本，正式报表上线前建议再抽 2026 样本做同样 QA。

#### `总销售额1`

- FineBI 公式：`总销售额1 = if(ISNULL(销售额), "", 销售额 / IF(store="WF-FOP",0.93,0.94))`。
- `销售额` 物理源：FineBI `zzdnb_wf上架sku每日销售额` 对应数据库物理表 `wf上架sku每日销售额`。
- 字段：`store`、`日期`、`supplier_part`、`currency`、`total_revenue`、`units_sold`、`链接sku`、`wayfair_sku`。
- FineBI 匹配：`store = store`、`LEFT(date_est,7) = LEFT(日期,7)`、`sku = wayfair_sku`，`销售额 = SUM(total_revenue)`。
- 2026-06-23 组件级修正：最终要出的 WSP 广告组数据表，以及 `WF-WSP广告组广告分析 -> 广告组活动 -> WSP广告组每月汇总` 里的 `总销售额` 相关字段，应使用这条 `zzdnb_wf上架sku每日销售额 / wf上架sku每日销售额` 链路，而不是用 `wf每月产品和广告数据` 或 `wf每月产品和广告数据汇总` 反推。后两张月度表主要对应整体 / 分类广告看板。
- 2026-06-23 Chrome 复核路径：在 `WF广告数据` 主题中先打开 `zzdnb_wf上架sku每日销售额`，字段为 `store/日期/supplier_part/currency/total_revenue/units_sold/链接sku/wayfair_sku`；再进入 `广告` 表右侧流程，`总销售额1` 前置“其他表添加列”节点明文显示：从 `zzdnb_wf上架sku每日销售额` 选择 `销售额`，按求和，匹配 `store=store`、`LEFT(date_est,7)=LEFT(日期,7)`、`sku=wayfair_sku`。切到 `WSP广告组每月汇总` 组件后，展示字段 `总销售额` 的字段信息为 `来源字段名：总销售额修`、`来源表：广告`。血缘视图同步可见 `zzdnb_wf上架sku每日销售额 -> 广告 -> WSP广告组每月汇总 -> WSP广告组广告分析`。
- 最终表接入规则：先把广告事实按 `store + 月份 + campaign_id/campaign_name + sku + bid(product_max_bid)` 等目标粒度聚合，再按 `store + 月份 + sku = wayfair_sku` 左连 SKU 月销售额；`总销售额1` 是 SKU 月销售额扣点还原后的字段，可对到广告组和月份，但不是 bid 原生指标。若同一 SKU 在同月同广告组下拆成多个 bid 行，`总销售额1` 会被带到多行，后续汇总时必须按唯一 `store + 月份 + campaign + sku` 去重或复用 FineBI 的 `AVG_AGG(总销售额1)` 口径，不能直接把 bid 行上的 `总销售额1` 求和。
- 与 `wf_supplier_parts_performance` 差异：2026-01 至 2026-06 美国店 `WF-FOP/WF-TD` 月店铺汇总完全一致；`WF-TD-CA/WF-TD-UK` 行数和销量一致但金额不同。若报表包含 CA/UK，建议替换为 FineBI 同源 `wf上架sku每日销售额`。
- 2026-06-19 再核实：同名过程 `wf上架sku每日销售额` 每日执行，写入时从 `wf_supplier_parts_performance` 左连 `平台产品详情表` 补 `链接sku/wayfair_sku`，并按最新 `货币汇率` 折算 `GBP/CAD`。2026-01 至 2026-05 对比：`WF-FOP/WF-TD` 两表金额差异为 `0`；`WF-TD-CA` 为 `325510.05 -> 233097.75`，`WF-TD-UK` 为 `89459.98 -> 120323.68`。
- 2026-06-26 再核实：事件最近执行 `2026-06-26 08:27:00`，但目标表与源表最新业务日期均为 `2026-06-23`；本表只确认销售额和销量，不确认总成本 / 总利润字段来源。
- 2026-06-30 数据库复核补充：`wf上架sku每日销售额.supplier_part` 是销售源 Supplier Part Number / 销售 SKU 层，按产品管理可为 `Kit`、`Standard`、`Sellable Component` 或少量 `Nonsellable Component`；`总账单详情_extract.上架sku` 是 WF 订单数据拆到最细 part number 后的子件。若把总销售额继续分摊到子件 / 采购 SKU，必须先识别 Product_Type，再用 `Related_Kit_Part -> Supplier_Part_Number` 展开，不能直接 `supplier_part = 上架sku`。

#### 原 `断货百分比` 字段

- 已确认生成链路：`WF店铺绩效2.0/wf广告` 从 `各sku每月断货天数` 补 `可用`，匹配 `UPPER(采购SKU)=UPPER(sku)`、`date_est=date`、`国家=国家`。
- 中间公式：
  - `上架sku可用库存 = DEF(MIN_AGG(可用), [extend, isB2b, 店铺SKU])`
  - `断货part个数 = NVL(DEF(COUNTD_AGG(店铺SKU), [isB2b, extend], [上架sku可用库存=0]), 0)`
  - `活动的上架sku数量 = DEF(COUNTD_AGG(店铺SKU), [isB2b, extend])`
  - `断货百分比 = SUM_AGG(断货part个数) / SUM_AGG(活动的上架sku数量)`
- 2026-06-22 当前主 `广告` 页单点公式显示 `断货part个数 = DEF(COUNTD_AGG(店铺SKU), [isB2b,extend], [上架sku可用库存=0])`，未显示 `NVL`，且该节点预览提示“字段丢失”；`活动的上架sku数量 = 按 [isB2b,extend] 组内去重计数 [店铺SKU]` 可正常预览。复刻原断货百分比前需以导出样本确认当前版本是否仍在组件层补 0。
- 2026-06-19 再核实：数据库未发现 `断货part个数`、`活动的上架sku数量`、`上架sku可用库存` 同名物理列；当前库可见底层库存表为 `warehouse_stock`，含 `date/sku/part/可用/仓库` 等字段，`wf_sku不断货销量` 含 `断货销量/不断货销量/断货销量占比`，但该销量辅助口径不能替代广告组断货 part 口径。
- 是否接入当前报表：用户当前报表已改用链接 SKU 有货率，原 `断货百分比` 仅在需要复刻 FineBI WSP 原组件时接入。

#### 分类分区规则：`室内` / `灯具` / `户外` / `其他`

- 已确认：FineBI 广告主题中的分区字段是中文 `分类`，不是直接按数据库 `category`、`first_class`、`second_class` 筛选。
- 来源链路：`WF广告数据/广告` 由 `wf广告` / 广告事实字段 `class_name`，通过 FineBI 左侧表 `zzdnb_wf_class中文分类` 回补中文 `分类`；数据库物理表名为 `wf_class中文分类`。
- 字段与 join key：`advertising_product_report_by_day.class_name = wf_class中文分类.\`Class Name\``，取 `wf_class中文分类.分类`。该映射表字段为 `Class Name`、`类目`、`分类`。
- 映射值：数据库当前 `分类` 包含 `室内家具`、`户外家具`、`灯具`、`厨房卫浴`、`其他`、`家居装饰`。看板/组件名中的“室内”“户外”是展示简称；字段值层通常对应 `室内家具`、`户外家具`。
- 抽样验证：2026-01 至 2026-05 广告日报按该映射聚合后有 `室内家具`、`户外家具`、`灯具`、`家居装饰`、`其他`；因此 WSP 分类 / 广告组分区应优先用 `分类` 字段，不要改用商品维表 `first_class/second_class`。
- 2026-06-23 页签过滤补核：`灯具广告组汇总数据` 组件编辑弹窗已确认结果过滤为 `分类 属于 固定值 灯具`。`家具装饰广告组汇总数据` 报表页签在 `店铺=WF-TD、日期>=2026-01-01` 下的展示值，与数据库按 `分类='家居装饰'` 聚合完全对上，例如 2026-01 曝光 `17,430`、点击 `414`、花费 `104.57`、WSC `226.39`；因此该页签展示名“家具装饰”对应字段值 `家居装饰`。`其他广告组汇总数据` 编辑弹窗本轮未成功打开；数据库按广告事实 2026-01 至 2026-05 全范围聚合只出现 `室内家具/户外家具/灯具/家居装饰/其他`，其中 `其他` 仅 24 行、花费 `34.64`、WSC `0`，对应 `Class Name=Multichannel-Only (MCO) Fulfillment` / `类目=多渠道专供商品`，该范围未出现 `厨房卫浴`。因此当前报表范围可按 `分类='其他'` 接入，不需要把 `厨房卫浴` 并入其他；后续若更长时间或新数据出现 `厨房卫浴`，仍需确认 FineBI 是否隐藏、单独页签或并入其他。
- 粒度：映射表为 `Class Name -> 分类` 静态/准静态维表；广告事实仍按 `store + date_est + campaign_id/campaign_name + sku + product_max_bid` 聚合后补分类。
- 是否可接入脚本：可直接左连 `wf_class中文分类` 补 `分类`。未匹配的 `class_name` 应输出异常清单；不要把未匹配行默认归入 `其他`。

#### 2026-06-23 剩余 4 项只读补核

- `总成本 / 总利润` 2026 同期样本：
  - FineBI 路径：`WF店铺绩效2.0/广告`。
  - 当前页面可见字段：`每个sku平均成本`、`ddp平均成本`、`每个上架skuddp`、`总成本`、`总利润`、`单买数量`、`活动的垂直标签` 等。
  - 页面限制：预览第 `50/50` 页仍为 `2024-04-16~2024-04-29`，未能直接取得 2026-01 至 2026-05 已计算样本。数据库可查 2026 广告事实字段，但没有 `每个sku平均成本/总成本/总利润` 物理列，不能冒充 FineBI 已计算样本。
  - 状态：公式链已确认；2026 同期 FineBI 已计算样本仍待导出或过滤样本复核。
- `是否有价格优势`：
  - FineBI `WF广告数据/广告` 添加列确认：从 `本竞品价格对比` 选择 `本品最低前台价`、`竞品最低前台价`，均按最小值；匹配键 `Display_SKUs = link_sku`、`store = 店铺`。
  - FineBI 公式确认：`IF(ISNULL(竞品最低前台价), null, IF(竞品最低前台价 < 本品最低前台价, "否", "是"))`。等价时 FineBI 原式返回“是”，严格“本品低于竞品才有优势”口径返回“否”。
  - FineBI `本竞品价格对比` 数据源字段可见 `更新日期`、`竞品最低前台价`、`website`、`本品最新最低价`、`国家`、`采购sku`；数据库物理快照表 `产品近两天的最小前台价` 没有 `website` 列，`website` 来自上游 `link_sku_price_qtj_fabric`。
  - 三类样本：低于 `WF-FOP / model=240 / W110576662 / 149.99<189.99 / FineBI=是 / 严格=是`；等于 `WF-FOP / BARST-2SET / EVYN4068 / 113.99=113.99 / FineBI=是 / 严格=否`；高于 `WF-FOP / model=220 / W117783779 / 159.99>133.99 / FineBI=否 / 严格=否`。
- `评分 / 评论数`：
  - FineBI `WF广告数据/广告` 添加列确认：从 `zzdnb_detailed_listing_health` 选择 `评分`、`评论数`，均按最大值；匹配键 `store = store`、`年月 = 年月`、`sku = Wayfair SKU`。
  - 源表字段确认：`zzdnb_detailed_listing_health` 字段列表保留英文原始字段 `Lifetime Review Count`、`Lifetime Average Customer Review Rating`，中文 `评论数/评分` 是广告自助表添加列/展示命名。
  - 数据库重复键样本：`WF-TD / 2026-03 / OARI1153` 有 `192` 行，最大 `评论数=375`、最大 `评分=4.54`，验证最大值去重合理，不能明细求和。
- `其他广告组汇总数据`：
  - 看板页签在 `日期>=2026-01-01`、`店铺=WF-TD` 下显示一行汇总，核心值为 `impressions=3332`、`clicks=115`、`spend=34.64`、`WSC=0`、`总利润=-34.64`。
  - 数据库按 `advertising_product_report_by_day.class_name -> wf_class中文分类.分类` 聚合确认：`分类='其他'` 仅 `Class Name=Multichannel-Only (MCO) Fulfillment`，`24` 行，汇总值与看板一致。
  - 状态：当前报表可按 `分类='其他'` 落地；组件编辑过滤弹窗仍未打开，页面配置层待核实。

#### 2026-06-25 剩余 3 项只读补核

- 范围：只核实 Wayfair WSP 广告报表剩余 FineBI 问题；不重复核实 bid、WSC、总销售额、评分评论、分类来源、活动垂直标签。
- `总成本 / 总利润` 2026 同期样本：
  - FineBI 路径：`WF店铺绩效2.0/广告`。
  - 可见节点/字段：`单买数量`、`总成本`、`总利润`、`每个sku平均成本`、`ddp平均成本`、`每个上架skuddp`、`每个上架sku采购sku数量`、`每个采购skuddp`。
  - 可见公式：`总利润 = (attributed_wholesale_cost_window_view_through_USD_Day_14*IF(store="WF-FOP",0.93,0.94))-总成本`。
  - 状态：待核实。预览层仍只显示部分数据，未取得 `2026-01` 至 `2026-05` 且包含成本利润全字段的 FineBI 已计算样本；数据库只读连接本轮超时，未完成辅助对账。
- `本竞品价格对比 / 是否有价格优势`：
  - FineBI 路径：`WF广告数据/广告`，来源字段链路为 `本竞品价格对比`。
  - 可见公式：`是否有价格优势 = IF(ISNULL(竞品最低前台价),null,IF(竞品最低前台价<本品最低前台价,"否","是"))`。按公式推断，等价时返回 `是`，但本轮未拿到 FineBI 等价样本。
  - 可见样本：前 50 页预览中，本品低于竞品样本包括 `UXIF3413 / 149.29 < 152.99 / 是`、`UXIF3408 / 108.07 < 139.99 / 是`、`UXIF3107 / 289.98 < 319.99 / 是`；本品高于竞品样本包括 `W011205775 / 179.99 > 172.99 / 否`、`W009453364 / 309.99 > 279.98 / 否`。
  - 状态：待核实。`本竞品价格对比` 表打开后提示 `数据不活跃，暂不抽取`，未点击更新；当前页面未导出完整字段，也未完成与数据库 `产品近两天的最小前台价` 的逐行对账。
- `其他广告组汇总数据` 过滤：
  - FineBI 路径：`WF-WSP广告组广告分析 / 广告组活动 / 其他广告组汇总数据`。
  - 报表预览：当前筛选区间 `2026-01-01 - 无限制`、店铺 `WF-TD`，示例行 `2026-01` 曝光 `3,332`、点击 `115`、广告花费 `34.64`、广告销售额 `0`、广告利润 `-34.64`、广告 ROI `-1`。
  - 状态：待核实。尝试进入组件编辑/过滤配置时提示 `当前主题正在被刘壵杰(lzj)编辑`，只能进入预览状态；未能确认结果过滤是否只筛 `分类=其他`，也未能确认是否包含 `厨房卫浴`、空分类或未匹配分类。

#### 2026-06-25 继续核实补充

- `总成本 / 总利润` 2026 同期样本：
  - 状态：已确认。
  - FineBI 路径：`WF广告数据 / 广告` 数据预览。`WF店铺绩效2.0 / 广告` view 预览第 `50/50` 页仍停在 `2024-04-18~2024-05-01`，但 `WF广告数据 / 广告` 可见 2026 样本。
  - 字段：`WSC` 对应 `attributed_wholesale_cost_window_view_through_USD_Day_14`；同时可见 `store/date_est/campaign_id/campaign_name/sku/extend/product_max_bid/spend_USD/attributed_units_window_view_through_Day_14/每个sku平均成本/单买数量/总成本/总利润`。
  - 复算样例：`WF-TD / 2026-05-22 / CH0033 / UXIF3675`，`spend_USD=27.84`、`attributed_units=1`、`每个sku平均成本=104.36`，`总成本=132.20`；`WSC=148.80`、系数 `0.94`，`总利润=7.67`，与 FineBI 一致。
  - 样本覆盖：本轮抽到 `2026-01-15`、`2026-02-02`、`2026-02-13`、`2026-02-15`、`2026-02-25`、`2026-02-28`、`2026-03-10`、`2026-05-20`、`2026-05-22`、`2026-05-26` 等 10 行。
- `本竞品价格对比 / 是否有价格优势`：
  - 状态：有差异 / 待核实。FineBI `广告` 表页面样本已覆盖低于、等于、高于三类；数据库 `产品近两天的最小前台价` 逐行对账本轮连接超时，仍待补。
  - FineBI 等价样本：`WF-TD / model=LS0009 / Display_SKUs=OARI1791 / 本品最低前台价=269.99 / 竞品最低前台价=269.99 / 是否有价格优势=是`，确认 FineBI 原式等价返回 `是`。
  - 低于样本：`WF-TD / CH0036 / W111675914 / 169.99 < 299.99 / 是`、`WF-TD / 53333 / W111760112 / 86.99 < 111.99 / 是`。
  - 高于样本：`WF-FOP / TV STAND-01 / W112178227 / 137.99 > 67.99 / 否`、`WF-TD / SF0003 / W112285361 / 310 > 279.98 / 否`、`WF-FOP / BARST-2SET / EVYN4068 / 159.99 > 139.99 / 否`。
  - 限制：`WF广告数据 / 广告` 字段搜索未找到 `link_sku`、`平台sku`、`website`、`更新日期`；`本竞品价格对比` 表仍提示 `数据不活跃，暂不抽取`，未点击更新。
- `其他广告组汇总数据`：
  - 状态：已确认。
  - 配置路径：`WF-WSP广告组广告分析 / 广告组活动`，选中 `其他广告组汇总数据` 组件，打开浮动编辑入口。
  - 结果过滤：`为 广告.分类 添加过滤条件`，`分类 属于 固定值 其他`，固定值数量 `1`。
  - 结论：只筛 `分类=其他`，不包含 `厨房卫浴`、空分类或未匹配分类。

## 4. 指标口径待核对清单

- 广告基础指标：`广告花费`、`广告销售额`、`广告订单量`、`广告件数`、`广告点击`、`广告曝光`
- 自然与总量指标：`自然销售额`、`自然订单量`、`自然销量`、`总销售额`、`总订单量`、`总销量`
- 比率指标：`广告CPC`、`广告CTR`、`广告CVR`、`广告CR`、`广告CPM`、`广告CPA`、`广告花费占比`、`自然CVR`、`自然CR`
- 平台原生指标：`ROAS`、`ACoS`、`CPC`、`CTR`、`CVR`、`Retail ROAS`、`WSC ROAS`
- 利润相关指标：`总成本`、`总利润`、`最小利润`、`平均利润`、`最大利润`
- 广告销售额口径：Retail sales 与 WSC/wholesale cost 不能混用。
- 汇总比率口径：ROAS、ACoS、CPC、CTR、CVR、CPA 等应在汇总层级按分子分母重算，不直接平均明细比率。
- Campaign 主键：优先记录 `campaign_id`；如果页面只展示 `campaign_name`，需标注名称变更风险。

## 5. 待继续核实

### 看板到数据集

- `WF-WSP整体广告分析` 是否直接使用 `总广告数据`，还是经过自助数据集二次计算。
- `WF-WSP分类广告分析` 是否直接使用 `分类目广告数据`、分类组件公式和底层 SQL；`分类` 字段来源已确认由 `class_name -> wf_class中文分类.分类` 映射生成。
- `WF-WSP广告组广告分析` 的所有组件对应数据集，尤其 `广告组活动明细表`、`全部广告组广告分析组合图1`。
- `wf广告` 的组件级数据集和 SQL，重点确认滞销、清仓、可用字段来自哪个广告自助数据集。
- `wf新广告表现` 中 Enhanced Attribution、NTW/NTB 的数据集来源。
- `OS广告` 是否使用 `os广告重新计算`、`os广告2`，还是另有自助数据集承接。

### 数据集与字段

- `总广告数据_广告组数据`、`分类目广告数据_广告组数据` 的自助数据集公式、字段重命名和引用关系。
- `zzdnb_wf广告出价表` 的完整字段、下游节点名称、应用关系。
- `WSP广告组每月汇总` 组件维度 `bid` 已确认映射到 `广告.product_max_bid`。2026-06-19 外网 FineBI 复核 `广告组bid趋势` 组件字段，维度为 `store`、`date_est`、`14天周期`、`sku`、`model`、`活动垂直标签`、`垂直标签`，指标为 `Bid adjustment applied on B2B bids`、`Default bid(现在的bid)`、`Suggested Bid detail(具体金额)`、`最小建议bid`、`最大建议bid`；因此该组件展示当前/建议 bid 快照字段，不是历史效果分组用的 `product_max_bid`。
- `zzdnb_hd广告` 更新信息页无记录的原因，以及它在 FineBI 看板中的实际应用关系。
- `滞销品列表`、`仓库销量可用预估2` 是否被广告看板引用；只确认引用关系，不重复梳理库存链路。

### SQL 与公式

- `总广告数据`、`分类目广告数据` 编辑页当前只见字段选择列表，未看到可复制 SQL；如后续 FineBI 仍不展示 SQL，需持续标注为“FineBI 未展示 SQL”。
- `总成本`、`总利润`、`最小利润`、`平均利润`、`最大利润` 的公式来源。
- `广告花费占比` 的分母：总销售额、广告销售额或其他销售口径。

### 2026-07-14 上架 SKU 利润权限边界与新映射表检索

- FineBI 公共抽取表 `公共数据/王威/上架sku利润` 可只读预览，当前显示 `532,830` 行；字段搜索未找到 `扣除平均仓库仓储费的利润率`，说明该字段不是这张公共抽取表的直接输出列。
- 血缘图显示该公共抽取表下游存在自助数据集 `上架sku利润`，位置为 `wangwei的分析/上架sku利润/店铺sku利润`，最近更新时间为 2026-07-14 15:17:22；当前账号打开后明确显示“当前表无权限”。
- 因权限不足，本轮没有进入编辑态、没有点击更新，也没有取得目标利润率的公式；该字段来源继续标注 `待核实`，不能根据数据库表名反推。
- FineBI 公共数据全局搜索 `advertising_product_report_mapped` 返回“无匹配项”。这只表示当前公共数据区未见同名入口，不能排除私有数据集、别名数据集或库外应用。
- 数据库侧已确认该新表会拆行复制广告指标；即使后续接入 FineBI，也必须先确定采购 SKU 分摊 / 去重规则，不能直接求和。
