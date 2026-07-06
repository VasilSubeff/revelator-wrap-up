# Plan: Add SoundCloud download to the WMG downloader (3rd source)

## Context
The WMG downloader is now a multi-source pipeline (Spotify + Apple) driven by `WmgDownloaderOptions.Sources[]`,
where each report's COPY lives in an embedded, tokenized `.sql` (loaded by `SnowflakeWmgDownloadProvider.LoadReport`,
executed via the DAL's generic `executeWmgCommand`, then a post-COPY count-gate). We now add **SoundCloud** as a
third source, ingesting 3 reports: **Customer Data**, **Stream Level Report**, **Track Information**.

Verified facts (Snowflake + stage, read-only):
- Prod tables in `REVELATOR_ALL.LANDING`: `SOUNDCLOUD_CUSTDATA_TSV_LOAD` (13 data cols), `SOUNDCLOUD_STREAMLEVEL_TSV_LOAD`
  (18), `SOUNDCLOUD_TRACKINFO_TSV_LOAD` (13). All columns `VARCHAR(1000)` + `LOAD_DATETIME`, `FILENAME` last. Sampling the
  files confirms **straight positional maps** (`$1..$N` in table order) — no reordering like Apple's `CONTAINER_TYPE`.
- Files are gzipped **tab-delimited TSV with a header row**. Legacy format `SOUNDCLOUD_TSV` = `FIELD_DELIMITER='\t'`,
  `SKIP_HEADER=1` (no other options) — we clone it.
- Files sit in the **same WMG stage/bucket** under one shared directory:
  `SoundCloudMarket/raw/year={0:yyyy}/month={0:MM}/day={0:yyyyMMdd}/`.
- `WellKnownDistributors.Soundcloud = 85`. Tracking uses `REPORTTYPE='WMG'`, `SERVICEID=85` (same convention as Spotify=10/Apple=61).

**Key delta from Apple/Spotify:** all SoundCloud reports (and 2 others we don't ingest — Customeranalytics,
Playlistreporting — plus `.marker` files) share **one directory**, distinguished only by filename suffix. A directory-wide
`PATTERN='.*[.]gz'` would pull every report into one table (column-count mismatch → COPY abort). So each SoundCloud report
needs a **filename-specific pattern**: `.*Customerdata[.]tsv[.]gz`, `.*Streamlevelreport[.]tsv[.]gz`, `.*Trackinformation[.]tsv[.]gz`.

## Part A — Snowflake objects (create in `REVELATOR_ALL_STAGING` via the SYSADMIN connection; script prod guarded)
Reuse the existing stage `DEPLOYMENT.AWS_S3_WMG_STREAMING_DATA`.
1. **File format** clone of legacy `SOUNDCLOUD_TSV`:
   `CREATE FILE FORMAT IF NOT EXISTS REVELATOR_ALL_STAGING.DEPLOYMENT.SOUNDCLOUD_TSV FIELD_DELIMITER='\t' SKIP_HEADER=1;`
2. **3 landing tables** `REVELATOR_ALL_STAGING.LANDING.WMG_SOUNDCLOUD_{CUSTDATA,STREAMLEVEL,TRACKINFO}_TSV_LOAD` — exact
   copies of the prod DDL (already `VARCHAR(1000)` + `FILENAME VARCHAR(1000)`).
3. **Grants**: not needed — schema-level FUTURE GRANTS to `ROLE_ADF_STAGING` cover the new format + tables automatically.

## Part B — Shared-pipeline changes (apply to all distributors)

**B1. Per-report `FilePattern`** (SoundCloud needs it; Apple/Spotify keep the global `.*[.]gz`).
- `WmgSourceOption` and `WmgReportOption` gain an optional `string? FilePattern` (report overrides source overrides the
  global `WmgDownloaderOptions.FilePattern` default `.*[.]gz`).
- `ISnowflakeWmgDownloadProvider.LoadReport` gains a `filePattern` parameter; the provider uses it for **both** the `{PATTERN}`
  token substitution **and** the count-gate `matchFilter` (replacing the two current `_options.FilePattern` usages).
- `WmgDownloaderMessageHandler` resolves `report.FilePattern ?? source.FilePattern ?? Options.FilePattern` and passes it.
- Backward compatible: Apple/Spotify set no `FilePattern`, so they keep `.*[.]gz` and are unchanged.

