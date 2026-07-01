# Plan: Replicate WMG Spotify Orchestrator + Downloader into the ReportIngestion stack

## Context
The WMG Spotify orchestrator + downloader currently live in the Analytics stack
(`Netcore.Analytics` workers + `RepositoryDataAccessor.Analytics` DAL). We want the same
two-tier "Netcore + DAL" pattern replicated into the ReportIngestion stack. The Analytics
originals are **left in place** (copy, not move).

Target repos (each gets a **feature branch from `main`**, **no commits until review**):
- **Netcore.ReportIngestion** — worker tier (mirrors Netcore.Analytics). Currently on `main`, clean.
- **RepositoryDataAccessor.ReportIngestion** — DAL/GraphQL tier (mirrors RepositoryDataAccessor.Analytics). Currently on `vasil/implement-rest-of-table` → branch from `main`.

Both repos already reference `Revelator.Common.WorkersHost` / `Revelator.CommonLibs.DataAccess`, and the
common lib already defines `IReportIngestionDataAccessLayerService` + a `ReportIngestionDataAccessLayerService`
client resolved from `Microservices:ReportIngestionDataAccessLayerService`. So no common-lib changes are needed.

Proposed branch name (both repos): `vasil/implement-wmg-spotify-download`.

---

## Behavior (unchanged from the Analytics implementation)
- **Orchestrator** (`CronJob`/`RunOnce`): for `EnterpriseId = 1` / Spotify, loop the last N days; for each
  day not already recorded (`DAILYTRENDSDOWNLOADS`, `REPORTTYPE='WMG'`), enqueue a `WmgSpotifyDownloaderMessage`.
- **Downloader** (`RabbitMq` consumer): for the message date, run one recursive directory load per configured
  report from the Snowflake external stage into the landing table, filtered to `.gz`; if all reports load, write
  the `DAILYTRENDSDOWNLOADS` tracking row (`REPORTTYPE='WMG'`).
- Per report, **delete-before-insert** for idempotency on restart, then load:
  - `DELETE FROM <landingTable> WHERE FILENAME LIKE '%<path>%' AND FILENAME RLIKE '<filePattern>'`
  - `INSERT INTO <landingTable> (JSON_VALUE, FILENAME, LOAD_DATETIME) SELECT $1, METADATA$FILENAME,
    CURRENT_TIMESTAMP() FROM @<stage>/<path> (FILE_FORMAT => '<fmt>', PATTERN => '<filePattern>')`
  - Delete and insert match the same files (report path + the appsettings `FilePattern`), so a mid-process
    restart re-purges and reloads without duplicating.
- Reports + path templates identical to the Analytics version (tracks, users, aggregatedstreams, streams,
  sub30), driven by `WmgDownloaderOptions` in appsettings.
