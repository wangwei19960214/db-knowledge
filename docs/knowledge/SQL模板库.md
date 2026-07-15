# SQL 模板库

本文件只保存可复用 SQL 骨架。第一版不连接数据库、不校验真实字段；不确定字段统一写 `-- TODO: 待核实`。

## 使用约定

- `:start_date`、`:end_date`：日期参数。
- `:month_start`、`:month_end`：月份参数。
- `:store_list`：店铺列表。
- `:sku_list`：SKU 列表。
- `:platform`：平台。
- 汇总比率必须使用分子分母重算，不直接平均明细比率。

## SQL-WF-AD-MONTHLY：Wayfair 广告组月度表现骨架

```sql
WITH ad_daily AS (
  SELECT
    DATE_FORMAT(report_date, '%Y-%m') AS month,
    store,
    campaign_id,
    campaign_name,
    wayfair_sku,
    SUM(spend) AS spend,
    SUM(wsc_sales) AS wsc_sales,
    SUM(orders) AS orders,
    SUM(units) AS units,
    SUM(clicks) AS clicks,
    SUM(impressions) AS impressions
    -- TODO: 待核实：真实日期字段、WSC 字段、订单字段、销量字段名称
  FROM advertising_product_report_by_day
  WHERE report_date >= :start_date
    AND report_date < :end_date
    -- TODO: 待核实：WSP 过滤字段
    -- AND store IN (:store_list)
  GROUP BY
    DATE_FORMAT(report_date, '%Y-%m'),
    store,
    campaign_id,
    campaign_name,
    wayfair_sku
)
SELECT
  month,
  store,
  campaign_id,
  campaign_name,
  wayfair_sku,
  spend,
  wsc_sales,
  orders,
  units,
  clicks,
  impressions,
  CASE WHEN impressions = 0 THEN NULL ELSE clicks / impressions END AS ctr,
  CASE WHEN clicks = 0 THEN NULL ELSE orders / clicks END AS cvr,
  CASE WHEN clicks = 0 THEN NULL ELSE spend / clicks END AS cpc,
  CASE WHEN impressions = 0 THEN NULL ELSE spend * 1000 / impressions END AS cpm,
  CASE WHEN orders = 0 THEN NULL ELSE spend / orders END AS cpa,
  CASE WHEN spend = 0 THEN NULL ELSE wsc_sales / spend END AS roas
  -- TODO: 待核实：total_profit 来源后再计算 roi = total_profit / spend
FROM ad_daily;
```

## SQL-WF-AVAILABILITY：Wayfair 有货率骨架

```sql
WITH listing_scope AS (
  SELECT
    store,
    wayfair_sku,
    purchase_sku,
    listing_start_date,
    listing_end_date
    -- TODO: 待核实：真实字段名
  FROM 平台产品详情表
  WHERE platform = 'Wayfair'
),
warehouse_country AS (
  SELECT
    `后台仓库名称` AS warehouse,
    MAX(`所在国家`) AS country,
    MAX(`是否停用`) AS warehouse_status
  FROM 仓库映射表
  GROUP BY `后台仓库名称`
),
stock_daily AS (
  SELECT
    s.date AS stock_date,
    s.`仓库` AS warehouse,
    w.country,
    s.sku AS purchase_sku,
    SUM(s.`可用`) AS available_qty
  FROM warehouse_stock_在售 s
  LEFT JOIN warehouse_country w
    ON s.`仓库` = w.warehouse
  WHERE s.date >= :start_date
    AND s.date < :end_date
    AND (w.warehouse_status IS NULL OR w.warehouse_status = '正常')
  GROUP BY s.date, s.`仓库`, w.country, s.sku
)
SELECT
  l.store,
  l.wayfair_sku,
  COUNT(*) AS active_days,
  SUM(CASE WHEN s.available_qty > 0 THEN 1 ELSE 0 END) AS available_days,
  CASE WHEN COUNT(*) = 0 THEN NULL
       ELSE SUM(CASE WHEN s.available_qty > 0 THEN 1 ELSE 0 END) / COUNT(*)
  END AS availability_rate
FROM listing_scope l
LEFT JOIN stock_daily s
  ON s.purchase_sku = l.purchase_sku
WHERE s.stock_date >= l.listing_start_date
  AND (l.listing_end_date IS NULL OR s.stock_date < l.listing_end_date)
GROUP BY l.store, l.wayfair_sku;
```

## SQL-WF-AVAILABILITY-MAPPING-PRECHECK：Wayfair 有货率映射预检骨架

适用：正式计算 Wayfair 上架 SKU / 链接 SKU 有货率前，先检查上架范围、专用映射、回退采购 SKU 和缺映射规模。

```sql
WITH map_keys AS (
  SELECT
    store,
    part_number,
    COUNT(*) AS component_count,
    SUM(CAST(`数量` AS DECIMAL(18,4))) AS quantity_sum
  FROM `wayfair上架sku销售映射`
  GROUP BY store, part_number
),
listing_scope AS (
  SELECT
    store,
    `上架sku` AS listing_sku,
    `平台sku` AS platform_sku,
    `链接sku` AS link_sku,
    `采购sku` AS fallback_purchase_sku,
    `上下架状态` AS listing_status,
    `上架日期` AS listing_start_date,
    `下架日期` AS listing_end_date
  FROM `平台产品详情表`
  WHERE platform = 'Wayfair'
)
SELECT
  COUNT(*) AS listing_rows,
  SUM(CASE WHEN listing_status = '上架' THEN 1 ELSE 0 END) AS active_listing_rows,
  SUM(CASE WHEN listing_status = '上架' AND m.part_number IS NOT NULL THEN 1 ELSE 0 END) AS active_rows_with_mapping,
  SUM(CASE WHEN listing_status = '上架' AND m.part_number IS NULL THEN 1 ELSE 0 END) AS active_rows_without_mapping,
  SUM(CASE WHEN listing_status = '上架'
            AND m.part_number IS NULL
            AND fallback_purchase_sku IS NOT NULL
            AND TRIM(fallback_purchase_sku) <> ''
           THEN 1 ELSE 0 END) AS active_rows_can_fallback,
  SUM(CASE WHEN listing_status = '上架'
            AND m.part_number IS NULL
            AND (fallback_purchase_sku IS NULL OR TRIM(fallback_purchase_sku) = '')
           THEN 1 ELSE 0 END) AS active_rows_missing_purchase_sku,
  SUM(CASE WHEN listing_status = '上架'
            AND (link_sku IS NULL OR TRIM(link_sku) = '' OR TRIM(link_sku) = '无')
           THEN 1 ELSE 0 END) AS active_rows_invalid_link_sku,
  SUM(CASE WHEN listing_status = '上架'
            AND m.part_number IS NOT NULL
            AND m.quantity_sum <> m.component_count
           THEN 1 ELSE 0 END) AS active_rows_quantity_not_equal_component_count
FROM listing_scope l
LEFT JOIN map_keys m
  ON l.store = m.store
 AND l.listing_sku = m.part_number;
```

注意：`wayfair上架sku销售映射.数量` 当前不能简单等同为组件个数；有货判断仍按采购 SKU 组件逐个判断。链接 SKU 聚合前需过滤 `NULL`、空字符串和 `无`。

## SQL-SKU-SALES-BASIC：SKU 销量查询骨架

2026-07-02 复核：`order_details_tb` 已确认可作为通用订单销售明细主表。日期优先 `订单日期`，销量用 `数量`，销售额用 `总销售额`，SKU 用 `sku修`。不要默认把 `总销售额` 重算为 `数量 * 售价`；本表未见取消 / 退款字段。

```sql
SELECT
  `订单日期` AS order_date,
  `出单平台` AS platform,
  `出单店铺修` AS store,
  `sku修` AS sku,
  SUM(`数量`) AS units,
  SUM(`总销售额`) AS sales_amount
FROM `order_details_tb`
WHERE `订单日期` >= :start_date
  AND `订单日期` < :end_date
  AND `sku修` IN (:sku_list)
GROUP BY `订单日期`, `出单平台`, `出单店铺修`, `sku修`;
```

注意：

- 若需要补 `订单日期` 为空或早于 2023-12-01 的历史记录，可参考 `order_details_tb_tj` 的规则：`CASE WHEN 订单日期 IS NULL OR 订单日期 < '2023-12-01' THEN 做单日2 ELSE 订单日期 END`。
- 净销量、取消订单、退款扣减仍为 `待核实`；`order_details_tb` 本身未见显式取消 / 退款字段。

