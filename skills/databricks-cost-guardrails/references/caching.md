# Caching and reuse — key discipline

Reuse is the one lever that cuts cost with zero quality loss, but only when the key is right.

## The key law
**The cache key holds exactly the facts the computation sees. No more, no less.**
- **Too wide** (a field that moves daily but does not change the output, such as an "as of" date or a
  timestamp): the hit rate collapses. Observed: only 71 of 150 daily AI briefings were reused, because
  an as-of date and a running total sat in the key.
- **Too coarse** (rounded or bucketed inputs while the output prints exact values): a user sees a
  **stale exact figure**. That is a correctness bug, not a cost one.
- **Fix:** move volatile display fields out of the computation and render them fresh at display time.
  Then hash the exact remaining inputs.

## Patterns
| Layer | Pattern |
|---|---|
| Paid calls (LLM, vendor API) | Cache table keyed by `sha2(model \|\| exact_input)`; union hits with misses; never store NULL or failure (see [ai.md](ai.md)) |
| SQL result cache | Deterministic SQL: no `current_*()` / `rand()` in the query **or its views**; pass truncated literals from the caller (see [warehouses.md](warehouses.md)) |
| Expensive rebuilds | Incremental MVs (serverless + row tracking); skip an unchanged build (compare the input hash or the source `MAX(version)`) |
| Hot dashboard tiles | Small materialized aggregate tables on a schedule |
| Heavy tests and checks | Make their SQL cacheable, and scope local runs to what changed (see [agents-ci.md](agents-ci.md)) |

## Make a cache measurable
- Log hits and misses per run, and alert when the hit rate drops sharply. A key regression is silent
  otherwise.
- Use the hit-rate formula `1 - misses / distinct_inputs`. A healthy daily job over mostly unchanged
  inputs should be near 1.

## Pitfalls
- **Spark `round(x, scale)` needs a constant scale.** A per-row computed scale fails only at run time
  with `NON_FOLDABLE_INPUT`. For significant-figure keys use `format_string('%.2g', x)`, or better, do
  not round keys at all.
- **Serverless does not support `df.cache()` / `persist()`.** Persist to a table, or `collect()` small
  data.
- **A hash over a non-deterministic serialization misses forever.** Map ordering and float formatting
  are examples. Canonicalize first: sorted keys, fixed precision.
- **Invalidate on the model too.** A model or prompt-template change must change the key.
