# AI functions, model serving, vector search

Official basis: `databricks-ai-functions`, `databricks-model-serving`, `databricks-unity-gateway`, `databricks-vector-search`.

## Official floor
- **Prefer a task-specific AI function** (`ai_classify`, `ai_extract`, …) over `ai_query`.
- **Run once per row and persist the result to a Delta table.** Use `failOnError => false` in batch.
- **Use pay-per-token foundation models** (`system.ai.*`) to start. Provisioned throughput buys guaranteed
  throughput at a higher, always-on cost.
- **Vector search:** start with **Storage-Optimized** and **TRIGGERED** sync. Use Standard or CONTINUOUS
  only for a stated need such as sub-100 ms latency, high QPS or real-time sync. You cannot change the
  endpoint type later. `target_qps` is billed whether traffic comes or not. **Ask before creating
  endpoints or indexes.**
- **Known official exception:** the AI-functions skill suggests a classic ML LTS cluster for batch
  inference throughput. Follow it only when throughput actually needs it, and measure it.

## Cost rules
1. **Never put `ai_query` inside a rebuild that recomputes every run.** That covers MVs,
   `CREATE OR REPLACE`, and full refreshes. It is the most common AI leak: the same rows are re-sent to
   the model every day. In one case a daily MV re-scored about 170 entities with a top-tier model every
   day, for a report that people read three times in six weeks.
2. **Cache by the exact input.**
   - Key on `sha2(concat(model, '|', exact_prompt), 256)`.
   - Hits and misses run as **separate branches unioned together**: hits are read from the cache table,
     and only misses call the model. Never wrap `ai_query` in a CASE and hope the optimizer skips it.
   - **Never cache NULL or failed results**, so they are retried.
   - The key holds exactly the facts the model sees, no more and no less (see [caching.md](caching.md)).
3. **Price from the serving SKU, not the vendor list.** Databricks bills models in DBUs per million
   tokens, and models can share one SKU. Two models' vendor-list price ratio is not their Databricks
   ratio. Read `system.billing.list_prices` and measure.
4. **Pay-per-token has no idle cost.** Provisioned endpoints and vector search endpoints bill 24/7 until
   deleted or stopped.
   - Enable scale-to-zero where offered.
   - Use on-demand up/down for non-production endpoints.
   - Know what CD does *not* restore. An endpoint created out of band can vanish silently.
5. **Decide quality against cost per surface, explicitly.**
   - User-facing answers and client-facing numbers: keep the validated model. Change it only with an
     eval comparison, never for an estimated saving.
   - Batch internal text: a cheaper model is fine, but **after** caching. Caching usually saves more
     than a model swap, with zero quality risk.
   - Cap eval-loop spend: scheduled evals are runs too.
6. **Use LIMIT while developing.** Every dev query over a full table is billed per token.

## Measure
- Per-day AI spend by endpoint: `usage_metadata.endpoint_name`. See [measure.md](measure.md).
- Token volumes: `system.serving.endpoint_usage`. With Unity Gateway, also its usage table.
- Divide AI spend by distinct inputs per day. A high ratio means you are recomputing unchanged inputs.
