# WmgDownloadOrchestrator Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create a CronJob orchestrator that checks the Snowflake `DAILYTRENDSDOWNLOADS` table for missing WMG Spotify report path records (enterprise 1, service 10, ReportType "WMG") and enqueues `WmgSpotifyDownloaderMessage` for each gap — using the Analytics DAL microservice instead of a direct Snowflake connection.

**Architecture:** The orchestrator is a .NET 8 CronJob that loops over a configurable date range for a single enterprise (1). For each date, it calls `ISnowflakeWmgDownloadProvider.ReportPathExists()`, which sends a SQL command to `RepositoryDataAccessor.Analytics` via GraphQL (`IAnalyticsDataAccessLayerService`). The DAL executes the query against a new `SnowflakeDbContext`. If no record exists, a `WmgSpotifyDownloaderMessage` is enqueued for the future downloader worker.

**Tech Stack:** .NET 8, xUnit 2.5.3, Moq 4.20.70, HotChocolate (RepositoryDataAccessor.Analytics), EF Core + Snowflake EF provider, `Revelator.Common.WorkersHost`, `IAnalyticsDataAccessLayerService`

## Global Constraints

- EnterpriseId = `1` (WMG internal enterprise)
- ServiceId = `10` (`WellKnownDistributors.Spotify`)
- ReportType = `"WMG"` (string constant, uppercase)
- Start date = `DateTime.Now.AddDays(-2)` (same as all other orchestrators)
- Days to download from config key `DownloadSettings:DaysToDownloadTrends`
- All Snowflake access goes through the Analytics DAL (`RepositoryDataAccessor.Analytics`), not direct `SnowflakeDbConnection`
- Orchestrator project targets `net8.0`; DAL targets its existing framework version (check before modifying)
- All new files in `RepositoryDataAccessor.Analytics` follow the existing partial class pattern (`public partial class AnalyticsQuery`)
- Test project: `c:\Repos\Netcore.Analytics\src\Revelator.Analytics.Tests` — xUnit, Moq

---

## File Map

**RepositoryDataAccessor.Analytics** (new files):
- `RepositoryDataAccessor.Analytics/WmgAnalytics/Models/WmgCountResult.cs` — EF response model for COUNT(*) result
- `RepositoryDataAccessor.Analytics/WmgAnalytics/GraphQLOperations/AnalyticsQuery.cs` — partial class extending root query with `WmgReportPathExists`
- `RepositoryDataAccessor.Analytics/WmgAnalytics/GraphQLOperations/AnalyticsMutation.cs` — partial class extending root mutation with `WriteWmgReportPath`
- `RepositoryDataAccessor.Analytics/Infrastructure/SnowflakeDbContext.cs` — new EF Core DbContext for Snowflake
- `RepositoryDataAccessor.Analytics/RepositoryDataAccessor.Analytics.csproj` — add Snowflake EF package
- `RepositoryDataAccessor.Analytics/Program.cs` or `Startup.cs` — register `SnowflakeDbContext`

**Revelator.Analytics.Abstractions** (new files):
- `src/Revelator.Analytics.Abstractions/MessageDtos/WmgSpotifyDownloadOrchestratorMessage.cs`
- `src/Revelator.Analytics.Abstractions/MessageDtos/WmgSpotifyDownloaderMessage.cs`

**Revelator.Analytics.Domain** (new file):
- `src/Revelator.Analytics.Domain/Interfaces/ISnowflakeWmgDownloadProvider.cs`

**Revelator.Analytics.Infrastructure** (new + modified files):
- `src/Revelator.Analytics.Infrastructure/Providers/SnowflakeWmgDownloadProvider.cs`
- `src/Revelator.Analytics.Infrastructure/DAL/Queries/Graphql/WmgReportPathExists.graphql` — embedded resource
- `src/Revelator.Analytics.Infrastructure/DependencyInjection.cs` — register new provider + DAL HTTP client

**Revelator.Analytics.WmgDownloadOrchestrator** (new project):
- `src/Revelator.Analytics.WmgDownloadOrchestrator/Revelator.Analytics.WmgDownloadOrchestrator.csproj`
- `src/Revelator.Analytics.WmgDownloadOrchestrator/Program.cs`
- `src/Revelator.Analytics.WmgDownloadOrchestrator/Hosting/WmgDownloadOrchestratorConfigurator.cs`
- `src/Revelator.Analytics.WmgDownloadOrchestrator/MessageHandlers/WmgSpotifyDownloadOrchestratorMessageHandler.cs`
- `src/Revelator.Analytics.WmgDownloadOrchestrator/appsettings.json`
- `src/Revelator.Analytics.WmgDownloadOrchestrator/bootstrap.json`

