# Jobs and serverless compute

Official basis: `databricks-serverless`, `databricks-jobs` and the docs page on serverless performance modes.

## Official floor (do not go under it)
- **Serverless is the default for every job and pipeline.** Classic is only for hard blockers: R,
  `%scala`, Docker, OS packages, custom data source JARs, or a stated instance or DBR requirement. The
  official skill says **cost is not a reason for classic**.
- **Set `performance_target` on every job, and state the reason in one line.**
  - `STANDARD`: scheduled batch, nightly or hourly runs, backfills, nothing waiting.
  - `PERFORMANCE_OPTIMIZED`: a person waiting, a tight SLA, file-arrival or table-update triggers, very
    frequent runs.
- **STANDARD consumes fewer DBUs at the same price per DBU, with identical results.** It is not a cheaper
  rate. Startup is typically 4–6 minutes, against under a minute for performance-optimized (docs).
- **Performance-optimized is the platform default.** STANDARD is opt-in.

## The cost levers inside that floor
1. **Count starts, not work.** Each task's compute start is billed. When tasks are short, starts dominate
   the bill. A notebook that exits early still paid for its start, so a self-skip saves almost nothing.
   Observed: 139 of 170 runs of one job finished in under 23 seconds and cost most of that project's bill.
   Measure it on your workspace; see the query below.
2. **Gate by the kind of cost.**
   - When the cost is *external work* (an LLM or a vendor API), gate inside the code.
   - When the cost is the *compute start*, gate with a `condition_task` before the expensive task.
   - Every gate needs a freshness check, because a skipped job reports SUCCESS.
3. **Consolidate short tasks.** A job with N short tasks pays N starts. Running them through one runner
   (`dbutils.notebook.run` in dependency order, with lanes in threads if wall time matters) pays one.
   - Keep the dependency edges in the runner.
   - Fail loudly on the first error.
   - Keep the DAG when tasks are long, need independent retry, or have different compute needs.
   - Observed: 22 tasks → 1 cut the run cost by 25–40%, with a longer wall time.
4. **STANDARD multiplies by task count.** Each task in a STANDARD job waits its own 4–6 minute start. On
   a 10-task chain that can add 40+ minutes. Pair STANDARD with consolidation, and measure duration before
   and after. Never claim a STANDARD saving you have not measured.
5. **Base schedule cuts on measured arrivals.** Check when the source data actually lands, and check the
   timezone (source timestamps are often UTC). Trim only then. Weekend or weekday-only schedules interact
   with freshness SLAs. Calendar trims usually save little; ingestion and ad-hoc runs are where the money is.
6. **Use `availableNow` or triggered runs over continuous ones** unless a latency need is stated.

## Measure starts
```sql
SELECT job_id, task_key, COUNT(*) AS runs,
       SUM(CASE WHEN timestampdiff(SECOND, period_start_time, period_end_time) < 60 THEN 1 ELSE 0 END) AS sub_minute
FROM system.lakeflow.job_task_run_timeline
WHERE period_start_time >= current_date() - INTERVAL 14 DAYS
GROUP BY ALL ORDER BY runs DESC;
```
Join cost per job from [measure.md](measure.md) (`usage_metadata.job_id`). A high `sub_minute` share
means you pay for starts.

## Scheduling hygiene
- Never put a year in a quartz cron. Once the date passes, the Jobs API rejects it even when the job is
  PAUSED, and a single-bundle deploy then fails for every resource.
- One-off jobs get no schedule; run them with run-now.
- "Merged" is not "deployed". Check that the deploy run is green.
