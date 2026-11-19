# Lakeflow pipelines and streaming

Official basis: `databricks-pipelines` (including `references/performance.md`),
`databricks-spark-structured-streaming` and `databricks-serverless`.

## Official floor
- Default to serverless pipelines. Classic is only for R, RDD, or JAR/Maven needs.
- Set `continuous: false` unless continuous processing is explicitly required. A continuous pipeline runs,
  and bills, all the time, and retries failed updates forever.
- Real-Time Mode needs an always-on classic cluster, so use it only when explicitly requested.
- A pipeline has no performance mode of its own. Set `performance_target` on the **job that triggers it**.
  Scheduled MV and streaming-table refreshes default to standard (docs).
- **Incremental MV refresh needs serverless plus row tracking.** Without them it silently falls back to
  a full recompute, which you pay for on every run.
- A serverless update can sit in INITIALIZING for minutes. Do not cancel and restart it, because that
  pays the start twice.

## Cost levers
1. **Triggered over continuous.** Use `trigger(availableNow=True)` or a scheduled pipeline update, and
   keep latency as a stated requirement.
2. **Keep MVs incremental-eligible.** Non-deterministic expressions (`current_timestamp()`, `rand()`,
   `ai_query`, …) in an MV definition force a full recompute. Move volatile logic downstream, or into a
   cached step (see [ai.md](ai.md)).
3. **Refresh only on arrival.** Use file-arrival or table-update triggers where the source is irregular.
   Those triggers want performance-optimized compute (docs).
4. **Layout and maintenance:** prefer Liquid Clustering and turn on Predictive Optimization (official).
   Do not repeat unverified percentage claims; measure on your own tables.

## The reliability-vs-cost trap
`expect_or_drop`, `AT_LEAST_ONE_SUCCESS` and arrival triggers can turn real data loss into a green run.
A cheaper pipeline that drops rows silently is broken.

Next to any cost change to a pipeline, add a degradation check that is a different instrument from the
failure alert: row-count or freshness bounds on the output, and expectation counts from the event log:

```sql
SELECT timestamp, details:flow_progress.data_quality.expectations
FROM event_log(TABLE(<catalog>.<schema>.<table>))
WHERE event_type = 'flow_progress' ORDER BY timestamp DESC LIMIT 50;
```

## Measure
Pipeline cost per update: `usage_metadata.dlt_pipeline_id` in billing (see [measure.md](measure.md)).
Update duration and frequency: `system.lakeflow.pipeline_update_timeline`.
Cost divided by updates gives cost per refresh, which is the number to cut.