**Test file** (new):
- `src/Revelator.Analytics.Tests/WmgDownloadOrchestrator/WmgSpotifyDownloadOrchestratorMessageHandlerTests.cs`

---

## Task 1: DAL — Snowflake support in RepositoryDataAccessor.Analytics

**Files:**
- Create: `C:\Repos\RepositoryDataAccessor.Analytics\RepositoryDataAccessor.Analytics\WmgAnalytics\Models\WmgCountResult.cs` ✅
- Create: `C:\Repos\RepositoryDataAccessor.Analytics\RepositoryDataAccessor.Analytics\WmgAnalytics\GraphQLOperations\AnalyticsQuery.cs` ✅
- Create: `C:\Repos\RepositoryDataAccessor.Analytics\RepositoryDataAccessor.Analytics\WmgAnalytics\GraphQLOperations\AnalyticsMutation.cs` ✅
- ~~Create: `Infrastructure/SnowflakeDbContext.cs`~~ — `SnowflakeDbContext` already exists in `Revelator.CommonLibs.DataAccess` (global namespace); no new file needed
- Modify: `C:\Repos\RepositoryDataAccessor.Analytics\RepositoryDataAccessor.Analytics\Program.cs` ✅ — added `AnalyticsMutation` type registration

**Interfaces:**
- Produces: `WmgReportPathExists(string sqlCommand, [Service] SnowflakeDbContext db)` → `List<WmgCountResult>` (GraphQL query field)
- Produces: `WriteWmgReportPath(string sqlCommand, [Service] SnowflakeDbContext db)` → `bool` (GraphQL mutation field)

- [x] **Step 1: Create `WmgCountResult.cs`** ✅

  ```csharp
  // C:\Repos\RepositoryDataAccessor.Analytics\RepositoryDataAccessor.Analytics\WmgAnalytics\Models\WmgCountResult.cs
  using System.ComponentModel.DataAnnotations.Schema;

  // global namespace — no 'namespace' declaration
  [Table("WmgCountResult")]
  public class WmgCountResult : StoredProcedureModel
  {
      public int Count { get; set; }
  }
  ```
  `StoredProcedureModel` and `SnowflakeDbContext` are in global namespace in `Revelator.CommonLibs.DataAccess`. `WmgCountResult` is auto-registered as a keyless entity by `DbContextSettings.OnModelCreating` (scans assemblies containing "RepositoryDataAccessor.") — no `OnModelCreating` override needed.

- [x] **Step 2: Create `WmgAnalytics/GraphQLOperations/AnalyticsQuery.cs`** ✅

  ```csharp
  // C:\Repos\RepositoryDataAccessor.Analytics\RepositoryDataAccessor.Analytics\WmgAnalytics\GraphQLOperations\AnalyticsQuery.cs
  using HotChocolate;
  using Microsoft.EntityFrameworkCore;

  // global namespace — matches existing partial class pattern
  public partial class AnalyticsQuery
  {
      public async Task<List<WmgCountResult>> WmgReportPathExists(
          string sqlCommand,
          [Service] SnowflakeDbContext db)
      {
          return await db.Set<WmgCountResult>()
              .FromSqlRaw(sqlCommand)
              .ToListAsync();
      }
  }
  ```

- [x] **Step 3: Create `WmgAnalytics/GraphQLOperations/AnalyticsMutation.cs`** ✅

  ```csharp
  // C:\Repos\RepositoryDataAccessor.Analytics\RepositoryDataAccessor.Analytics\WmgAnalytics\GraphQLOperations\AnalyticsMutation.cs
  using HotChocolate;
  using Microsoft.EntityFrameworkCore;

  // global namespace — new partial class (first mutation in this DAL)
  public partial class AnalyticsMutation
  {
      public async Task<bool> WriteWmgReportPath(
          string sqlCommand,
          [Service] SnowflakeDbContext db)
      {
          var rows = await db.Database.ExecuteSqlRawAsync(sqlCommand);
          return rows > 0;
      }
  }
  ```