**B2. Tracking-write guards** (in `WmgDownloaderMessageHandler`, so every distributor benefits):
- **Completeness:** write the `DAILYTRENDSDOWNLOADS` row only when **every report configured for the source in appsettings**
  loaded (all count-gates > 0). This is the existing `allLoaded` gate — keep it authoritative: if any configured report
  returns `loaded=false` or throws, do not write, so the date is retried on the next orchestrator run. (No markers / no
  stage-manifest guessing — "all files" = all configured reports.)
- **Idempotency:** immediately before `WriteReportPath`, re-check `ReportPathExists(enterpriseId, date, serviceId)` and skip
  the insert if it already exists — guards against a redelivered/duplicate message writing a second tracking row, which
  matters now that the downloader runs `Count: 3` concurrent consumers. (Optionally also make the insert conditional —
  `INSERT … SELECT … WHERE NOT EXISTS (…)` — for a fully atomic guard.)

Files: `Application/Wmg/WmgDownloaderOptions.cs`, `Application/Wmg/ISnowflakeWmgDownloadProvider.cs`,
`Application/Wmg/SnowflakeWmgDownloadProvider.cs`, `WmgDownloader/MessageHandlers/WmgDownloaderMessageHandler.cs`.

## Part C — SoundCloud content (data/config only, no new code paths)
1. **3 embedded COPY `.sql`** in `Application/Wmg/Sql/Copy/` (C1 style, tokenized `{DATABASE}`/`{PATH}`/`{PATTERN}`), e.g.
   `SoundCloud.CustomerData.copy.sql`:
   ```sql
   COPY INTO {DATABASE}.LANDING.WMG_SOUNDCLOUD_CUSTDATA_TSV_LOAD
       (REPORTING_START_DATE, ... , SUBS_LAST_SUBSCRIPTION_TIER, LOAD_DATETIME, FILENAME)
   FROM (SELECT $1,$2,$3,$4,$5,$6,$7,$8,$9,$10,$11,$12,$13, CURRENT_TIMESTAMP(), METADATA$FILENAME
         FROM @{DATABASE}.DEPLOYMENT.AWS_S3_WMG_STREAMING_DATA/{PATH})
   FILE_FORMAT = (FORMAT_NAME = '{DATABASE}.DEPLOYMENT.SOUNDCLOUD_TSV')
   PATTERN = '{PATTERN}' ON_ERROR = ABORT_STATEMENT
   ```
   Likewise `SoundCloud.StreamLevelReport.copy.sql` (18 cols → `WMG_SOUNDCLOUD_STREAMLEVEL_TSV_LOAD`) and
   `SoundCloud.TrackInformation.copy.sql` (13 cols → `WMG_SOUNDCLOUD_TRACKINFO_TSV_LOAD`). Column lists come verbatim from
   the prod DDLs; all straight `$1..$N`. (Embedded automatically via the existing `Wmg\Sql\Copy\**\*.sql` glob.)
2. **appsettings** — add a `SoundCloud` source (`ServiceId: 85`) to the WmgDownloader `Sources[]`, one shared
   `PathTemplate`, and per-report `LandingTable` + `CopyFile` + `FilePattern`:
   ```jsonc
   { "Name": "SoundCloud", "ServiceId": 85, "Reports": [
     { "Name": "CustomerData",      "PathTemplate": "SoundCloudMarket/raw/year={0:yyyy}/month={0:MM}/day={0:yyyyMMdd}/",
       "LandingTable": "LANDING.WMG_SOUNDCLOUD_CUSTDATA_TSV_LOAD",   "CopyFile": "SoundCloud.CustomerData.copy.sql",     "FilePattern": ".*Customerdata[.]tsv[.]gz" },
     { "Name": "StreamLevelReport", "PathTemplate": "SoundCloudMarket/raw/year={0:yyyy}/month={0:MM}/day={0:yyyyMMdd}/",
       "LandingTable": "LANDING.WMG_SOUNDCLOUD_STREAMLEVEL_TSV_LOAD","CopyFile": "SoundCloud.StreamLevelReport.copy.sql", "FilePattern": ".*Streamlevelreport[.]tsv[.]gz" },
     { "Name": "TrackInformation",  "PathTemplate": "SoundCloudMarket/raw/year={0:yyyy}/month={0:MM}/day={0:yyyyMMdd}/",
       "LandingTable": "LANDING.WMG_SOUNDCLOUD_TRACKINFO_TSV_LOAD", "CopyFile": "SoundCloud.TrackInformation.copy.sql",  "FilePattern": ".*Trackinformation[.]tsv[.]gz" } ] }
   ```
   Add `{ "Name": "SoundCloud", "ServiceId": 85 }` to the **orchestrator** appsettings `Sources[]`.
