# Analytics stage vs prod full parity run — 2026-10-02

**Repo:** Netcore.Analytics  
**Branch:** `vasil/analytics-parity-fixes` → [PR #118](https://github.com/Revelator-il/Netcore.Analytics/pull/118) (code fixes). The report lives only in this folder; it was removed from Netcore.Analytics.  

## What we did

- Ran a full parity test of stage **2.0.7** (combined `/v1/*?dimension=` routes) against prod **0.0.96** (old `/byX` routes) over the
  Revelator Analytics MCP servers. It ran from 2026-09-30 to 2026-10-02.
  - Parent **127038**: 1,993 stage/prod pairs. 1,648 PASS, 107 EXPECTED, 190 ties (listed for review, not counted as matches),
    10 VACUOUS / IGNORED-BOTH, 38 FAIL.
  - Child **222139** (Royal Oakie Records): 2,297 stage-only contract tests. 2,035 PASS, 166 EXPECTED, 75 VACUOUS / IGNORED,
    19 FAIL, 2 NOT RUN (the token expired).
  - Covered every domain except YouTube (Consumption, Revenue, Engagement, AS, AS dashboard, Playlists, playlist movements,
    Movement) across base, every sort column, granularity, paging, dates, enums / required params and filters, with filter values
    seeded from each dimension.
- Built a resumable harness:
  - `runner.py` hands out one call at a time: prod ≥ 3 s, stage ≥ 2 s, a per-sitting prod cap, 5-minute back-off on 429 or
    timeout, and infra retry.
  - `parity_compare.py` normalises (N1–N8) and classifies each result. It checks result-set invariance, duplicates, oversize
    pages and filter assertions.
  - `gen_chunk1-4.py` generate the tests; `report.py` and `curls.py` build the outputs.
  - Ran as up to 4 parallel Sonnet MCP-subagent lanes.
- Looked into 2 days of prod Playlists errors in Datadog:
  - One real user 500 → **F-18**.
  - The rest were our own test calls (F-06, F-09).
  - Ingestion: the playlists downloader fails to open Snowflake (81× NullReferenceException) and logs an AMQP unknown delivery tag
    error.
- Report, per-F stage/prod curls and finding log are in this folder (`report.md`, `findings-with-curls.md`, `findings.md` with the final outcome of every F, `report-readme.md`). The harness scripts are not kept in any repo; they were only in the session scratchpad.
- Went through every F case with the user, one at a time, and fixed the real bugs in [PR #118](https://github.com/Revelator-il/Netcore.Analytics/pull/118) (549 unit tests pass).

## Findings (almost all also happen on prod, so they are not regressions)

| # | Finding | Where |
|---|---|---|
| F-02 | Engagement deliveryType sort validated against the distributor mapping | prod + stage |
| F-07 | Country PK collision id 2000 (ZZ/Unknown): duplicate row, page > pageSize | prod + stage |
| F-09 | Playlist movements pageNumber=0 → 500 | prod + stage |
| F-11 | Engagement consumptionTypeIds ignored | prod + stage |
| F-12 | Playlists excludeDistributorIds / excludeTikTok ignored | prod + stage |
| F-13 | sourceOfStream + granularity overflows pageSize | prod + stage |
| F-14 | Movement paging not validated | prod + stage |
| F-15 | Movement quarterly/yearly by track/release > 5 s on stage only | **likely regression** |
| F-16 | Movement country ignores excludeDistributorIds | prod + stage |
| F-17 | AS dashboard MONTHLY totalItemsCount = client × month | prod + stage |
| F-18 | Playlists metric=All + trackIds, 3 months → 500 after 15 s (enterprise 971905) | prod |

Minor findings: F-01 no deterministic default sort, F-03 the 403 says "Payee", F-05 Revenue id/name rename (intentional),
F-06 onlyNew > 5 s, F-08 granular sort ranks by best period, F-10 inconsistent error bodies.

## Outcome per finding

| # | Outcome |
|---|---|
| F-01 | **Fixed:** no-granularity eventdate sort → main metric DESC; `PrimaryKey ASC` tie-breaker on every ORDER BY (Consumption, Engagement, AS, Revenue) |
| F-02 | **Fixed:** handler uses `OrderByDeliveryTypeFieldMapping` |
| F-03 | Accepted |
| F-04 | Withdrawn (stage SingleStore outage) |
| F-05 | Accepted (intentional rename) |
| F-06 | Accepted. The real cause: with onlyNew the range-family playlists get **no date filter**, so the query scans all history |
| F-07 | **Fixed:** `Iso2Code` in the country PrimaryKey. The DAL `[Key]` is PrimaryKey, so EF folded the two rows. The Engagement UGC + granularity variant (arbitrary label per period) is accepted |
| F-08 | Accepted. Prod sorts the same way |
| F-09 / F-14 | **Fixed:** `[Range]` paging on `PlaylistMovementsFilterDto` and `MovementFilterDto` |
| F-10 | **Fixed:** Revenue dimension enum binder; Cyrillic ММ (still on stage in the AS dashboard) → MM; dates formatted yyyy-MM-dd |
| F-11 | **Fixed:** handler forwards `consumptionTypeIds` |
| F-12 | **Fixed:** exclude-distributor placeholder in the playlists fragment |
| F-13 | **Fixed:** timeline query gets the page's `UniqueKeys` pairs |
| F-15 | Left as-is. Identical responses and the same SQL as prod; stage SingleStore is slower (p95 5.2 s vs 2.2 s) |
| F-16 | Withdrawn. False positive: the harness checked which rows were present, not their values |
| F-17 | **Fixed:** dashboard item count taken from the ungrouped page query |
| F-18 | Accepted. Not code: prod SingleStore WLM admission rejections at 15 s during capacity bursts |

## Key decisions

- Rate-limit prod at 3 s per call with a cap per sitting. The user later approved raising it to 1,400 without stopping.
- ASC sorts only on text and id columns.
- Skipped: YouTube, targetEnterpriseId, AS organizationIds, metricsByDate.
- The child ran stage only (no prod child token). The parent got filters on the main dimensions only and no WEEKLY/QUARTERLY
  granularity.
- Ties do not count as a match; they are listed for review instead.
- Heavy queries were made stage-only after prod timeouts (YEARLY, high-cardinality DAILY, Movement track yearly).
- After prod's mid-run data load, we did not refresh stage. Differences caused only by late-loaded data are marked DATA.
- "Timeouts" are the MCP client's 5 s limit (stage spans show HTTP 499 at 5.00 s), not server failures.

## Files changed

| Where | Change |
|---|---|
| Netcore.Analytics PR #118 | 36 files: 4 query services, 5 country templates + playlists fragment, Engagement/Consumption/AS handlers, movements/Revenue filter DTOs, FilterUtilityHelper, controller docs, tests |
| This folder | summary, plan, report, findings (final), findings-with-curls, report-readme |
| `C:\Users\vasil\analytics-parity-F-cases.postman_collection.json` (local only, contains tokens) | Postman collection of every F case, with `stage_base`/`prod_base` variables |

## Still to do / follow-up

- Merge PR #118, deploy to stage, re-run the F folders of the Postman collection on stage.
- Expected differences from prod after deploy: default/tie ordering, country 2000 as two rows, pageSize > 2000 → 400 on movements, Revenue invalid dimension → ValidationProblemDetails.
- Infra (not code): give `api_user_prod` its own SingleStore resource pool and stagger the pipeline maintenance steps (F-18); the DAL should log the real SingleStore error.
- BI: why country id 2000 carries both ZZ and Unknown.
- Ingestion: Snowflake NullReferenceException in the playlists downloader (`SnowflakeTrendsDownloadProvider.cs:34-37`).
