# WMG Spotify Downloader + move to ReportIngestion — 2026-07-01

**Repos:** Netcore.ReportIngestion, RepositoryDataAccessor.ReportIngestion (also Netcore.Analytics + RepositoryDataAccessor.Analytics as origin/template)
**Branches:** `vasil/implement-wmg-spotify-download` (both ReportIngestion repos, from `main`); `vasil/implement-spotify-wmg-download` (Netcore.Analytics)
**No commits** — all work left uncommitted on the branches for review.

## What we did

1. **Built the WMG Spotify downloader** in Netcore.Analytics (consumer of `WmgSpotifyDownloaderMessage`, the orchestrator from the prior session already enqueued it). Loads WMG files already in S3 via a Snowflake external stage into `…LANDING.WMG_SPOTIFY_TRENDS_JSON_LOAD` **through the DAL** (GraphQL → `SnowflakeDbContext`), not blob/HTTP.
2. **Created Snowflake objects** in `REVELATOR_ALL_STAGING`: `DEPLOYMENT.JSON_GZ_FORMAT`, `DEPLOYMENT.AWS_S3_WMG_STREAMING_DATA` (stage, `STORAGE_INTEGRATION = AWS_S3_WMG`, `s3://1004-data-forge-us-east-1-snowstage-prod/data/`), `LANDING.WMG_SPOTIFY_TRENDS_JSON_LOAD`. DDL committed in the DAL repo.
3. **Config-driven load**: `WmgDownloaderOptions` with a single abstracted `Database` + `schema.object` names (`QualifiedStage/FileFormat/LandingTable/DailyTrendsDownloads`); per-report path templates (tracks, users, aggregatedstreams, streams, sub30); `FilePattern` (`.*[.]gz`).
4. **Migrated the pattern to the ReportIngestion stack** (same "Netcore + DAL" shape): workers → **Netcore.ReportIngestion**, DAL → **RepositoryDataAccessor.ReportIngestion**. Self-contained (no cross-repo refs); DAL calls go via `IReportIngestionDataAccessLayerService`. Analytics originals left in place.
5. **Long-running message handling**: worker acks up front (`AutoAckBehavior.None` + `Consumer.Ack`) so a load can exceed RabbitMQ's 30-min consumer timeout; idempotency + orchestrator re-enqueue make it safe. (ReportIngestion only.)
6. **Timeouts** raised to 2h and made configurable: HTTP client (`MicroservicesTimeout:…`), EF/Snowflake command (`Wmg:CommandTimeoutSeconds`), HotChocolate `GraphQL:ExecutionTimeoutSeconds`.
7. **Planned + executed COPY INTO** to replace `INSERT … SELECT FROM @stage` (faster bulk load + native load-history dedup); success now determined by a post-COPY landing-row count. Removed delete-before-insert.
8. **Truncated** the landing table (had grown to ~2.38B duplicate rows under the old INSERT path) and **verified** `ROLE_ADF_STAGING` has USAGE on the stage, integration, and file format (LIST + stage SELECT both succeeded; `CURRENT_TIMESTAMP()` works in a stage read).
9. Added `appsettings.Staging.json` to both ReportIngestion workers.

## Key decisions

- **Ack up front** (not on completion) — RabbitMQ `consumer_timeout` is delivery→ack and can't be extended; early-ack + idempotency + orchestrator retry beats it. Modeled on `SpotifyPlaylistsDownloaderMessageHandler`.
- **COPY over INSERT** — bulk path + per-file load history gives dedup/restart-safety natively; success signal switched to "rows present for path" (count) because a dedup'd re-run loads 0 rows.
- **`DAILYTRENDSDOWNLOADS` fully-qualified from config** (`{Database}.STAGE.DAILYTRENDSDOWNLOADS`) — DAL sessions default to different DBs.
- **`REPORTPATH` now `1/10/<yyyyMMdd>/WMG`** (appended report type; `REPORTTYPE` stays `'WMG'`).
- **`ExecutionTimeout` is server-wide** — raising it affects all ReportIngestion DAL operations (accepted).
- **ReportIngestion-only** for behavioral changes (ack, COPY, report-path); Analytics kept as the untouched template.
- **Directory-level recursive load** per report (COPY `PATTERN` = gz), so per-country `streams`/`sub30` load in one statement; `METADATA$FILENAME` captures full path (returns `data/…`-prefixed; count uses contains-match so still matches).

## Files changed (key)

| File | Change |
|---|---|
| Netcore.ReportIngestion `Application/Wmg/*` | Options, `ISnowflakeWmgDownloadProvider` + impl (COPY + count), `AddWmgDownloader` DI, `Queries/*.graphql` |
| Netcore.ReportIngestion `Abstractions/MessageDtos/WmgSpotify*Message.cs` | Two message DTOs |
| Netcore.ReportIngestion `WmgDownloadOrchestrator/`, `WmgDownloader/` | Two worker projects (Program/bootstrap/appsettings×4/Configurator/Handler); added to sln |
| Netcore.ReportIngestion `Application.csproj` | Embed `Wmg\Queries\**\*.graphql` |
| RepositoryDataAccessor.ReportIngestion `Wmg/` | `ExecuteWmgCommand` mutation (configurable command timeout), `WmgReportPathExists` query, `WmgCountResult`, 3 DDL scripts |
| RepositoryDataAccessor.ReportIngestion `Program.cs`, `appsettings.json` | Configurable `GraphQL:ExecutionTimeoutSeconds` + `Wmg:CommandTimeoutSeconds` (both 7200) |
| Netcore.Analytics WMG (template) | Original downloader + DB abstraction + delete-before-insert→count-gate (COPY/ack NOT applied here) |

## Still to do / follow-up

- **First real COPY** via the worker will confirm `INSERT` privilege for `ROLE_ADF_STAGING` and that `CURRENT_TIMESTAMP()` works inside a COPY transformation (fallback: table `LOAD_DATETIME DEFAULT`).
- **Prod (`REVELATOR_ALL`) grants**: the prod DAL role needs USAGE on `AWS_S3_WMG` + stage/format and INSERT/SELECT/DELETE on the landing + `DAILYTRENDSDOWNLOADS`. Prod objects not yet created.
- Decide whether to mirror COPY/ack/report-path changes to the Netcore.Analytics template or retire it.
- Nothing committed on any branch — review then commit/PR.
