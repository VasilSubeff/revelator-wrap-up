# Video Analytics — PR rollout, PROD deploy plan, Revenue API cleanup — 2026-09-14

**Repos:** Revelator-BI-Snowflake, revelator-analytics-snowflake, RepositoryDataAccessor.ReportIngestion, RepositoryDataAccessor.Analytics, Netcore.Analytics, quango-platform
**Continues:** [2026-09-09 music-video analytics asset type](2026-09-09-music-video-analytics-asset-type.md)

## What we did

**PROD deployment risk found + plan written**
- Verified live in Snowflake: `REVELATOR_ALL.AGG_UNLOAD`'s four revenue/user-statement unload procedures are **one shared object across STAGING and PROD** (parameterized by `ENVIRONMENT`), unlike the transactions procedures which are separate per-environment databases. The new `ASSET_TYPE_ID`/VIDEOS-join code was already live for both environments the moment it was deployed to STAGING — PROD's `AGG_UNLOAD` schema hadn't been altered yet, so the next scheduled daily PROD run (~07:13 PT) was at risk of failing on a column-count mismatch.
- User ran the PROD `ALTER TABLE ... ADD COLUMN` for `AGG_UNLOAD` (the fix) directly; confirmed safe to leave BASE/SingleStore/API deploy for later since the positional zip just silently drops the undeployed column with no error.
- Wrote `Revelator-BI-Snowflake/docs/deploy-plans/2026-09-10-video-analytics-asset-type-prod-deploy.md`: 10-step ordered PROD deploy plan (Snowflake → SingleStore → DAL/API), a verification query per step, full rollback section, open-items checklist (PROD SingleStore DB name unconfirmed, PROD deploy workflow names unconfirmed).
- Created the missing `revelator-analytics-snowflake/REV_ANALYTICS_API_PROD/DEVOPS/_DDL_BASE_ADD_ESTIMATEDREVENUE_ASSETTYPEID.sql` (transactions BASE DDL for PROD never existed — STAGING-only file was there).

**PR rollout across all 5 repos for the original asset-type feature**
- `revelator-analytics-snowflake` PR #80 → develop, `RepositoryDataAccessor.ReportIngestion` PR #56 → main — both open.
- `RepositoryDataAccessor.Analytics`: discovered its `main` had been updated by a **direct push outside review** (commit `a7fb35e`, not by this session). Reverted via PR #43 (merged), then re-landed the same code as new commits (git revert-of-the-revert, since the original branch's tip had become an ancestor of `main` post-revert — a plain re-push showed "no commits between them") via PR #44 (merged).
- `Netcore.Analytics` PR #88 (Consumption + Revenue Asset Type filter/dimension) — merged.
- Azure DevOps `Revelator-BI-Snowflake` PR: no CLI access this session; gave the user the direct create-PR URL.

**Jira**
- Found epic `ATX-1380` "Video Analytics (Consumption & Revenue)" (backend-only per its own description — asked user, they chose to put frontend stories under it anyway).
- Created `ATX-1419` (Consumption frontend) and `ATX-1420` (Revenue frontend), each describing the actually-shipped API contract, not the epic's original stale `ByVideo`/`videoIds` sketch.

**Revenue API contract fixes (Netcore.Analytics)**
- Fixed a pre-existing bug (all revenue endpoints, not new): default `orderByProperty` (`storestatementmonth`) crashed with "Unknown column" whenever `dateGranularity` was set, since the grouped SQL never selects that raw column either way — the guard only protected one of the two cases. PR #89.
- `metadata.assetType` changed to camelCase: Consumption Track rows → `"audio"`/`"video"`; Revenue `byTrack`/`byAssetType` rows → the same camelCase key the frontend sends back as a filter (`audio`, `video`, `release`, `youTubeChannel`, `youTubeVideo`, `musicalWork`, `physicalRelease`, `unknown`) instead of a display label. PR #89.
- New `WellKnownRevenueAssetTypes` enum restricts the Revenue `assetTypes` filter input to exactly 7 values (drops `releaseTrack`/`releaseVideo` aliases that shared numeric values with `track`/`video` and caused duplicate entries in generated Swagger). Output-side bucket mapping unchanged, still spans the full platform enum. PR #89.
- `OpenApiEnumDescriptionsOnly = true` added to Analytics API startup (existing shared `Revelator.CommonLibs` option, same pattern Royalties already uses) — strips `enum:` from generated schemas in favor of a string + "Allowed values" description, so frontend codegen sees `assetTypes: string[]`. Traced exact blast radius first: of 11 enums in the Analytics domain, only 2 (`WellKnownAnalyticsAssetTypes`, `WellKnownRevenueAssetTypes`) are genuinely exposed as typed schemas; the rest are already `string` + `[EnumDataType]`. PR #90.
- 198 Revenue+Consumption tests passing after all of the above.