3. **DAL `Wmg/Sql/`** DDL scripts for traceability (staging + prod-guarded): `SOUNDCLOUD_TSV.sql`,
   `WMG_SOUNDCLOUD_LANDING_TABLES.sql`. No GraphQL change (generic passthrough reused).

## Verification
1. Create Part A objects + grants; build the `Netcore.ReportIngestion` solution (0 errors).
2. **Per-report COPY smoke test** as `ROLE_ADF_STAGING` for `day=20260622`: for each report, run the token-substituted `.sql`
   and confirm (a) **only** its file loads (the specific pattern excludes the other reports + `.marker`), (b) header skipped,
   (c) columns align vs the raw file, (d) count-gate > 0, (e) a re-run loads 0 files (dedup).
3. **End-to-end:** DAL on 5251 + both workers; publish `WmgDownloaderMessage { EnterpriseId=1, Date=2026-06-22, Source="SoundCloud" }`;
   confirm all 3 tables populate and **exactly one** `DAILYTRENDSDOWNLOADS` row (`SERVICEID=85, REPORTTYPE='WMG'`) is written;
   re-run is skipped and writes **no duplicate** row (B2 idempotency); a date missing any of the 3 files writes **no** tracking row (B2 completeness).
4. **Regression:** confirm Spotify + Apple sources still load (they use the global `.*[.]gz`, untouched by the enhancement).
5. Leave changes uncommitted on `vasil/implement-apple-wmg-download` (or a new branch) for review.

---

# Plan: Add Apple Music download to the WMG downloader (generalize to multi-source)

## Context
The WMG Spotify downloader (built in the 2026-07-01 session) loads gzipped **JSON** from a
Snowflake external stage into a single landing table `LANDING.WMG_SPOTIFY_TRENDS_JSON_LOAD` via
`COPY INTO`, orchestrated by two workers in `Netcore.ReportIngestion` that call the
`RepositoryDataAccessor.ReportIngestion` GraphQL DAL. We now want Apple Music reports loaded
**through the same mechanism, integration and stage**.

Apple differs from Spotify in three ways that shape the design:
- Apple files are **tab-delimited TSV with a header row** (`.txt.gz`), not JSON.
- Apple has **6 report types, each with its own landing table and parsed columns** (Spotify was one JSON table).
- For two reports the file column order does **not** match the landing-table column order (a `CONTAINER_TYPE`
  field sits mid-file but is a trailing table column), so those need an explicit positional COPY mapping.

The Apple files already sit in the **same WMG bucket/stage** (`AWS_S3_WMG_STREAMING_DATA` →
`s3://1004-data-forge-us-east-1-snowstage-prod/data/`) under `AppleMusic*/raw/<Report>/day=YYYYMMDD/`,
with split files `[1-n]` and multiple vendor ids per day-directory. A directory-level
`PATTERN='.*[.]gz'` COPY loads all splits/vendors for a day in one statement (same approach Spotify
used for per-country files).

**Decisions (confirmed with user):**
1. **Generalize the existing WMG downloader** into a multi-source pipeline (Spotify + Apple) rather than
   creating a separate Apple worker pair. One orchestrator, one downloader queue, source-driven config.
2. **Tracking:** Apple uses `REPORTTYPE='WMG'`, `SERVICEID=61` (`WellKnownDistributors.AppleMusic`),
   `ENTERPRISEID=1` — same convention as Spotify (`SERVICEID=10`), differentiated only by service id.
   One `DAILYTRENDSDOWNLOADS` row per date after all 6 Apple reports load.

---

## Part A — Snowflake objects (create in `REVELATOR_ALL_STAGING`; script prod guarded)

Created directly using the SYSADMIN connection the user provided (`USER_ADF_ETL`, `role=SYSADMIN`,
account `vva53450`) — the key is held only in the session scratchpad, never committed or saved to memory.
Reuse the existing stage `DEPLOYMENT.AWS_S3_WMG_STREAMING_DATA` (no change — the COPY overrides
`FILE_FORMAT` per statement).