## SQL-SKU-INVENTORY-BASIC：SKU 库存查询骨架

```sql
SELECT
  date AS stock_date,
  `仓库` AS warehouse,
  sku,
  SUM(`可用`) AS available_qty,
  SUM(`待出库`) AS pending_out_qty,
  SUM(`实际标品库存`) AS actual_standard_stock_qty,
  SUM(`在途`) AS inbound_qty
FROM warehouse_stock_在售
WHERE date = :stock_date
  AND sku IN (:sku_list)
GROUP BY date, `仓库`, sku;
```

## SQL-SKU-INVENTORY-CURRENT：当前库存查询骨架

适用：用户问“现在库存”“当前可用库存”时，优先查当前快照表。输入若是 Model、平台 SKU、链接 SKU 或上架 SKU，必须先映射到采购 SKU。

```sql
SELECT
  date AS stock_date,
  `仓库` AS warehouse,
  sku,
  SUM(`可用`) AS available_qty,
  SUM(`待出库`) AS pending_out_qty,
  SUM(`实际标品库存`) AS actual_standard_stock_qty,
  SUM(`在途`) AS inbound_qty
FROM warehouse_stock_最新
WHERE sku IN (:sku_list)
GROUP BY date, `仓库`, sku
ORDER BY date, `仓库`, sku;
```

## SQL-SKU-INVENTORY-HISTORICAL-DAILY：历史每日库存查询骨架

适用：用户问“某 SKU 某月每天库存”“历史库存趋势”时，查历史库存事实表；如果用户输入的是 Model，需要先展开采购 SKU。

```sql
SELECT
  date AS stock_date,
  `仓库` AS warehouse,
  sku,
  SUM(`可用`) AS available_qty,
  SUM(`待出库`) AS pending_out_qty,
  SUM(`实际标品库存`) AS actual_standard_stock_qty,
  SUM(`在途`) AS inbound_qty
FROM warehouse_stock_在售
WHERE date >= :start_date
  AND date < :end_date
  AND sku IN (:sku_list)
GROUP BY date, `仓库`, sku
ORDER BY date, `仓库`, sku;
```

## SQL-SKU-INVENTORY-SPLIT-DAILY：库存拆分日查询骨架

适用：用户问 CG 仓、其他仓、AMZ、垂直库存拆分时使用；该表是 `sku + date` 粒度，没有原始 `仓库` 字段。

```sql
SELECT
  date AS stock_date,
  sku,
  SUM(`总可用库存`) AS total_available_qty,
  SUM(`其他仓库可用库存`) AS other_available_qty,
  SUM(`cg仓可用库存`) AS cg_available_qty,
  SUM(`amz可用库存`) AS amz_available_qty,
  SUM(`垂直可用库存`) AS vertical_available_qty,
  SUM(`总入库数量`) AS inbound_qty,
  SUM(`总出库数量`) AS outbound_qty
FROM `库存拆分记录`
WHERE date >= :start_date
  AND date < :end_date
  AND sku IN (:sku_list)
GROUP BY date, sku
ORDER BY date, sku;
```

注意：`库存拆分记录` 已发现少量 `总可用库存 < 0`，正式输出需保留异常提示或另做异常清单；是否过滤负库存由业务口径决定，当前标 `待核实`。

## SQL-MODEL-SALES-BASIC：Model 层级销售骨架

```sql
WITH sku_model AS (
  SELECT
    `采购sku` AS purchase_sku,
    model
  FROM `采购sku_model映射表`
)
SELECT
  o.`出单平台` AS platform,
  o.`出单店铺修` AS store,
  m.model,
  SUM(o.`数量`) AS units,
  SUM(o.`总销售额`) AS sales_amount
FROM `order_details_tb` o
LEFT JOIN sku_model m
  ON UPPER(TRIM(o.`sku修`)) = UPPER(TRIM(m.purchase_sku))
WHERE o.`订单日期` >= :start_date
  AND o.`订单日期` < :end_date
GROUP BY o.`出单平台`, o.`出单店铺修`, m.model;
```

注意：Model 映射中非标准 model、空 model 和一对多聚合规则仍需按 `Q-MAP-001` 继续复核；利润字段不在本模板中计算。

## SQL-AD-PROFIT-MERGE：广告与总销售额 / 利润合并骨架

适用：Wayfair WSP 广告月度表现、总销售额回补和利润字段预留。广告事实字段已确认，利润字段来源仍需 FineBI 已计算样本或预聚合成本链路复算。

```sql
WITH ad AS (
  SELECT
    LEFT(date_est, 7) AS month,
    store,
    campaign_id,
    campaign_name,
    sku AS wayfair_sku,
    product_max_bid,
    isB2b,
    SUM(CAST(NULLIF(spend_USD, '') AS DECIMAL(18,4))) AS spend_usd,
    SUM(CAST(NULLIF(attributed_wholesale_cost_window_view_through_USD_Day_14, '') AS DECIMAL(18,4))) AS wsc_14,
    SUM(CAST(NULLIF(attributed_retail_sales_window_view_through_USD_Day_14, '') AS DECIMAL(18,4))) AS retail_14,
    SUM(CAST(NULLIF(attributed_orders_window_view_through_Day_14, '') AS DECIMAL(18,4))) AS orders_14,
    SUM(CAST(NULLIF(attributed_units_window_view_through_Day_14, '') AS DECIMAL(18,4))) AS units_14
  FROM advertising_product_report_by_day
  WHERE date_est >= :start_date
    AND date_est < :end_date
  GROUP BY
    LEFT(date_est, 7),
    store,
    campaign_id,
    campaign_name,
    sku,
    product_max_bid,
    isB2b
),
sales AS (
  SELECT
    LEFT(`日期`, 7) AS month,
    store,
    wayfair_sku,
    SUM(CAST(NULLIF(total_revenue, '') AS DECIMAL(18,4))) AS total_revenue,
    SUM(units_sold) AS units_sold
  FROM `wf上架sku每日销售额`
  WHERE `日期` >= :start_date
    AND `日期` < :end_date
  GROUP BY LEFT(`日期`, 7), store, wayfair_sku
),
profit AS (
  SELECT
    month,
    store,
    campaign_id,
    wayfair_sku,
    product_max_bid,
    isB2b,
    total_cost,
    total_profit
    -- TODO: 待核实：总成本、总利润物理来源。
    -- 当前不得从 advertising_product_report_by_day、wf上架sku每日销售额 或 上架sku利润表 直接推断。
  FROM profit_source_to_confirm_after_finebi_or_preagg_qa
)
SELECT
  ad.month,
  ad.store,
  ad.campaign_id,
  ad.campaign_name,
  ad.wayfair_sku,
  ad.product_max_bid,
  ad.isB2b,
  ad.spend_usd,
  ad.wsc_14,
  ad.retail_14,
  ad.orders_14,
  ad.units_14,
  sales.total_revenue,
  sales.units_sold,
  profit.total_cost,
  profit.total_profit,
  CASE WHEN ad.spend_usd = 0 THEN NULL ELSE ad.wsc_14 / ad.spend_usd END AS roas_wsc,
  CASE WHEN ad.spend_usd = 0 THEN NULL ELSE profit.total_profit / ad.spend_usd END AS roi
FROM ad
LEFT JOIN sales
  ON ad.month = sales.month
 AND ad.store = sales.store
 AND ad.wayfair_sku = sales.wayfair_sku
LEFT JOIN profit
  ON ad.month = profit.month
 AND ad.store = profit.store
 AND ad.campaign_id = profit.campaign_id
 AND ad.wayfair_sku = profit.wayfair_sku
 AND ad.product_max_bid = profit.product_max_bid
 AND ad.isB2b = profit.isB2b;
```

注意：

- `advertising_product_report_by_day` 的核心数值字段是字符串，汇总前要 `CAST(NULLIF(...,'') AS DECIMAL(...))`。
- `product_max_bid` 是历史广告事实 bid，不是 `wf广告出价表.Default bid(现在的bid)`。
- `wf上架sku每日销售额` 只确认了销售额和销量，`total_revenue` 已在过程写入时对 `CAD/GBP` 按最新汇率折算；总成本 / 总利润不能从该表字段名推断。

