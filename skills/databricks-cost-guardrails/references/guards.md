# Guards, budgets, alerting

Official basis: `databricks-unity-gateway` (budgets) and the docs pages on serverless usage policies and
serverless jobs.

## What does NOT cap spend (official)
- **Budgets alert; they do not cap or block.** Alerts arrive by email, at list price, with lag.
- **Serverless usage policies** (formerly budget policies) **tag usage for attribution**. They do not
  enforce a limit.
- **A compute policy limits autoscaling but does not cap total job cost** (docs).

So a hard cap must be built by you, from levers that act on compute state.

## Enforcement levers, cheapest first
1. Auto-stop on every warehouse; see [warehouses.md](warehouses.md).
2. Schedules: business-hours stop and start pairs for apps; see [apps-lakebase.md](apps-lakebase.md).
3. Nothing always-on without an owner (law 4).
4. A **guard job** for whatever is left.

## Guard design — it must not fail open
A real guard from the field never caught a single leak, because of four defects:
1. It read yesterday's billing only.
2. Its threshold had been loosened 5× to quiet alerts.
3. An auth error made the check `exit()` with success, which skipped enforcement.
4. It had no workspace filter.

So:
1. **Enforce on compute state** (what is running now). Use billing only as a second opinion, since it
   lags about 12 hours.
2. **Enforcement runs first, and there is one exit at the end.** No early `exit()` or `return` before the
   stop runs.
3. **A failed read raises.** Never treat "could not read" as "nothing to do". Match error types, not
   substrings.
4. **Assert the workspace.** Billing is account-wide, but a stop acts only locally.
5. **Never loosen a threshold to stop noise.** Fix the noise source. A cap set above normal spend plus
   the leak size catches nothing.
6. **Use two layers:** a daily spike cap, plus a monthly run-rate projection (trailing 7 days × 30/7)
   against the cap.
7. **Make every stop poll to a resting state** (STOPPED), and fail when it does not get there.
8. **A guard is cheap.** An hourly state check costs cents a day, so run it often enough to matter.

## Alerting that is real end to end
An alert counts only when it is **configured, fires, is sent, is delivered and is seen**. Test all five.
- Alerts filtered into a mail folder or trash are dead alerts.
- Prefer sinks a person and an agent can both read back, such as a webhook or a channel.

## Dead-man checks
- Look for "job X has not succeeded within its SLA" with a **LEFT JOIN from the expected-jobs manifest**,
  counting NULL as failure. An inner join from the run timeline cannot see a job that never ran.
- Derive SLAs from observed run cadence. Do not hand-type constants.
- A gate whose pass state only the gate itself can write deadlocks at its first failure. Give it an
  external reset or baseline.

## Governance
- Shared compute settings, such as warehouse size, auto-stop and max clusters, have **one owning
  process**.
- A size change is a cost decision and needs explicit approval. Shared identities make rogue edits
  untraceable, so the process is the only control.
- Audit config changes via `system.access.audit`.