**1. TSV file format** — clone `REVELATOR_ALL.DEPLOYMENT.APPLE_STREAMS_TXT` **verbatim** (same options, no
additions) so staging loads behave exactly like prod. Its exact `GET_DDL` body:
```sql
CREATE FILE FORMAT IF NOT EXISTS REVELATOR_ALL_STAGING.DEPLOYMENT.APPLE_STREAMS_TXT
    FIELD_DELIMITER = '\t'
    SKIP_HEADER = 1
    ESCAPE_UNENCLOSED_FIELD = 'NONE';
-- TYPE defaults to CSV; COMPRESSION defaults to AUTO (verified it reads the .gz files).
```

**2. Six landing tables** `REVELATOR_ALL_STAGING.LANDING.WMG_APPLE_*` — exact copies of the live prod
`REVELATOR_ALL.LANDING.APPLE_*_LOAD` structures (column names/order/types verified via `GET_DDL`), with
`FILENAME` widened to `VARCHAR(1000)` (prod uses 200; the full stage path fits but 1000 is safe and matches
`WMG_SPOTIFY_TRENDS_JSON_LOAD`):
- `WMG_APPLE_TRENDS_STREAMS_SUMMARY_LOAD` (17 cols + LOAD_DATETIME, FILENAME, **trailing** CONTAINER_TYPE)
- `WMG_APPLE_CONTAINER_SUMMARY_LOAD` (…, **trailing** CONTAINER_TYPE)
- `WMG_APPLE_CONTENT_DEMOGRAPHICS_LOAD`
- `WMG_APPLE_LIBRARY_EVENTS_LOAD`
- `WMG_APPLE_EDITORIAL_PLAYLISTADDS_SUMMARY_LOAD`
- `WMG_APPLE_CONTENT_DETAILED_LOAD`

**3. Grants** to the staging DAL role `ROLE_ADF_STAGING` (prior session confirmed it has USAGE on the stage +
integration): `GRANT USAGE ON FILE FORMAT …APPLE_STREAMS_TXT`, and `GRANT INSERT, SELECT` on the 6 new tables.

### Verified per-report COPY column mappings
The COPY template per report is:
```sql
COPY INTO {Database}.{LandingTable} ({Columns})
FROM (SELECT {SourceColumns} FROM @{Database}.DEPLOYMENT.AWS_S3_WMG_STREAMING_DATA/{path})
FILE_FORMAT = (FORMAT_NAME = '{Database}.DEPLOYMENT.APPLE_STREAMS_TXT')
PATTERN = '.*[.]gz' ON_ERROR = ABORT_STATEMENT
```
Only the two "streams" reports reorder (Container Type → trailing column); the other four are straight positional.

| Report / table | Stage path (relative) | `SourceColumns` (SELECT) |
|---|---|---|
| **SummaryStreams** → `WMG_APPLE_TRENDS_STREAMS_SUMMARY_LOAD` | `AppleMusicSummaryStreams/raw/SummaryStreams/day={0:yyyyMMdd}/` | `$1,$2,$3,$4,$5,$6,$7,$8,$9,$10,$12,$13,$14,$15,$16,$17,$18,CURRENT_TIMESTAMP(),METADATA$FILENAME,$11` |
| **ContainerSummary** → `WMG_APPLE_CONTAINER_SUMMARY_LOAD` | `AppleMusicContainerSummary/raw/ContainerSummary/day={0:yyyyMMdd}/` | `$1,$2,$3,$5,$6,$7,$8,CURRENT_TIMESTAMP(),METADATA$FILENAME,$4` |
| **ContentDemographics** → `WMG_APPLE_CONTENT_DEMOGRAPHICS_LOAD` | `AppleMusicDemographicReport/raw/ContentDemographics/day={0:yyyyMMdd}/` | `$1,$2,$3,$4,$5,$6,$7,$8,$9,$10,$11,$12,$13,CURRENT_TIMESTAMP(),METADATA$FILENAME` |
| **LibraryEvents** → `WMG_APPLE_LIBRARY_EVENTS_LOAD` | `AppleMusicLibraryEventsAdds/raw/LibraryEvents/day={0:yyyyMMdd}/` | `$1,$2,$3,$4,$5,$6,$7,$8,$9,CURRENT_TIMESTAMP(),METADATA$FILENAME` |
| **EditorialPlaylistAdds** → `WMG_APPLE_EDITORIAL_PLAYLISTADDS_SUMMARY_LOAD` | `AppleMusicEditorialPlaylistAdds/raw/EditorialPlaylistAdds/day={0:yyyyMMdd}/` | `$1,$2,$3,$4,$5,$6,$7,$8,$9,$10,$11,CURRENT_TIMESTAMP(),METADATA$FILENAME` |
| **Content (detailed)** → `WMG_APPLE_CONTENT_DETAILED_LOAD` | `AppleMusicSummaryStreams/raw/Content/day={0:yyyyMMdd}/` | `$1,$2,$3,$4,$5,$6,$7,$8,$9,$10,$11,$12,CURRENT_TIMESTAMP(),METADATA$FILENAME` |