## SQL-WF-AD-BID-SAMPLE-PRECHECK：Wayfair 广告 BID 样本匹配预检

适用：核对当前 / 建议 bid 是否能补到广告事实。全量覆盖率查询较慢时，先按目标日期或小样本执行；正式报表需输出未匹配清单。

```sql
SELECT
  a.store,
  a.date_est,
  a.campaign_id,
  a.campaign_name,
  a.sku,
  a.product_max_bid,
  b.`Default bid(现在的bid)` AS current_default_bid,
  b.`Suggested Bid detail(具体金额)` AS suggested_bid_detail,
  b.`Suggested Bid（金额范围)` AS suggested_bid_range,
  b.`Bid adjustment applied on B2B bids` AS b2b_bid_adjustment,
  CASE WHEN b.campaignlayer_id IS NULL THEN '未匹配' ELSE '已匹配' END AS bid_match_status
FROM advertising_product_report_by_day a
LEFT JOIN `wf广告出价表` b
  ON b.store = a.store
 AND b.`日期` = a.date_est
 AND b.`wayfair SKU` = a.sku
 AND b.campaignlayer_id = a.campaign_id
WHERE a.date_est = :target_date
  AND a.store = :store
ORDER BY a.campaign_id, a.sku
LIMIT 200;
```

注意：`Campaign name` 只作展示或兜底。2026-07-01 样本验证中，存在按 `campaignlayer_id` 可匹配但按名称不匹配的记录。

## SQL-WF-AD-CLASS-COVERAGE-PRECHECK：Wayfair 广告中文分类覆盖预检

适用：按中文分类汇总广告数据前，检查 `class_name` 是否全部命中 `wf_class中文分类`。

```sql
SELECT
  COALESCE(c.`分类`, '未匹配') AS category_cn,
  COUNT(*) AS rows_count,
  SUM(CAST(NULLIF(a.spend_USD, '') AS DECIMAL(18,4))) AS spend_usd,
  SUM(CAST(NULLIF(a.attributed_wholesale_cost_window_view_through_USD_Day_14, '') AS DECIMAL(18,4))) AS wsc_14
FROM advertising_product_report_by_day a
LEFT JOIN `wf_class中文分类` c
  ON a.class_name = c.`Class Name`
WHERE a.date_est >= :start_date
  AND a.date_est < :end_date
GROUP BY COALESCE(c.`分类`, '未匹配')
ORDER BY rows_count DESC;
```

注意：未匹配 `class_name` 应输出异常清单，不默认归为 `其他`。

## SQL-STOCKOUT-RISK-BASIC：断货风险骨架

适用：用户问“某采购 SKU / Model 未来是否断货”“未来几个月可用风险”时使用。输入为 Model 时必须先展开到采购 SKU。当前预测事实表未见 store / 国家 / 仓库字段，按采购 SKU 全局预测解释。

```sql
SELECT
  date AS forecast_date,
  sku,
  `可用` AS available_qty,
  `预估可用` AS forecast_available_qty,
  `日均销量` AS avg_daily_sales_qty,
  `销量` AS actual_sales_qty,
  `断货日期` AS stockout_date,
  `到仓数量` AS inbound_qty,
  `计划批次号` AS inbound_batch_no
FROM tb_kyxl_forecast_and_reality
WHERE date >= :start_date
  AND date < :end_date
  AND sku IN (:sku_list)
  AND (`可用` = 0 OR `预估可用` = 0 OR `断货日期` IS NOT NULL)
ORDER BY date, sku;
```

注意：未来预测区间常见 `可用` 为 `NULL`、`预估可用` 有值；不要把未来 `可用=NULL` 强制补 0。需要排除 / 标注放弃 SKU 时，再左连 `考虑放弃sku清单`，链接 SKU / 组合 SKU 场景不要静默排除。

## SQL-FINEBI-RECON：FineBI 口径复核骨架

```sql
-- 先从 FineBI 文档确认：看板名称、组件名称、数据集名称、过滤器、公式字段。
-- TODO: 待核实：不要在未确认数据集和过滤器前直接写最终 SQL。

SELECT
  /* dimensions */,
  /* numerator */,
  /* denominator */
FROM /* dataset_or_table */
WHERE /* same filters as FineBI */
GROUP BY /* same dimensions as FineBI */;
```

## SQL-SKU-MODEL-PLATFORM-MAP：SKU / Model / 平台映射骨架

```sql
WITH input_token AS (
  SELECT :keyword AS keyword
),
model_hit AS (
  SELECT
    'model' AS hit_type,
    NULL AS platform,
    NULL AS store,
    m.model,
    NULL AS link_sku,
    NULL AS platform_sku,
    NULL AS listing_sku,
    NULL AS product_sku,
    m.`采购sku` AS purchase_sku,
    NULL AS component_qty,
    NULL AS purchase_sku_text,
    NULL AS listing_status,
    NULL AS product_status
  FROM `采购sku_model映射表` m
  JOIN input_token i
    ON UPPER(m.model) = UPPER(i.keyword)
),
platform_detail_hit AS (
  SELECT
    'platform_detail' AS hit_type,
    p.platform,
    p.store,
    p.model,
    p.`链接sku` AS link_sku,
    p.`平台sku` AS platform_sku,
    p.`上架sku` AS listing_sku,
    NULL AS product_sku,
    NULL AS purchase_sku,
    NULL AS component_qty,
    p.`采购sku` AS purchase_sku_text,
    p.`上下架状态` AS listing_status,
    p.`产品状态` AS product_status
  FROM `平台产品详情表` p
  JOIN input_token i
    ON UPPER(p.model) = UPPER(i.keyword)
    OR UPPER(p.`采购sku`) = UPPER(i.keyword)
    OR UPPER(p.`上架sku`) = UPPER(i.keyword)
    OR UPPER(p.`平台sku`) = UPPER(i.keyword)
    OR UPPER(p.`链接sku`) = UPPER(i.keyword)
  WHERE (:platform IS NULL OR p.platform = :platform)
),
wayfair_component_hit AS (
  SELECT
    'wayfair_listing_component' AS hit_type,
    'Wayfair' AS platform,
    w.store,
    NULL AS model,
    NULL AS link_sku,
    NULL AS platform_sku,
    w.part_number AS listing_sku,
    w.product_sku,
    w.`采购sku` AS purchase_sku,
    w.`数量` AS component_qty,
    NULL AS purchase_sku_text,
    NULL AS listing_status,
    w.product_status
  FROM `wayfair上架sku销售映射` w
  JOIN input_token i
    ON UPPER(w.part_number) = UPPER(i.keyword)
    OR UPPER(w.product_sku) = UPPER(i.keyword)
    OR UPPER(w.`采购sku`) = UPPER(i.keyword)
)
SELECT * FROM model_hit
UNION ALL
SELECT * FROM platform_detail_hit
UNION ALL
SELECT * FROM wayfair_component_hit;

-- TODO: 待核实：
-- 1. 实际执行时可先按 `hit_type` 展示命中来源，再由业务场景决定继续查库存、销售或广告。
-- 2. `平台产品详情表.采购sku` 可能包含逗号分隔多个采购 SKU，正式 SQL 需先拆分。
-- 3. Wayfair 上架 SKU 到采购 SKU 组件优先用 `wayfair上架sku销售映射`；
--    该表的 `part_number` 对应 `平台产品详情表.上架sku`，不是 `平台sku` 或 `链接sku`。
-- 4. Model 查询优先走 `采购sku_model映射表`；FineBI 的 `采购sku_model映射_2`
--    是库存预估链路中的数据集名称，当前业务库元数据未见同名物理表。
```

## SQL-WF-INVENTORY-THEME-AVAILABILITY-FIRST-PASS：库存主题 / Wayfair 有货率第一轮骨架

