# databricks-cost-guardrails

**An agent skill for building on Databricks at the cheapest *correct* cost.**

Coding agents such as Claude Code and Cursor now build most of our Databricks work. The
[official Databricks agent skills](https://github.com/databricks/databricks-agent-skills) teach them *how*
to build: serverless, bundles, pipelines, apps, Genie. They say very little about **what it will cost
to run**:
- apps that never scale to zero
- warehouses that stay warm all day
- jobs that pay a compute start per task
- `ai_query` calls that re-send the same rows every night
- guards that fail open

This skill is that missing layer. It sits **on top of** the official skills and never replaces them:

> **Cheapest correct.** Correctness and the official Databricks pattern are floors. Cost is optimized
> *inside* them, never by going under them. A saving that weakens a number, freshness, or a guard is a
> cost, not a saving.

## What's inside

| File | What it gives the agent |
|---|---|
| `SKILL.md` | 10 ordered laws, a mandatory flow for every new service, the **cost card**, the "bill jumped" playbook |
| `references/measure.md` | Billing truth from system tables: the correct `usage × list_prices` join and recipes |
| `references/warehouses.md` | Idle burn, auto-stop per audience, right-sizing without spill, result-cache rules |
| `references/jobs.md` | Task-start economics, STANDARD vs PERFORMANCE_OPTIMIZED, gates, consolidation, schedules |
| `references/pipelines.md` | Triggered over continuous, staying incremental, degradation checks |
| `references/apps-lakebase.md` | Choosing the surface, app stop/start pairs, Lakebase scale-to-zero |
| `references/ai.md` | AI functions, model serving, vector search: cache by exact input, price from the SKU |
| `references/caching.md` | Cache-key discipline: too wide kills the hit rate, too coarse shows stale numbers |
| `references/guards.md` | Budgets don't cap; how to build a guard that cannot fail open |
| `references/agents-ci.md` | Your agents and CI are a cost source, and how to contain them |
| `references/traps.md` | Cost cuts that look cheap and silently break quality |

Every rule is either
- an official Databricks position (cited to the official skill or docs page), or
- a lesson from a real production workspace, with the mechanism and how to check it on yours.

It holds no company data or IDs. You put those in your own project instructions.

## The cost card

Every new job, pipeline, app or endpoint gets this card *before* code is written, and again in the PR:

```
Official skill(s):  databricks-serverless, databricks-jobs (v0.2.x)
Who waits?          nobody (nightly)  → serverless, STANDARD
Idles billed?       none
Starts per day:     1 task × 1 run
Paid calls:         ai_classify, cached by sha2(model|input), NULLs not cached
Estimate:           ~0.4 DBU/day (list-price estimate)
Quality guard:      freshness check on the output table
Measure:            daily cost by job_id — re-check in 7 days
Tags:               project, owner (in bundle code)
```

## Install

1. Install the official Databricks agent skills first:
   <https://github.com/databricks/databricks-agent-skills>. This skill routes to them.
2. Copy the skill folder:

   ```bash
   git clone https://github.com/DavidShukrun/databricks-cost-guardrails
   cp -r databricks-cost-guardrails/skills/databricks-cost-guardrails ~/.claude/skills/
   ```

   Any agent that supports the [Agent Skills](https://agentskills.io) format can load it.
3. Optional: add one line to your project's `CLAUDE.md` / `AGENTS.md`:
   > Any Databricks work: load the official `databricks-core` + product skill, then
   > `databricks-cost-guardrails`. Fill the cost card before building.

Put your own facts (warehouse IDs, budget caps, tag taxonomy, who may edit shared compute) in that
project file. The skill stays generic.

## Freshness

Checked against `databricks-agent-skills` **v0.2.50** and docs.databricks.com as of **November 2026**.
The platform moves fast. Where this skill and a newer official skill or doc disagree, **the official
source wins**. Open an issue or PR.

## License

MIT