- [x] **Step 4: Update `Program.cs` to register `AnalyticsMutation`** ✅

  Changed:
  ```csharp
  builder.Services.AddRevelatorDataAccessor<AnalyticsQuery>(builder.Configuration, "Analytics");
  ```
  To:
  ```csharp
  builder.Services.AddRevelatorDataAccessor<AnalyticsQuery, AnalyticsMutation>(builder.Configuration, "Analytics");
  ```

- [x] **Step 5: Build RepositoryDataAccessor.Analytics** ✅ — Build succeeded, 0 errors.

---

## Task 2: Message DTOs in Revelator.Analytics.Abstractions

**Files:**
- Create: `src/Revelator.Analytics.Abstractions/MessageDtos/WmgSpotifyDownloadOrchestratorMessage.cs`
- Create: `src/Revelator.Analytics.Abstractions/MessageDtos/WmgSpotifyDownloaderMessage.cs`

**Interfaces:**
- Produces: `WmgSpotifyDownloadOrchestratorMessage` (empty — trigger signal for the CronJob)
- Produces: `WmgSpotifyDownloaderMessage` with `EnterpriseId` (long) and `Date` (DateTime) from inherited `DownloaderMessage`

- [x] **Step 1: Create `WmgSpotifyDownloadOrchestratorMessage.cs`** ✅

  ```csharp
  namespace Revelator.Analytics.Abstractions.MessageDtos
  {
      public class WmgSpotifyDownloadOrchestratorMessage { }
  }
  ```

- [x] **Step 2: Create `WmgSpotifyDownloaderMessage.cs`** ✅

  ```csharp
  namespace Revelator.Analytics.Abstractions.MessageDtos
  {
      public class WmgSpotifyDownloaderMessage : DownloaderMessage { }
  }
  ```
  `DownloaderMessage` is in `Revelator.Analytics.Abstractions.MessageDtos` — provides `long EnterpriseId` and `DateTime Date`.

---

## Task 3: Provider in Revelator.Analytics.Infrastructure

**Files:**
- Create: `src/Revelator.Analytics.Domain/Interfaces/ISnowflakeWmgDownloadProvider.cs`
- Create: `src/Revelator.Analytics.Infrastructure/Providers/SnowflakeWmgDownloadProvider.cs`
- Create: `src/Revelator.Analytics.Infrastructure/DAL/Queries/Graphql/WmgReportPathExists.graphql`
- Modify: `src/Revelator.Analytics.Infrastructure/DependencyInjection.cs`
- Modify: `src/Revelator.Analytics.Infrastructure/Revelator.Analytics.Infrastructure.csproj` (ensure graphql is embedded resource — check existing pattern)

**Interfaces:**
- Consumes: `IAnalyticsDataAccessLayerService.SendQueryAsync(string graphql, object variables, CancellationToken ct)` → `string` (JSON)
- Produces: `ISnowflakeWmgDownloadProvider.ReportPathExists(long enterpriseId, DateTime date, long serviceId, CancellationToken ct)` → `Task<bool>`

  > `serviceId` was parametrized at user request (originally hardcoded as `10`).

- [x] **Step 1: Create `ISnowflakeWmgDownloadProvider.cs`** ✅

  ```csharp
  namespace Revelator.Analytics.Domain.Interfaces
  {
      public interface ISnowflakeWmgDownloadProvider
      {
          Task<bool> ReportPathExists(long enterpriseId, DateTime date, long serviceId, CancellationToken cancellationToken = default);
      }
  }
  ```

- [x] **Step 2: Create `WmgReportPathExists.graphql`** ✅

  New file is picked up by the existing `**/*.graphql` glob in `Revelator.Analytics.Infrastructure.csproj`.
  ```graphql
  query WmgReportPathExists($sqlCommand: String!) {
    wmgReportPathExists(sqlCommand: $sqlCommand) {
      count
    }
  }
  ```