```sql
WITH listing_scope AS (
  SELECT
    p.store,
    p.link_sku,
    p.platform_sku AS listing_sku,
    COALESCE(m.`采购sku`, p.`采购sku`) AS purchase_sku,
    COALESCE(m.`数量`, 1) AS component_qty,
    p.listing_start_date,
    p.listing_end_date
    -- TODO: 待核实：平台产品详情表真实上架 SKU、链接 SKU、上架/下架日期字段名
    -- 已确认：wayfair上架sku销售映射存在一对多组件映射；2026-06-29 用户确认数量是销售个数。
  FROM 平台产品详情表 p
  LEFT JOIN wayfair上架sku销售映射 m
    ON p.store = m.store
   AND p.platform_sku = m.part_number
  WHERE p.platform = 'Wayfair'
),
warehouse_country AS (
  SELECT
    `后台仓库名称` AS warehouse,
    MAX(`所在国家`) AS country,
    MAX(`是否停用`) AS warehouse_status
  FROM 仓库映射表
  GROUP BY `后台仓库名称`
),
stock_by_country AS (
  SELECT
    s.date AS stock_date,
    s.sku AS purchase_sku,
    w.country,
    SUM(s.`可用`) AS available_qty
  FROM warehouse_stock_在售 s
  LEFT JOIN warehouse_country w
    ON s.`仓库` = w.warehouse
  WHERE s.date >= :start_date
    AND s.date < :end_date
    AND (w.warehouse_status IS NULL OR w.warehouse_status = '正常')
  GROUP BY s.date, s.sku, w.country
),
listing_daily AS (
  SELECT
    l.store,
    l.link_sku,
    l.listing_sku,
    d.stock_date,
    MIN(CASE WHEN COALESCE(d.available_qty, 0) > 0 THEN 1 ELSE 0 END) AS is_available
    -- TODO: 待核实：缺失库存日是否向前取最近有效库存；此处仅为第一轮骨架
  FROM listing_scope l
  LEFT JOIN stock_by_country d
    ON d.purchase_sku = l.purchase_sku
   AND d.country = CASE
     WHEN l.store IN ('WF-TD', 'WF-FOP') THEN '美国'
     WHEN l.store IN ('WF-TD-CA', 'WF-FOP-CA') THEN '加拿大'
     WHEN l.store = 'WF-TD-UK' THEN '英国'
     ELSE 'TODO: 待核实'
   END
  WHERE d.stock_date >= l.listing_start_date
    AND (l.listing_end_date IS NULL OR d.stock_date < l.listing_end_date)
  GROUP BY l.store, l.link_sku, l.listing_sku, d.stock_date
)
SELECT
  store,
  listing_sku,
  COUNT(*) AS active_days,
  SUM(is_available) AS available_days,
  CASE WHEN COUNT(*) = 0 THEN NULL ELSE SUM(is_available) / COUNT(*) END AS listing_availability_rate
FROM listing_daily
GROUP BY store, listing_sku;
```

## SQL-WF-CG-INVENTORY-FIRST-PASS：CG 仓库存骨架

```sql
SELECT
  b.`采购sku` AS sku,
  a.part_number AS part,
  a.in_stock AS actual_stock,
  a.available AS available_qty,
  a.on_order AS on_order_qty,
  CONCAT(b.store, '_CG') AS warehouse,
  DATE_FORMAT(a.create_time, '%Y-%m-%d') AS stock_date,
  a.warehouse AS source_warehouse,
  a.status
FROM wayfair后台cg可用 a
LEFT JOIN cg_part_sku映射 b
  ON a.store = b.store
 AND a.part_number = b.`上架sku`
WHERE DATE_FORMAT(a.create_time, '%Y-%m-%d') = (
  SELECT DATE_FORMAT(MAX(create_time), '%Y-%m-%d')
  FROM wayfair后台cg可用
);
```

注意：

- `wayfair后台cg可用` 源表粒度接近 `store + part_number + warehouse + 日期`，不是 `store + part_number`。
- `cg_part_sku映射` 可能一条 `store + 上架sku` 对多个采购 SKU；如果按采购 SKU 展开后再汇总库存，会产生行数扩展。只看 part / 店铺 / 源仓库库存时，应先在源表粒度聚合，或明确使用组件展开口径。
- `status` 是否过滤 `INACTIVE` 当前待核实；正式报表需输出状态分布或与 FineBI 对账后固定。

## SQL-WF-CG-INVENTORY-PRECHECK：CG 仓库存预检骨架

```sql
WITH latest AS (
  SELECT *
  FROM `wayfair后台cg可用`
  WHERE DATE(create_time) = (
    SELECT DATE(MAX(create_time))
    FROM `wayfair后台cg可用`
  )
),
map_one AS (
  SELECT
    store,
    `上架sku` AS part_number,
    COUNT(*) AS mapping_rows,
    COUNT(DISTINCT `采购sku`) AS purchase_sku_count
  FROM `cg_part_sku映射`
  GROUP BY store, `上架sku`
)
SELECT
  COUNT(*) AS source_rows,
  COUNT(DISTINCT latest.store) AS store_count,
  COUNT(DISTINCT latest.part_number) AS part_count,
  COUNT(DISTINCT latest.warehouse) AS source_warehouse_count,
  SUM(latest.available) AS source_available_qty,
  SUM(latest.on_order) AS source_on_order_qty,
  SUM(CASE WHEN latest.status = 'ACTIVE' THEN latest.available ELSE 0 END) AS active_available_qty,
  SUM(CASE WHEN latest.status <> 'ACTIVE' OR latest.status IS NULL THEN latest.available ELSE 0 END) AS non_active_available_qty,
  SUM(CASE WHEN map_one.part_number IS NULL THEN 1 ELSE 0 END) AS unmapped_rows,
  SUM(CASE WHEN map_one.part_number IS NULL THEN latest.available ELSE 0 END) AS unmapped_available_qty,
  SUM(CASE WHEN map_one.mapping_rows > 1 THEN 1 ELSE 0 END) AS rows_with_multi_purchase_sku_mapping
FROM latest
LEFT JOIN map_one
  ON latest.store = map_one.store
 AND latest.part_number = map_one.part_number;
```

## SQL-WF-CG-INVENTORY-RECONCILE-LATEST：CG 仓后台源与当前快照对账骨架

用途：当用户问“后台 CG 可用和当前库存快照为什么不一致”时，先用该骨架解释 `status`、未映射、90670 源仓和 `warehouse_stock_最新` 的差异。  
状态：2026-06-26 样本验证可用；生成血缘仍需待核实。

```sql
WITH latest AS (
  SELECT *
  FROM `wayfair后台cg可用`
  WHERE DATE(create_time) = (
    SELECT DATE(MAX(create_time))
    FROM `wayfair后台cg可用`
  )
),
map_one AS (
  SELECT
    store,
    `上架sku` AS part_number,
    COUNT(*) AS mapping_rows
  FROM `cg_part_sku映射`
  GROUP BY store, `上架sku`
),
source_active_mapped AS (
  SELECT
    l.store,
    SUM(l.available) AS active_mapped_available,
    SUM(CASE WHEN l.warehouse LIKE '%90670%' THEN l.available ELSE 0 END) AS active_mapped_90670_available,
    SUM(CASE WHEN l.warehouse NOT LIKE '%90670%' OR l.warehouse IS NULL THEN l.available ELSE 0 END) AS active_mapped_non_90670_available,
    SUM(l.on_order) AS active_mapped_on_order
  FROM latest l
  JOIN map_one m
    ON l.store = m.store
   AND l.part_number = m.part_number
  WHERE l.status = 'ACTIVE'
  GROUP BY l.store
),
warehouse_snapshot AS (
  SELECT
    REPLACE(`仓库`, '_CG', '') AS store,
    SUM(`可用`) AS snapshot_available,
    SUM(`在途`) AS snapshot_inbound
  FROM `warehouse_stock_最新`
  WHERE `仓库` LIKE '%_CG'
  GROUP BY REPLACE(`仓库`, '_CG', '')
)
SELECT
  s.store,
  s.active_mapped_available,
  s.active_mapped_90670_available,
  s.active_mapped_non_90670_available,
  w.snapshot_available,
  s.active_mapped_non_90670_available - w.snapshot_available AS non_90670_diff,
  s.active_mapped_on_order,
  w.snapshot_inbound,
  s.active_mapped_on_order - w.snapshot_inbound AS inbound_diff
FROM source_active_mapped s
LEFT JOIN warehouse_snapshot w
  ON s.store = w.store
ORDER BY s.store;
```

注意：

- `wayfair后台cg可用` 源口径、FineBI `cg仓库存` SQL 口径、`warehouse_stock_最新` 快照口径不是一回事。
- 当前样本显示 `warehouse_stock_最新` 更接近 `ACTIVE + 已映射 + 排除 90670 源仓`，但该结论必须继续标注为生成线索 / 待核实血缘。
- 90670 是否应归为 CG、其他仓或 shipout 需要业务确认。

