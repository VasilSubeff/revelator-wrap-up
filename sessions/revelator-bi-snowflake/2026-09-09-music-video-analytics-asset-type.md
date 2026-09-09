# Music-video analytics: Apple video catalog, DSP video routing, ASSET_TYPE_ID chain to SingleStore — 2026-09-04 → 09-09

**Repos:** Revelator-BI-Snowflake (`develop`), RepositoryDataAccessor.Analytics (`vasil/apple-mismatch-videos`, PR #42), revelator-analytics-snowflake (`develop`), RepositoryDataAccessor.ReportIngestion (`main`)
**Environments touched:** Snowflake `REVELATOR_ALL` / `REVELATOR_ALL_STAGING` / `REVELATOR_BI` (PROD account vva53450), `REVELATOR_ANALYTICS_DEV` / `_STAGING`, `REV_ANALYTICS_API_DEV` / `_STAGING`; SingleStore `REV_ANALYTICS_API_STAGE`

## What we did

### 1. Apple video catalog (Revelator-BI-Snowflake)
- **`SP_REFRESH_APPLE_CATALOG_APPLEID_MISMATCHES_VIDEOS`** (committed `98591ea`): video twin of the track mismatch SP. Sources: `LANDING.APPLE_CONTENT_DETAILED_LOAD` + latest Merlin feed, both `MEDIA_TYPE = 2`; catalog = `REVELATOR_ALL.STAGE.CATALOGVIDEOS` joined to `STAGE.RELEASES_VIDEOS` on VIDEO_ID + RELEASE_ID. ISRC fallback not gated on the external usage id; `NOT EXISTS` in direct-matched + `QUALIFY COUNT(DISTINCT RELEASE_ID) OVER (PARTITION BY VIDEO_ID) = 1`. Output table in REVELATOR_BI.STAGE, REVELATOR_ALL.STAGE, REVELATOR_ALL_STAGING.STAGE (84 rows: 80 Apple, 4 Merlin).
- **`SP_REFRESH_APPLE_CATALOG_VIDEOS_V1`** (new, deployed PROD, uncommitted): video twin of `SP_REFRESH_APPLE_CATALOG_V1` → `REVELATOR_ALL.STAGE.APPLE_CATALOG_VIDEOS` (VIDEOTYPE cast NUMBER, only rows with non-null APPLE_ID, Apple direct+ISRC and Merlin UPC+ISRC branches, SOURCE_PRIORITY tie-break).
- **Analytics DAL PR #42**: `AppleCatalogAppleIdMismatchVideo` model + `appleCatalogAppleIdMismatchVideos` GraphQL query (offset paging, VideoId instead of TrackId). Postman collection at `C:\Temp\AppleCatalogAppleIdMismatches.postman_collection.json`.

### 2. Route DSP music-video streams into VIDEO_VIEWS (Revelator-BI-Snowflake, STAGING then PROD)
- **Spotify** `SP_LOAD_SPOTIFY_TRENDS_REPORTPATH_V1`: second match of parsed rows against CATALOGVIDEOS where `CONTENT_TYPE_FLAG = 'video'` (UPC via `TRY_TO_NUMBER`) → insert into `STAGE.VIDEO_VIEWS` (video id in TrackId, 20 cols incl. new `SOURCE_OF_STREAM_ID`); audio inserts filter `IFNULL(CONTENT_TYPE_FLAG,'') != 'video'`; pre-stage keeps all unmatched content types (audio ∪ video).
- **Apple** `SP_LOAD_APPLE_TRENDS_REPORTPATH_V1`: media lookup temp from `APPLE_CONTENT_DETAILED_LOAD` → per Apple ID `IS_MUSIC` / `IS_VIDEO` flags (MEDIA_TYPE '1' / '2'); TRANSACTIONS gets `IS_MUSIC = 1`, VIDEO_VIEWS gets `IS_VIDEO = 1` via deduped `APPLE_CATALOG_VIDEOS` + CATALOGVIDEOS. An ID present as both types goes to BOTH (no collapsing). Delivery type 102/109 same as audio.
- **Merlin** `SP_LOAD_MERLIN_APPLE_TRENDS_REPORTPATH_V1` (+ new STAGING copy): same two-flag approach; media lookup built once over the whole Merlin content feed before parse (1.9 s vs 24 min for the IN-subquery variant); tenant filter widened with `OR IS_VIDEO = 1` so video rows survive the track-catalog tenant join.
- `SP_LOAD_VEVO_VIDEO_VIEWS_REPORTPATH` → explicit 19-column insert list; `SP_PROCESS_VIDEO_VIEWS_V1` (PROD + new STAGING) carries `SOURCE_OF_STREAM_ID`; `SP_PROCESS_VIDEOS` carries `VIDEO_RECORDING_TYPE`. Helper ALTER scripts for `STAGE.VIDEO_VIEWS` (both envs) and `TENANTS.VIDEOS`.
- Landing data for Apple 1/61/20260906 and Merlin 127038/961/20260906 copied to `REVELATOR_ALL_STAGING.LANDING`; user ran all loaders manually. Validation via read-only template extraction: Merlin 26,967 → 26,994 rows when an ID is both types; Apple 1/61 1,621,946 → 1,621,953.

### 3. ASSET_TYPE_ID through the analytics aggregation (revelator-analytics-snowflake, DEV + STAGING)
- `SP_CREATE_BASE_TRANSACTIONS`: `1 AS ASSET_TYPE_ID` for tracks + `UNION ALL` from `REVELATOR_ALL.TENANTS.VIDEO_VIEWS` (`VV_REVELATOR_ASSET_ID`, `VIDEO_VIEWS` as quantity, `6 AS ASSET_TYPE_ID`, `FALSE AS TRANSACTION_IS_SUB30`, same tenant/retention/`DSP_ANALYTICS_DISTRIBUTION = 'Transactions'` filter).
- `SP_CREATE_BASE_DATA`: ASSET_TYPE_ID through all three branches (1 for downloads/saves); revenue estimate IFF NOT gated by asset type.
- Six `SP_AGGREGATE_{TCD,DSP,DT,SOS,TRV,CT}_TRANSACTIONS`: `LEFT JOIN TENANTS.ASSETS ON ASSET_TYPE_ID = 1` + `LEFT JOIN TENANTS.VIDEOS ON ASSET_TYPE_ID = 6 AND TRANSACTION_ASSET_ID = REVELATOR_ASSET_ID`; COALESCE'd attributes (TRACK_ID ← VIDEO_ID); TRV recording type `COALESCE(A.ASSET_RECORDING_VERSION_TYPE, V.VIDEO_RECORDING_TYPE)`; new last column `AGG_x_TRN_ASSET_TYPE_ID`. Orchestrator restored to call `SP_CREATE_BASE_TRANSACTIONS` first.
- DDL run: `_DDL_ASSET_TYPE_TRANSACTIONS.sql` (DEV + STAGING), RPS tables/columns in DEV (`_DDL_RPS_TABLES.sql`, was missing), `REV_ANALYTICS_API_<env>.BASE.TRANSACTIONS_BY_*` + `ESTIMATEDREVENUE NUMBER(38,6)`, `ASSETTYPEID NUMBER(38,0)` (5 tables × DEV + STAGING). All nine procedures deployed to DEV and STAGING; **orchestrators not run** (user runs them).
- Read-only chain validation on tenant 135664 (Vevo rows as video stand-in): totals reconcile, 172 video rows per aggregate.

### 4. SingleStore publishing chain (RepositoryDataAccessor.ReportIngestion)
- Established the chain: AGG table → `REV_ANALYTICS_API_<env>.BASE` column map (positional zip in `SP_UNLOAD_TRANSACTIONS_TO_INACTIVE`) → parquet on S3 → `PIPE_<TABLE>_A/B` → `<TABLE>_A/_B` → `FLIP_VIEWS`. Per-tenant mirror loader `SP_PY_POPULATE_TENANT_TABLE` is retired (memory saved). `TRANSACTIONS_BY_DISTRIBUTOR` is not published.
- Table DDL (5 files) + `_ALTER_TRANSACTIONS_ADD_ESTIMATEDREVENUE_ASSETTYPEID.sql` (20 ALTERs with `AFTER …` positioning) — user executed in STAGE and flipped twice. Views need no change (they are `SELECT *` re-created by FLIP_VIEWS).
- Pipeline definitions: `ESTIMATEDREVENUE` / `ASSETTYPEID` mappings added to the 10 transactions pipelines; `RESOURCE POOL pipelines_pool` added to all 30 remaining files (live had it on 22 of them; 8 ARTIFICIAL_STREAMS/PLAYLIST/STREAMS_REVENUE_MOVEMENTS pipelines gained it).
- **All 40 STAGE pipelines recreated** (2026-09-09): STOP → CREATE OR REPLACE → `ALTER PIPELINE … SET OFFSETS LATEST` → START. All Running, file/cursor bookkeeping and all 40 table row counts unchanged.

### 5. ADF / RPS guidance
- RPS monthly build = `REVELATOR_ANALYTICS_<env>.AGG_DATA.SP_AGGREGATE_RPS_ORCHESTRATOR()` (statements → consumption → rates). STAGING fully deployed (562,387 rate rows, scope 102/103 STREAM, 120 VIEW). DEV lacks the scope seed and three RPS procedures; PROD has none.
- ADF: If Condition `@equals(dayOfMonth(pipeline().TriggerTime), 1)` (or `convertFromUtc` for local zone), True branch = the CALL, placed before the daily transactions orchestrator; optional `forceRps` parameter.

## Key decisions

- Asset type convention: 1 = track, 6 = video (`REVELATORASSETTYPEID`, `CONFIG.AGG_DSP_FILTER`). Video recording type is derived from `TENANTS.VIDEOS` in the TRV aggregate only, not carried in base rows.
- Discovery type explicitly out of scope ("group of source of stream, added downstream"); only `SOURCE_OF_STREAM_ID` added. Listeners out of scope.
- Duplicated Apple IDs (music + video) are NOT collapsed — both fact tables get the row.
- Pre-stage receives every unmatched content type. Movement SPs disregarded.
- `SET OFFSETS LATEST` after every `CREATE OR REPLACE PIPELINE`: the cursor was observed to reset despite docs, which would reload every file already in the tables.
- Secrets never embedded in shell commands (auto-mode classifier blocks them); kept in scratch files and deleted after use.

## Files changed (all uncommitted unless noted)

| Repo / file | Change |
|---|---|
| BI-Snowflake `Stored Procedures/Catalog Matching/SP_REFRESH_APPLE_CATALOG_APPLEID_MISMATCHES_VIDEOS.sql` | New, committed 98591ea |
| BI-Snowflake `Landing V1/Apple/SP_REFRESH_APPLE_CATALOG_VIDEOS_V1.sql` | New, deployed PROD |
| BI-Snowflake `Landing V1/Spotify/{PROD,STAGING}/SP_LOAD_SPOTIFY_TRENDS_REPORTPATH_V1.sql` | Video match + VIDEO_VIEWS insert + SOURCE_OF_STREAM_ID |
| BI-Snowflake `Landing V1/Apple/{PROD,STAGING}/SP_LOAD_APPLE_TRENDS_REPORTPATH_V1.sql` | IS_MUSIC / IS_VIDEO routing, deployed both |
| BI-Snowflake `Landing V1/Apple/SP_LOAD_MERLIN_APPLE_TRENDS_REPORTPATH_V1.sql` + `STAGING/` copy | Same for Merlin, deployed both |
| BI-Snowflake `Landing V1/UGC/Vevo/SP_LOAD_VEVO_VIDEO_VIEWS_REPORTPATH.sql` | Explicit column list, deployed PROD |
| BI-Snowflake `ADF V1/Facts/UGC/SP_PROCESS_VIDEO_VIEWS_V1.sql` + new `ADF V1/Facts/STAGING/` copy | + SOURCE_OF_STREAM_ID |
| BI-Snowflake `ADF V1/Dimensions/UGC/SP_PROCESS_VIDEOS.sql` | + VIDEO_RECORDING_TYPE (user redeployed) |
| BI-Snowflake `Helper Scripts/{STAGING,PROD}_VIDEO_VIEWS_ADD_SOURCE_OF_STREAM.sql`, `TENANTS_VIDEOS_ADD_VIDEO_RECORDING_TYPE.sql` | ALTER scripts (applied) |
| DAL Analytics `WmgAnalytics/Models/AppleCatalogAppleIdMismatchVideo.cs`, `WmgAnalytics/GraphQLOperations/AnalyticsQuery.cs` | PR #42 (open) |
| analytics-snowflake `REVELATOR_ANALYTICS_{DEV,STAGING}/AGG_DATA/TRENDS/TRANSACTIONS/*` (9 SPs + `_DDL_ASSET_TYPE_TRANSACTIONS.sql`) | ASSET_TYPE_ID chain, deployed |
| analytics-snowflake `REV_ANALYTICS_API_{DEV,STAGING}/DEVOPS/_DDL_BASE_ADD_ESTIMATEDREVENUE_ASSETTYPEID.sql` | Executed |
| DAL ReportIngestion `AnalyticsPipelines/Sql/Tables/TRANSACTIONS_BY_*.sql` (5) + `_ALTER_TRANSACTIONS_ADD_ESTIMATEDREVENUE_ASSETTYPEID.sql` | SingleStore DDL (executed STAGE) |
| DAL ReportIngestion `AnalyticsPipelines/Sql/Pipelines/PIPE_*.sql` (40) | Mappings + resource pool (executed STAGE) |

## Still to do / follow-up

- **Commit nothing yet without permission**: BI-Snowflake (11 files), analytics-snowflake (21), DAL ReportIngestion (46 incl. local `appsettings.json` with secrets — exclude), DAL Analytics local `appsettings.json` — exclude. Merge PR #42.
- User to run `SP_AGGREGATE_TRANSACTIONS_ORCHESTRATOR` in DEV and STAGING, then the STAGING analytics pipeline (unload → SingleStore → flip) and verify ASSETTYPEID / ESTIMATEDREVENUE land.
- Resolve which API read path is live: SingleStore `REV_ANALYTICS_API_STAGE` holds only 5 procedures (FLIP_VIEWS, PREPARE_INACTIVE_TABLES, VERIFY_INACTIVE_LOAD, warm_blob_cache*) while Snowflake `REV_ANALYTICS_API_STAGING.ENDPOINTS` has 158 stale `SP_GET_*` reading TENANT_ tables (last written 2026-05-26). Then add asset-type handling (`IFNULL(ASSETTYPEID,1) = 1` default vs parameter) to the 25 consumption/engagement endpoints.
- DEV RPS: seed `CONFIG.RPS_DELIVERY_TYPE_SCOPE` ((102,'STREAM',TRUE),(103,'STREAM',TRUE),(120,'VIEW',TRUE)) and deploy the three RPS procedures; PROD RPS not deployed at all. Build the ADF monthly If Condition.
- PROD: repeat SingleStore table ALTERs + pipeline recreate for `REV_ANALYTICS_API_PROD` once STAGE is validated; PROD analytics procedures not yet touched.
- Security: `AnalyticsPipelines/Scripts/create_nosub_database.py` holds a plaintext SingleStore admin password and AWS keys — rotate/move to secrets.
- Decide whether the 8 ARTIFICIAL_STREAMS / PLAYLIST / STREAMS_REVENUE_MOVEMENTS pipelines should stay in `pipelines_pool` (they were outside it before 2026-09-09).