- [x] **Step 3: Create `SnowflakeWmgDownloadProvider.cs`** ✅

  Uses `EmbeddedResourceLoader.Load("WmgReportPathExists.graphql")` from `Revelator.Common` — not `Assembly.GetManifestResourceStream` — because the existing codebase uses that helper.

  ```csharp
  namespace Revelator.Analytics.Infrastructure.Providers
  {
      public class SnowflakeWmgDownloadProvider : ISnowflakeWmgDownloadProvider
      {
          private static readonly string GraphqlQuery = EmbeddedResourceLoader.Load("WmgReportPathExists.graphql");

          private readonly ILogger<SnowflakeWmgDownloadProvider> _logger;
          private readonly IAnalyticsDataAccessLayerService _analyticsDataAccessLayerService;

          public SnowflakeWmgDownloadProvider(
              ILogger<SnowflakeWmgDownloadProvider> logger,
              IAnalyticsDataAccessLayerService analyticsDataAccessLayerService)
          {
              _logger = logger;
              _analyticsDataAccessLayerService = analyticsDataAccessLayerService;
          }

          public async Task<bool> ReportPathExists(long enterpriseId, DateTime date, long serviceId, CancellationToken cancellationToken = default)
          {
              var sql = $"SELECT COUNT(*) AS \"Count\" FROM DAILYTRENDSDOWNLOADS " +
                        $"WHERE ENTERPRISEID = {enterpriseId} " +
                        $"AND DATE = '{date:yyyy-MM-dd}' " +
                        $"AND SERVICEID = {serviceId} " +
                        $"AND REPORTTYPE = 'WMG'";

              _logger.LogDebug("WmgReportPathExists SQL: {Sql}", sql);

              try
              {
                  var json = await _analyticsDataAccessLayerService.SendQueryAsync(
                      GraphqlQuery, new { sqlCommand = sql }, cancellationToken);

                  using var doc = JsonDocument.Parse(json);
                  var items = doc.RootElement
                      .GetProperty("data")
                      .GetProperty("wmgReportPathExists")
                      .EnumerateArray();

                  foreach (var item in items)
                  {
                      if (item.GetProperty("count").GetInt32() > 0)
                          return true;
                  }

                  return false;
              }
              catch (Exception ex)
              {
                  _logger.LogError(ex, "Failed to check WMG report path for EnterpriseId={EnterpriseId}, Date={Date}",
                      enterpriseId, date.ToString("yyyy-MM-dd"));
                  return false;
              }
          }
      }
  }
  ```

- [x] **Step 4: Register the provider in `DependencyInjection.cs`** ✅

  Added `services.AddTransient<ISnowflakeWmgDownloadProvider, SnowflakeWmgDownloadProvider>();` in `AddWorkerInfrastructure`.

---

## Task 4: WmgDownloadOrchestrator project + handler (TDD)

**Files:**
- Create: `src/Revelator.Analytics.WmgDownloadOrchestrator/Revelator.Analytics.WmgDownloadOrchestrator.csproj`
- Create: `src/Revelator.Analytics.WmgDownloadOrchestrator/Program.cs`
- Create: `src/Revelator.Analytics.WmgDownloadOrchestrator/Hosting/WmgDownloadOrchestratorConfigurator.cs`
- Create: `src/Revelator.Analytics.WmgDownloadOrchestrator/MessageHandlers/WmgSpotifyDownloadOrchestratorMessageHandler.cs`
- Create: `src/Revelator.Analytics.WmgDownloadOrchestrator/appsettings.json`
- Create: `src/Revelator.Analytics.WmgDownloadOrchestrator/bootstrap.json`
- Create: `src/Revelator.Analytics.Tests/WmgDownloadOrchestrator/WmgSpotifyDownloadOrchestratorMessageHandlerTests.cs`

**Interfaces:**
- Consumes: `ISnowflakeWmgDownloadProvider.ReportPathExists(long, DateTime, long serviceId, CancellationToken)` → `Task<bool>` (from Task 3)
- Consumes: `WmgSpotifyDownloaderMessage` from Task 2
- Consumes: `IConfiguration` key `DownloadSettings:DaysToDownloadTrends` → `int`
- Consumes: `IMessageBroker.EnqueueMessage<T>(T message)`