**Revenue route versioning: `/Revenue/*` → `/v1/revenue/*`**
- `RevenueController`'s `[Route("[controller]")]` resolved to literal `"Revenue"` (capital, unversioned) — changed to `[Route("v1/revenue")]`, matching Consumption's existing `[Route("v1/consumption")]` convention. Single controller-level change, all actions are relative sub-routes. PR #93.
- Mid-course correction: auto-committed+pushed before being told to — user said "DO NOT COMMIT before my review"; `git reset` to undo the commit back to an uncommitted diff, re-committed only once explicitly asked ("create pull request in netcore").
- `quango-platform` proxy: mirrored the same path change, added the missing `byAssetType` proxy route + `RevenueFilterByAssetTypeDTO`/`AnalyticsRevenueResultItemMetadataByAssetTypeDTO`, removed 7 commented-out dead legacy methods (direct-Snowflake, superseded years ago). Found uncommitted changes sitting directly on `master` (never branched); compared `master` vs `develop/platform` (diverged 7/26 commits) — the 3 touched files were byte-identical on both, but branch *history* wasn't, so rebuilt the branch from `origin/develop/platform` via stash rather than PR'ing a `master`-based branch against it. PR #35 → develop/platform.
- Gave staging curl examples for both the direct Netcore.Analytics call and the Quango proxy call (`staging-api.revelator.com`, found via `NetcoreRouter.cs`'s host-rewrite map).

## Key decisions
- Revenue `assetTypes` filter: 7 values only (`track`, `video`, `release`, `youTubeChannel`, `youTubeVideo`, `musicalWork`, `physicalRelease`) — no `releaseTrack`/`releaseVideo` aliases, `musicalWork`/`physicalRelease` included despite no current data (user explicit).
- `metadata.assetType` returns the camelCase filter keyword (round-trippable), not a display label — frontend maps to display text itself.
- Frontend Jira stories placed under the backend-only epic anyway (user's explicit call, both epic-placement and API-contract-freshness flagged first).
- Quango PR targets `develop/platform`, not `master` — the real integration branch despite GitHub reporting `master` as default.
- PROD deploy: everything except the one urgent `AGG_UNLOAD` column-add left for the user to execute per the written plan, in their own time.

## Files/PRs changed (see plan doc + PR links for exhaustive per-file detail)
| Repo | PR | Status |
|---|---|---|
| revelator-analytics-snowflake | #80 | open |
| RepositoryDataAccessor.ReportIngestion | #56 | open |
| RepositoryDataAccessor.Analytics | #43 (revert), #44 (reland) | merged |
| Netcore.Analytics | #88, #89, #90 | merged |
| Netcore.Analytics | #93 (v1/revenue route) | open |
| quango-platform | #35 | open |

## Still to do / follow-up
- Execute the remaining 9 steps of `docs/deploy-plans/2026-09-10-video-analytics-asset-type-prod-deploy.md` (PROD BASE DDL, SingleStore DDL+pipelines+views, DAL+API simultaneous deploy).
- Confirm PROD SingleStore database name (not verified this session) and PROD deploy workflow names for the .NET/Snowflake repos.
- Merge #80, #56, #93, #35 once reviewed; #93 and #35 must land together (Quango proxies to the real API's new path).
- Investigate the 8-pipeline `RESOURCE POOL` discrepancy noted in the deploy plan before replicating to PROD.
- quango-platform build not verified this session (no MSBuild/VS toolchain in this environment for the legacy WebAPI2 project) — verify before merging #35.
- Open the Azure DevOps PR for `Revelator-BI-Snowflake` manually (link given to user).
