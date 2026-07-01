# Plan: switch WMG load from `INSERT … SELECT FROM @stage` to `COPY INTO`

## Context
The WMG Spotify downloader loads gzipped JSON from a Snowflake external stage into
`…LANDING.WMG_SPOTIFY_TRENDS_JSON_LOAD`. Today it uses `INSERT … SELECT $1 FROM @stage`, which
runs through the general query engine and is slow for large files (the tracks gz is ~233 MB).
Switching to `COPY INTO` uses Snowflake's optimized bulk loader and gives native load-history
dedup, which also lets us remove the manual delete-before-insert logic.

This document is a PLAN only — no code is to be changed yet.

## Why COPY is better
- **Optimized bulk path**, parallelized per file — much faster than `INSERT … SELECT` for big files.
- **Native load-history dedup** (per table, ~64 days): re-running skips already-loaded files. Better
  restart safety than the current delete-before-insert, which can then be removed.
- **Per-file error handling** (`ON_ERROR`) and load statistics.

## SQL change (in `SnowflakeWmgDownloadProvider.LoadSpotifyTrends`)
From:
```sql
INSERT INTO {landing} (JSON_VALUE, FILENAME, LOAD_DATETIME)
SELECT $1, METADATA$FILENAME, CURRENT_TIMESTAMP()
FROM @{stage}/{path} (FILE_FORMAT => '{fmt}', PATTERN => '{pattern}')
```
To:
```sql
COPY INTO {landing} (JSON_VALUE, FILENAME, LOAD_DATETIME)
FROM (SELECT $1, METADATA$FILENAME, CURRENT_TIMESTAMP() FROM @{stage}/{path})
FILE_FORMAT = (FORMAT_NAME = '{fmt}')
PATTERN = '{pattern}'
ON_ERROR = ABORT_STATEMENT
```
Syntax differences: `FILE_FORMAT = (FORMAT_NAME = …)` (not `=> '…'`); `PATTERN` is a top-level clause.

## Key design consequence — the success signal
Currently `loaded` = rows the INSERT added. With COPY dedup, a re-run loads **0 rows even though data
is present**, which is indistinguishable from "files not delivered yet" — that would break the
orchestrator retry loop (tracking row never written).

Fix: determine success by a **post-COPY count of landing rows for the path**, not by what COPY loaded:
```csharp
await ExecuteCommandAsync(copySql, ct);   // COPY; throws on load error -> retry
var rows = await CountAsync($"SELECT COUNT(*) AS \"Count\" FROM {landing} {matchFilter}", ct);
return rows > 0;                          // present (now or earlier) = success
```
- Files present (this run or a prior partial run) -> count > 0 -> success.
- Never delivered -> count 0 -> retried next orchestrator run.
- No duplicates, because COPY skips already-loaded files.

This **reuses existing `CountAsync`** and **removes** the pre-delete count + `DELETE` block. `FilePattern`
stays (used in COPY `PATTERN` and in `matchFilter`).

## Also: tracking REPORTPATH format
`WriteReportPath` currently writes `REPORTPATH = {enterpriseId}/{serviceId}/{yyyyMMdd}` (e.g. `1/10/20260626`).
Change it to append the report type: `{enterpriseId}/{serviceId}/{yyyyMMdd}/WMG` (e.g. `1/10/20260626/WMG`).
`REPORTTYPE` stays `'WMG'`. Only the `REPORTPATH` string changes.

## Execution decisions (chosen)
- Scope: **Netcore.ReportIngestion only** (Analytics template left unchanged, consistent with the
  ReportIngestion-only directive for behavioral changes).
- `ON_ERROR = ABORT_STATEMENT`.
- `CURRENT_TIMESTAMP()` kept in the COPY transformation.
- One-time `TRUNCATE` of the landing table is **not** performed automatically (table is owned by another
  role; it's a manual/owner step) — flagged for the user.

## Files to change (when implemented)
1. `Netcore.ReportIngestion/.../Application/Wmg/SnowflakeWmgDownloadProvider.cs` — replace delete+insert
   in `LoadSpotifyTrends` with COPY + post-count; drop the delete + pre-count.
   - Decision: ReportIngestion only, or also mirror to the Netcore.Analytics template.
2. **No DAL signature change** — COPY still goes through the existing `executeWmgCommand` mutation
   (`ExecuteSqlRawAsync`); its bool return is now ignored (success comes from the count). The configurable
   2-hour `SetCommandTimeout` already applies.

## Confirm before/while implementing
- **`CURRENT_TIMESTAMP()` inside a COPY transformation** — usually supported; if rejected, rely on a
  `LOAD_DATETIME DEFAULT CURRENT_TIMESTAMP()` on the table. The table is owned by another role (can't
  `ALTER` as `SYSADMIN`), so verify with one test COPY first.
- **`ON_ERROR`** — `ABORT_STATEMENT` (fail+retry the report) vs `CONTINUE` (skip bad rows). Default
  `ABORT_STATEMENT` so bad data isn't silently dropped.
- **One-time transition** — rows previously loaded via the old `INSERT` path are not in COPY's load
  history, so the first COPY could duplicate them. Recommend a one-time `TRUNCATE` of the landing table.

## Verification
1. Run one COPY manually for `2026-06-24` tracks: confirm rows land with `FILENAME` populated and
   `CURRENT_TIMESTAMP()` works.
2. Run it again: confirm 0 files reprocessed (dedup) and the post-count still > 0.
3. End-to-end via the worker: publish a `WmgSpotifyDownloaderMessage`, confirm all reports load and the
   `DAILYTRENDSDOWNLOADS` (REPORTTYPE='WMG') row is written; a re-run is skipped.