- [x] **Step 1: Create all project files** ✅ — Files created and added to `Revelator.Analytics.sln`:

  Create `src/Revelator.Analytics.Tests/WmgDownloadOrchestrator/WmgSpotifyDownloadOrchestratorMessageHandlerTests.cs`:

  ```csharp
  using Microsoft.Extensions.Configuration;
  using Microsoft.Extensions.Logging;
  using Moq;
  using Revelator.Analytics.Abstractions.MessageDtos;
  using Revelator.Analytics.Domain.Interfaces;
  using Revelator.Analytics.WmgDownloadOrchestrator.MessageHandlers;
  using Revelator.Common.WorkersHost;
  using Xunit;

  namespace Revelator.Analytics.Tests.WmgDownloadOrchestrator;

  public class WmgSpotifyDownloadOrchestratorMessageHandlerTests
  {
      private readonly Mock<ISnowflakeWmgDownloadProvider> _providerMock;
      private readonly Mock<IMessageBroker> _brokerMock;
      private readonly IConfiguration _configuration;

      public WmgSpotifyDownloadOrchestratorMessageHandlerTests()
      {
          _providerMock = new Mock<ISnowflakeWmgDownloadProvider>();
          _brokerMock = new Mock<IMessageBroker>();
          _configuration = new ConfigurationBuilder()
              .AddInMemoryCollection(new Dictionary<string, string?>
              {
                  ["DownloadSettings:DaysToDownloadTrends"] = "2"
              })
              .Build();
      }

      [Fact]
      public async Task HandleMessageAsync_EnqueuesMessage_ForEachDateWithNoReportPath()
      {
          // All dates have no path → 3 messages enqueued (i=2,1,0)
          _providerMock
              .Setup(p => p.ReportPathExists(0, It.IsAny<DateTime>(), It.IsAny<CancellationToken>()))
              .ReturnsAsync(false);

          var handler = new WmgSpotifyDownloadOrchestratorMessageHandler(
              Mock.Of<ILogger<WmgSpotifyDownloadOrchestratorMessageHandler>>(),
              _brokerMock.Object,
              _configuration,
              _providerMock.Object);

          var context = new HandleMessageContext<WmgSpotifyDownloadOrchestratorMessage>(
              new WmgSpotifyDownloadOrchestratorMessage());

          await handler.HandleMessageAsync(context, CancellationToken.None);

          _brokerMock.Verify(
              b => b.EnqueueMessage(It.IsAny<WmgSpotifyDownloaderMessage>()),
              Times.Exactly(3));
      }

      [Fact]
      public async Task HandleMessageAsync_SkipsDate_WhenReportPathAlreadyExists()
      {
          // All dates already have a path → 0 messages enqueued
          _providerMock
              .Setup(p => p.ReportPathExists(0, It.IsAny<DateTime>(), It.IsAny<CancellationToken>()))
              .ReturnsAsync(true);

          var handler = new WmgSpotifyDownloadOrchestratorMessageHandler(
              Mock.Of<ILogger<WmgSpotifyDownloadOrchestratorMessageHandler>>(),
              _brokerMock.Object,
              _configuration,
              _providerMock.Object);

          var context = new HandleMessageContext<WmgSpotifyDownloadOrchestratorMessage>(
              new WmgSpotifyDownloadOrchestratorMessage());

          await handler.HandleMessageAsync(context, CancellationToken.None);

          _brokerMock.Verify(
              b => b.EnqueueMessage(It.IsAny<WmgSpotifyDownloaderMessage>()),
              Times.Never);
      }

      [Fact]
      public async Task HandleMessageAsync_EnqueuesWithCorrectEnterpriseId()
      {
          WmgSpotifyDownloaderMessage? captured = null;
          _providerMock
              .Setup(p => p.ReportPathExists(It.IsAny<long>(), It.IsAny<DateTime>(), It.IsAny<CancellationToken>()))
              .ReturnsAsync(false);
          _brokerMock
              .Setup(b => b.EnqueueMessage(It.IsAny<WmgSpotifyDownloaderMessage>()))
              .Callback<WmgSpotifyDownloaderMessage>(msg => captured = msg);

          var handler = new WmgSpotifyDownloadOrchestratorMessageHandler(
              Mock.Of<ILogger<WmgSpotifyDownloadOrchestratorMessageHandler>>(),
              _brokerMock.Object,
              _configuration,
              _providerMock.Object);

          var context = new HandleMessageContext<WmgSpotifyDownloadOrchestratorMessage>(
              new WmgSpotifyDownloadOrchestratorMessage());

          await handler.HandleMessageAsync(context, CancellationToken.None);

          Assert.NotNull(captured);
          Assert.Equal(1, captured.EnterpriseId);
      }
  }
  ```

