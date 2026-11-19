# Measure — billing truth and recipes

Official basis: `databricks-unity-catalog` (system tables) and the docs pages for billing and pricing system tables.

## Rules
- **Billing truth** is `system.billing.usage × system.billing.list_prices`, joined on `sku_name`, `cloud`
  and `usage_unit`, with `usage_start_time` inside `[price_start_time, price_end_time)`. Leaving out the time
  window double-counts across price changes, which is the most common mistake.
- **The result is a list-price estimate.** It excludes discounts, cloud storage and egress. Label it that way.
- **Filter `workspace_id`.** Billing is account-wide.
- **Group by `billing_origin_product`** before `sku_name`.
- **Sum `usage_quantity`.** RETRACTION and RESTATEMENT rows then net out.
- **Billing lags about 12h.** It cannot be a real-time ceiling (see [guards.md](guards.md)).
- **Attribute by the field the product fills in:**

  | Resource | Field |
  |---|---|
  | Jobs | `usage_metadata.job_id` |
  | Warehouses | `usage_metadata.warehouse_id` |
  | Apps | `usage_metadata.app_name` |
  | Pipelines | `usage_metadata.dlt_pipeline_id` |
  | Serving and vector search | `usage_metadata.endpoint_name` |
  | Lakebase | `usage_metadata.database_instance_id` |

  Only compute tags reach `custom_tags`. Dashboard, Genie and table tags do not. A dashboard built on tags
  alone can miss most of the bill.
- **Split any rolling window at a fix date.** A 30-day card can be mostly spend from before the fix.

## Recipes (replace `:ws`)

**Daily cost by product:**
```sql
SELECT u.usage_date, u.billing_origin_product,
       ROUND(SUM(u.usage_quantity * COALESCE(p.pricing.effective_list.default, p.pricing.default)), 2) AS usd_list
FROM system.billing.usage u
JOIN system.billing.list_prices p
  ON u.sku_name = p.sku_name AND u.cloud = p.cloud AND u.usage_unit = p.usage_unit
 AND u.usage_start_time >= p.price_start_time
 AND (p.price_end_time IS NULL OR u.usage_start_time < p.price_end_time)
WHERE u.workspace_id = :ws AND u.usage_date >= current_date() - INTERVAL 30 DAYS
  AND p.currency_code = 'USD'
GROUP BY ALL ORDER BY 1 DESC, 3 DESC;
```

**Top resources.** In the query above, swap the product column for
`COALESCE(u.usage_metadata.job_id, u.usage_metadata.warehouse_id, u.usage_metadata.app_name, u.usage_metadata.dlt_pipeline_id, u.usage_metadata.endpoint_name) AS resource`.

**Untagged spend:**
`... WHERE u.custom_tags['<your-tag>'] IS NULL GROUP BY u.billing_origin_product`

**Run-rate projection:** take the last 7 complete days and compute `SUM(usd_list) * 30 / 7`.

**Billing freshness:**
`SELECT max(usage_end_time) FROM system.billing.usage WHERE workspace_id = :ws`

**Before and after a change:** run the same query with `usage_date < :fix_date` and `>= :fix_date`.
Compare per-day averages, not totals.

## Reporting a number
State the window, the workspace, the query and "list-price estimate". "About $X/day" without a query is
speculation, and law 3 forbids it.
