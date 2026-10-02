# Analytics stage vs prod full parity run — 2026-10-02

**Repo:** Netcore.Analytics  
**Branch:** none — report kept uncommitted in Netcore.Analytics; full copy lives in this folder  

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
- Report, per-F stage/prod curls and finding log copied into this folder (`report.md`, `findings-with-curls.md`, `findings.md`, `report-readme.md`). Harness scripts stay uncommitted in `Netcore.Analytics/docs/parity/2026-10-02-analytics-stage-vs-prod/harness/`.

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

| File | Change |
|---|---|
| `docs/parity/2026-10-02-analytics-stage-vs-prod/README.md` (Netcore, uncommitted) | Overview, headline findings, how to re-run |
| `.../report.md` | Full results, repro params, tie list, not-covered list |
| `.../findings-with-curls.md` | Stage and prod curl per F case (tokens are env vars) |
| `.../findings.md` | Finding log with root causes |
| `.../harness/*` | Runner, comparator, generators, report/curl builders |
| `docs/superpowers/plans/2026-09-29-analytics-stage-vs-prod-parity-test-plan.md` | Test plan (rev 4); copied here as plan.md |

## Still to do / follow-up

- F-15: profile stage Movement quarterly/yearly track/release SQL against prod before the stage → prod deploy.
- F-18: reproduce with a token that can see 971905. Trace `6abe76f000000000ea73cd4e7e8322b4`.
- Fix candidates: F-02, F-07, F-09, F-11, F-12, F-13, F-14, F-16, F-17.
- Ingestion: Snowflake NullReferenceException in the playlists downloader (`SnowflakeTrendsDownloadProvider.cs:34-37`).
- Review the 190 parent ties. Re-run the 2 NOT RUN child tests with a fresh token.
- The curls need fresh tokens (all had expired at commit time).
