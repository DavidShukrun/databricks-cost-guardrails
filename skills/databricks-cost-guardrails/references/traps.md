# Traps — cuts that look cheap and break things

Run this list before shipping any cost change. Each line happened in production.

## Cost cuts that silently cost quality
| Looks like a saving | What actually breaks | Do instead |
|---|---|---|
| Self-skip inside the notebook | The start is still paid; the saving is about 0 | `condition_task` before the expensive task |
| `condition_task` / skip gate | The job reports SUCCESS having done nothing | Add a freshness check on the output |
| Trimming a schedule window on assumption | It misses real arrivals (a UTC misread is classic) | Measure arrivals first, in the right timezone |
| Weekday-only schedule | A 48h freshness SLA meets a 72h gap, and the guard trips or gets loosened | Check SLAs before trimming days |
| Rounding a cache key | Stale exact figures for users, or a run-time failure | Exact key; render volatile fields fresh |
| Cheaper model on a user-facing surface | Answer quality, and trust | Cache first; swap only with an eval |
| Loosening a guard threshold | The guard never fires again | Fix the noise source |
| Shipping a stop without its start | An outage the next morning | Ship the pair in one change |
| STANDARD on a many-task chain | Each task adds a 4–6 minute start; the SLA slips | Consolidate tasks, then STANDARD |
| Smallest warehouse by rule | Spill to disk wastes more DBUs, and queues | Smallest with zero spill (official) |
| 1-minute auto-stop on a user warehouse | Cold-start latency on every visit | Short stop for agents; measured stop for users |

## Runtime traps (pass tests, fail only in the live job)
- **`round(x, scale)` with a computed scale** fails at analysis time with `NON_FOLDABLE_INPUT`.
- **`df.cache()` / `persist()` on serverless** fails at run time.
- **A quartz cron with a year**: once the date passes, it is rejected even when PAUSED, and the bundle
  deploy fails.
- **A notebook whose first `%md` cell has no cell separator before the code** runs nothing and reports
  SUCCESS.
- **A SQL `COMMENT '…'` with an apostrophe**: the string is valid in the host language, but the DDL fails.
- **A print-only notebook** leaves no run output. Return a summary with `dbutils.notebook.exit`.
- **A Spark Connect self-join loop** referencing bare columns becomes ambiguous from the second hop.
  Alias both sides.

**Countermeasure:** run `EXPLAIN` on every new SQL statement against the real workspace before merging.
Have tests read the actual SQL the job runs, not a copy.

## Status traps
- "PR open" is not merged, and merged is not deployed. A deploy run can be red for hours while everyone
  believes "it's in". Verify the deploy run and the deployed content.
- A green job is not complete data, and a green job is not a cheap job.
- "Compute dead, CI green" usually means account credits or the SKU. A trial SKU meters against trial
  credits. Check billing and account status before debugging code.

## Measurement traps
- Report a list-price estimate, not the invoice. Say so.
- A rolling window that straddles a fix makes old spend look current. Split it at the fix date.
- An estimate is not a result. Write "estimated, to be measured" until a re-measure exists.
- Rates from a vendor's list are not Databricks rates. Price from the SKU.
- An official skill claim is not docs fact. Mark numbers you could not verify in the docs.
