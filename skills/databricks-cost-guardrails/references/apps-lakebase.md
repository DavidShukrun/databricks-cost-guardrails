# Apps, Lakebase, dashboards — choosing the surface

Official basis: `databricks-apps` (platform guide), `databricks-lakebase`, `databricks-aibi-dashboards`, and
the docs pages on app compute size and Lakebase scale-to-zero.

## Cheapest correct surface first
Ask what the user needs before choosing a surface:
- **Read-only numbers or charts:** use an AI/BI (Lakeview) dashboard over a materialized table. It costs
  only the queries run while someone has it open, and a plain "create a dashboard" request maps to it
  officially.
- **Natural-language questions:** use a Genie space or agent on the user warehouse.
- **Writes, workflow, custom UX:** use an app. It is the most expensive surface, because it bills by the hour.

## Apps
- **Billed per hour while running, by provisioned size** (docs). Medium = 0.5 DBU/h (the default and
  the floor), Large = 1 DBU/h, XLarge = 3 DBU/h. **There is no scale-to-zero.** A Medium app that nobody
  opens still burns 12 DBU a day. In one workspace an idle app was 88% of the bill. Horizontal scaling
  adds instances; it never removes the floor.
- **The only lever is stop/start, shipped as a pair.**
  - Ship a business-hours stop and its morning start in **one** change.
  - A stop without a start is an outage. A start without a stop is a leak.
  - A safe pattern is to land the stop active and the start paused, then enable the start as one reviewed
    line.
- **Make the stop guard poll.**
  - `apps stop` is accepted only from ACTIVE.
  - STOPPING is not a resting state. Poll until STOPPED and fail when it never gets there.
  - Never `stop || true`.
  - In the Python SDK, compare `state.value`, not `str(state)`, because enums are plain `Enum`.
- A bare `bundle deploy` creates the app stopped, while `apps deploy` starts it. Know which one your CD
  runs, and make any deploy-time stop guard use the same hours as the shutdown schedule.
- Pick the size from measured memory and CPU, never from habit. Large doubles the floor.
- Attribute by `usage_metadata.app_name`. Apps cannot take bundle compute tags; use a serverless usage
  policy for attribution.

## Lakebase (Autoscaling)
- Never create the retired provisioned Lakebase (official).
- **Scale-to-zero is on by default, but the default timeout is 24 hours** (docs). It saves little until you
  shorten it. Use 60 seconds to 30 minutes for dev and branch computes. Wake-up takes a few hundred
  milliseconds.
- Scale-to-zero is not available with HA or above 32 CU. Size the minimum CU to the working set, not
  higher.
- Delete dev branches when they are done.

## Dashboards
- Point dashboards at the user warehouse and at small pre-aggregated tables.
- Turn off auto-refresh unless someone actually watches it live.
- Scheduled dashboard snapshots or emails each wake the warehouse, so count them as runs.