`{Columns}` for each is the full target column list in table order (see `GET_DDL`), ending
`…,LOAD_DATETIME,FILENAME[,CONTAINER_TYPE]`. The `SourceColumns` above are aligned 1:1 with those lists.

---

## Part B — `Netcore.ReportIngestion` (generalize the worker tier)

Restructure `WmgDownloaderOptions` from *single table + implicit JSON mapping* to *sources → reports*, with
report-level values falling back to source-level defaults (so Spotify stays terse, Apple is per-report).

**1. Options** — `src/Revelator.ReportIngestion.Application/Wmg/WmgDownloaderOptions.cs`
- Keep shared: `Database`, `Stage`, `DailyTrendsDownloadsTable`, `FilePattern`.
- Replace `FileFormat`/`LandingTable`/`Reports[]` with `List<WmgSourceOption> Sources`.
- `WmgSourceOption`: `Name`, `long ServiceId`, `FileFormat`, optional default `LandingTable`/`Columns`/`SourceColumns`, `List<WmgReportOption> Reports`.
- `WmgReportOption`: `Name`, `PathTemplate`, optional `LandingTable`/`Columns`/`SourceColumns` (override the source default).
- Keep the `Qualified*` compose helpers (`{Database}.{…}`).

**2. Provider** — `Application/Wmg/SnowflakeWmgDownloadProvider.cs` (+ interface)
- Keep `ReportPathExists(enterpriseId, date, serviceId)` and `WriteReportPath(enterpriseId, date, serviceId)`
  unchanged (already service-id parameterised; both use `REPORTTYPE='WMG'`).
- Replace `LoadSpotifyTrends(reportPath)` with `LoadReport(reportPath, landingTable, columns, sourceColumns, fileFormat, ct)`:
  builds the COPY from the template above (explicit `({columns})` + `SELECT {sourceColumns}`), then the same
  **count-gate** success check (`SELECT COUNT(*) … WHERE FILENAME LIKE '%{reportPath}%' AND FILENAME RLIKE '{FilePattern}'`).
- Reuse the existing embedded `ExecuteWmgCommand.graphql` (mutation) and `WmgReportPathExists.graphql` (query)
  and `IReportIngestionDataAccessLayerService` — **no DAL API change**.

**3. Messages** — `src/Revelator.ReportIngestion.Abstractions/MessageDtos/`
- Rename `WmgSpotifyDownloaderMessage` → `WmgDownloaderMessage` and add `string Source`. Keep `EnterpriseId`, `Date`.
- Keep `WmgSpotifyDownloadOrchestratorMessage` (empty trigger) — optionally rename to `WmgDownloadOrchestratorMessage`.
  (Feature is a single internal deployment unit, so the queue rename is safe.)

**4. Orchestrator handler** — `WmgDownloadOrchestrator/MessageHandlers/…OrchestratorMessageHandler.cs`
- Loop `Options.Sources`; for each source loop `DaysToDownloadTrends` days; if `!ReportPathExists(1, date, source.ServiceId)`
  enqueue `WmgDownloaderMessage { EnterpriseId=1, Date=date, Source=source.Name }`.

**5. Downloader handler** — `WmgDownloader/MessageHandlers/…DownloaderMessageHandler.cs`
- Keep ack-up-front (`AutoAckBehavior.None` + `Consumer.Ack`) for long loads.
- Resolve `source = Options.Sources.First(s => s.Name == message.Source)`; skip if `ReportPathExists(…, source.ServiceId)`.
- For each `report` in `source.Reports`: resolve effective `LandingTable/Columns/SourceColumns` (report ?? source),
  format the path, call `LoadReport(...)`; track `allLoaded`.
- If all loaded → `WriteReportPath(1, date, source.ServiceId)`; else return so the orchestrator retries.