- [x] **Step 2: Handler — `WmgSpotifyDownloadOrchestratorMessageHandler.cs`** ✅

  Uses `WellKnownDistributors.Spotify` (= 10) for `distributorId`, passes it to `ReportPathExists`. Logs `1/{distributorId}/{date:yyyyMMdd}/WMG` on enqueue (matching other orchestrators' debug format). Catches `OperationCanceledException` explicitly to avoid false-error logging.

  ```csharp
  public override async Task HandleMessageAsync(HandleMessageContext<WmgSpotifyDownloadOrchestratorMessage> context, CancellationToken cancellationToken)
  {
      Logger.LogInformation("WMG Spotify Download Orchestrator Started.");

      var distributorId = Convert.ToInt64(WellKnownDistributors.Spotify);
      var daysToDownload = configuration.GetValue<int>(AnalyticsConstants.DAYS_TO_DOWNLOAD_TRENDS);
      var now = DateTime.Now.AddDays(-2);

      for (int i = daysToDownload; i >= 0; i--)
      {
          var date = now.AddDays(-i);
          bool exists;
          try
          {
              exists = await wmgDownloadProvider.ReportPathExists(1, date, distributorId, cancellationToken);
          }
          catch (OperationCanceledException)
          {
              Logger.LogWarning("WMG operation was cancelled for EnterpriseId=1, Date={Date}.", date.ToString("yyyyMMdd"));
              throw;
          }

          if (!exists)
          {
              MessageBroker.EnqueueMessage(new WmgSpotifyDownloaderMessage { EnterpriseId = 1, Date = date });
              Logger.LogDebug($"1/{distributorId}/{date:yyyyMMdd}/WMG");
          }
      }

      Logger.LogDebug("WMG Spotify Download Orchestrator Finished.");
  }
  ```

- [x] **Step 3: Configurator** ✅ — `WmgDownloadOrchestratorConfigurator` calls `services.AddWorkerInfrastructure(...)` and `services.AddMicroservices(...)`.

- [x] **Step 4: bootstrap.json** ✅ — `ConsumerType: RunOnce`, no `CronIntervalMilliseconds` (WorkersHost has no native time-of-day cron; scheduling handled externally by Kubernetes CronJob).

- [x] **Step 5: dev.bootstrap.json + launchSettings.json** ✅ — `BOOTSTRAP_FILE=bootstrap.json;dev.bootstrap.json` env var prevents interactive VS prompt on startup.

- [x] **Step 6: appsettings per-environment** ✅ — `DaysToDownloadTrends`: Development=2, Staging=10, Production=20.

---

## Self-Review Checklist

- [x] DAL: `WmgReportPathExists` query — Task 1, Step 2
- [x] DAL: `WriteWmgReportPath` mutation (kept for future downloader) — Task 1, Step 3
- [x] DAL: `WmgCountResult` model (global namespace, inherits `StoredProcedureModel`, auto-registered) — Task 1, Step 1
- [x] DAL: `AnalyticsMutation` registered in `Program.cs` — Task 1, Step 4
- [x] DTOs: `WmgSpotifyDownloadOrchestratorMessage` + `WmgSpotifyDownloaderMessage` — Task 2
- [x] Interface: `ReportPathExists(enterpriseId, date, serviceId, ct)` — serviceId parametrized — Task 3
- [x] Provider uses `EmbeddedResourceLoader.Load(...)` + `IAnalyticsDataAccessLayerService` — Task 3, Step 3
- [x] GraphQL file as embedded resource (auto-picked by existing glob) — Task 3, Step 2
- [x] Provider DI registration in `AddWorkerInfrastructure` — Task 3, Step 4
- [x] Configurator registers `AddWorkerInfrastructure` + `AddMicroservices` — Task 4, Step 3
- [x] Handler: `EnterpriseId = 1`, `serviceId = WellKnownDistributors.Spotify`, `DateTime.Now.AddDays(-2)`, configurable days — Task 4, Step 2
- [x] Handler: debug log format `1/{distributorId}/{date:yyyyMMdd}/WMG` — Task 4, Step 2
- [x] bootstrap.json with correct FQNs, `ConsumerType: RunOnce` — Task 4, Step 4
- [x] launchSettings.json + dev.bootstrap.json to bypass VS startup prompt — Task 4, Step 5
- [ ] Unit tests for `WmgSpotifyDownloadOrchestratorMessageHandler` — **still to do**
- [ ] 1 AM scheduling — WorkersHost has no native time-of-day cron; needs Kubernetes CronJob config — **still to do**