- **Database abstraction**: `WmgDownloaderOptions` carries a single `Database` plus `schema.object`-relative
  `Stage`/`FileFormat`/`LandingTable`/`DailyTrendsDownloadsTable`; the provider composes fully-qualified names
  (`{Database}.{...}`). Only `Database` differs per env (`REVELATOR_ALL_STAGING` dev/staging, `REVELATOR_ALL`
  prod). `DAILYTRENDSDOWNLOADS` lives at `{Database}.STAGE.DAILYTRENDSDOWNLOADS` (exists in both DBs) and is now
  fully-qualified from config (it is no longer resolved against the DAL session's default database).
- These refinements were applied to the Netcore.Analytics template first (built, 0 errors); the ReportIngestion
  copy inherits them.

---

## Part A — Netcore.ReportIngestion (worker tier)
Solution: `src/Revelator.ReportIngestion.sln`. Workers follow the existing
`Revelator.ReportIngestion.ReportDiscovery` pattern.

1. **Message DTOs** → `Revelator.ReportIngestion.Abstractions/MessageDtos/`
   - `WmgSpotifyDownloadOrchestratorMessage` (empty trigger).
   - `WmgSpotifyDownloaderMessage` (`long EnterpriseId`, `DateTime Date`).

2. **Options + provider + GraphQL** → `Revelator.ReportIngestion.Infrastructure`
   - `Configuration/WmgDownloaderOptions.cs` (Database, schema.object Stage/FileFormat/LandingTable/
     DailyTrendsDownloadsTable, FilePattern=`.*[.]gz`, Reports[], + `Qualified*` computed props). Copy from the
     refined Netcore.Analytics version.
   - `Providers/ISnowflakeWmgDownloadProvider.cs` + `SnowflakeWmgDownloadProvider.cs` — same shape as the
     Analytics provider (`ReportPathExists`, `LoadSpotifyTrends`, `WriteReportPath`) but injects
     **`IReportIngestionDataAccessLayerService`** and calls the ReportIngestion DAL. Uses `EmbeddedResourceLoader`
     + `GraphQLHelper.DeserializeGraphQL` (the convention already used by `RunAnalyticsPipelineHandler`).
   - Embedded `.graphql`: `WmgReportPathExists.graphql` (query), `ExecuteWmgCommand.graphql` (mutation) — add an
     `<EmbeddedResource Include="...\*.graphql" />` glob to the Infrastructure `.csproj`.
   - Register the provider + options in `Infrastructure/DependencyInjection.cs` (`AddInfrastructure`).
   - **Fully-qualify `DAILYTRENDSDOWNLOADS`** in the SQL (e.g. `REVELATOR_ALL_STAGING.<schema>.DAILYTRENDSDOWNLOADS`)
     — the ReportIngestion DAL's Snowflake session defaults to a different DB than the Analytics DAL, so the
     previously-unqualified reference would not resolve. Confirm the table's schema during implementation.

3. **New worker project** `src/Revelator.ReportIngestion.WmgDownloadOrchestrator/`
   - `Program.cs` (standard `ConfigureWorkersHost`), `bootstrap.json` (`HostType: CronJob`, `RunOnce`),
     `dev.bootstrap.json`, `appsettings*.json`, `Properties/launchSettings.json` (BOOTSTRAP_FILE env).
   - `Hosting/WmgDownloadOrchestratorConfigurator.cs`: `Configure<MicroservicesOptions>` +
     `AddDataAccessLayerApiClient<IReportIngestionDataAccessLayerService, ReportIngestionDataAccessLayerService>()`
     + `AddInfrastructure` + bind `WmgDownloaderOptions` + bind `DownloadSettings`.
   - `MessageHandlers/WmgSpotifyDownloadOrchestratorMessageHandler.cs` (loop days → enqueue).

4. **New worker project** `src/Revelator.ReportIngestion.WmgDownloader/`
   - Same scaffolding; `bootstrap.json` (`HostType: Consumer`, `ConsumerType: RabbitMq`,
     `QueueName: WmgSpotifyDownloaderMessage`).
   - `MessageHandlers/WmgSpotifyDownloaderMessageHandler.cs` (per-report load + tracking write).

5. **Wiring**: add both projects to `Revelator.ReportIngestion.sln`; ensure `docker/Dockerfile.workers`
   build-all picks them up (it globs `src/*/*.csproj`, so automatic). Add `Microservices:ReportIngestionDataAccessLayerService`
   (already present in Api appsettings) + `DownloadSettings:DaysToDownloadTrends` + `WmgDownloaderOptions` to the
   new workers' appsettings. `WellKnownDistributors.Spotify` comes from `Revelator.CommonLibs.Enums` (no Analytics ref).

## Part B — RepositoryDataAccessor.ReportIngestion (DAL tier)
GraphQL server uses partial `ReportIngestionMutation` / `ReportIngestionQuery` and a registered `SnowflakeDbContext`.

1. **Feature folder** `Wmg/` (mirrors `WmgAnalytics/` in the Analytics DAL):
   - `Wmg/GraphQLOperations/WmgMutation.cs` — `public partial class ReportIngestionMutation` with
     `ExecuteWmgCommand(string sqlCommand, [Service] SnowflakeDbContext db)` → `ExecuteSqlRawAsync`, returns `rows > 0`.
   - `Wmg/GraphQLOperations/WmgQuery.cs` — `public partial class ReportIngestionQuery` with
     `WmgReportPathExists(string sqlCommand, [Service] SnowflakeDbContext db)` → `FromSqlRaw(...).ToListAsync()`.
   - `Wmg/Models/WmgCountResult.cs` — `[Table("WmgCountResult")] class WmgCountResult : StoredProcedureModel { int Count }`
     (confirm `StoredProcedureModel` is exposed by `Revelator.CommonLibs.DataAccess`, as it is for the Analytics DAL).

2. **Snowflake DDL** → `Wmg/Sql/` (idempotent, committed here for traceability; objects already exist in
   `REVELATOR_ALL_STAGING`): `WMG_SPOTIFY_TRENDS_JSON_LOAD.sql`, `JSON_GZ_FORMAT.sql`, `AWS_S3_WMG_STREAMING_DATA.sql`
   (`STORAGE_INTEGRATION = AWS_S3_WMG`, `URL = s3://1004-data-forge-us-east-1-snowstage-prod/data/`). Copy from the
   Analytics DAL versions.

3. No `Program.cs`/DI change needed — HotChocolate auto-discovers the partial-class extensions; `SnowflakeDbContext`
   is already registered via `AddRevelatorDataAccessor<ReportIngestionQuery, ReportIngestionMutation>`.

---

## Key risks / to confirm during implementation
- **Snowflake grants**: the ReportIngestion DAL's Snowflake user (per env) must have `USAGE` on integration
  `AWS_S3_WMG` and access to `REVELATOR_ALL_STAGING.LANDING` + `.DEPLOYMENT` + `.STAGE.DAILYTRENDSDOWNLOADS`.
  The staging Analytics DAL used `USER_ADF_ETL`/`SYSADMIN`; the ReportIngestion DAL may use a different user — verify.
- **Dev appsettings** point the workers at the local ReportIngestion DAL:
  `Microservices:ReportIngestionDataAccessLayerService = http://localhost:5251/report-ingestion/graphql`.
- **`StoredProcedureModel`** availability in `Revelator.CommonLibs.DataAccess`.
- `DAILYTRENDSDOWNLOADS` is now fully-qualified from config (`{Database}.STAGE.DAILYTRENDSDOWNLOADS`); it exists in
  both `REVELATOR_ALL_STAGING.STAGE` and `REVELATOR_ALL.STAGE`, so no DB-default reliance.
- **.NET TFMs**: RepositoryDataAccessor.ReportIngestion is net9.0; Netcore.ReportIngestion workers are net8.0
  (match the existing workers' TFM).

## Verification
1. Branch from `main` in both repos; build each solution (0 errors).
2. Run the ReportIngestion DAL locally; in Snowflake confirm the `Wmg` query/mutation execute (smoke `LIST`/INSERT).
3. Run the two new workers locally (RabbitMq up), publish a `WmgSpotifyDownloaderMessage { EnterpriseId=1, Date=2026-06-24 }`,
   confirm all 5 reports load into the landing table and a `REPORTTYPE='WMG'` row is written; re-run is skipped.
4. Leave all changes uncommitted on the feature branches for review.