## SQL-STOCKOUT-MONTHLY-FINEBI-FIRST-PASS：月度断货明细骨架

```sql
WITH sku_month AS (
  SELECT
    DATE_FORMAT(date, '%Y-%m') AS month,
    sku,
    SUM(CASE WHEN `可用` = 0 OR `预估可用` = 0 THEN 1 ELSE 0 END) AS stockout_days,
    MIN(CASE WHEN `可用` = 0 OR `预估可用` = 0 THEN date ELSE NULL END) AS stockout_start_date,
    MAX(CASE WHEN `可用` = 0 OR `预估可用` = 0 THEN date ELSE NULL END) AS stockout_end_date,
    ROUND(SUM(CASE WHEN `预估可用` = 0 THEN `日均销量` ELSE 0 END), 0) AS stockout_units_future,
    ROUND(SUM(CASE WHEN `可用` != 0 THEN `销量` ELSE 0 END), 0) AS non_stockout_units,
    SUM(CASE WHEN `可用` != 0 THEN 1 ELSE 0 END) AS non_stockout_days
  FROM tb_kyxl_forecast_and_reality
  WHERE date >= :start_date
    AND date < :end_date
  GROUP BY DATE_FORMAT(date, '%Y-%m'), sku
),
filtered AS (
  SELECT a.*
  FROM sku_month a
  LEFT JOIN 考虑放弃sku清单 b
    ON UPPER(a.sku) = UPPER(b.SKU)
  WHERE b.SKU IS NULL
)
SELECT
  month,
  sku,
  stockout_days,
  stockout_start_date,
  stockout_end_date,
  stockout_units_future,
  non_stockout_units,
  CASE
    WHEN non_stockout_units + stockout_units_future = 0 THEN 0
    ELSE non_stockout_units / (non_stockout_units + stockout_units_future)
  END AS availability_rate
  -- TODO: 待核实：当前月/历史月是否需要叠加三个月不断货销量估算的断货数量_3
FROM filtered;
```

注意：如果只回答整体月度 KPI，优先查数据库物理表 `断货sku数量` / `断货sku数量_合并`；FineBI 中的 `zzdnb_断货sku数量` 是数据集显示名。不要用月度 KPI 表反推 SKU 分日明细。

## SQL-SALES-COMBO-EXTRACT-BASIC：组合产品销量权重 / 总账单抽取骨架

```sql
WITH base AS (
  SELECT
    出单平台,
    出单店铺,
    PO,
    订单日期,
    上架sku,
    SKU AS 采购sku,
    数量 AS 采购sku数量,
    售价,
    售价 * 数量 AS 组件分摊销售额,
    sku总价,
    订单总价,
    订单状态,
    取消数量,
    订单来源
  FROM `总账单详情_extract`
  WHERE 订单日期 >= :start_date
    AND 订单日期 < :end_date
    -- TODO: 待业务确认：是否排除或扣减取消、拒绝、退货状态
    -- AND (订单状态 IS NULL OR 订单状态 NOT IN ('Cancelled', 'CANCELLED', 'Rejected', 'Returned'))
),
component_count AS (
  SELECT
    出单店铺,
    PO,
    上架sku,
    COUNT(DISTINCT 采购sku) AS 实际组件数
  FROM base
  GROUP BY 出单店铺, PO, 上架sku
),
component_sales AS (
  SELECT
    b.出单平台,
    b.出单店铺,
    b.订单日期,
    b.上架sku,
    b.采购sku,
    c.实际组件数,
    SUM(b.采购sku数量) AS 采购sku销量,
    SUM(b.组件分摊销售额) AS 组件分摊销售额,
    COUNT(DISTINCT b.PO) AS po数
  FROM base b
  JOIN component_count c
    ON b.出单店铺 = c.出单店铺
   AND b.PO = c.PO
   AND b.上架sku = c.上架sku
  GROUP BY
    b.出单平台,
    b.出单店铺,
    b.订单日期,
    b.上架sku,
    b.采购sku,
    c.实际组件数
)
SELECT *
FROM component_sales;
```

注意：

- 组件级金额用 `售价 * 数量`；不要直接汇总 `sku总价` 或 `订单总价`。
- 上架 SKU / 订单层金额应另建去重 CTE，按 `出单店铺 + PO + 上架sku` 保留一行。
- Wayfair 场景下，`总账单详情_extract.上架sku` 是订单子件 part number；`wf上架sku每日销售额.supplier_part` 是销售源 Supplier Part Number / 销售 SKU 层，可为 `Kit`、`Standard`、`Sellable Component` 或少量 `Nonsellable Component`。不要直接用 `supplier_part = 上架sku` 当同粒度关联；组合件分摊前需先识别 Product_Type 并通过 `Related_Kit_Part -> Supplier_Part_Number` 展开。
- 若要把 `wf上架sku每日销售额` 的组合件销售额分摊到子件 / 采购 SKU，需要先建立组合件 -> 子件展开口径，再用 `总账单详情_extract.SKU` 或 Wayfair 子件映射接采购 SKU。
- 与有货率结合时，仍需接库存事实表并按组合产品“任一组件缺货即断货”的规则判断。
- 2026-07-01 补充：`取消数量` 在少数组合件订单上会按组件行重复，不能直接逐行 `数量 - 取消数量`。有效销量建议先汇总到 `出单平台 + 出单店铺 + PO + 上架sku` 粒度确认订单上架 SKU 取消件数，再按映射数量或组件数量扣回；正式通用 SQL 仍需样本对账后固化，未确认处标 `TODO: 待核实`。

## SQL-SALES-EXTRACT-EFFECTIVE-QTY：OS / HD 组合件有效销量扣减骨架

适用：Overstock / Homedepot 需要从 `总账单详情_extract` 计算采购 SKU 有效销量、销量权重、断货补偿销量时使用。2026-07-01 已用 2026-06 样本验证：Homedepot 无取消影响；Overstock 直接逐行 `数量 - 取消数量` 会多算。

```sql
WITH platform_maps AS (
  SELECT
    'Overstock' AS platform,
    store,
    part_number,
    `采购sku` AS purchase_sku,
    CAST(`数量` AS DECIMAL(18,4)) AS map_qty
  FROM `overstock上架sku销售映射`
  UNION ALL
  SELECT
    'Homedepot' AS platform,
    store,
    part_number,
    `采购sku` AS purchase_sku,
    CAST(`数量` AS DECIMAL(18,4)) AS map_qty
  FROM `homedepot上架sku销售映射`
),
row_base AS (
  SELECT
    e.出单平台 AS platform,
    e.出单店铺 AS store,
    e.PO AS po,
    DATE(e.订单日期) AS sales_date,
    e.上架sku AS listing_sku,
    UPPER(TRIM(e.SKU)) AS purchase_sku,
    COALESCE(e.数量, 0) AS gross_component_qty,
    COALESCE(e.取消数量, 0) AS cancel_units_row,
    m.map_qty,
    CASE
      WHEN m.map_qty IS NULL OR m.map_qty = 0 THEN NULL
      ELSE COALESCE(e.数量, 0) / m.map_qty
    END AS listing_units_row,
    CASE
      WHEN m.purchase_sku IS NULL THEN 'MISSING_MAP'
      WHEN m.map_qty IS NULL OR m.map_qty = 0 THEN 'INVALID_MAP_QTY'
      ELSE 'OK'
    END AS map_status
  FROM `总账单详情_extract` e
  LEFT JOIN platform_maps m
    ON e.出单平台 = m.platform
   AND e.出单店铺 = m.store
   AND e.上架sku = m.part_number
   AND UPPER(TRIM(e.SKU)) = UPPER(TRIM(m.purchase_sku))
  WHERE e.出单平台 IN ('Overstock', 'Homedepot')
    AND e.订单日期 >= :start_date
    AND e.订单日期 < :end_date
),
order_listing AS (
  SELECT
    platform,
    store,
    po,
    sales_date,
    listing_sku,
    MAX(listing_units_row) AS listing_gross_units,
    MAX(cancel_units_row) AS listing_cancel_units,
    COUNT(*) AS component_rows,
    SUM(map_status <> 'OK') AS exception_rows
  FROM row_base
  GROUP BY platform, store, po, sales_date, listing_sku
),
component_net AS (
  SELECT
    r.platform,
    r.store,
    r.sales_date,
    r.po,
    r.listing_sku,
    r.purchase_sku,
    r.map_qty,
    r.gross_component_qty,
    r.cancel_units_row,
    o.listing_gross_units,
    o.listing_cancel_units,
    GREATEST(o.listing_gross_units - o.listing_cancel_units, 0) AS listing_net_units,
    CASE
      WHEN r.map_status <> 'OK' THEN NULL
      ELSE GREATEST(o.listing_gross_units - o.listing_cancel_units, 0) * r.map_qty
    END AS purchase_sku_net_qty,
    r.map_status
  FROM row_base r
  JOIN order_listing o
    ON r.platform = o.platform
   AND r.store = o.store
   AND r.po = o.po
   AND r.sales_date = o.sales_date
   AND r.listing_sku = o.listing_sku
)
SELECT
  platform,
  store,
  sales_date,
  listing_sku,
  purchase_sku,
  SUM(gross_component_qty) AS gross_component_qty,
  SUM(purchase_sku_net_qty) AS purchase_sku_net_qty,
  MAX(map_status) AS map_status,
  COUNT(DISTINCT po) AS po_count
FROM component_net
GROUP BY platform, store, sales_date, listing_sku, purchase_sku;
```

