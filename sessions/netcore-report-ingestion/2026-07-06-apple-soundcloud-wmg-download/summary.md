# Apple + SoundCloud WMG Downloads (multi-source generalization) — 2026-07-06

**Repos:** Netcore.ReportIngestion (worker tier), RepositoryDataAccessor.ReportIngestion (DAL — DDL scripts only)
**Branch:** `vasil/implement-apple-wmg-download` (both repos) — all work uncommitted for review.
**Builds:** both worker projects 0 errors. Snowflake side fully smoke-tested as the runtime role.

Builds on the 2026-07-01 WMG Spotify downloader. Added **Apple** (6 reports) and **SoundCloud** (3 reports)
as additional sources by generalizing the single-source downloader into a config-driven multi-source pipeline.

## What we did

1. **Generalized `WmgDownloaderOptions`** from single-table/JSON to `Sources[] → Reports[]` with report→source
   fallback for defaults. One orchestrator loops all sources; one downloader queue processes all via a `Source`
   field on the message. Renamed DTOs/handlers `WmgSpotify*` → `Wmg*`; queue `WmgSpotifyDownloaderMessage` → `WmgDownloaderMessage`.
2. **Apple (6 reports, TSV `.txt.gz` with header):** created staging file format `APPLE_STREAMS_TXT` (verbatim clone of
   prod) + 6 `WMG_APPLE_*` landing tables. Two reports (`SummaryStreams`, `ContainerSummary`) need a positional
   reorder — `Container Type` is mid-file but a trailing table column; the other 4 are straight `$1..$N`. All 6
   mappings verified by reading real stage files. `ServiceId=61`, `REPORTTYPE='WMG'`.
3. **C1 refactor — COPY-as-embedded-SQL:** moved the per-report column mappings out of verbose appsettings strings
   into runnable, tokenized `.sql` files (`Wmg/Sql/Copy/*.sql`, tokens `{DATABASE}/{PATH}/{PATTERN}`), loaded via
   `EmbeddedResourceLoader` (same pattern as `.graphql`) and executed through the DAL. Chosen over a mapping table
   / stored procs because the concern was config verbosity/error-proneness.
4. **SoundCloud (3 reports, TSV `.tsv.gz` with header):** cloned `SOUNDCLOUD_TSV` format + 3 `WMG_SOUNDCLOUD_*`
   tables. All straight positional maps. `ServiceId=85`.
5. **B1 — per-report `FilePattern`:** SoundCloud's 3 reports (plus 2 unused) share ONE directory distinguished only
   by filename suffix, so each needs a specific pattern (`.*Customerdata[.]tsv[.]gz`, …). Added optional `FilePattern`
   (report→source→global `.*[.]gz`), threaded into `LoadReport` for both the COPY `PATTERN` and the count-gate.
   Backward-compatible (Apple/Spotify keep `.*[.]gz`).
6. **B2 — tracking-write guards (all distributors):** keep the `allLoaded` completeness gate (write
   `DAILYTRENDSDOWNLOADS` only when every configured report loaded); added a pre-write `ReportPathExists` re-check
   for idempotency under the now `Count: 3` concurrent consumers.
7. **Error hardening:** empty/404 DAL responses now throw a clear `InvalidOperationException` instead of a cryptic
   `JsonReaderException` (`ParseDalResponse` guard in the provider).
8. **DAL:** no code/GraphQL change — generic `executeWmgCommand`/`wmgReportPathExists` reused. Added idempotent DDL
   scripts (`APPLE_STREAMS_TXT.sql`, `WMG_APPLE_LANDING_TABLES.sql`, `SOUNDCLOUD_TSV.sql`, `WMG_SOUNDCLOUD_LANDING_TABLES.sql`).

## Key decisions

- **Generalize the existing downloader** (not a separate worker pair per source) — user choice.
- **Tracking convention:** every WMG source uses `REPORTTYPE='WMG'`, differentiated by `SERVICEID` (Spotify=10, Apple=61, SoundCloud=85).
- **C1 (full COPY per embedded `.sql`)** over a Snowflake mapping table or stored procs — most directly removes the
  error-prone hand-authored `$N`/column strings; each file is paste-into-Snowflake runnable.
- **Grants:** not scripted — schema-level FUTURE GRANTS to `ROLE_ADF_STAGING` cover new objects.
- Staging objects created now via SYSADMIN (`USER_ADF_ETL`); prod objects scripted+guarded for later (prod grants already exist).

## Files changed (key)

| File | Change |
|---|---|
| `Application/Wmg/WmgDownloaderOptions.cs` | `Sources[]→Reports[]`; `CopyFile`, `LandingTable`, `FilePattern` + resolvers |
| `Application/Wmg/ISnowflakeWmgDownloadProvider.cs` + `SnowflakeWmgDownloadProvider.cs` | `LoadReport(path, table, copyFile, filePattern)` loads embedded `.sql` + tokens; `ParseDalResponse` guard |
| `Application/Wmg/Sql/Copy/*.copy.sql` | 6 Apple + 3 SoundCloud + 1 shared Spotify embedded COPY files; csproj `**/*.sql` glob |
| `Abstractions/MessageDtos/WmgDownloaderMessage.cs`, `WmgDownloadOrchestratorMessage.cs` | renamed; `Source` field |
| `WmgDownloadOrchestrator` + `WmgDownloader` handlers/bootstrap/appsettings | source loop; per-report pattern; B2 guards; Spotify+Apple+SoundCloud config |
| RepositoryDataAccessor.ReportIngestion `Wmg/Sql/*.sql` | 4 DDL scripts (staging+prod guarded) |
| Snowflake `REVELATOR_ALL_STAGING` | `APPLE_STREAMS_TXT`, `SOUNDCLOUD_TSV` formats; 6 `WMG_APPLE_*` + 3 `WMG_SOUNDCLOUD_*` tables |

## Verified

- All 6 Apple + 3 SoundCloud COPYs load 0-error as `ROLE_ADF_STAGING`; column alignment checked vs raw files;
  load-history dedup (re-run = 0 files); count-gate > 0; per-report patterns load ONLY their own file.
- Embedded `.sql` present in built assembly; token-substituted COPY (as the provider builds it) runs end-to-end.

## Still to do / follow-up

- **Full worker E2E** (RabbitMQ + DAL on 5251) not run here — every dependent layer verified independently.
  Local gotcha: ReportIngestion DAL must run on **5251**, Analytics DAL on **5250** (they collided).
- **Prod rollout:** run the prod-guarded DDL; grants already exist.
- **Apple 6/30–7/3 not downloading = upstream S3 delivery gap** (not a bug): `SummaryStreams`+`ContainerSummary`
  (and some `Content`) files absent for those dates, so the all-reports completeness gate correctly withholds the
  tracking row. Auto-completes when files land. Design note: a permanently-missing report blocks a date forever —
  consider a per-report required/optional flag or max-age cutoff if that becomes an issue.
- Nothing committed — review then commit/PR.