**6. Bootstrap/queue** — update `WmgDownloader/bootstrap.json` `QueueName` + `MessageFqn` to the renamed
message; update the orchestrator `RunOnceMessage` type if renamed.

**7. appsettings** (both new workers) — new `WmgDownloaderOptions` shape:
```jsonc
"WmgDownloaderOptions": {
  "Database": "REVELATOR_ALL_STAGING",
  "Stage": "DEPLOYMENT.AWS_S3_WMG_STREAMING_DATA",
  "DailyTrendsDownloadsTable": "STAGE.DAILYTRENDSDOWNLOADS",
  "FilePattern": ".*[.]gz",
  "Sources": [
    { "Name": "Spotify", "ServiceId": 10,
      "FileFormat": "DEPLOYMENT.JSON_GZ_FORMAT",
      "LandingTable": "LANDING.WMG_SPOTIFY_TRENDS_JSON_LOAD",
      "Columns": "JSON_VALUE,FILENAME,LOAD_DATETIME",
      "SourceColumns": "$1,METADATA$FILENAME,CURRENT_TIMESTAMP()",
      "Reports": [ { "Name": "tracks", "PathTemplate": "Spotify/raw/tracks/year={0:yyyy}/month={0:MM}/day={0:yyyyMMdd}/" }, /* users, aggregatedstreams, streams, sub30 as before */ ] },
    { "Name": "Apple", "ServiceId": 61,
      "FileFormat": "DEPLOYMENT.APPLE_STREAMS_TXT",
      "Reports": [ /* 6 entries, each with Name, PathTemplate, LandingTable, Columns, SourceColumns from Part A table */ ] }
  ]
}
```
The orchestrator only needs `Database`/`DailyTrendsDownloadsTable` + each source's `ServiceId`/`Reports` names;
the downloader needs the full block. `MicroservicesTimeout` (7200s) and `DownloadSettings:DaysToDownloadTrends`
stay as-is.

---

## Part C — `RepositoryDataAccessor.ReportIngestion` (DAL)
- **No GraphQL change** — `ExecuteWmgCommand` (runs arbitrary SQL, 7200s timeout) and `WmgReportPathExists`
  (count query) are generic and handle Apple's COPY/count as-is.
- Add DDL scripts under `Wmg/Sql/` for traceability (idempotent, staging + prod-guarded, objects created in
  staging now): `APPLE_STREAMS_TXT.sql` and one file per `WMG_APPLE_*` table (or one `WMG_APPLE_LANDING_TABLES.sql`).

---

## Key risks / to confirm during implementation
- **Load parity:** the staging format is a verbatim clone of prod `APPLE_STREAMS_TXT`, which already loads
  these exact reports in prod — so no column-count/enclosure tuning is needed; behaviour matches prod by construction.
- **`CURRENT_TIMESTAMP()` inside the COPY transformation** — used successfully by the Spotify COPY; expected fine.
- **Prod (`REVELATOR_ALL`) rollout:** the DAL role's grants on `LANDING`/`DEPLOYMENT` already exist, so a future
  prod rollout only needs the 6 `WMG_APPLE_*` tables + the `APPLE_STREAMS_TXT` format created (scripts are prod-guarded).

## Verification
1. Create the Part A objects in `REVELATOR_ALL_STAGING` + grants; build the `Netcore.ReportIngestion` solution (0 errors).
2. **Manual COPY smoke test** (via the DAL or SYSADMIN) for one Apple report/day, e.g. ContainerSummary `day=20260624`:
   confirm rows land, `FILENAME` populated, `CONTAINER_TYPE`/`CONTAINER_SUB_TYPE` correctly separated
   (validate against the raw file), and a re-run loads 0 files (load-history dedup) while the count-gate stays >0.
3. Run the ContentDemographics/LibraryEvents/EditorialPlaylistAdds/Content COPYs similarly (straight mappings).
4. **End-to-end:** run the DAL + both workers locally (RabbitMq up), publish `WmgDownloaderMessage { EnterpriseId=1,
   Date=2026-06-24, Source="Apple" }`; confirm all 6 tables populate and one `DAILYTRENDSDOWNLOADS` row
   (`SERVICEID=61, REPORTTYPE='WMG'`) is written; re-run is skipped. Then confirm the **Spotify** source still loads
   (regression) via `Source="Spotify"`.
5. Leave changes uncommitted on a feature branch (`vasil/implement-apple-wmg-download`) for review.