异常输出建议：

```sql
-- 输出缺映射 / 映射数量异常，不要静默丢弃
SELECT *
FROM component_net
WHERE map_status <> 'OK';
```

注意：

- `总账单详情_extract.数量` 在 OS / HD 中通常已经是 `订单上架 SKU 件数 * 映射数量` 后的组件销量。
- `取消数量` 是订单上架 SKU 层取消件数，不是组件数量；先还原 `listing_gross_units = 数量 / map_qty`，再扣 `listing_cancel_units`，最后乘回 `map_qty`。
- 2026-07-01 只读校验：2026-06 Homedepot 公式结果与直接扣减一致；Overstock 直接逐行扣减比公式多算 `20` 件，其中 `7` 件来自 `map_qty > 1` 的取消扣减错误，`13` 行来自缺映射 / 映射异常需单独输出。
- 本模板适用于 OS / HD；Wayfair 的 `wf上架sku每日销售额` 是销售 SKU 层，分摊前必须先做 Product_Type 和父子关系预检。

## SQL-WF-SALES-SUPPLIER-PART-EXPANSION-PRECHECK：Wayfair 销售 SKU 分摊预检

适用：把 `wf上架sku每日销售额.units_sold / total_revenue` 下钻到子件或采购 SKU 前，先识别 `supplier_part` 是 `Kit`、`Standard`、`Sellable Component`、`Nonsellable Component` 还是未匹配。

```sql
WITH sales_parts AS (
  SELECT
    store,
    supplier_part,
    COUNT(*) AS sales_rows,
    SUM(CAST(NULLIF(units_sold, '') AS DECIMAL(18,4))) AS units_sold,
    SUM(CAST(NULLIF(total_revenue, '') AS DECIMAL(18,4))) AS revenue
  FROM `wf上架sku每日销售额`
  WHERE STR_TO_DATE(日期, '%Y-%m-%d') >= :start_date
    AND STR_TO_DATE(日期, '%Y-%m-%d') < :end_date
  GROUP BY store, supplier_part
),
child_profile AS (
  SELECT
    c.店铺 AS store,
    c.Related_Kit_Part AS parent_part,
    COUNT(DISTINCT c.Supplier_Part_Number) AS child_part_count,
    COUNT(DISTINCT m.`采购sku`) AS child_purchase_sku_count
  FROM `wfproduct_management_latest` c
  LEFT JOIN `wayfair上架sku销售映射` m
    ON c.店铺 = m.store
   AND c.Supplier_Part_Number = m.part_number
  WHERE c.Product_Type IN ('Sellable Component', 'Nonsellable Component')
    AND c.Related_Kit_Part IS NOT NULL
    AND TRIM(c.Related_Kit_Part) <> ''
  GROUP BY c.店铺, c.Related_Kit_Part
),
direct_map AS (
  SELECT
    store,
    part_number,
    COUNT(DISTINCT `采购sku`) AS direct_purchase_sku_count
  FROM `wayfair上架sku销售映射`
  GROUP BY store, part_number
)
SELECT
  sp.store,
  sp.supplier_part,
  COALESCE(p.Product_Type, '<unmatched>') AS product_type,
  sp.sales_rows,
  sp.units_sold,
  sp.revenue,
  COALESCE(cp.child_part_count, 0) AS child_part_count,
  COALESCE(cp.child_purchase_sku_count, 0) AS child_purchase_sku_count,
  COALESCE(dm.direct_purchase_sku_count, 0) AS direct_purchase_sku_count,
  CASE
    WHEN p.Product_Type = 'Kit' AND COALESCE(cp.child_purchase_sku_count, 0) > 0 THEN 'KIT_CHILD_EXPANSION_CANDIDATE'
    WHEN p.Product_Type = 'Kit' AND COALESCE(dm.direct_purchase_sku_count, 0) > 0 THEN 'KIT_DIRECT_MAP_FALLBACK_TODO'
    WHEN p.Product_Type IN ('Standard', 'Sellable Component') AND COALESCE(dm.direct_purchase_sku_count, 0) > 0 THEN 'DIRECT_MAP_CANDIDATE'
    WHEN p.Product_Type = 'Nonsellable Component' THEN 'EXCEPTION_NONSELLABLE_SALES_SOURCE'
    WHEN p.Product_Type IS NULL THEN 'EXCEPTION_UNMATCHED_PRODUCT_TYPE'
    ELSE 'TODO_待核实'
  END AS expansion_status
FROM sales_parts sp
LEFT JOIN `wfproduct_management_latest` p
  ON sp.store = p.店铺
 AND sp.supplier_part = p.Supplier_Part_Number
LEFT JOIN child_profile cp
  ON sp.store = cp.store
 AND sp.supplier_part = cp.parent_part
LEFT JOIN direct_map dm
  ON sp.store = dm.store
 AND sp.supplier_part = dm.part_number;
```

注意：

- 这是预检模板，不是已确认的销售额分摊模板。
- `Kit` 且能通过 Component 行 `Related_Kit_Part -> Supplier_Part_Number` 找到子件时，才是优先的父件展开候选。
- `Standard`、`Sellable Component` 默认不要当父件展开；可作为直接映射候选，但仍需检查一对多和金额分摊口径。
- `Nonsellable Component` 和 `<unmatched>` 必须进入异常输出。
- 2026-07-01 只读校验：2026-06 Wayfair 销售 SKU 中 `Kit` 有 `761` 个 `store + supplier_part`，其中 `487` 个有子件采购 SKU 映射；`Nonsellable Component` 有 `26` 个、未匹配 `12` 个，需异常输出。

## SQL-PLATFORM-LISTING-PURCHASE-MAP-PRECHECK：三平台上架 SKU 销售映射预检骨架

适用：正式做组合产品销量权重、有货率或订单拆采购 SKU 前，先检查三平台映射表的组件数、数量字段和订单抽取未命中规模。

