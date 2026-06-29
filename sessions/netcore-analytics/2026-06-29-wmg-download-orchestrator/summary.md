# WMG Download Orchestrator — 2026-06-29

**Repo:** `c:\Repos\Netcore.Analytics`  
**Branch:** `vasil/implement-1-query-revenue`  
**Related repo:** `C:\Repos\RepositoryDataAccessor.Analytics`

---

## What we did

- Added Snowflake DAL support to `RepositoryDataAccessor.Analytics` (first time Snowflake is used in that service)
- Created new `WmgAnalytics` domain area in the DAL with a read query and write mutation
- Added `ISnowflakeWmgDownloadProvider` + implementation in `Revelator.Analytics.Infrastructure` — uses the Analytics DAL microservice via HTTP/GraphQL instead of direct Snowflake connection
- Created two new message DTOs in `Revelator.Analytics.Abstractions`
- Created new `Revelator.Analytics.WmgDownloadOrchestrator` CronJob project from scratch (net8.0, mirrors TrendsDownloadOrchestrator pattern)
- Added project to the solution file
- Wired `launchSettings.json` + `dev.bootstrap.json` so VS doesn't show the bootstrap prompt on local run
- Added environment-specific `appsettings.{env}.json` with `DaysToDownloadTrends` matching TrendsDownloadOrchestrator values

---

## Key decisions

- **DAL over direct Snowflake connection** — WMG orchestrator is the first in Netcore.Analytics to use `IAnalyticsDataAccessLayerService` (HTTP/GraphQL) instead of `SnowflakeDbConnection`. Keeps Snowflake credentials out of the orchestrator.
- **EnterpriseId = 1** — WMG data is system-wide for a single internal enterprise, not per-distributor-manager like Spotify.
- **ServiceId = 10** (`WellKnownDistributors.Spotify`) — WMG data arrives as Spotify streaming data; no new enum entry needed.
- **ReportType = "WMG"** — distinguishes WMG rows in `DAILYTRENDSDOWNLOADS` from regular Spotify rows.
- **Read-only orchestrator** — the write mutation (`WriteWmgReportPath`) is in the DAL but the orchestrator only reads; a future `WmgSpotifyDownloader` will call the mutation after processing files.
- **`serviceId` parametrized** — `ReportPathExists(enterpriseId, date, serviceId)` rather than hardcoding `10`, at user request.

---

## Files changed

### RepositoryDataAccessor.Analytics

| File | Change |
|---|---|
| `WmgAnalytics/Models/WmgCountResult.cs` | New — keyless EF entity inheriting `StoredProcedureModel`, registered auto by `DbContextSettings` |
| `WmgAnalytics/GraphQLOperations/AnalyticsQuery.cs` | New — `WmgReportPathExists(sqlCommand, SnowflakeDbContext)` HotChocolate query |
| `WmgAnalytics/GraphQLOperations/AnalyticsMutation.cs` | New — `WriteWmgReportPath(sqlCommand, SnowflakeDbContext)` HotChocolate mutation |
| `Program.cs` | Updated `AddRevelatorDataAccessor<AnalyticsQuery>` → `AddRevelatorDataAccessor<AnalyticsQuery, AnalyticsMutation>` |

### Revelator.Analytics.Abstractions

| File | Change |
|---|---|
| `MessageDtos/WmgSpotifyDownloadOrchestratorMessage.cs` | New — empty trigger message |
| `MessageDtos/WmgSpotifyDownloaderMessage.cs` | New — inherits `DownloaderMessage` (EnterpriseId, Date) |

### Revelator.Analytics.Domain

| File | Change |
|---|---|
| `Interfaces/ISnowflakeWmgDownloadProvider.cs` | New — `ReportPathExists(enterpriseId, date, serviceId, ct)` |

### Revelator.Analytics.Infrastructure

| File | Change |
|---|---|
| `Providers/SnowflakeWmgDownloadProvider.cs` | New — builds SQL, calls DAL, parses JSON response |
| `DAL/Queries/Graphql/WmgReportPathExists.graphql` | New — GraphQL query shape |
| `DependencyInjection.cs` | Registered `ISnowflakeWmgDownloadProvider` in `AddWorkerInfrastructure` |

### Revelator.Analytics.WmgDownloadOrchestrator (new project)

| File | Change |
|---|---|
| `Revelator.Analytics.WmgDownloadOrchestrator.csproj` | New project |
| `Program.cs` | Standard WorkersHost entry point |
| `Hosting/WmgDownloadOrchestratorConfigurator.cs` | Registers `AddWorkerInfrastructure` + `AddMicroservices` |
| `MessageHandlers/WmgSpotifyDownloadOrchestratorMessageHandler.cs` | CronJob handler — loops dates, enqueues `WmgSpotifyDownloaderMessage` for missing paths |
| `bootstrap.json` | CronJob config, 4-hour interval |
| `dev.bootstrap.json` | Local dev hostname override |
| `appsettings.json` | Base config + Microservices DAL URL |
| `appsettings.Development.json` | DaysToDownloadTrends = 2 |
| `appsettings.Staging.json` | DaysToDownloadTrends = 10 |
| `appsettings.Production.json` | DaysToDownloadTrends = 20 |
| `Properties/launchSettings.json` | Sets `BOOTSTRAP_FILE=bootstrap.json;dev.bootstrap.json` so VS doesn't prompt |

### Solution

| File | Change |
|---|---|
| `src/Revelator.Analytics.sln` | Added `Revelator.Analytics.WmgDownloadOrchestrator` project |

---

## Still to do / follow-up

- `WmgSpotifyDownloader` worker — will consume `WmgSpotifyDownloaderMessage`, process Snowflake data, call `WriteWmgReportPath` mutation to mark path as done
- Bootstrap schedule: user asked about running at 1 AM — need to check if WorkersHost supports time-of-day cron expression (current config only uses `CronIntervalMilliseconds`)
- No unit tests written yet for `WmgSpotifyDownloadOrchestratorMessageHandler`
