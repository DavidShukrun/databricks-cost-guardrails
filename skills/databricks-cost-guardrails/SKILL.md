---
name: databricks-cost-guardrails
description: "Cost layer on top of the official Databricks agent skills: find the cheapest correct way to build, run or change anything on Databricks — compute choice, idle burn, task starts, caching, AI spend, guards, billing truth. WHEN: \"new job\", \"new pipeline\", \"new app\", \"warehouse\", \"Databricks cost\", \"bill jumped\", \"DBU\", \"auto_stop\", \"serverless\", \"budget\", \"cost guard\", \"ai_query cost\"."
---

# Databricks cost guardrails — cheapest correct

**Prime directive: the cheapest solution that is fully correct, on the official pattern.**
Correctness and the official Databricks pattern are floors. Cost is optimized *inside* them, never by
going under them. A saving that weakens a number, freshness, or a guard is a cost, not a saving.

This skill does not replace the official `databricks-*` skills. It is the layer you apply after them.

## The laws (in precedence order — a higher law wins a conflict)

1. **The official skills are the Torah.** Before any Databricks work, load `databricks-core` and then the
   product skill (`databricks-serverless`, `-jobs`, `-pipelines`, `-dbsql`, `-apps`, `-lakebase`,
   `-model-serving`, `-vector-search`, `-ai-functions`, `-unity-catalog`, …). Never pick a less advanced or
   off-pattern approach to save money. Pick the cost lever the official pattern already offers. If a skill
   is silent, read docs.databricks.com before you state how the platform behaves. Check that the installed
   skills match the latest upstream release.
2. **Correct and reliable before cheap.** Every cost cut ships with the check that proves it broke
   nothing: freshness, row counts, the numbers gate. No skipping, gating or schedule trim without one.
3. **Measure, never speculate.** Every cost claim comes from `system.billing.usage` joined to
   `list_prices` on the price window, filtered by workspace, and labeled "list-price estimate". No query,
   no claim. → [measure.md](references/measure.md)
4. **Nothing bills idle without an owner and a stop path.** Before you create anything that bills while
   idle (an app, a warehouse, provisioned serving, vector search, an interactive cluster, a continuous
   pipeline), say what it costs per idle day and how it stops. Ask first.
5. **Pay for work, not for starts or uptime.** Count compute starts per run × runs per day, and warm
   hours against busy hours. These, not query count, are the usual bill.
6. **Compute once, reuse forever.** Pay for a paid call (LLM, vendor API) or an expensive rebuild once per
   distinct input. Key the cache on the exact input, and never cache a failure.
7. **Enforce by state, not by budget.** Budgets and usage policies alert and attribute; they do not block.
   Enforcement means auto-stop, schedules, and a guard that cannot fail open.
8. **Attribute everything, in code.** Tag every resource in the bundle so the tag survives deploy.
   Untagged spend is a defect.
9. **Price before you approve, in money.** Approve "~$X/day (list-price estimate)", never "N rows" or
   "a few runs".
10. **Shipped means deployed and measured.** "Merged" is not "deployed". A saving is real only after a
    re-measure against a clean baseline, with the window split at the change date.

## Mandatory flow for every new service or change

1. Load the official skills (law 1). Take their default: serverless, the recommended mode, auto-stop on.
2. Fill in the **cost card** below *before* writing code. A blank field means a design gap.
3. Pick the cheapest lever per product → the router below.
4. Run the traps check → [traps.md](references/traps.md).
5. Put the cost card in the PR body. After about a week of data, re-measure and record the result.

```
COST CARD
Official skill(s):  <loaded, version>
Who waits?          <nobody | person | SLA hh:mm>  → compute/mode: <…> because <one line>
Idles billed?       <none | what, $/idle-day, stop path, owner>
Starts per day:     <tasks × runs/day>
Paid calls:         <none | what, cache key, NULL policy>
Estimate:           <~$/day, arithmetic, "list-price estimate">
Quality guard:      <freshness / count / numbers check that proves nothing broke>
Measure:            <query> — re-check on <date>
Tags:               <keys in bundle code>
```

## Router — read the matching reference

| Work | Reference |
|---|---|
| Billing truth, attribution, recipes | [measure.md](references/measure.md) |
| SQL warehouses, result cache | [warehouses.md](references/warehouses.md) |
| Jobs, serverless modes, schedules | [jobs.md](references/jobs.md) |
| Lakeflow pipelines, streaming | [pipelines.md](references/pipelines.md) |
| Apps, Lakebase, dashboards | [apps-lakebase.md](references/apps-lakebase.md) |
| AI functions, serving, vector search | [ai.md](references/ai.md) |
| Caching and keys | [caching.md](references/caching.md) |
| Guards, budgets, alerting | [guards.md](references/guards.md) |
| Agents and CI as a cost source | [agents-ci.md](references/agents-ci.md) |
| Cheap-looking cuts that break things | [traps.md](references/traps.md) |

## When the bill jumps

1. Query, don't guess: run the daily cost by `billing_origin_product`, then by resource. → [measure.md](references/measure.md)
2. Split the window at any recent fix, and check git for a fix that already landed.
3. Check in this order:
   - idle warehouses (warm hours ≫ busy hours)
   - idle apps or endpoints
   - start-dominated jobs (many sub-minute runs)
   - paid calls recomputed on unchanged input
   - agent or CI traffic on user compute
   - untagged interactive compute
4. Fix the largest line with the smallest change that keeps quality. Use the cost card and re-measure.

## Never

- Claim a cost or a saving without a billing query.
- Choose classic compute, a cheaper model or a skipped check on cost grounds alone. The official
  serverless skill lists cost as *not* a reason for classic.
- Loosen a guard threshold to stop the noise.
- Ship a stop without its start, or a skip without its freshness check.
- Edit shared compute (warehouse size) outside its owner's process. Size is a cost decision.
