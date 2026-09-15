# Video Analytics PROD deploy plan v2 (live-verified) + checkpoint annotations — 2026-09-15

**Repos:** Revelator-BI-Snowflake, revelator-analytics-snowflake, RepositoryDataAccessor.ReportIngestion, RepositoryDataAccessor.Analytics, Netcore.Analytics, quango-platform
**Continues:** [2026-09-14 video analytics PR rollout](2026-09-14-video-analytics-prs-and-v1-revenue-route.md)

## What we did

**Rewrote the PROD deploy plan against live-verified current state** (all requested: pull every repo's main branch, plan zero-downtime PROD rollout, fold in the colleague's UGC royalty-estimates work).

- Pulled `develop`/`main`/`develop/platform` fresh in all 6 repos; confirmed via `gh pr view` that every PR from the 2026-09-14 rollout is now merged (Revelator-BI-Snowflake Azure DevOps PR 304, revelator-analytics-snowflake #80, ReportIngestion #56, Analytics DAL #43/#44, Netcore.Analytics #88/#89/#90/#93, quango-platform #35). Nothing left to merge for the feature itself — deploy-only from here.
- **Queried PROD SingleStore directly** (read-only, reusing the existing `REV_ANALYTICS_API_PROD` credential from `C:\Temp\CheckProd`) via a new throwaway inspector project in the scratchpad: confirmed zero of the new columns (`ESTIMATEDREVENUE`, `ASSETTYPEID`) exist anywhere in PROD yet, across all `TRANSACTIONS_BY_*`/`UGC_BY_*`/`REVENUE_BY_*`/`USER_STATEMENTS_BY_*` tables and views (`_A`/`_B` + live view) — and this also confirmed the PROD SingleStore DB name (`REV_ANALYTICS_API_PROD`), resolving a v1 open item.
- **Read every CI/CD workflow file** instead of assuming deploy mechanics:
  - The three .NET/DAL repos: GitHub Actions, ArgoCD-templated, auto-deploy-to-dev on merge (auto-tags), then manual `workflow_dispatch` "3 Promote to env" (stage/prod) — confirmed multi-replica + PDB on the DAL (`minReplicas: 8`), so a rolling promote is zero-downtime by construction.
  - quango-platform: legacy WebAPI2/Azure Web App, **not** k8s. Found two non-obvious facts: (1) `api-to-prod.yaml` deploys to a `ghdeploy` *slot* but has **no automatic slot-swap step** (unlike staging's workflow, which does) — production traffic doesn't cut over until someone manually swaps the slot; (2) PR #35 merged into `develop/platform`, but PROD deploy triggers only on merge to `master`, and the two branches have diverged (28 vs 7 commits) — getting the change to prod needs either a full `develop/platform→master` merge or a narrow cherry-pick of the single commit, flagged as the user's call, not executed.
- **Investigated "royalties estimates for UGC, already in prod"** (user's phrasing): traced to `revelator-analytics-snowflake` commit `5a31150` (Martin Tsvetkov, "RPS"/Revenue Daily Estimates), found it unmerged and, per the user's clarification, not executed anywhere — code changes only. It touches the **same six transactions AGG tables** this workstream's Consumption feature touches, with a real positional-column-order dependency (`..., ESTIMATED_REVENUE, ASSET_TYPE_ID`) that this workstream's own PROD BASE DDL already correctly assumes. Documented as a coordination section (not something this plan executes) — also surfaced two inconsistencies found while reading Martin's own RPS files (a referenced "section 5" UGC-ALTER block missing from `_DDL_RPS_TABLES.sql`; a stale warning about `SP_PY_POPULATE_TENANT_TABLE`, which [[analytics-tenant-mirror-loader-retired]] confirms no longer runs).
- **Added explicit checkpoint annotations** per user request (second follow-up ask): went through all 10 steps and classified every one as a safe stopping point or not. Found and fixed a real sequencing bug in the process — the BASE name-map DDL steps were documented as running *after* the pipeline recreate; the files' own header comments say table-columns → BASE DDL → pipelines, the opposite. Also corrected the "DAL+API simultaneity" framing: it's an *ordering* requirement (DAL must not lag API), not true simultaneity — split into two independently-safe checkpoints (10a(i) DAL, 10a(ii) API) with only one hard rule: never deploy the API before the DAL.
- Net result: of 10 steps (11 counting the DAL/API split), **all but the last are safe to pause after indefinitely** — the system behaves exactly as today until the API promotion step, which is the only "feature goes live" moment, and even that has a clean rollback.

**Committed the plan** — both `2026-09-10-video-analytics-asset-type-prod-deploy.md` (marked superseded) and the new `2026-09-14-video-analytics-prod-deploy-v2.md` — to a fresh branch (`vasil/prod-deploy-plan-video-analytics`) off `develop`, pushed; gave the user the Azure DevOps create-PR link (no CLI access to that org this session).

## Key decisions

- Checkpoint table lives in a new §6.0 in the plan doc, decoupled from the existing step numbers (renumbering all 10 steps' cross-references was judged riskier than adding an execution-order table on top).
- RPS/UGC coordination is documented as a heads-up for whoever deploys it, not something this plan takes ownership of deploying — it's unmerged, untested end-to-end, and not this workstream's feature.
- quango's `master`-promotion path (bulk `develop/platform→master` merge vs. narrow cherry-pick) is left as an open decision for the user/team, not decided unilaterally.

## Files changed
| File | Change |
|---|---|
| `Revelator-BI-Snowflake/docs/deploy-plans/2026-09-10-video-analytics-asset-type-prod-deploy.md` | Marked superseded by v2, pointer link added |
| `Revelator-BI-Snowflake/docs/deploy-plans/2026-09-14-video-analytics-prod-deploy-v2.md` | New — full re-verified plan, §3 RPS coordination, §5 per-repo deploy mechanism, §6.0 checkpoint table, ordering bug fix in Steps 3/4, DAL/API split in Step 10a |

Branch `vasil/prod-deploy-plan-video-analytics` pushed to `origin`, PR not yet opened (Azure DevOps, no CLI this session — link given to user).

## Still to do / follow-up
- Open the Azure DevOps PR (link given to user): `https://dev.azure.com/Revelator-BI-Org/Revelator-BI/_git/Revelator-BI-Snowflake/pullrequestcreate?sourceRef=vasil/prod-deploy-plan-video-analytics&targetRef=develop`
- Execute the plan itself (still entirely unexecuted beyond the already-done Step 1) — user reviews and chooses where to start/stop using the new checkpoint table.
- Resolve v2's open items: RESOURCE POOL discrepancy sign-off, quango `master`-promotion path choice, Netcore.Analytics API prod replica count confirmation, deploy-window scheduling, RPS sequencing coordination with Martin.