```sql
WITH platform_maps AS (
  SELECT 'Wayfair' AS platform, store, part_number, `采购sku`, `数量`, `来源`
  FROM `wayfair上架sku销售映射`
  UNION ALL
  SELECT 'Overstock' AS platform, store, part_number, `采购sku`, `数量`, `来源`
  FROM `overstock上架sku销售映射`
  UNION ALL
  SELECT 'Homedepot' AS platform, store, part_number, `采购sku`, `数量`, `来源`
  FROM `homedepot上架sku销售映射`
),
map_profile AS (
  SELECT
    platform,
    COUNT(*) AS map_rows,
    COUNT(DISTINCT store) AS store_count,
    COUNT(DISTINCT part_number) AS part_count,
    COUNT(DISTINCT CONCAT(store, '||', part_number)) AS store_part_count,
    COUNT(DISTINCT `采购sku`) AS purchase_sku_count,
    SUM(CASE WHEN `采购sku` IS NULL OR TRIM(`采购sku`) = '' THEN 1 ELSE 0 END) AS blank_purchase_sku_rows,
    SUM(CASE WHEN `数量` IS NULL OR TRIM(CAST(`数量` AS CHAR)) = '' THEN 1 ELSE 0 END) AS blank_qty_rows
  FROM platform_maps
  GROUP BY platform
),
component_profile AS (
  SELECT
    platform,
    store,
    part_number,
    COUNT(*) AS component_count,
    COUNT(DISTINCT `采购sku`) AS purchase_sku_count,
    SUM(CAST(`数量` AS DECIMAL(18,4))) AS quantity_sum
  FROM platform_maps
  GROUP BY platform, store, part_number
),
extract_keys AS (
  SELECT
    出单平台 AS platform,
    出单店铺 AS store,
    上架sku AS part_number,
    SKU AS purchase_sku,
    COUNT(*) AS extract_rows
  FROM `总账单详情_extract`
  WHERE 订单日期 >= :start_date
    AND 订单日期 < :end_date
    AND 出单平台 IN ('Wayfair', 'Overstock', 'Homedepot')
  GROUP BY 出单平台, 出单店铺, 上架sku, SKU
),
missing_from_extract AS (
  SELECT
    e.platform,
    e.store,
    e.part_number,
    e.purchase_sku,
    e.extract_rows
  FROM extract_keys e
  LEFT JOIN platform_maps m
    ON e.platform = m.platform
   AND e.store = m.store
   AND e.part_number = m.part_number
   AND e.purchase_sku = m.`采购sku`
  WHERE m.`采购sku` IS NULL
),
component_summary AS (
  SELECT
    platform,
    SUM(CASE WHEN component_count > 1 THEN 1 ELSE 0 END) AS multi_component_store_parts,
    MAX(component_count) AS max_component_count
  FROM component_profile
  GROUP BY platform
),
missing_summary AS (
  SELECT
    platform,
    COUNT(*) AS missing_extract_keys,
    SUM(extract_rows) AS missing_extract_rows
  FROM missing_from_extract
  GROUP BY platform
)
SELECT
  p.*,
  c.multi_component_store_parts,
  c.max_component_count,
  COALESCE(m.missing_extract_keys, 0) AS missing_extract_keys,
  COALESCE(m.missing_extract_rows, 0) AS missing_extract_rows
FROM map_profile p
LEFT JOIN component_summary c
  ON p.platform = c.platform
LEFT JOIN missing_summary m
  ON p.platform = m.platform
```

注意：

- 数据库真实表名是 `...上架sku销售映射`；“销量映射表”是业务口头称呼。
- Overstock / Homedepot 上架 SKU 销售映射表没有订单日期 / 月份字段，只能作为 `part_number -> 采购sku` 映射来源；近 90 天销量和月销量应以 `总账单详情_extract.订单日期` 为事实来源，再扣减 `取消数量` 后归因到采购 SKU。
- Wayfair 映射表的 `part_number` 是店铺 SKU / Supplier Part Number，可对应 `Kit`、`Standard` 或 Component，并映射到采购 SKU；父子层级需结合 `wfproduct_management_latest.Product_Type` / `Related_Kit_Part` 判断。Overstock / Homedepot 映射表是组合件 part number -> 采购 SKU。
- `wf上架sku每日销售额.supplier_part` 是销售源 Supplier Part Number / 销售 SKU 层，不要直接用它等值关联 `总账单详情_extract.上架sku` 当作同粒度；也不要把所有 `supplier_part` 都当 `Kit` 父件展开。和 `wayfair上架sku销售映射.part_number` 关联前需检查店铺、Product_Type、一对多和未匹配。
- Wayfair 组合件展开优先看 `wfproduct_management_latest` 的 Component 行：`Related_Kit_Part` 是组合名字 / 组合件 part number，`Supplier_Part_Number` 是子件名称。`Kit` 是完整可售套组 SKU；`Sellable Component` 可随套组出售也可单独售卖；`Nonsellable Component` 只能随套组出售；`Standard` 是标准独立产品，不属于套组组件。
- 2026-06-29 用户确认 `数量` 是销售个数；本模板只做画像和覆盖率预检，正式有效销量还需扣减 `取消数量`。

## SQL-WF-PRICE-ADVANTAGE-CURRENT：Wayfair 当前价格优势骨架

适用：查询当前本品前台价、竞品最低价、FineBI 原式价格优势或严格价格优势。注意本模板只适合当前快照，不适合回填历史月份。

```sql
WITH type_check AS (
  SELECT
    `产品类型`,
    COUNT(*) AS row_count
  FROM `产品近两天的最小前台价`
  GROUP BY `产品类型`
),
own_price AS (
  SELECT
    store,
    platform,
    model,
    link_sku,
    MIN(`最近日期最小前台价`) AS own_min_price,
    MAX(create_time) AS snapshot_time
  FROM `产品近两天的最小前台价`
  WHERE `产品类型` = '本品'
    AND store IN (:stores)
    AND link_sku = :link_sku
    -- TODO: 待核实：如果输入是 model、平台 SKU 或采购 SKU，先走映射后再过滤 link_sku / model。
  GROUP BY store, platform, model, link_sku
),
competitor_price AS (
  SELECT
    platform,
    model,
    MIN(`最近日期最小前台价`) AS competitor_min_price
  FROM `产品近两天的最小前台价`
  WHERE `产品类型` = '竞品'
  GROUP BY platform, model
)
SELECT
  o.store,
  o.platform,
  o.model,
  o.link_sku,
  o.snapshot_time,
  o.own_min_price AS 本品最低前台价,
  c.competitor_min_price AS 竞品最低前台价,
  CASE
    WHEN c.competitor_min_price IS NULL THEN NULL
    WHEN c.competitor_min_price < o.own_min_price THEN '否'
    ELSE '是'
  END AS FineBI原式_是否有价格优势,
  CASE
    WHEN c.competitor_min_price IS NULL THEN NULL
    WHEN o.own_min_price < c.competitor_min_price THEN '是'
    ELSE '否'
  END AS 严格口径_是否有价格优势,
  CASE
    WHEN NOT EXISTS (SELECT 1 FROM type_check WHERE `产品类型` = '竞品' AND row_count > 0)
      THEN '待核实：当前快照缺竞品价格行'
    WHEN c.competitor_min_price IS NULL
      THEN '待核实：该 model 未匹配竞品最低价'
    ELSE '已匹配'
  END AS 校验状态
FROM own_price o
LEFT JOIN competitor_price c
  ON o.platform = c.platform
 AND o.model = c.model;
```

注意：

- 2026-07-01 只读复核：`产品近两天的最小前台价` 和上游 `link_sku_price_qtj_fabric` 当前都只见 `产品类型='本品'`，未见 `竞品` 行；此时不要返回“无价格优势”，应返回竞品价缺失 / 待核实。
- 2026-07-02 用户确认：本次缺竞品原因是相关函数/例程执行失败且已修复；再次启用价格优势 SQL 前，先复核快照表是否恢复 `产品类型='竞品'`、最新日期和样本对账。
- 2026-07-02 修复后只读复核：`产品近两天的最小前台价` 已恢复 `竞品 4,649` 行，当前快照可重新启用价格优势 SQL；仍需提醒这是当前快照，不可直接回填历史月份。
- `link_sku_price_qtj_fabric` 主键为 `产品id + 日期 + website`，按日期范围直接聚合可能较慢；普通智能取数默认不要扫该大表。
- `wfprice` 是 Wayfair 成本 / 促销成本相关表，不是竞品最低前台价入口。

## SQL-WF-PRICE-SOURCE-PRECHECK：Wayfair 价格优势源表预检骨架

适用：在回答“是否有价格优势 / 竞品最低价”前，先确认当前数据库是否有可用竞品价格输入。该模板用于预检，不直接生成业务结论。

