# Playlists downloader "Failed to open Snowflake connection" NRE — 2026-10-12

**Repo:** Netcore.Analytics (investigation only, no code changed)
**Branch:** vasil/atx-1442-unknown-asset-type-page-size (unrelated, untouched)
**Trigger:** Datadog prod error, `analytics-workers-playlistsdownloader`, 2026-10-12 08:34:44 UTC, trace `6acc93a8000000005ce38ca5fe78dea6`

## Symptom

```
Failed to open Snowflake connection. | NullReferenceException
  at Snowflake.Data.Client.SnowflakeDbDataReader.PopulateSchemaTable
  at SnowflakeDbCommand.GetResultsFromQueryIdAsync
  at SnowflakeTrendsDownloadProvider.ToggleProcessingPlaylist   (line 456; also UpdatePlaylistsProcessed line 151)
```

Occurrences (hot retention ~15d, so "first seen 09-28" is just the retention edge — bug is old):
- ToggleProcessingPlaylist NRE: 202; UpdatePlaylistsProcessed NRE: 108; Snowflake 625 "number of waiters for
  this lock exceeds the 20 statements limit" on DAILYTRENDSDOWNLOADS: 25 (09-28, 10-02). All 08:00-08:45 UTC.

## Root cause (verified end to end)

1. The log text is a catch-all — the connection opened fine. The UPDATE was **BLOCKED on the
   `DAILYTRENDSDOWNLOADS` table lock**.
2. Provider runs DML as `ExecuteAsyncInAsyncMode()` + `GetResultsFromQueryIdAsync()`. In Snowflake.Data
   `QueryStatusExtensions.IsStillRunning` does **not** include `BLOCKED` (identical in 5.1.0 and 6.1.0 — the
   .NET 10 driver bump is NOT the cause). Awaiter returns, `GET /queries/{id}/result` long-polls 45s, returns
   "in progress" with no `rowType` -> `sfResultSetMetaData` null -> NRE.
3. The UPDATE still commits later. Query `01c7ad41-0a0d-ec70-001e-6783387e7bc2`: blocked 230.7s, SUCCESS, 1 row,
   ended 08:37:42.6.
4. **Lock holder:** `01c7ad41-0a0d-ff6b-001e-6783387e992a` = final
   `UPDATE STAGE.DAILYTRENDSDOWNLOADS SET ISPROCESSED/ISPLAYLISTSPROCESSED/ISSAVESPROCESSED = TRUE` of
   `SP_LOAD_AMAZON_PULSE_REPORTPATH_V1('1/451/20261010/GB/WMG')` on **WMG_WH (Large)**: exec 0.8s but
   **247s queued_overload** while holding the lock (ended 08:37:42.1).
5. WMG_WH was saturated by ADF-triggered WMG loads running in parallel: SP_LOAD_WMG_APPLE_TRENDS_V1 +
   SP_LOAD_APPLE_TRENDS_REPORTPATH_SUMMARY_V1 (~71 min each), Spotify WMG CTAS (~57 min), ~8 parallel
   SP_LOAD_AMAZON_PULSE_REPORTPATH_V1 (20-30 min each).

Trace shape to recognise it: `POST query-request` -> several `GET /monitoring/queries/{id}` 1-2s apart ->
`GET /queries/{id}/result` lasting **~45s** -> NRE at the end of that span.

Impact: ToggleProcessingPlaylist swallows (noise). UpdatePlaylistsProcessed rethrows -> handler Nacks the final
page and skips `ToggleProcessingPlaylist(false)`; redelivery before the blocked UPDATE commits can re-run
`PopulateStageDailyPlaylists*` (not verified).

## Timing facts (UTC, 10-05..10-12)

| What | When |
|---|---|
| AwsDownloadOrchestrator cron | `0 3 * * *` |
| WMG downloads complete (DAILYTRENDSDOWNLOADS inserts, ~80/day) | 03:01-05:03; outlier 10-12 04:31-06:12 |
| Amazon Pulse 359/451/73 (76 paths) | done ~03:06-03:15 (10-07 03:43, 10-12 06:12) |
| Spotify (10) / Apple (61) | last, up to 05:03 (10-12: 06:12 / 04:31) |
| ADF WMG triggers (before change) | Amazon Pulse ~06:10, Apple/Spotify/Pandora/SoundCloud 07:30 |
| Playlists PROCESSINGPLAYLISTS window | 04:00 - ~10:20 |
| DML on DAILYTRENDSDOWNLOADS blocked >=45s | 0-45/day of ~200-290; max 1213s |

