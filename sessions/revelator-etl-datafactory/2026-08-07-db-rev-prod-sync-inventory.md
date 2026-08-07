# db-rev-prod → Snowflake sync inventory (ADF) — 2026-08-07

**Repo:** revelator-etl-datafactory + revelator-analytics-datafactory (read-only analysis, branch `origin/develop`)
**Branch:** n/a (no code changes; both repos' `master` holds only a README — all ADF resources live on `develop`)

## What we did

- Inventoried every extraction from **db-rev-prod** (`quango-db.database.windows.net`, LS `LS_Azure_SQL_Revenue` in etl factory; `LS_RevDb` env-parameterized in analytics factory) across both Data Factory repos by statically parsing linkedService/dataset/dataflow/pipeline/trigger JSON via `git show origin/develop:...`.
- Mapped each SQL source (dataset table or inline/dynamic dataflow query — regex over dataflow script blocks split at `~> name` anchors) to its consuming dataflows and pipelines, including Lookup activities and dynamic `query:` expressions with `$p_Last_Load_Date`/`$p_Tenant_Id` params.
- Built trigger ancestry: parsed all 17 triggers and walked `ExecutePipeline` parent chains to classify every extraction pipeline as scheduled / stopped-trigger / unscheduled.
- Resolved sink targets to fully qualified Snowflake names (account `vva53450`, DBs `REVELATOR_BI` and `REVELATOR_ALL`) from sink datasets and inline Snowflake sinks.
- Resolved the 3 `ADF_*` SQL views (user pasted definitions) to base tables → final **25 distinct base tables** actually synced on active schedules.
- Estimated the Airbyte + Snowflake-SP replacement effort: **6–10 weeks realistic, 4–6 aggressive** for one domain-familiar dev.
- Deliverable: **`C:\Users\vasil\db-rev-prod_scheduled_syncs.xlsx`** — 7 tabs: Scheduled syncs (39 flows), Base tables (25), ADF_ views resolved, Dataflows (~50 incl. unscheduled), Triggers, Airbyte migration estimate + risks, Notes.

## Key findings

- Active trigger chains driving prod extraction: `TR_Main_PL_6PM` (18:00), `TR_Transform_All` (15:00 → PL_REV_00_Startup_Check), `TR_DB_Analytics_Prod` (17:00 → PL_00_V2_Orchestrator), `TR_Ingest_All` (04:00/09:00), `TR Weekly` (Sun 13:00, ref tables), `TR_Match_Merlin_Apple_IDs` (07:00), `TR_YouTube_DataPro` (Sun 13:00), `TR_Validation_PL` (20:00). Times mostly FLE (EET).
- `revelator-analytics-datafactory` has **no triggers at all** (checked develop + adf_publish ARM) — its 2 rev-db pipelines are manual-only.
- `TR_Db_Rev_Prd_Sync` (operations sync: SaleStatements/UserStatements/UserStatementsQueue) is **Stopped**; several dataflows (Revenue_To_Snow_All_Tenants, Not_Approved_Revenue_V2, Royalties_UserStatements) referenced by **no pipeline**.
- 25 distinct base tables synced on schedule; `ADF_AllAssets_RecVersion` → Releases/Releases_Tracks/Tracks/Artists/Enterprises/TrackRecordingVersions/Labels; `ADF_Videos` → Releases/Labels/Artists/ReleaseTypes (type 5); `ADF_YouTubeVideos` → YouTubeVideos/YouTubeChannels/MultiChannelNetworks.
- Some asset queries reference the DB cross-database as `[db-rev-prod].dbo.ADF_AllAssets_RecVersion`.

## Key decisions

- Estimate framing: simple flows ~15 × 0.5d; assets/catalog ~8 × 1d; revenue/statements ~10 × 1.5–2d (hardest: watermarks, negative-StatementId UNIONs, delete-detection); orchestration/cutover 1–2 wks.
- Top migration risks flagged: (1) hard deletes need CDC or periodic full refresh — decide first; (2) `SaleItemsAllApproved` / `Releases_Tracks` may themselves be views (unverified); (3) watermark state must move from ADF params to a Snowflake control table.

## Files changed

| File | Change |
|---|---|
| `C:\Users\vasil\db-rev-prod_scheduled_syncs.xlsx` | Created — 7-tab inventory + migration estimate (outside git) |

## Still to do / follow-up

- Verify whether `dbo.SaleItemsAllApproved` and `dbo.Releases_Tracks` are tables or views; if views, resolve base tables.
- Confirm live trigger states in the ADF UI (git `runtimeState` may be stale).
- Decide Airbyte delete-propagation strategy (CDC vs full refresh) for statement tables before committing to the migration estimate.
