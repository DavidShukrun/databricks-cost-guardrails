# Agents and CI as a cost source

Coding agents and CI are real Databricks spenders. They run bursts of queries, rerun heavy test suites
many times a day, and keep warehouses warm.

Observed: agent CLI sessions reached about 1,750 queries a day in bursts of 300–700 an hour. Their
warehouse went from about 1× to 8× its previous daily cost. One regression suite, run about 10 times a
day, had **0 of 123** result-cache hits, because its SQL used `current_timestamp()`.

## Rules
1. **Give agents their own warehouse.** Keep it off the user-facing warehouse, at the smallest size that
   does not spill, with **auto-stop at 1 minute** (API).
   - **Pin the warehouse id on every call**, for example with an env var for the CLI or SDK.
   - A "default warehouse" lookup is how agent traffic leaks onto user compute.
2. **Ask before starting anything idle-billed.** The official stance on interactive clusters is the same:
   never start one without asking, because it takes minutes and costs money. Prefer Databricks Connect,
   or a serverless job, for ad-hoc compute.
3. **Make test and check SQL cacheable.**
   - Inject a truncated "now" (to the hour) from the test harness instead of `current_timestamp()`, and
     keep the production default byte-identical.
   - Watch for transitive views that read `current_date()`. They defeat the cache too, and changing them
     can change production values, which is a correctness decision and not a test tweak.
4. **Scope local runs.** Run the live heavy checks that the change touches, and leave the full suite to CI.
   - Give expensive checks a marker or flag.
   - Merge repeated runs of the same heavy query into one.
5. **Mind agents that go live by accident.** A test helper that calls the CLI when it is on PATH turns
   every local test run into warehouse traffic. Make live mode explicit.
6. **Separate measurement from noise.** Agent queries pollute before-and-after cost comparisons. Filter
   `system.query.history` by `executed_by`, and keep agents off the compute you are measuring.
7. **Do no speculative querying.** Explore with `LIMIT`, `DESCRIBE` and metadata first. Use schema
   discovery tools rather than `SELECT *` over large tables.
8. **Freeze deterministic loops.**
   - Recurring agent work that is deterministic should be a zero-token script, failing closed on
     novelty.
   - Long-lived LLM agent sessions must compact; cost per call grows with session age.

## Measure
```sql
SELECT executed_by, compute.warehouse_id, to_date(start_time) AS d,
       COUNT(*) AS n, SUM(total_duration_ms)/3.6e6 AS busy_h,
       AVG(CASE WHEN from_result_cache THEN 1 ELSE 0 END) AS cache_hit
FROM system.query.history
WHERE start_time >= current_date() - INTERVAL 7 DAYS
GROUP BY ALL ORDER BY n DESC;
```
Any agent or service-principal identity on a user warehouse is a routing bug.
