# SQL warehouses

Official basis: `databricks-dbsql` (best practices), `databricks-execution-compute`, and the docs pages on
creating warehouses, warehouse behavior and query caching.

## The cost model
**A warehouse bills while it is warm, not per query.** A trickle of sparse queries keeps resetting
auto-stop, so the warehouse never sleeps.

The idle-burn signature is "warm for 10 hours or more while executing less than 5% of that time". One team
saw its SQL line grow about 14× in a week from configuration drift alone: a bigger size and a longer
auto-stop, with the same queries.

## Rules
1. **Serverless warehouses** (official). They give fast start and stop, IWM scaling and the remote result cache.
2. **Auto-stop is always on, and set per audience.** The serverless default is 10 minutes. The UI minimum
   is 5. The **API minimum is 1**.
   - Programmatic traffic (agents, CLI, CI): use 1 minute.
   - Interactive users: use 5–10 minutes. Base it on measured cold start (`waiting_for_compute_duration_ms`)
     against idle cost.
3. **Right-size by measurement.** The official guidance is to start larger and size down, because
   spilling to disk wastes more DBUs than a larger size. The cheapest correct size is **the smallest with
   zero spill and acceptable queue time** (watch Peak Queued Queries). Size up for spill, never by habit.
4. **Split warehouses by audience.** Keep user-facing warehouses (BI, apps, Genie) apart from
   agent/CLI/CI warehouses. Otherwise programmatic bursts keep the user warehouse warm and blur
   measurement. Set the warehouse id explicitly on every programmatic call; never rely on a "default".
5. **Configuration drifts, because a warehouse is often not in your bundle.** Pin size, auto-stop and
   cluster count. Check them on a schedule, and treat any size change as a reviewed cost decision with one
   owner. `warehouses edit` needs the **full** configuration; fields you leave out revert to defaults.
6. **Max clusters is a concurrency decision.** Raise it only when measured queueing requires it.

## Result cache (free reuse)
- Serverless has a **remote result cache**. It is workspace-wide, survives warehouse stop, lasts 24 hours,
  and is invalidated by table changes (docs).
- Non-deterministic SQL is not served from cache. Avoid `now()`, `current_timestamp()`, `current_date()`,
  `rand()` (skill and observed), **including inside views your query reads**.
- To make a recurring query cacheable, pass a truncated literal such as `TIMESTAMP '2026-01-01 10:00'`
  from the caller instead of `current_timestamp()`. Keep the production default unchanged if the value
  semantics matter.
- Materialize hot dashboard aggregates into small tables, refreshed on a schedule.
- To verify, check `system.query.history.from_result_cache` per statement.

## Diagnose
```sql
WITH q AS (
  SELECT compute.warehouse_id AS wh, to_date(start_time) AS d,
         COUNT(*) AS n, SUM(total_duration_ms)/3.6e6 AS busy_h,
         AVG(CASE WHEN from_result_cache THEN 1 ELSE 0 END) AS cache_hit,
         AVG(waiting_for_compute_duration_ms)/1000 AS avg_wait_s
  FROM system.query.history
  WHERE start_time >= current_date() - INTERVAL 14 DAYS GROUP BY ALL)
SELECT * FROM q ORDER BY d DESC, busy_h DESC;
```
Compare `busy_h` with the warehouse's billed DBUs per day (`usage_metadata.warehouse_id`, see
[measure.md](measure.md)) or its running time in `system.compute.warehouse_events`. High billing with
low `busy_h` means idle burn. Group by `executed_by` to find which audience keeps it warm.