## Key decisions

- User **rescheduled ADF 3h earlier** (Pulse ~03:10, others ~04:30). **Risk:** earlier than download completion
  on some days (Pulse paths 03:15/03:43/06:12; Spotify/Apple after 04:30 on 4 of 8 days). Check that late
  report paths are still picked up; better: ADF waits until all ~80 WMG rows for the date exist.
- Moving earlier reduces but doesn't remove overlap with the playlists window (04:00-10:20).

## Still to do / follow-up

- [ ] Verify after the ADF change: blocked DML count/day drops, no WMG report path skipped (queries below).
- [ ] Netcore fix (proposed, not done): replace `ExecuteAsyncInAsyncMode` + `GetResultsFromQueryIdAsync` with
      `ExecuteNonQueryAsync` for DML (9x SnowflakeTrendsDownloadProvider, 6x SnowflakePlaylistsDownloadProvider);
      merge final `ISPLAYLISTSPROCESSED=1` + `PROCESSINGPLAYLISTS=0` into one UPDATE; honest log text.
- [ ] WMG side: run the bookkeeping UPDATE outside the saturated WMG_WH / cap parallel Pulse CALLs / multi-cluster.
- [ ] Side finding: prod playlists downloader runs on `TEST_WH` (X-Small).

## Debugging recipe

1. Datadog logs: `service:analytics-workers-playlistsdownloader @SourceContext:*SnowflakeTrendsDownloadProvider
   status:error`, patterns on `@error.stack`.
2. Trace -> Snowflake spans: `trace_id:<id> @http.host:vva53450.snowflakecomputing.com`; query id is in the
   `/monitoring/queries/{id}` URL. A ~45s `/result` span = blocked query.
3. Snowflake (USER_ADF_ETL key-pair, role SYSADMIN, warehouse TEST_WH; ACCOUNT_USAGE lags ~15-45 min):
   ```sql
   -- the victim
   SELECT start_time,end_time,execution_status,transaction_blocked_time/1000 blk_s,rows_updated
   FROM snowflake.account_usage.query_history WHERE query_id='<id>';
   -- lock holders in the window: blk_s=0 but long elapsed/queued_overload = holder
   SELECT start_time,end_time,query_id,warehouse_name,transaction_blocked_time/1000 blk_s,
          queued_overload_time/1000 qo_s,execution_time/1000 ex_s,LEFT(query_text,80)
   FROM snowflake.account_usage.query_history
   WHERE start_time BETWEEN '<t-3m>' AND '<t+2m>' AND query_text ILIKE '%DAILYTRENDSDOWNLOADS%'
     AND query_type IN ('UPDATE','INSERT','MERGE','DELETE') ORDER BY start_time;
   -- holder's session -> which proc (same session_id, preceding statements)
   -- WMG download completion per day
   SELECT CONVERT_TIMEZONE('UTC',end_time)::date d,COUNT(*),MIN(..),MAX(..) ... WHERE query_type='INSERT'
     AND query_text ILIKE 'INSERT INTO%DAILYTRENDSDOWNLOADS (ENTERPRISEID, SERVICEID, DATE, REPORTPATH, COUNTRYCODE, REPORTTYPE)%'
     AND query_text ILIKE '%''WMG'')%' GROUP BY 1;
   -- blocked DML per day
   SELECT start_time::date,COUNT(*),COUNT_IF(transaction_blocked_time>=45000),MAX(transaction_blocked_time)/1000
   FROM snowflake.account_usage.query_history WHERE query_text ILIKE 'UPDATE DAILYTRENDSDOWNLOADS%' GROUP BY 1;
   ```
4. Driver source to confirm behaviour: snowflake-connector-net `Snowflake.Data/Core/QueryResultsAwaiter.cs`,
   `SFStatement.cs` (`IsStillRunning`), `Client/SnowflakeDbDataReader.cs` (`PopulateSchemaTable`).