```sql
-- 1. 当前快照是否存在竞品价格行
SELECT
  `产品类型`,
  COUNT(*) AS row_count,
  COUNT(DISTINCT platform) AS platform_count,
  COUNT(DISTINCT store) AS store_count,
  COUNT(DISTINCT model) AS model_count,
  COUNT(DISTINCT link_sku) AS link_sku_count,
  MIN(`最近日期最小前台价`) AS min_recent_price,
  MAX(`最近日期最小前台价`) AS max_recent_price
FROM `产品近两天的最小前台价`
GROUP BY `产品类型`;

-- 2. temp_qtj 中间表回连映射后是否存在竞品
SELECT
  COALESCE(m.platform, 'UNMAPPED') AS platform,
  COALESCE(m.store, 'UNMAPPED') AS store,
  COALESCE(m.`产品类型`, 'UNMAPPED') AS product_type,
  t.`来源` AS source,
  COUNT(*) AS row_count,
  COUNT(DISTINCT t.`产品id`) AS product_id_count,
  COUNT(DISTINCT m.model) AS model_count,
  COUNT(DISTINCT m.link_sku) AS link_sku_count,
  MIN(t.price) AS min_price,
  MAX(t.price) AS max_price,
  MIN(t.`日期`) AS min_date,
  MAX(t.`日期`) AS max_date
FROM temp_link_sku_price_qtj_fabric t
LEFT JOIN link_sku_table_2 m
  ON t.`产品id` = m.`产品id`
GROUP BY
  COALESCE(m.platform, 'UNMAPPED'),
  COALESCE(m.store, 'UNMAPPED'),
  COALESCE(m.`产品类型`, 'UNMAPPED'),
  t.`来源`;

-- 3. temp_scj 中间表回连映射后是否存在竞品
SELECT
  COALESCE(m.platform, 'UNMAPPED') AS platform,
  COALESCE(m.store, 'UNMAPPED') AS store,
  COALESCE(m.`产品类型`, 'UNMAPPED') AS product_type,
  t.`来源` AS source,
  COUNT(*) AS row_count,
  COUNT(DISTINCT t.`产品id`) AS product_id_count,
  COUNT(DISTINCT m.model) AS model_count,
  COUNT(DISTINCT m.link_sku) AS link_sku_count,
  MIN(t.price) AS min_price,
  MAX(t.price) AS max_price,
  MIN(t.`日期`) AS min_date,
  MAX(t.`日期`) AS max_date
FROM temp_latest_link_sku_price_scj t
LEFT JOIN link_sku_table_2 m
  ON t.`产品id` = m.`产品id`
GROUP BY
  COALESCE(m.platform, 'UNMAPPED'),
  COALESCE(m.store, 'UNMAPPED'),
  COALESCE(m.`产品类型`, 'UNMAPPED'),
  t.`来源`;

-- 4. 竞品关系表只确认映射，不确认价格
SELECT
  COUNT(*) AS row_count,
  COUNT(DISTINCT model) AS model_count,
  COUNT(DISTINCT `竞品sku`) AS competitor_sku_count
FROM `竞品对标表`;
```

注意：

- 2026-07-02 只读复核：`temp_link_sku_price_qtj_fabric` 和 `temp_latest_link_sku_price_scj` 当前回连后也只见 `本品` 或未映射，不能补 `竞品最低前台价`。
- `最新价格表` 没有 `产品类型`、`竞品sku`、`model` 字段，且价格记录日期偏旧；不要把它当作当前价格优势入口。
- `竞品对标表` 和 `本竞品失效sku` 是竞品关系 / 状态线索，不是价格事实表。
- 2026-07-02 过程补充：如果要继续排查根因，优先检查 `link_sku_price_qtj_fabric` 过程的竞品分支。已知 `product_fabric` 近 90 天可按 `link_sku_table_2.link_sku = product_fabric.sku` 命中部分最新竞品映射样本，但当前目标表仍只见 `本品`；不要把预检结论简化为“上游无竞品价”。
- 2026-07-02 用户确认：本次根因为相关函数/例程执行失败，现已修复；本模板后续重点从“找替代表”转为“复核恢复后的竞品行和 FineBI 样本一致性”。
- 2026-07-02 修复后复核：下游快照已恢复竞品行；预检模板后续用于确认当前快照是否仍有 `竞品`，而不是每次都重查上游大表。

## SQL-SALES-DDP-PROFIT-CANDIDATE：DDP 利润候选复算骨架

适用：用户问“某 SKU / Model / 店铺 / 月份利润是多少”时，先做候选复算和口径说明。当前仍不是最终 `ddp_利润查询` 精确复刻，未确认处必须保留 `TODO: 待核实`。

```sql
WITH sales AS (
  SELECT
    LEFT(`订单日期`, 7) AS month,
    `出单平台` AS platform,
    `出单店铺修` AS store,
    `国家` AS country,
    `sku修` AS purchase_sku,
    SUM(`数量`) AS units,
    SUM(`总销售额`) AS sales_amount,
    SUM(`平台折后销售额`) AS platform_net_sales_amount
  FROM `order_details_tb_profit`
  WHERE `订单日期` >= :start_date
    AND `订单日期` < :end_date
    AND `sku修` IN (:purchase_sku_list)
  GROUP BY LEFT(`订单日期`, 7), `出单平台`, `出单店铺修`, `国家`, `sku修`
),
sku_cost AS (
  -- 候选平均成本层：`上架sku利润表` 主键是 store + 上架sku + 日期。
  -- TODO: 待核实：月度按采购 SKU 平均会混合多个上架 SKU / 包装 / 套数组合，不等同订单行精确成本。
  SELECT
    LEFT(`日期`, 7) AS month,
    platform,
    store,
    `国家` AS country,
    `采购sku` AS purchase_sku,
    AVG(`平台佣金`) AS avg_platform_commission,
    AVG(`发货费`) AS avg_ship_fee,
    AVG(`ddp`) AS avg_ddp,
    AVG(`成本`) AS avg_cost_excluding_commission,
    AVG(`利润率上限`) AS avg_profit_rate_limit
  FROM `上架sku利润表`
  WHERE `日期` >= :start_date
    AND `日期` < :end_date
    AND `采购sku` IN (:purchase_sku_list)
  GROUP BY LEFT(`日期`, 7), platform, store, `国家`, `采购sku`
),
monthly_return AS (
  SELECT
    `年月` AS month,
    `平台` AS platform,
    `国家` AS country,
    sku AS purchase_sku,
    SUM(`退货金额`) AS return_amount,
    SUM(`补件费用`) AS replacement_fee
  FROM `退货补件费用`
  WHERE `年月` >= LEFT(:start_date, 7)
    AND `年月` < LEFT(:end_date, 7)
    AND sku IN (:purchase_sku_list)
  GROUP BY `年月`, `平台`, `国家`, sku
)
SELECT
  s.month,
  s.platform,
  s.store,
  s.country,
  s.purchase_sku,
  s.units,
  s.sales_amount,
  s.platform_net_sales_amount,
  c.avg_platform_commission,
  c.avg_ship_fee,
  c.avg_ddp,
  c.avg_cost_excluding_commission,
  r.return_amount,
  r.replacement_fee,
  -- TODO: 待核实：最终利润分母用总销售额还是平台折后销售额。
  -- TODO: 待核实：退货金额在各平台表中的正负号是否需要统一取反。
  -- TODO: 待核实：成本按均值乘销量，还是按上架 SKU / 订单行更细粒度匹配。
  s.platform_net_sales_amount
    - COALESCE(c.avg_platform_commission, 0) * s.units
    - COALESCE(c.avg_cost_excluding_commission, 0) * s.units
    - COALESCE(r.return_amount, 0)
    - COALESCE(r.replacement_fee, 0) AS candidate_profit
FROM sales s
LEFT JOIN sku_cost c
  ON c.month = s.month
 AND c.store = s.store
 AND c.purchase_sku = s.purchase_sku
LEFT JOIN monthly_return r
  ON r.month = s.month
 AND r.country = s.country
 AND r.purchase_sku = s.purchase_sku
ORDER BY s.month, s.store, s.purchase_sku;
```

注意：

- 2026-07-03 样本验证：`上架sku利润表.成本 ≈ 发货费 + ddp`，不含 `平台佣金`。
- 2026-07-03 匹配键验证：`order_details_tb_profit` 无 `上架sku`；`上架sku利润表` 主键为 `store + 上架sku + 日期`。按 `store + 国家 + 日期 / 月份 + 采购sku` 只能做候选平均成本辅助，不能把明细 join 后直接求和。
- 2026-06 样本提醒：`上架sku利润表` 相关店铺只到 `2026-06-26`，HD 店铺本轮未命中；按日非 HD 店铺仍有未匹配和一对多，按月覆盖率高但平均放大约 `17` 到 `50` 行。
- `利润率上限` 不是逐行最终利润率，不能直接当作 `ddp_利润查询` 的利润率。
- `退货补件费用` 是月度汇总，不是订单明细；正式复算需要确认 `平台`、`国家`、SKU 和金额正负号规则。
