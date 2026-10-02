# Analytics stage vs prod parity: results

Generated 2026-10-02 07:03. Stage 2.0.7, prod 0.0.96. Plan: `Netcore.Analytics/docs/superpowers/plans/2026-09-29-analytics-stage-vs-prod-parity-test-plan.md`.

Accounts: **parent 127038** (every test as a stage/prod pair), **child 222139** (stage only, contract checks). Tokens are recorded only as their first 8 characters in the run log.


## parent 127038 (stage vs prod): 1993 tests

| Section | PASS | EXPECTED | TIE | TIE-BOUNDARY | TIE-UNORDERED | VACUOUS | IGNORED-BOTH | FAIL |
|---|---|---|---|---|---|---|---|---|
| Smoke + base | 163 | 9 | 6 | 1 | 22 | 0 | 0 | 1 |
| Sorting | 680 | 17 | 41 | 8 | 92 | 0 | 0 | 5 |
| Granularity | 224 | 7 | 2 | 0 | 3 | 0 | 0 | 5 |
| Paging | 113 | 19 | 0 | 1 | 3 | 0 | 0 | 6 |
| Dates | 24 | 18 | 0 | 0 | 0 | 0 | 0 | 0 |
| Enums / required params | 73 | 35 | 0 | 0 | 7 | 0 | 0 | 18 |
| Filters | 371 | 2 | 1 | 3 | 0 | 6 | 4 | 3 |
| **Total** | **1648** | **107** | **50** | **13** | **127** | **6** | **4** | **38** |

| Domain | PASS | EXPECTED | TIE | TIE-BOUNDARY | TIE-UNORDERED | VACUOUS | IGNORED-BOTH | FAIL |
|---|---|---|---|---|---|---|---|---|
| AS dashboard | 51 | 17 | 0 | 0 | 0 | 0 | 0 | 1 |
| Artificial Streams | 154 | 11 | 0 | 1 | 8 | 0 | 0 | 0 |
| Consumption | 411 | 17 | 5 | 3 | 29 | 0 | 0 | 4 |
| Engagement | 395 | 18 | 6 | 1 | 33 | 0 | 1 | 6 |
| Movement | 147 | 11 | 1 | 1 | 9 | 0 | 1 | 22 |
| Playlist movements | 74 | 8 | 1 | 0 | 0 | 0 | 0 | 2 |
| Playlists | 199 | 12 | 31 | 0 | 42 | 6 | 2 | 3 |
| Revenue | 216 | 13 | 6 | 7 | 6 | 0 | 0 | 0 |
| Smoke | 1 | 0 | 0 | 0 | 0 | 0 | 0 | 0 |

## child 222139 (stage only): 2297 tests

| Section | PASS | EXPECTED | VACUOUS | IGNORED | FAIL | NOT RUN |
|---|---|---|---|---|---|---|
| Smoke + base | 169 | 17 | 0 | 0 | 0 | 0 |
| Sorting | 740 | 11 | 0 | 0 | 4 | 2 |
| Granularity | 381 | 10 | 0 | 0 | 11 | 0 |
| Paging | 99 | 17 | 0 | 0 | 2 | 0 |
| Dates | 20 | 15 | 0 | 0 | 0 | 0 |
| Enums / required params | 13 | 19 | 0 | 0 | 0 | 0 |
| Filters | 613 | 0 | 56 | 19 | 2 | 0 |
| **Total** | **2035** | **166** | **56** | **19** | **19** | **2** |

| Domain | PASS | EXPECTED | VACUOUS | IGNORED | FAIL | NOT RUN |
|---|---|---|---|---|---|---|
| AS dashboard | 0 | 4 | 0 | 0 | 0 | 0 |
| Artificial Streams | 105 | 21 | 0 | 0 | 0 | 0 |
| Consumption | 630 | 16 | 0 | 0 | 5 | 0 |
| Engagement | 623 | 16 | 6 | 7 | 11 | 1 |
| Movement | 0 | 1 | 0 | 0 | 0 | 0 |
| Playlist movements | 91 | 7 | 0 | 1 | 2 | 0 |
| Playlists | 297 | 12 | 50 | 7 | 1 | 0 |
| Probe | 0 | 77 | 0 | 0 | 0 | 0 |
| Revenue | 289 | 12 | 0 | 4 | 0 | 1 |

## Findings

Run started 2026-09-30. Parent 127038 (stage vs prod), child 222139 (stage only). Stage 2.0.7, prod 0.0.96.

## F-01: Default sort has no real order without dateGranularity (prod AND stage)
- Default `orderByProperty` = `EventDate`. Without a granularity, `EventDate` is NULL on every row, so
  `ORDER BY ISNULL(EventDate), EventDate` ties on every row and the page content is arbitrary.
- Seen as TIE-UNORDERED:
  - Consumption: streaming track; ugc track, release, artist, label, country
  - Engagement: streaming track, label; ugc track, release, artist, label
- The same dimensions sorted by their main metric DESC all PASS, and the totals are identical, so the data matches.
- Not a regression (same on prod). Anyone relying on the default order gets an arbitrary page. Fix idea: a deterministic
  fallback (main metric DESC, then id).
- Code: `ConsumptionQueryService.BuildOrderClause`, `BaseFilterDto` default order.

## F-02: Engagement streaming deliveryType validates sort against the DISTRIBUTOR mapping (stage)
- `GetEngagementByDeliveryTypeQueryHandler.cs:40-42`: `IsUgc ? OrderByDeliveryTypeUgcFieldMapping : OrderByDistributorFieldMapping`.
  It should be `OrderByDeliveryTypeFieldMapping`, which exists in `EngagementMappingHelper.cs:132`.
- Effect: `orderByProperty=deliverytypeid|deliverytypename` → 400. `distributorid|distributorname` is accepted but has no
  column in the deliveryType SQL.
- Probe: the stage 400 whitelist for streaming deliveryType lists distributorid/distributorname (cs0).
- **Confirmed effect (stage, child 222139):** `orderByProperty=distributorid|distributorname` (asc/desc) on streaming deliveryType
  returns an **empty 200** (`totalItemsCount 0`). The same dimension with any other sort returns 3 rows. The SQL error is swallowed.
- **Prod has the same bug (pre-existing, not a regression):** parent 127038, 2b2 pairs S-EN-streaming-deliveryType-distributor{id,name}-{asc,desc}: prod AND stage both return an empty 200 (base: 3 rows, 442,008,978 streams).

## F-03: 403 message names the wrong role for child sessions (stage)
- Child 222139 on AS client/organization (list + dashboard) gets 403 (correct, parent-only), but the title reads
  `"TenantEnterpriseId: 127038, EnterpriseId: 222139 - Payee  doesn't have permissions to access Client endpoint"`.
  The caller is a child enterprise, not a payee, and the double space suggests an empty placeholder.

## F-04: WITHDRAWN. AS dashboard by client PASSES on healthy stage (the empty 200 was the outage)
- `GET /artificial-streams/dashboard?dimension=client`, parent 127038, W2 (2026-04-01..06-30), with and without
  `orderByProperty=artificialstreamscount desc`: stage `totalItemsCount=0`, all totals 0. Prod: 14 clients, 13,590 streams.
- Re-run once (plan rule for "stage empty / prod has data"): identical empty 200.
- The stage AS *list* `dimension=client` with the same window returns exactly 14 clients / 13,590 streams, so the data is on stage.
- No `mysql.query` error spans on stage in Datadog in the 15 minutes around the calls, so it's not a SQL error the DAL masked.
  The SQL assembly (`ArtificialStreamsQueryService.BuildDashboardCombined` + shared `BuildFilters`) looks equivalent to the list.
  Suspect the dashboard handler/provider path (mapping or a swallowed exception). Not root-caused yet. A direct SingleStore
  check failed on DNS at the time.
- Parent `dimension=organization` (list + dashboard) is 403 on both prod and stage (parity PASS).

## F-05: Revenue metadata keys renamed (intentional, normalised as N8)
- Prod `<dim>Id/<dim>Name/<dim>Version/<dim>ImageId` → stage `id/name/version/imageId`. assetType: prod `{assetType}` → stage `{id, name}`.
- Stage assetType ids: track=4, release=2, video=6. The "track" bucket (raw 1+4) reports id 4, so worth a look if the FE ever uses the id.

## OUTAGE: stage data layer down from ~18:04 (2026-09-30)
- From 18:04 every stage call returned an empty 200 (Consumption, AS), a 500 (Playlists) or a ReadTimeout (the first one).
  Before that, the same queries returned data.
- A direct connection to the stage SingleStore host failed with DNS `getaddrinfo failed`. That matches the known
  "stage SingleStore suspended" pattern: no mysql spans, fast empty 200s.
- 108 stage results saved after 18:03:30 are quarantined in `results/_outage/`, and those calls are re-issued on the next run.
  Prod results from that window are kept, so no prod calls are repeated.
- Parent chunk 1d was stopped mid-way. Prod calls this sitting: 178/700.

- Prod `Playlists byMetadata onlyNew=true` timed out twice (ReadTimeout) and was not sent a third time. Known prod bug (F5 in skill), stage returns 200.

## F-06 (CONFIRMED, pre-existing): Playlists onlyNew takes > 5 s for the parent on stage AND prod
- `GET /playlists?dimension=playlist&metric=streams&onlyNew=true&pageSize=5`, parent 127038: stage ReadTimeout (after the
  SingleStore restart). The child 222139 got a 200 on the same query before the outage. The prod query also times out (known prod bug).
- The onlyNew date filter is `EventDate >= (SELECT MAX(EventDate) FROM <table> WHERE tenant)`, a full-table MAX per call.
  2026-10-01 re-run (warm cache, refreshed data): ReadTimeout again on stage AND prod. A performance problem for large tenants that already exists on prod, not a regression.

## F-07: PrimaryKey collision duplicates one row and drops another (prod AND stage, REAL BUG)
- Consumption `dimension=country` groups by `CountryId, Iso2Code`, but `PrimaryKey` = `CountryId + date`
  (`streaming/consumption/country.sqlt:11`; the ugc template is the same pattern).
- Data: countryId 2000 has two ISO labels in `TRANSACTIONS_BY_TRACK_COUNTRY_DSP`: `ZZ` (2,609,340 streams, parent June) and `Unknown` (2,844).
- API page for `orderByProperty=countryid desc`: prod returns `2000 ZZ` **twice**; stage returns `2000 Unknown` **twice**
  (child 222139 on stage: `2000 ZZ` twice). The other row is lost. Consistent with the DAL client merging rows by their key.
  Which row wins depends on tie order, which caused the prod/stage FAIL S-CS-streaming-country-countryid-desc.
- Impact: one of the two id-2000 rows (~2.6M or ~2.8k streams) disappears from the page; totals stay right.
- Fix ideas: include Iso2Code in PrimaryKey (or group by CountryId only). Also a BI data question: why does id 2000 carry both ZZ and 'Unknown'?
- **Also on ugc + granularity (child 222139, stage):** `engagement?dimension=country&mediaType=ugc&dateGranularity=QUARTERLY`
  `&orderByProperty=creationscount desc&pageSize=5` returns **6 items for pageSize 5**. `Unknown` (id 2000) and `ZZ` (id 2000)
  carry identical metrics [17466, 0, 0], so one row's data is copied onto the other and the page size and order break.
- Other dimensions whose GROUP BY has more columns than their PrimaryKey may have the same issue (to check in later chunks).

## F-08 (design question): with dateGranularity, sort orders items by their best single period, not their total (stage; prod TBD)
- `consumption?dimension=track&mediaType=streaming&dateGranularity=MONTHLY&orderByProperty=streamscount&orderByDescending=true`
  (child 222139, Apr-Jun): items come out by their highest single-month value (63,943 / 6,664 / 5,014 / 4,492 / 4,337).
  Track 6560000 (total 9,189) ranks above 3383285 (total 10,772).
- The ORDER BY runs on the per-period rows and the items are assembled in order of first appearance. A user sorting by streams
  probably expects row totals. Parent pairs in chunk 3g will show whether prod behaves the same.

## F-09: Playlist movements paging validation (prod AND stage, pre-existing)
- `GET /playlists/movements?movementType=active&period=28&pageNumber=0` → **HTTP 500** (unhandled). Every other endpoint returns 400. Real bug.
- `pageSize=0` → 200 with an empty page (`pageSize 0`, `totalPagesCount 0`). Every other endpoint returns 400. Inconsistent validation.
- `pageSize=2001` → clamped to 2000 (documented: "defaults to and capped at 2000"), not a bug.
- Parent pairs P-PM-active-page0 / -size0: **prod returns the same 500 / empty 200**, so it's a pre-existing bug, not a regression.
- Related, prod-only: Playlists byMetadata `pageNumber=0` → **HTTP 500 on prod**, 400 on stage (fixed by standards work).
  Prod also accepts `pageSize=0` / `pageSize=2001` / `pageNumber=0` with 200 on Consumption, Engagement, Revenue, AS; stage rejects
  them with 400 (intentional standards contract change).

## F-10 (minor): inconsistent error bodies (stage)
- Revenue unknown dimension → `{'error': 'The provided dimension is not valid.'}` (no ProblemDetails, no traceId). Every other endpoint
  returns ValidationProblemDetails "Invalid value 'zzdim' for Dimension".
- Prod's malformed-date error reads `yyyy-ММ-dd` with **Cyrillic М** (U+041C) characters; stage reads `yyyy-MM-dd` (fixed by the standards work).
- Date validation messages print culture-formatted dates (`6/30/2026 12:00:00 AM` with U+202F narrow no-break space) instead of yyyy-MM-dd.
- Note (expected, monthly data): Revenue `fromDate == toDate` on a mid-month day returns 0 rows (statements are dated the 1st of the month).

## F-11: Engagement `consumptionTypeIds` filter is ignored (prod AND stage, pre-existing)
- `engagement?dimension=consumptionType&mediaType=streaming&consumptionTypeIds=41,44` (child 222139, June) returns all 11 consumption
  types with the unfiltered totals (12,134 sub30s), exactly as with no filter. Consumption with the same kind of filter applies it.
- Cause: `GetEngagementByConsumptionTypeQueryHandler.cs:59-62` copies the request with
  `CopyTrendsFilter<TrendsQueryFilterDto, TrendsFilterDto>` and never sets `queryFilter.ConsumptionTypeIds = filter.ConsumptionTypeIds`.
  Every sibling handler (Engagement deliveryType / recordingVersion / sourceOfStream / discoveryType, Consumption consumptionType)
  copies with its dimension-specific DTO and sets the id list explicitly.
- Parent pairs (4f3): prod ignores it too (`consumptionTypeIds=41,44` returns 45, 47, 48), so it is pre-existing, not a regression.
- Also on ugc: `engagement?dimension=consumptionType&mediaType=ugc&consumptionTypeIds=999999999` (non-existent) still returns data.
- `EngagementControllerDispatchTests` asserts the controller passes `ConsumptionTypeIds` into the query, but nothing asserts the
  handler forwards it to the query filter. Same failure pattern as PR #99 (CopyTrendsFilterBase dropping assetTypes).

## F-12: Playlists ignores `excludeDistributorIds` and `excludeTikTok` (prod AND stage, pre-existing)
- `playlists?dimension=distributor&metric=streams&excludeDistributorIds=10` (child 222139, June) still returns Spotify (10) and the
  unfiltered total (4,169 playlist streams). `distributorIds=10` filters correctly (4,062).
- Same on `dimension=playlist`: the result is identical to no filter.
- Cause: `PlaylistsFilterDto : FilterDto` accepts the exclude params (the stage tool schema advertises them), but neither
  `PlaylistsQueryService` nor the `playlists/*.sqlt` templates contain any exclude-distributor filter, so they're silently dropped.
  `excludeTikTok` (which feeds the same list) is ignored too.
- Parent pairs (4f6): prod and stage both ignore it on playlist, distributor and country (Spotify 10 still present), so it is pre-existing.

## F-13: sourceOfStream + dateGranularity returns more rows than pageSize (prod AND stage, pre-existing)
- `consumption?dimension=sourceOfStream&mediaType=streaming&dateGranularity=DAILY&orderByProperty=streamscount desc&pageSize=5`
  (parent, June): **8 rows** on page 1, identical on prod and stage. Without granularity, page 1 has the expected 5 (sos, distributor) pairs
  (4,10) (6,10) (7,10) (11,61) (3,61).
- The 3 extra rows (3,10) (6,11) (3,11) are combinations of those same sourceOfStream ids and distributor ids. The timeline-fill second query
  (`GetConsumptionBySourceOfStreamQueryHandler.cs:80-89`, `DimensionIdSetters`) filters by the id lists separately, not by the composite key,
  so rows from other pages bleed onto page 1 (and still appear on their own pages: duplicates across pages, broken page size).
- `uniqueKeys` (sourceOfStreamId/distributorId pairs) would be the right filter for the second query.
- Scope (all results re-scanned with a page-size check): only `dimension=sourceOfStream` with a dateGranularity, Consumption (8 tests) and
  Engagement (5 tests), parent and child. No other dimension returns more rows than pageSize.

## F-14 (minor): Movement accepts out-of-range paging (prod AND stage)
- `movement?dimension=track&pageSize=0`, `pageSize=2001`, `pageNumber=0` all return 200 on both prod and stage. Every other combined endpoint
  returns 400 for these on stage. The Movement contract documents only the 2000 cap ("capped at 2000"), so `pageSize=0` and `pageNumber=0`
  being accepted is a validation gap, not a regression.

## F-15 (performance, LIKELY REGRESSION): stage Movement quarterly/yearly by track/release takes > 5 s; prod answers in < 5 s
- `movement?dimension=track&metricType=revenueChangeAbsolute&movementType=rising&periodType=quarterly` (parent): **prod 200**
  (4,854 items), **stage ReadTimeout twice**. Stage also timed out on `revenueChangePercent / yearly / falling`. Prod timed out on some yearly
  variants too, so both environments are close to the MCP timeout on these; stage is the slower one for quarterly.
- Chunk 3e1 (parent, 2026-10-02 ~00:00): **11 of 11** stage calls `movement?dimension=track&periodType=quarterly|yearly` (metricType
  streamChangePercent / streamChangeAbsolute / revenueChangePercent, rising and falling) timed out twice; **prod returned 200 for all 11**.
  daily / weekly / monthly work on stage. Other dimensions (release, artist, label, country, distributor) at quarterly/yearly also returned on stage.
- Chunk 3e2: also **release** at quarterly and yearly times out on stage twice (prod: release quarterly 200 after one retry).
  label and country quarterly time out once on stage, then succeed. So the slowness grows with dimension cardinality (track > release > label/country).
- Also seen: Movement `pageNumber=2429` (past the end of 11,849 tracks), prod 200 after one retry, stage ReadTimeout twice.

## F-16: Movement `dimension=country` ignores `excludeDistributorIds` (prod AND stage, pre-existing)
- `movement?dimension=country&metricType=streamChangeAbsolute&movementType=rising&periodType=monthly&excludeDistributorIds=85` (parent):
  result identical to the unfiltered call (15 countries, same values) on both sides. On the same dimension `distributorIds=85` does
  change the result, and on track and distributor `excludeDistributorIds` works. So the exclude filter is dropped only for the country template.

## F-17: AS dashboard with MONTHLY reports the wrong totalItemsCount (prod AND stage, pre-existing)
- `artificial-streams/dashboard?dimension=client&dateGranularity=MONTHLY&fromDate=2026-04-01&toDate=2026-06-30&pageSize=5` (parent):
  `totalItemsCount=31` on both sides. Without granularity it is 14 (the number of clients). The page itself is right: 5 distinct clients,
  each with 3 monthly entries. So the count is the number of client×month rows, and `totalPagesCount` (stage) / client paging overstate the
  pages (7 instead of 3; trailing pages come back empty). The AS *list* endpoint with MONTHLY reports the right count.

## Note on 'timeouts' (Datadog, 2026-10-02)
- Every timeout in this run is the **MCP client's 5 s read timeout**, not a server failure. Stage spans for the Movement calls end with
  **HTTP 499 at 5.00 s** (client closed); prod `onlyNew` errors are `TaskCanceledException` at ~4,985 ms. No server-side Movement
  errors were logged in any environment. So F-06 and F-15 mean "slower than 5 s", and F-15 is still a stage-vs-prod performance gap.

## F-18 (prod, real user): Playlists metric=All with trackIds over 3 months fails after 15 s
- 2026-10-01 15:06 UTC, enterprise 971905 (child of 960929), FE call
  `playlists/byMetadata?metric=All&metadata=Distributor&trackIds[0]=8728903&fromDate=2026-07-04&toDate=2026-10-01&dateGranularity=All&orderByDescending=true&pageSize=100`
  → HTTP 500. The DAL GraphQL `PlaylistByMetadata` span returns "Unexpected Execution Error" after exactly 15.0 s (trace 6abe76f000000000ea73cd4e7e8322b4).
- Not covered by the parity run (we tested metric=all on 1 month, pageSize 5). The 15 s limit is not the DAL's GraphQL timeout
  (7200 s); most likely a SingleStore query/resource-pool limit on the metric=All (combined range + snapshot) query.
- Not in the F-list before: new.

## Ingestion (not API, outside the parity scope): playlists downloader
- `analytics-workers-playlistsdownloader`, prod, 2026-09-30 08:02 → 2026-10-01 08:53: **81× "Failed to open Snowflake connection"**,
  `NullReferenceException` thrown inside `SnowflakeDbConnection.OpenAsync` (`SnowflakeTrendsDownloadProvider.cs:34-37`). Recurs on each daily run,
  so it looks like a config/secret or Snowflake driver problem, not a transient one.
- Follow-on errors: 'Error occurred while fetching playlist report paths' / 'populating playlist fact tables' (canceled), a Deezer download
  (EnterpriseId 1, DistributorId 11, 2026-09-28) failing, and one worker fault on an AMQP `PRECONDITION_FAILED - unknown delivery tag`
  (a RabbitMQ ack after the channel was closed).

## E4 (documented divergence)
- Movement `dimension=client`: prod silently returns track-shaped rows (200). Stage returns 400 "Invalid value 'client' for Dimension".

## Reviewed filter results explained by data (not findings)
- Child 222139 Engagement ugc consumptionType/deliveryType: the account's only engagement row there is YouTube Content ID (distributor 19):
  distributorIds incl. 19, excluding 407/TikTok, and seeds from other dimensions legitimately return the unfiltered/empty result.
- Child has a single label (291187), single payee (228333) and audio only: label/payee filters equal the unfiltered result; assetTypes=video is empty.

- Child Revenue deliveryType: countryIds / distributorIds / excludeDistributorIds return the unfiltered result. Documented: the
  deliveryType data has no country or distributor columns ("ignored for deliveryType").
- Child Playlists metric=followers: no follower data for this child in June (it was empty in chunk 1 too), so all 50 filter tests are VACUOUS.
- Child single label 291187: labelIds on Playlists / playlist movements equal the unfiltered result.

## Notes
- Revenue `dimension=date` totals == prod byAssetType totals exactly. They are higher than byTrack by release-level downloads + video, as they should be.
- Prod now returns 400, not 500, for Revenue `labelname` on track and `deliverytypenname` (the skill's E8 note is stale).
- Revenue stage whitelists include `storestatementmonth` on every dimension (known default-sort + granularity bug area).
- Playlists accepts all 16 sort columns on every dimension (e.g. playlistname on country), so chunk 2e checks what that does.
- MCP "session expired" on the first prod call of 1a was a connection issue. The runner now re-issues it once.
- Sonnet copying verified: 3 saved files match independent fetches byte for byte; 85+ prod/stage pairs match.


## Every FAIL / IGNORED / VACUOUS (with repro params)

- **PL-E6-onlyNew** (parent 127038, FAIL): prod 599, stage 599 (expected {'prod_status': 'any', 'stage_status': 200}); HTTP request timed out (ReadTimeout)
  - stage params: `{"dimension": "playlist", "metric": "streams", "onlyNew": true, "pageSize": 5}`
  - prod params: `{"metadata": "playlist", "metric": "streams", "onlyNew": true, "pageSize": 5}`
- **S-CS-streaming-country-countryid-desc** (parent 127038, FAIL): prod: the same row appears 2x on one page (duplicate row, see F-07); items[0].metadata.iso2Code: prod=ZZ stage=Unknown; items[0].metrics[0].streamsCount: prod=2609340 stage=2844
  - stage params: `{"dimension": "country", "mediaType": "streaming", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "countryid", "orderByDescending": true, "pageSize": 5}`
  - prod params: `{"dimension": "country", "mediaType": "streaming", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "countryid", "orderByDescending": true, "pageSize": 5}`
- **S-EN-streaming-deliveryType-distributorid-asc** (parent 127038, FAIL): prod: totalItemsCount 0 != base 3 (EN-streaming-deliveryType-base): sort changed the result set; prod: totals.streamsCount 0 != base 442008978 (EN-streaming-deliveryType-base); prod: totals.sub30sCount 0 != base 52034020 (EN-streaming-deliveryType-base)
  - stage params: `{"dimension": "deliveryType", "mediaType": "streaming", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "distributorid", "orderByDescending": false, "pageSize": 5}`
  - prod params: `{"isUgc": false, "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "distributorid", "orderByDescending": false, "pageSize": 5}`
- **S-EN-streaming-deliveryType-distributorid-desc** (parent 127038, FAIL): prod: totalItemsCount 0 != base 3 (EN-streaming-deliveryType-base): sort changed the result set; prod: totals.streamsCount 0 != base 442008978 (EN-streaming-deliveryType-base); prod: totals.sub30sCount 0 != base 52034020 (EN-streaming-deliveryType-base)
  - stage params: `{"dimension": "deliveryType", "mediaType": "streaming", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "distributorid", "orderByDescending": true, "pageSize": 5}`
  - prod params: `{"isUgc": false, "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "distributorid", "orderByDescending": true, "pageSize": 5}`
- **S-EN-streaming-deliveryType-distributorname-asc** (parent 127038, FAIL): prod: totalItemsCount 0 != base 3 (EN-streaming-deliveryType-base): sort changed the result set; prod: totals.streamsCount 0 != base 442008978 (EN-streaming-deliveryType-base); prod: totals.sub30sCount 0 != base 52034020 (EN-streaming-deliveryType-base)
  - stage params: `{"dimension": "deliveryType", "mediaType": "streaming", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "distributorname", "orderByDescending": false, "pageSize": 5}`
  - prod params: `{"isUgc": false, "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "distributorname", "orderByDescending": false, "pageSize": 5}`
- **S-EN-streaming-deliveryType-distributorname-desc** (parent 127038, FAIL): prod: totalItemsCount 0 != base 3 (EN-streaming-deliveryType-base): sort changed the result set; prod: totals.streamsCount 0 != base 442008978 (EN-streaming-deliveryType-base); prod: totals.sub30sCount 0 != base 52034020 (EN-streaming-deliveryType-base)
  - stage params: `{"dimension": "deliveryType", "mediaType": "streaming", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "distributorname", "orderByDescending": true, "pageSize": 5}`
  - prod params: `{"isUgc": false, "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "distributorname", "orderByDescending": true, "pageSize": 5}`
- **E-MV-track-streamChangePercent-quarterly-rising** (parent 127038, FAIL): stage status 599, expected 200: Error calling tool 'Movement_GetMovement': HTTP request timed out (ReadTimeout)
  - stage params: `{"dimension": "track", "metricType": "streamChangePercent", "movementType": "rising", "periodType": "quarterly", "orderByProperty": "absolutechange", "orderByDescending": true, "pageSize": 5}`
- **E-MV-track-streamChangePercent-quarterly-falling** (parent 127038, FAIL): stage status 599, expected 200: Error calling tool 'Movement_GetMovement': HTTP request timed out (ReadTimeout)
  - stage params: `{"dimension": "track", "metricType": "streamChangePercent", "movementType": "falling", "periodType": "quarterly", "orderByProperty": "absolutechange", "orderByDescending": true, "pageSize": 5}`
- **E-MV-track-streamChangePercent-yearly-rising** (parent 127038, FAIL): stage status 599, expected 200: Error calling tool 'Movement_GetMovement': HTTP request timed out (ReadTimeout)
  - stage params: `{"dimension": "track", "metricType": "streamChangePercent", "movementType": "rising", "periodType": "yearly", "orderByProperty": "absolutechange", "orderByDescending": true, "pageSize": 5}`
- **E-MV-track-streamChangePercent-yearly-falling** (parent 127038, FAIL): stage status 599, expected 200: Error calling tool 'Movement_GetMovement': HTTP request timed out (ReadTimeout)
  - stage params: `{"dimension": "track", "metricType": "streamChangePercent", "movementType": "falling", "periodType": "yearly", "orderByProperty": "absolutechange", "orderByDescending": true, "pageSize": 5}`
- **E-MV-track-streamChangeAbsolute-quarterly-rising** (parent 127038, FAIL): stage status 599, expected 200: Error calling tool 'Movement_GetMovement': HTTP request timed out (ReadTimeout)
  - stage params: `{"dimension": "track", "metricType": "streamChangeAbsolute", "movementType": "rising", "periodType": "quarterly", "orderByProperty": "absolutechange", "orderByDescending": true, "pageSize": 5}`
- **E-MV-track-streamChangeAbsolute-quarterly-falling** (parent 127038, FAIL): stage status 599, expected 200: Error calling tool 'Movement_GetMovement': HTTP request timed out (ReadTimeout)
  - stage params: `{"dimension": "track", "metricType": "streamChangeAbsolute", "movementType": "falling", "periodType": "quarterly", "orderByProperty": "absolutechange", "orderByDescending": true, "pageSize": 5}`
- **E-MV-track-streamChangeAbsolute-yearly-rising** (parent 127038, FAIL): stage status 599, expected 200: Error calling tool 'Movement_GetMovement': HTTP request timed out (ReadTimeout)
  - stage params: `{"dimension": "track", "metricType": "streamChangeAbsolute", "movementType": "rising", "periodType": "yearly", "orderByProperty": "absolutechange", "orderByDescending": true, "pageSize": 5}`
- **E-MV-track-streamChangeAbsolute-yearly-falling** (parent 127038, FAIL): stage status 599, expected 200: Error calling tool 'Movement_GetMovement': HTTP request timed out (ReadTimeout)
  - stage params: `{"dimension": "track", "metricType": "streamChangeAbsolute", "movementType": "falling", "periodType": "yearly", "orderByProperty": "absolutechange", "orderByDescending": true, "pageSize": 5}`
- **E-MV-track-revenueChangePercent-quarterly-rising** (parent 127038, FAIL): stage status 599, expected 200: Error calling tool 'Movement_GetMovement': HTTP request timed out (ReadTimeout)
  - stage params: `{"dimension": "track", "metricType": "revenueChangePercent", "movementType": "rising", "periodType": "quarterly", "orderByProperty": "absolutechange", "orderByDescending": true, "pageSize": 5}`
- **E-MV-track-revenueChangePercent-quarterly-falling** (parent 127038, FAIL): stage status 599, expected 200: Error calling tool 'Movement_GetMovement': HTTP request timed out (ReadTimeout)
  - stage params: `{"dimension": "track", "metricType": "revenueChangePercent", "movementType": "falling", "periodType": "quarterly", "orderByProperty": "absolutechange", "orderByDescending": true, "pageSize": 5}`
- **E-MV-track-revenueChangePercent-yearly-rising** (parent 127038, FAIL): stage status 599, expected 200: Error calling tool 'Movement_GetMovement': HTTP request timed out (ReadTimeout)
  - stage params: `{"dimension": "track", "metricType": "revenueChangePercent", "movementType": "rising", "periodType": "yearly", "orderByProperty": "absolutechange", "orderByDescending": true, "pageSize": 5}`
- **E-MV-track-revenueChangePercent-yearly-falling** (parent 127038, FAIL): stage status 599, expected 200: Error calling tool 'Movement_GetMovement': HTTP request timed out (ReadTimeout)
  - stage params: `{"dimension": "track", "metricType": "revenueChangePercent", "movementType": "falling", "periodType": "yearly", "orderByProperty": "absolutechange", "orderByDescending": true, "pageSize": 5}`
- **E-MV-track-revenueChangeAbsolute-quarterly-rising** (parent 127038, FAIL): stage status 599, expected 200: Error calling tool 'Movement_GetMovement': HTTP request timed out (ReadTimeout)
  - stage params: `{"dimension": "track", "metricType": "revenueChangeAbsolute", "movementType": "rising", "periodType": "quarterly", "orderByProperty": "absolutechange", "orderByDescending": true, "pageSize": 5}`
- **E-MV-track-revenueChangeAbsolute-quarterly-falling** (parent 127038, FAIL): stage status 599, expected 200: Error calling tool 'Movement_GetMovement': HTTP request timed out (ReadTimeout)
  - stage params: `{"dimension": "track", "metricType": "revenueChangeAbsolute", "movementType": "falling", "periodType": "quarterly", "orderByProperty": "absolutechange", "orderByDescending": true, "pageSize": 5}`
- **E-MV-track-revenueChangeAbsolute-yearly-rising** (parent 127038, FAIL): stage status 599, expected 200: Error calling tool 'Movement_GetMovement': HTTP request timed out (ReadTimeout)
  - stage params: `{"dimension": "track", "metricType": "revenueChangeAbsolute", "movementType": "rising", "periodType": "yearly", "orderByProperty": "absolutechange", "orderByDescending": true, "pageSize": 5}`
- **E-MV-track-revenueChangeAbsolute-yearly-falling** (parent 127038, FAIL): stage status 599, expected 200: Error calling tool 'Movement_GetMovement': HTTP request timed out (ReadTimeout)
  - stage params: `{"dimension": "track", "metricType": "revenueChangeAbsolute", "movementType": "falling", "periodType": "yearly", "orderByProperty": "absolutechange", "orderByDescending": true, "pageSize": 5}`
- **E-MV-release-quarterly** (parent 127038, FAIL): prod 200, stage 599; prod: {"fromDate":null,"toDate":null,"previousFromDate":null,"previousToDate":null,"totalItemsCount":0,"pageNumber":1,"pageSize":0,"items":[]}; stage: Error calling tool 'Movement_GetMovement': HTTP request timed out (ReadTimeout)
  - stage params: `{"dimension": "release", "metricType": "streamChangeAbsolute", "movementType": "rising", "periodType": "quarterly", "orderByProperty": "absolutechange", "orderByDescending": true, "pageSize": 5}`
  - prod params: `{"dimension": "release", "metricType": "streamChangeAbsolute", "movementType": "rising", "periodType": "quarterly", "orderByProperty": "absolutechange", "orderByDescending": true, "pageSize": 5}`
- **E-MV-release-yearly** (parent 127038, FAIL): stage status 599, expected 200: Error calling tool 'Movement_GetMovement': HTTP request timed out (ReadTimeout)
  - stage params: `{"dimension": "release", "metricType": "streamChangeAbsolute", "movementType": "rising", "periodType": "yearly", "orderByProperty": "absolutechange", "orderByDescending": true, "pageSize": 5}`
- **G-CS-streaming-sourceOfStream-DAILY** (parent 127038, FAIL): prod: page has 8 rows for pageSize=5 (see F-13); stage: page has 8 rows for pageSize=5 (see F-13)
  - stage params: `{"dimension": "sourceOfStream", "mediaType": "streaming", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "streamscount", "orderByDescending": true, "pageSize": 5, "dateGranularity": "DAILY"}`
  - prod params: `{"dimension": "sourceOfStream", "mediaType": "streaming", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "streamscount", "orderByDescending": true, "pageSize": 5, "dateGranularity": "DAILY"}`
- **G-CS-streaming-sourceOfStream-MONTHLY** (parent 127038, FAIL): prod: page has 8 rows for pageSize=5 (see F-13); stage: page has 8 rows for pageSize=5 (see F-13)
  - stage params: `{"dimension": "sourceOfStream", "mediaType": "streaming", "fromDate": "2026-04-01", "toDate": "2026-06-30", "orderByProperty": "streamscount", "orderByDescending": true, "pageSize": 5, "dateGranularity": "MONTHLY"}`
  - prod params: `{"dimension": "sourceOfStream", "mediaType": "streaming", "fromDate": "2026-04-01", "toDate": "2026-06-30", "orderByProperty": "streamscount", "orderByDescending": true, "pageSize": 5, "dateGranularity": "MONTHLY"}`
- **G-CS-streaming-sourceOfStream-YEARLY** (parent 127038, FAIL): prod: page has 8 rows for pageSize=5 (see F-13); stage: page has 8 rows for pageSize=5 (see F-13)
  - stage params: `{"dimension": "sourceOfStream", "mediaType": "streaming", "fromDate": "2026-04-01", "toDate": "2026-06-30", "orderByProperty": "streamscount", "orderByDescending": true, "pageSize": 5, "dateGranularity": "YEARLY"}`
  - prod params: `{"dimension": "sourceOfStream", "mediaType": "streaming", "fromDate": "2026-04-01", "toDate": "2026-06-30", "orderByProperty": "streamscount", "orderByDescending": true, "pageSize": 5, "dateGranularity": "YEARLY"}`
- **G-EN-streaming-sourceOfStream-MONTHLY** (parent 127038, FAIL): prod: page has 11 rows for pageSize=5 (see F-13); stage: page has 11 rows for pageSize=5 (see F-13); items: same rows, different order
  - stage params: `{"dimension": "sourceOfStream", "mediaType": "streaming", "fromDate": "2026-04-01", "toDate": "2026-06-30", "orderByProperty": "sub30scount", "orderByDescending": true, "pageSize": 5, "dateGranularity": "MONTHLY"}`
  - prod params: `{"isUgc": false, "fromDate": "2026-04-01", "toDate": "2026-06-30", "orderByProperty": "sub30scount", "orderByDescending": true, "pageSize": 5, "dateGranularity": "MONTHLY"}`
- **G-ASD-client-MONTHLY** (parent 127038, FAIL): prod: totalItemsCount 31 != base 14 (ASD-client-artificialstreamscount-desc): sort changed the result set; stage: totalItemsCount 31 != base 14 (ASD-client-artificialstreamscount-desc): sort changed the result set
  - stage params: `{"dimension": "client", "fromDate": "2026-04-01", "toDate": "2026-06-30", "orderByProperty": "artificialstreamscount", "orderByDescending": true, "pageSize": 5, "dateGranularity": "MONTHLY"}`
  - prod params: `{"fromDate": "2026-04-01", "toDate": "2026-06-30", "orderByProperty": "artificialstreamscount", "orderByDescending": true, "pageSize": 5, "dateGranularity": "MONTHLY"}`
- **P-PM-active-size0** (parent 127038, FAIL): prod 200, stage 200 (expected {'prod_status': 'any', 'stage_status': 400})
  - stage params: `{"movementType": "active", "period": 28, "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 0}`
  - prod params: `{"movementType": "active", "period": 28, "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 0}`
- **P-PM-active-page0** (parent 127038, FAIL): prod 500, stage 500 (expected {'prod_status': 'any', 'stage_status': 400}); HTTP error 500: Internal Server Error - {'type': 'https://tools.ietf.org/html/rfc9110#section-15.6.1', 'title': 'An error occurred while processing your request
  - stage params: `{"movementType": "active", "period": 28, "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "pageNumber": 0}`
  - prod params: `{"movementType": "active", "period": 28, "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "pageNumber": 0}`
- **P-MV-track-pastend2429** (parent 127038, FAIL): prod 200, stage 599; prod: {"fromDate":"2026-08-28T00:00:00Z","toDate":"2026-09-27T00:00:00Z","previousFromDate":"2026-07-28T00:00:00Z","previousToDate":"2026-08-27T00:00:00Z","tota; stage: Error calling tool 'Movement_GetMovement': HTTP request timed out (ReadTimeout)
  - stage params: `{"dimension": "track", "metricType": "streamChangeAbsolute", "movementType": "rising", "periodType": "monthly", "orderByProperty": "absolutechange", "orderByDescending": true, "pageSize": 5, "pageNumber": 2429}`
  - prod params: `{"dimension": "track", "metricType": "streamChangeAbsolute", "movementType": "rising", "periodType": "monthly", "orderByProperty": "absolutechange", "orderByDescending": true, "pageSize": 5, "pageNumber": 2429}`
- **P-MV-track-size0** (parent 127038, FAIL): prod 200, stage 200 (expected {'prod_status': 'any', 'stage_status': 400})
  - stage params: `{"dimension": "track", "metricType": "streamChangeAbsolute", "movementType": "rising", "periodType": "monthly", "orderByProperty": "absolutechange", "orderByDescending": true, "pageSize": 0}`
  - prod params: `{"dimension": "track", "metricType": "streamChangeAbsolute", "movementType": "rising", "periodType": "monthly", "orderByProperty": "absolutechange", "orderByDescending": true, "pageSize": 0}`
- **P-MV-track-size2001** (parent 127038, FAIL): prod 200, stage 200 (expected {'prod_status': 'any', 'stage_status': 400})
  - stage params: `{"dimension": "track", "metricType": "streamChangeAbsolute", "movementType": "rising", "periodType": "monthly", "orderByProperty": "absolutechange", "orderByDescending": true, "pageSize": 2001}`
  - prod params: `{"dimension": "track", "metricType": "streamChangeAbsolute", "movementType": "rising", "periodType": "monthly", "orderByProperty": "absolutechange", "orderByDescending": true, "pageSize": 2001}`
- **P-MV-track-page0** (parent 127038, FAIL): prod 200, stage 200 (expected {'prod_status': 'any', 'stage_status': 400})
  - stage params: `{"dimension": "track", "metricType": "streamChangeAbsolute", "movementType": "rising", "periodType": "monthly", "orderByProperty": "absolutechange", "orderByDescending": true, "pageSize": 5, "pageNumber": 0}`
  - prod params: `{"dimension": "track", "metricType": "streamChangeAbsolute", "movementType": "rising", "periodType": "monthly", "orderByProperty": "absolutechange", "orderByDescending": true, "pageSize": 5, "pageNumber": 0}`
- **F-EN-streaming-consumptionType-consumptionTypeIds** (parent 127038, FAIL): prod: rows outside consumptionTypeIds=[41, 44]: [45, 47, 48]; stage: rows outside consumptionTypeIds=[41, 44]: [45, 47, 48]
  - stage params: `{"dimension": "consumptionType", "mediaType": "streaming", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "sub30scount", "orderByDescending": true, "pageSize": 5, "consumptionTypeIds": [41, 44]}`
  - prod params: `{"isUgc": false, "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "sub30scount", "orderByDescending": true, "pageSize": 5, "consumptionTypeIds": [41, 44]}`
- **F-EN-ugc-consumptionType-consumptionTypeIds-nonexistent** (parent 127038, IGNORED-BOTH): prod: consumptionTypeIds=[999999999] (non-existent) still returned data: filter not applied; stage: consumptionTypeIds=[999999999] (non-existent) still returned data: filter not applied
  - stage params: `{"dimension": "consumptionType", "mediaType": "ugc", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "creationscount", "orderByDescending": true, "pageSize": 5, "consumptionTypeIds": [999999999]}`
  - prod params: `{"isUgc": true, "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "creationscount", "orderByDescending": true, "pageSize": 5, "consumptionTypeIds": [999999999]}`
- **F-MV-country-excludeDistributorIds** (parent 127038, IGNORED-BOTH): prod: excludeDistributorIds=[85] returned exactly the unfiltered result (filter ignored?); stage: excludeDistributorIds=[85] returned exactly the unfiltered result (filter ignored?)
  - stage params: `{"dimension": "country", "metricType": "streamChangeAbsolute", "movementType": "rising", "periodType": "monthly", "orderByProperty": "absolutechange", "orderByDescending": true, "pageSize": 5, "excludeDistributorIds": [85]}`
  - prod params: `{"dimension": "country", "metricType": "streamChangeAbsolute", "movementType": "rising", "periodType": "monthly", "orderByProperty": "absolutechange", "orderByDescending": true, "pageSize": 5, "excludeDistributorIds": [85]}`
- **F-PL-streams-playlist-excludeDistributorIds** (parent 127038, IGNORED-BOTH): prod: excludeDistributorIds=[10] returned exactly the unfiltered result (filter ignored?); stage: excludeDistributorIds=[10] returned exactly the unfiltered result (filter ignored?)
  - stage params: `{"dimension": "playlist", "metric": "streams", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "excludeDistributorIds": [10]}`
  - prod params: `{"metric": "streams", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "metadata": "playlist", "excludeDistributorIds": [10]}`
- **F-PL-streams-distributor-excludeDistributorIds** (parent 127038, FAIL): prod: excluded ids still present: [10]; stage: excluded ids still present: [10]
  - stage params: `{"dimension": "distributor", "metric": "streams", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "excludeDistributorIds": [10]}`
  - prod params: `{"metric": "streams", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "metadata": "distributor", "excludeDistributorIds": [10]}`
- **F-PL-streams-country-excludeDistributorIds** (parent 127038, IGNORED-BOTH): prod: excludeDistributorIds=[10] returned exactly the unfiltered result (filter ignored?); stage: excludeDistributorIds=[10] returned exactly the unfiltered result (filter ignored?)
  - stage params: `{"dimension": "country", "metric": "streams", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "excludeDistributorIds": [10]}`
  - prod params: `{"metric": "streams", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "metadata": "country", "excludeDistributorIds": [10]}`
- **F-PL-followers-playlist-trackIds** (parent 127038, VACUOUS): prod: filtered result empty; stage: filtered result empty
  - stage params: `{"dimension": "playlist", "metric": "followers", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "trackIds": [7884343, 8182932]}`
  - prod params: `{"metric": "followers", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "metadata": "playlist", "trackIds": [7884343, 8182932]}`
- **F-PL-followers-playlist-releaseIds** (parent 127038, VACUOUS): prod: filtered result empty; stage: filtered result empty
  - stage params: `{"dimension": "playlist", "metric": "followers", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "releaseIds": [3338970, 3414352]}`
  - prod params: `{"metric": "followers", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "metadata": "playlist", "releaseIds": [3338970, 3414352]}`
- **F-PL-followers-distributor-excludeDistributorIds** (parent 127038, FAIL): prod: excluded ids still present: [10]; stage: excluded ids still present: [10]
  - stage params: `{"dimension": "distributor", "metric": "followers", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "excludeDistributorIds": [10]}`
  - prod params: `{"metric": "followers", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "metadata": "distributor", "excludeDistributorIds": [10]}`
- **F-PL-followers-distributor-trackIds** (parent 127038, VACUOUS): prod: filtered result empty; stage: filtered result empty
  - stage params: `{"dimension": "distributor", "metric": "followers", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "trackIds": [7884343, 8182932]}`
  - prod params: `{"metric": "followers", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "metadata": "distributor", "trackIds": [7884343, 8182932]}`
- **F-PL-followers-distributor-releaseIds** (parent 127038, VACUOUS): prod: filtered result empty; stage: filtered result empty
  - stage params: `{"dimension": "distributor", "metric": "followers", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "releaseIds": [3338970, 3414352]}`
  - prod params: `{"metric": "followers", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "metadata": "distributor", "releaseIds": [3338970, 3414352]}`
- **F-PL-followers-country-trackIds** (parent 127038, VACUOUS): prod: filtered result empty; stage: filtered result empty
  - stage params: `{"dimension": "country", "metric": "followers", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "trackIds": [7884343, 8182932]}`
  - prod params: `{"metric": "followers", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "metadata": "country", "trackIds": [7884343, 8182932]}`
- **F-PL-followers-country-releaseIds** (parent 127038, VACUOUS): prod: filtered result empty; stage: filtered result empty
  - stage params: `{"dimension": "country", "metric": "followers", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "releaseIds": [3338970, 3414352]}`
  - prod params: `{"metric": "followers", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "metadata": "country", "releaseIds": [3338970, 3414352]}`
- **C-S-EN-streaming-deliveryType-distributorid-asc** (child 222139, FAIL): stage: totalItemsCount 0 != base 3 (C-EN-streaming-deliveryType-base): sort changed the result set; stage: totals.streamsCount 0 != base 142472 (C-EN-streaming-deliveryType-base); stage: totals.sub30sCount 0 != base 12139 (C-EN-streaming-deliveryType-base)
  - stage params: `{"dimension": "deliveryType", "mediaType": "streaming", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "distributorid", "orderByDescending": false, "pageSize": 5}`
- **C-S-EN-streaming-deliveryType-distributorid-desc** (child 222139, FAIL): stage: totalItemsCount 0 != base 3 (C-EN-streaming-deliveryType-base): sort changed the result set; stage: totals.streamsCount 0 != base 142472 (C-EN-streaming-deliveryType-base); stage: totals.sub30sCount 0 != base 12139 (C-EN-streaming-deliveryType-base)
  - stage params: `{"dimension": "deliveryType", "mediaType": "streaming", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "distributorid", "orderByDescending": true, "pageSize": 5}`
- **C-S-EN-streaming-deliveryType-distributorname-asc** (child 222139, FAIL): stage: totalItemsCount 0 != base 3 (C-EN-streaming-deliveryType-base): sort changed the result set; stage: totals.streamsCount 0 != base 142472 (C-EN-streaming-deliveryType-base); stage: totals.sub30sCount 0 != base 12139 (C-EN-streaming-deliveryType-base)
  - stage params: `{"dimension": "deliveryType", "mediaType": "streaming", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "distributorname", "orderByDescending": false, "pageSize": 5}`
- **C-S-EN-streaming-deliveryType-distributorname-desc** (child 222139, FAIL): stage: totalItemsCount 0 != base 3 (C-EN-streaming-deliveryType-base): sort changed the result set; stage: totals.streamsCount 0 != base 142472 (C-EN-streaming-deliveryType-base); stage: totals.sub30sCount 0 != base 12139 (C-EN-streaming-deliveryType-base)
  - stage params: `{"dimension": "deliveryType", "mediaType": "streaming", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "distributorname", "orderByDescending": true, "pageSize": 5}`
- **C-G-CS-streaming-sourceOfStream-DAILY** (child 222139, FAIL): stage: page has 13 rows for pageSize=5 (see F-13)
  - stage params: `{"dimension": "sourceOfStream", "mediaType": "streaming", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "streamscount", "orderByDescending": true, "pageSize": 5, "dateGranularity": "DAILY"}`
- **C-G-CS-streaming-sourceOfStream-WEEKLY** (child 222139, FAIL): stage: page has 13 rows for pageSize=5 (see F-13)
  - stage params: `{"dimension": "sourceOfStream", "mediaType": "streaming", "fromDate": "2026-04-01", "toDate": "2026-06-30", "orderByProperty": "streamscount", "orderByDescending": true, "pageSize": 5, "dateGranularity": "WEEKLY"}`
- **C-G-CS-streaming-sourceOfStream-MONTHLY** (child 222139, FAIL): stage: page has 13 rows for pageSize=5 (see F-13)
  - stage params: `{"dimension": "sourceOfStream", "mediaType": "streaming", "fromDate": "2026-04-01", "toDate": "2026-06-30", "orderByProperty": "streamscount", "orderByDescending": true, "pageSize": 5, "dateGranularity": "MONTHLY"}`
- **C-G-CS-streaming-sourceOfStream-QUARTERLY** (child 222139, FAIL): stage: page has 14 rows for pageSize=5 (see F-13)
  - stage params: `{"dimension": "sourceOfStream", "mediaType": "streaming", "fromDate": "2026-01-01", "toDate": "2026-08-31", "orderByProperty": "streamscount", "orderByDescending": true, "pageSize": 5, "dateGranularity": "QUARTERLY"}`
- **C-G-CS-streaming-sourceOfStream-YEARLY** (child 222139, FAIL): stage: page has 14 rows for pageSize=5 (see F-13)
  - stage params: `{"dimension": "sourceOfStream", "mediaType": "streaming", "fromDate": "2026-01-01", "toDate": "2026-08-31", "orderByProperty": "streamscount", "orderByDescending": true, "pageSize": 5, "dateGranularity": "YEARLY"}`
- **C-G-EN-streaming-sourceOfStream-DAILY** (child 222139, FAIL): stage: page has 11 rows for pageSize=5 (see F-13)
  - stage params: `{"dimension": "sourceOfStream", "mediaType": "streaming", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "sub30scount", "orderByDescending": true, "pageSize": 5, "dateGranularity": "DAILY"}`
- **C-G-EN-streaming-sourceOfStream-WEEKLY** (child 222139, FAIL): stage: page has 11 rows for pageSize=5 (see F-13)
  - stage params: `{"dimension": "sourceOfStream", "mediaType": "streaming", "fromDate": "2026-04-01", "toDate": "2026-06-30", "orderByProperty": "sub30scount", "orderByDescending": true, "pageSize": 5, "dateGranularity": "WEEKLY"}`
- **C-G-EN-streaming-sourceOfStream-MONTHLY** (child 222139, FAIL): stage: page has 11 rows for pageSize=5 (see F-13)
  - stage params: `{"dimension": "sourceOfStream", "mediaType": "streaming", "fromDate": "2026-04-01", "toDate": "2026-06-30", "orderByProperty": "sub30scount", "orderByDescending": true, "pageSize": 5, "dateGranularity": "MONTHLY"}`
- **C-G-EN-streaming-sourceOfStream-QUARTERLY** (child 222139, FAIL): stage: page has 11 rows for pageSize=5 (see F-13)
  - stage params: `{"dimension": "sourceOfStream", "mediaType": "streaming", "fromDate": "2026-01-01", "toDate": "2026-08-31", "orderByProperty": "sub30scount", "orderByDescending": true, "pageSize": 5, "dateGranularity": "QUARTERLY"}`
- **C-G-EN-streaming-sourceOfStream-YEARLY** (child 222139, FAIL): stage: page has 11 rows for pageSize=5 (see F-13)
  - stage params: `{"dimension": "sourceOfStream", "mediaType": "streaming", "fromDate": "2026-01-01", "toDate": "2026-08-31", "orderByProperty": "sub30scount", "orderByDescending": true, "pageSize": 5, "dateGranularity": "YEARLY"}`
- **C-G-EN-ugc-country-QUARTERLY** (child 222139, FAIL): stage page not ordered by creationscount desc
  - stage params: `{"dimension": "country", "mediaType": "ugc", "fromDate": "2026-01-01", "toDate": "2026-08-31", "orderByProperty": "creationscount", "orderByDescending": true, "pageSize": 5, "dateGranularity": "QUARTERLY"}`
- **C-P-PM-active-size0** (child 222139, FAIL): stage status 200, expected 400: {"totals":{"playlistStreamsCount":0},"totalItemsCount":0,"pageNumber":1,"pageSize":0,"totalPagesCount":0,"items":[]}
  - stage params: `{"movementType": "active", "period": 28, "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 0}`
- **C-P-PM-active-page0** (child 222139, FAIL): stage status 500, expected 400: HTTP error 500: Internal Server Error - {'type': 'https://tools.ietf.org/html/rfc9110#section-15.6.1', 'title': 'An error occurr
  - stage params: `{"movementType": "active", "period": 28, "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "pageNumber": 0}`
- **C-F-EN-streaming-consumptionType-consumptionTypeIds** (child 222139, FAIL): stage: rows outside consumptionTypeIds=[41, 44]: [45, 53, 47]
  - stage params: `{"dimension": "consumptionType", "mediaType": "streaming", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "sub30scount", "orderByDescending": true, "pageSize": 5, "consumptionTypeIds": [41, 44]}`
- **C-F-EN-ugc-consumptionType-trackIds** (child 222139, VACUOUS): stage: filtered result empty
  - stage params: `{"dimension": "consumptionType", "mediaType": "ugc", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "creationscount", "orderByDescending": true, "pageSize": 5, "trackIds": [5224185, 3646212]}`
- **C-F-EN-ugc-consumptionType-releaseIds** (child 222139, VACUOUS): stage: filtered result empty
  - stage params: `{"dimension": "consumptionType", "mediaType": "ugc", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "creationscount", "orderByDescending": true, "pageSize": 5, "releaseIds": [2284129, 1319063]}`
- **C-F-EN-ugc-consumptionType-artistIds** (child 222139, VACUOUS): stage: filtered result empty
  - stage params: `{"dimension": "consumptionType", "mediaType": "ugc", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "creationscount", "orderByDescending": true, "pageSize": 5, "artistIds": [1213426, 785696]}`
- **C-F-EN-ugc-consumptionType-distributorIds** (child 222139, IGNORED): stage: distributorIds=[407, 19] returned exactly the unfiltered result (filter ignored?)
  - stage params: `{"dimension": "consumptionType", "mediaType": "ugc", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "creationscount", "orderByDescending": true, "pageSize": 5, "distributorIds": [407, 19]}`
- **C-F-EN-ugc-consumptionType-excludeDistributorIds** (child 222139, IGNORED): stage: excludeDistributorIds=[407] returned exactly the unfiltered result (filter ignored?)
  - stage params: `{"dimension": "consumptionType", "mediaType": "ugc", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "creationscount", "orderByDescending": true, "pageSize": 5, "excludeDistributorIds": [407]}`
- **C-F-EN-ugc-consumptionType-excludeTikTok** (child 222139, IGNORED): stage: excludeTikTok=True returned exactly the unfiltered result (filter ignored?)
  - stage params: `{"dimension": "consumptionType", "mediaType": "ugc", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "creationscount", "orderByDescending": true, "pageSize": 5, "excludeTikTok": true}`
- **C-F-EN-ugc-deliveryType-trackIds** (child 222139, VACUOUS): stage: filtered result empty
  - stage params: `{"dimension": "deliveryType", "mediaType": "ugc", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "creationscount", "orderByDescending": true, "pageSize": 5, "trackIds": [5224185, 3646212]}`
- **C-F-EN-ugc-deliveryType-releaseIds** (child 222139, VACUOUS): stage: filtered result empty
  - stage params: `{"dimension": "deliveryType", "mediaType": "ugc", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "creationscount", "orderByDescending": true, "pageSize": 5, "releaseIds": [2284129, 1319063]}`
- **C-F-EN-ugc-deliveryType-artistIds** (child 222139, VACUOUS): stage: filtered result empty
  - stage params: `{"dimension": "deliveryType", "mediaType": "ugc", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "creationscount", "orderByDescending": true, "pageSize": 5, "artistIds": [1213426, 785696]}`
- **C-F-EN-ugc-deliveryType-distributorIds** (child 222139, IGNORED): stage: distributorIds=[407, 19] returned exactly the unfiltered result (filter ignored?)
  - stage params: `{"dimension": "deliveryType", "mediaType": "ugc", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "creationscount", "orderByDescending": true, "pageSize": 5, "distributorIds": [407, 19]}`
- **C-F-EN-ugc-deliveryType-excludeDistributorIds** (child 222139, IGNORED): stage: excludeDistributorIds=[407] returned exactly the unfiltered result (filter ignored?)
  - stage params: `{"dimension": "deliveryType", "mediaType": "ugc", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "creationscount", "orderByDescending": true, "pageSize": 5, "excludeDistributorIds": [407]}`
- **C-F-EN-ugc-deliveryType-excludeTikTok** (child 222139, IGNORED): stage: excludeTikTok=True returned exactly the unfiltered result (filter ignored?)
  - stage params: `{"dimension": "deliveryType", "mediaType": "ugc", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "creationscount", "orderByDescending": true, "pageSize": 5, "excludeTikTok": true}`
- **C-F-EN-ugc-consumptionType-consumptionTypeIds-nonexistent** (child 222139, IGNORED): stage: consumptionTypeIds=[999999999] (non-existent) still returned data: filter not applied
  - stage params: `{"dimension": "consumptionType", "mediaType": "ugc", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "creationscount", "orderByDescending": true, "pageSize": 5, "consumptionTypeIds": [999999999]}`
- **C-F-RV-deliveryType-countryIds** (child 222139, IGNORED): stage: countryIds=[244, 254] returned exactly the unfiltered result (filter ignored?)
  - stage params: `{"dimension": "deliveryType", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "netrevenue", "orderByDescending": true, "pageSize": 5, "countryIds": [244, 254]}`
- **C-F-RV-deliveryType-distributorIds** (child 222139, IGNORED): stage: distributorIds=[359, 1] returned exactly the unfiltered result (filter ignored?)
  - stage params: `{"dimension": "deliveryType", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "netrevenue", "orderByDescending": true, "pageSize": 5, "distributorIds": [359, 1]}`
- **C-F-RV-deliveryType-excludeDistributorIds** (child 222139, IGNORED): stage: excludeDistributorIds=[359] returned exactly the unfiltered result (filter ignored?)
  - stage params: `{"dimension": "deliveryType", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "netrevenue", "orderByDescending": true, "pageSize": 5, "excludeDistributorIds": [359]}`
- **C-F-RV-track-assetTypes-track** (child 222139, IGNORED): stage: assetTypes=['track'] returned exactly the unfiltered result (filter ignored?)
  - stage params: `{"dimension": "track", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "netrevenue", "orderByDescending": true, "pageSize": 5, "assetTypes": ["track"]}`
- **C-F-PL-streams-playlist-excludeDistributorIds** (child 222139, IGNORED): stage: excludeDistributorIds=[10] returned exactly the unfiltered result (filter ignored?)
  - stage params: `{"dimension": "playlist", "metric": "streams", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "excludeDistributorIds": [10]}`
- **C-F-PL-streams-playlist-labelIds** (child 222139, IGNORED): stage: labelIds=[291187] returned exactly the unfiltered result (filter ignored?)
  - stage params: `{"dimension": "playlist", "metric": "streams", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "labelIds": [291187]}`
- **C-F-PL-streams-playlistType-excludeDistributorIds** (child 222139, IGNORED): stage: excludeDistributorIds=[10] returned exactly the unfiltered result (filter ignored?)
  - stage params: `{"dimension": "playlistType", "metric": "streams", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "excludeDistributorIds": [10]}`
- **C-F-PL-streams-playlistType-labelIds** (child 222139, IGNORED): stage: labelIds=[291187] returned exactly the unfiltered result (filter ignored?)
  - stage params: `{"dimension": "playlistType", "metric": "streams", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "labelIds": [291187]}`
- **C-F-PL-streams-distributor-excludeDistributorIds** (child 222139, FAIL): stage: excluded ids still present: [10]
  - stage params: `{"dimension": "distributor", "metric": "streams", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "excludeDistributorIds": [10]}`
- **C-F-PL-streams-distributor-labelIds** (child 222139, IGNORED): stage: labelIds=[291187] returned exactly the unfiltered result (filter ignored?)
  - stage params: `{"dimension": "distributor", "metric": "streams", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "labelIds": [291187]}`
- **C-F-PL-streams-country-excludeDistributorIds** (child 222139, IGNORED): stage: excludeDistributorIds=[10] returned exactly the unfiltered result (filter ignored?)
  - stage params: `{"dimension": "country", "metric": "streams", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "excludeDistributorIds": [10]}`
- **C-F-PL-streams-country-labelIds** (child 222139, IGNORED): stage: labelIds=[291187] returned exactly the unfiltered result (filter ignored?)
  - stage params: `{"dimension": "country", "metric": "streams", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "labelIds": [291187]}`
- **C-F-PL-followers-playlist-playlistIds** (child 222139, VACUOUS): stage: filtered result empty
  - stage params: `{"dimension": "playlist", "metric": "followers", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "playlistIds": [102482581, 223865089]}`
- **C-F-PL-followers-playlist-playlistTypeIds** (child 222139, VACUOUS): stage: filtered result empty
  - stage params: `{"dimension": "playlist", "metric": "followers", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "playlistTypeIds": [13]}`
- **C-F-PL-followers-playlist-countryIds** (child 222139, VACUOUS): stage: filtered result empty
  - stage params: `{"dimension": "playlist", "metric": "followers", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "countryIds": [244, 253]}`
- **C-F-PL-followers-playlist-distributorIds** (child 222139, VACUOUS): stage: filtered result empty
  - stage params: `{"dimension": "playlist", "metric": "followers", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "distributorIds": [10]}`
- **C-F-PL-followers-playlist-excludeDistributorIds** (child 222139, VACUOUS): stage: filtered result empty
  - stage params: `{"dimension": "playlist", "metric": "followers", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "excludeDistributorIds": [10]}`
- **C-F-PL-followers-playlist-trackIds** (child 222139, VACUOUS): stage: filtered result empty
  - stage params: `{"dimension": "playlist", "metric": "followers", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "trackIds": [8387737, 3383285]}`
- **C-F-PL-followers-playlist-releaseIds** (child 222139, VACUOUS): stage: filtered result empty
  - stage params: `{"dimension": "playlist", "metric": "followers", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "releaseIds": [3498935, 1164456]}`
- **C-F-PL-followers-playlist-artistIds** (child 222139, VACUOUS): stage: filtered result empty
  - stage params: `{"dimension": "playlist", "metric": "followers", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "artistIds": [737450, 722683]}`
- **C-F-PL-followers-playlist-labelIds** (child 222139, VACUOUS): stage: filtered result empty
  - stage params: `{"dimension": "playlist", "metric": "followers", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "labelIds": [291187]}`
- **C-F-PL-followers-playlist-excludeTikTok** (child 222139, VACUOUS): stage: filtered result empty
  - stage params: `{"dimension": "playlist", "metric": "followers", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "excludeTikTok": true}`
- **C-F-PL-followers-playlistType-playlistIds** (child 222139, VACUOUS): stage: filtered result empty
  - stage params: `{"dimension": "playlistType", "metric": "followers", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "playlistIds": [102482581, 223865089]}`
- **C-F-PL-followers-playlistType-playlistTypeIds** (child 222139, VACUOUS): stage: filtered result empty
  - stage params: `{"dimension": "playlistType", "metric": "followers", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "playlistTypeIds": [13]}`
- **C-F-PL-followers-playlistType-countryIds** (child 222139, VACUOUS): stage: filtered result empty
  - stage params: `{"dimension": "playlistType", "metric": "followers", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "countryIds": [244, 253]}`
- **C-F-PL-followers-playlistType-distributorIds** (child 222139, VACUOUS): stage: filtered result empty
  - stage params: `{"dimension": "playlistType", "metric": "followers", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "distributorIds": [10]}`
- **C-F-PL-followers-playlistType-excludeDistributorIds** (child 222139, VACUOUS): stage: filtered result empty
  - stage params: `{"dimension": "playlistType", "metric": "followers", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "excludeDistributorIds": [10]}`
- **C-F-PL-followers-playlistType-trackIds** (child 222139, VACUOUS): stage: filtered result empty
  - stage params: `{"dimension": "playlistType", "metric": "followers", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "trackIds": [8387737, 3383285]}`
- **C-F-PL-followers-playlistType-releaseIds** (child 222139, VACUOUS): stage: filtered result empty
  - stage params: `{"dimension": "playlistType", "metric": "followers", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "releaseIds": [3498935, 1164456]}`
- **C-F-PL-followers-playlistType-artistIds** (child 222139, VACUOUS): stage: filtered result empty
  - stage params: `{"dimension": "playlistType", "metric": "followers", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "artistIds": [737450, 722683]}`
- **C-F-PL-followers-playlistType-labelIds** (child 222139, VACUOUS): stage: filtered result empty
  - stage params: `{"dimension": "playlistType", "metric": "followers", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "labelIds": [291187]}`
- **C-F-PL-followers-playlistType-excludeTikTok** (child 222139, VACUOUS): stage: filtered result empty
  - stage params: `{"dimension": "playlistType", "metric": "followers", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "excludeTikTok": true}`
- **C-F-PL-followers-distributor-playlistIds** (child 222139, VACUOUS): stage: filtered result empty
  - stage params: `{"dimension": "distributor", "metric": "followers", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "playlistIds": [102482581, 223865089]}`
- **C-F-PL-followers-distributor-playlistTypeIds** (child 222139, VACUOUS): stage: filtered result empty
  - stage params: `{"dimension": "distributor", "metric": "followers", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "playlistTypeIds": [13]}`
- **C-F-PL-followers-distributor-countryIds** (child 222139, VACUOUS): stage: filtered result empty
  - stage params: `{"dimension": "distributor", "metric": "followers", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "countryIds": [244, 253]}`
- **C-F-PL-followers-distributor-distributorIds** (child 222139, VACUOUS): stage: filtered result empty
  - stage params: `{"dimension": "distributor", "metric": "followers", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "distributorIds": [10]}`
- **C-F-PL-followers-distributor-excludeDistributorIds** (child 222139, VACUOUS): stage: filtered result empty
  - stage params: `{"dimension": "distributor", "metric": "followers", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "excludeDistributorIds": [10]}`
- **C-F-PL-followers-distributor-trackIds** (child 222139, VACUOUS): stage: filtered result empty
  - stage params: `{"dimension": "distributor", "metric": "followers", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "trackIds": [8387737, 3383285]}`
- **C-F-PL-followers-distributor-releaseIds** (child 222139, VACUOUS): stage: filtered result empty
  - stage params: `{"dimension": "distributor", "metric": "followers", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "releaseIds": [3498935, 1164456]}`
- **C-F-PL-followers-distributor-artistIds** (child 222139, VACUOUS): stage: filtered result empty
  - stage params: `{"dimension": "distributor", "metric": "followers", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "artistIds": [737450, 722683]}`
- **C-F-PL-followers-distributor-labelIds** (child 222139, VACUOUS): stage: filtered result empty
  - stage params: `{"dimension": "distributor", "metric": "followers", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "labelIds": [291187]}`
- **C-F-PL-followers-distributor-excludeTikTok** (child 222139, VACUOUS): stage: filtered result empty
  - stage params: `{"dimension": "distributor", "metric": "followers", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "excludeTikTok": true}`
- **C-F-PL-followers-country-playlistIds** (child 222139, VACUOUS): stage: filtered result empty
  - stage params: `{"dimension": "country", "metric": "followers", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "playlistIds": [102482581, 223865089]}`
- **C-F-PL-followers-country-playlistTypeIds** (child 222139, VACUOUS): stage: filtered result empty
  - stage params: `{"dimension": "country", "metric": "followers", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "playlistTypeIds": [13]}`
- **C-F-PL-followers-country-countryIds** (child 222139, VACUOUS): stage: filtered result empty
  - stage params: `{"dimension": "country", "metric": "followers", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "countryIds": [244, 253]}`
- **C-F-PL-followers-country-distributorIds** (child 222139, VACUOUS): stage: filtered result empty
  - stage params: `{"dimension": "country", "metric": "followers", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "distributorIds": [10]}`
- **C-F-PL-followers-country-excludeDistributorIds** (child 222139, VACUOUS): stage: filtered result empty
  - stage params: `{"dimension": "country", "metric": "followers", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "excludeDistributorIds": [10]}`
- **C-F-PL-followers-country-trackIds** (child 222139, VACUOUS): stage: filtered result empty
  - stage params: `{"dimension": "country", "metric": "followers", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "trackIds": [8387737, 3383285]}`
- **C-F-PL-followers-country-releaseIds** (child 222139, VACUOUS): stage: filtered result empty
  - stage params: `{"dimension": "country", "metric": "followers", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "releaseIds": [3498935, 1164456]}`
- **C-F-PL-followers-country-artistIds** (child 222139, VACUOUS): stage: filtered result empty
  - stage params: `{"dimension": "country", "metric": "followers", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "artistIds": [737450, 722683]}`
- **C-F-PL-followers-country-labelIds** (child 222139, VACUOUS): stage: filtered result empty
  - stage params: `{"dimension": "country", "metric": "followers", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "labelIds": [291187]}`
- **C-F-PL-followers-country-excludeTikTok** (child 222139, VACUOUS): stage: filtered result empty
  - stage params: `{"dimension": "country", "metric": "followers", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "excludeTikTok": true}`
- **C-F-PL-followers-date-playlistIds** (child 222139, VACUOUS): stage: filtered result empty
  - stage params: `{"dimension": "date", "metric": "followers", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "playlistIds": [102482581, 223865089]}`
- **C-F-PL-followers-date-playlistTypeIds** (child 222139, VACUOUS): stage: filtered result empty
  - stage params: `{"dimension": "date", "metric": "followers", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "playlistTypeIds": [13]}`
- **C-F-PL-followers-date-countryIds** (child 222139, VACUOUS): stage: filtered result empty
  - stage params: `{"dimension": "date", "metric": "followers", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "countryIds": [244, 253]}`
- **C-F-PL-followers-date-distributorIds** (child 222139, VACUOUS): stage: filtered result empty
  - stage params: `{"dimension": "date", "metric": "followers", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "distributorIds": [10]}`
- **C-F-PL-followers-date-excludeDistributorIds** (child 222139, VACUOUS): stage: filtered result empty
  - stage params: `{"dimension": "date", "metric": "followers", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "excludeDistributorIds": [10]}`
- **C-F-PL-followers-date-trackIds** (child 222139, VACUOUS): stage: filtered result empty
  - stage params: `{"dimension": "date", "metric": "followers", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "trackIds": [8387737, 3383285]}`
- **C-F-PL-followers-date-releaseIds** (child 222139, VACUOUS): stage: filtered result empty
  - stage params: `{"dimension": "date", "metric": "followers", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "releaseIds": [3498935, 1164456]}`
- **C-F-PL-followers-date-artistIds** (child 222139, VACUOUS): stage: filtered result empty
  - stage params: `{"dimension": "date", "metric": "followers", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "artistIds": [737450, 722683]}`
- **C-F-PL-followers-date-labelIds** (child 222139, VACUOUS): stage: filtered result empty
  - stage params: `{"dimension": "date", "metric": "followers", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "labelIds": [291187]}`
- **C-F-PL-followers-date-excludeTikTok** (child 222139, VACUOUS): stage: filtered result empty
  - stage params: `{"dimension": "date", "metric": "followers", "fromDate": "2026-06-01", "toDate": "2026-06-30", "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "excludeTikTok": true}`
- **C-F-PM-active-labelIds** (child 222139, IGNORED): stage: labelIds=[291187] returned exactly the unfiltered result (filter ignored?)
  - stage params: `{"movementType": "active", "period": 28, "orderByProperty": "playliststreamscount", "orderByDescending": true, "pageSize": 5, "labelIds": [291187]}`

## Tie review list (user decision: ties do not count as a match; listed for review)

| Test | Class | Detail |
|---|---|---|
| CS-streaming-track-base | TIE-UNORDERED | items: every row ties on eventdate=None (effectively unordered): different rows on the page |
| CS-streaming-release-base | TIE | items: same rows, different order |
| CS-streaming-artist-base | TIE-UNORDERED | items: every row ties on eventdate=None (effectively unordered): different rows on the page |
| CS-streaming-label-base | TIE-UNORDERED | items: every row ties on eventdate=None (effectively unordered): different rows on the page |
| CS-streaming-country-base | TIE-UNORDERED | items: every row ties on eventdate=None (effectively unordered): different rows on the page |
| CS-ugc-track-base | TIE-UNORDERED | items: every row ties on eventdate=None (effectively unordered): different rows on the page |
| CS-ugc-release-base | TIE-UNORDERED | items: every row ties on eventdate=None (effectively unordered): different rows on the page |
| CS-ugc-artist-base | TIE-UNORDERED | items: every row ties on eventdate=None (effectively unordered): different rows on the page |
| CS-ugc-label-base | TIE-UNORDERED | items: every row ties on eventdate=None (effectively unordered): different rows on the page |
| CS-ugc-country-base | TIE-UNORDERED | items: every row ties on eventdate=None (effectively unordered): different rows on the page |
| EN-streaming-track-base | TIE-UNORDERED | items: every row ties on eventdate=None (effectively unordered): different rows on the page |
| EN-streaming-release-base | TIE-UNORDERED | items: every row ties on eventdate=None (effectively unordered): different rows on the page |
| EN-streaming-label-base | TIE-UNORDERED | items: every row ties on eventdate=None (effectively unordered): different rows on the page |
| EN-ugc-track-base | TIE-UNORDERED | items: every row ties on eventdate=None (effectively unordered): different rows on the page |
| EN-ugc-release-base | TIE-UNORDERED | items: every row ties on eventdate=None (effectively unordered): different rows on the page |
| EN-ugc-artist-base | TIE-UNORDERED | items: every row ties on eventdate=None (effectively unordered): different rows on the page |
| EN-ugc-label-base | TIE-UNORDERED | items: every row ties on eventdate=None (effectively unordered): different rows on the page |
| AS-track-base | TIE-UNORDERED | items: every row ties on eventdate=None (effectively unordered): different rows on the page |
| PL-playlist-streams-base | TIE-UNORDERED | items: every row ties on eventdate=None (effectively unordered): different rows on the page |
| PL-playlist-playlistsCount-base | TIE-UNORDERED | items: every row ties on eventdate=None (effectively unordered): different rows on the page |
| PL-playlistType-streams-base | TIE | items: same rows, different order |
| PL-playlistType-playlistsCount-base | TIE | items: same rows, different order |
| PL-distributor-streams-base | TIE | items: same rows, different order |
| PL-distributor-playlistsCount-base | TIE | items: same rows, different order |
| PL-country-streams-base | TIE-UNORDERED | items: every row ties on eventdate=None (effectively unordered): different rows on the page |
| PL-country-playlistsCount-base | TIE-UNORDERED | items: every row ties on eventdate=None (effectively unordered): different rows on the page |
| MV-track-base | TIE-UNORDERED | items: every row ties on absolutechange=1 (effectively unordered): different rows on the page |
| MV-release-base | TIE-BOUNDARY | items: page-boundary swap at absolutechange=3: prod-only=1 stage-only=1 |
| MV-artist-base | TIE | items: same rows, different order |
| S-CS-streaming-track-eventdate-asc | TIE-UNORDERED | items: every row ties on eventdate=None (effectively unordered): different rows on the page |
| S-CS-streaming-track-eventdate-desc | TIE-UNORDERED | items: every row ties on eventdate=None (effectively unordered): different rows on the page |
| S-CS-streaming-track-trackversion-asc | TIE-UNORDERED | items: every row ties on trackversion=None (effectively unordered): different rows on the page |
| S-CS-streaming-release-releasedownloadscount-desc | TIE | items: same rows, different order |
| S-CS-streaming-release-eventdate-asc | TIE-UNORDERED | items: every row ties on eventdate=None (effectively unordered): different rows on the page |
| S-CS-streaming-release-eventdate-desc | TIE-UNORDERED | items: every row ties on eventdate=None (effectively unordered): different rows on the page |
| S-CS-streaming-release-releaseversion-desc | TIE | items: same rows, different order |
| S-CS-streaming-artist-eventdate-asc | TIE-UNORDERED | items: every row ties on eventdate=None (effectively unordered): different rows on the page |
| S-CS-streaming-artist-eventdate-desc | TIE-UNORDERED | items: every row ties on eventdate=None (effectively unordered): different rows on the page |
| S-CS-streaming-artist-artistname-asc | TIE-BOUNDARY | items: page-boundary swap at artistname=$hadowflex: prod-only=1 stage-only=1 |
| S-CS-streaming-label-eventdate-asc | TIE-UNORDERED | items: every row ties on eventdate=None (effectively unordered): different rows on the page |
| S-CS-streaming-label-eventdate-desc | TIE-UNORDERED | items: every row ties on eventdate=None (effectively unordered): different rows on the page |
| S-CS-streaming-country-eventdate-asc | TIE-UNORDERED | items: every row ties on eventdate=None (effectively unordered): different rows on the page |
| S-CS-streaming-country-eventdate-desc | TIE-UNORDERED | items: every row ties on eventdate=None (effectively unordered): different rows on the page |
| S-CS-ugc-track-eventdate-asc | TIE-UNORDERED | items: every row ties on eventdate=None (effectively unordered): different rows on the page |
| S-CS-ugc-track-eventdate-desc | TIE-UNORDERED | items: every row ties on eventdate=None (effectively unordered): different rows on the page |
| S-CS-ugc-release-eventdate-asc | TIE-UNORDERED | items: every row ties on eventdate=None (effectively unordered): different rows on the page |
| S-CS-ugc-release-eventdate-desc | TIE | items: same rows, different order |
| S-CS-ugc-artist-eventdate-asc | TIE-UNORDERED | items: every row ties on eventdate=None (effectively unordered): different rows on the page |
| S-CS-ugc-artist-eventdate-desc | TIE-UNORDERED | items: every row ties on eventdate=None (effectively unordered): different rows on the page |
| S-CS-ugc-label-eventdate-asc | TIE-UNORDERED | items: every row ties on eventdate=None (effectively unordered): different rows on the page |
| S-CS-ugc-label-eventdate-desc | TIE-UNORDERED | items: every row ties on eventdate=None (effectively unordered): different rows on the page |
| S-CS-ugc-label-labelname-asc | TIE | items: same rows, different order |
| S-CS-ugc-country-eventdate-asc | TIE-UNORDERED | items: every row ties on eventdate=None (effectively unordered): different rows on the page |
| S-CS-ugc-country-eventdate-desc | TIE-UNORDERED | items: every row ties on eventdate=None (effectively unordered): different rows on the page |
| S-EN-streaming-track-skiprate-desc | TIE-UNORDERED | items: every row ties on skiprate=100 (effectively unordered): different rows on the page |
| S-EN-streaming-track-completerate-desc | TIE-UNORDERED | items: every row ties on completerate=100 (effectively unordered): different rows on the page |
| S-EN-streaming-track-eventdate-asc | TIE-UNORDERED | items: every row ties on eventdate=None (effectively unordered): different rows on the page |
| S-EN-streaming-track-eventdate-desc | TIE-UNORDERED | items: every row ties on eventdate=None (effectively unordered): different rows on the page |
| S-EN-streaming-release-skiprate-desc | TIE-UNORDERED | items: every row ties on skiprate=100 (effectively unordered): different rows on the page |
| S-EN-streaming-release-completerate-desc | TIE-UNORDERED | items: every row ties on completerate=100 (effectively unordered): different rows on the page |
| S-EN-streaming-release-eventdate-asc | TIE-UNORDERED | items: every row ties on eventdate=None (effectively unordered): different rows on the page |
| S-EN-streaming-release-eventdate-desc | TIE-UNORDERED | items: every row ties on eventdate=None (effectively unordered): different rows on the page |
| S-EN-streaming-artist-skiprate-desc | TIE-UNORDERED | items: every row ties on skiprate=100 (effectively unordered): different rows on the page |
| S-EN-streaming-artist-completerate-desc | TIE-UNORDERED | items: every row ties on completerate=100 (effectively unordered): different rows on the page |
| S-EN-streaming-artist-eventdate-asc | TIE-UNORDERED | items: every row ties on eventdate=None (effectively unordered): different rows on the page |
| S-EN-streaming-artist-eventdate-desc | TIE-UNORDERED | items: every row ties on eventdate=None (effectively unordered): different rows on the page |
| S-EN-streaming-label-skiprate-desc | TIE-UNORDERED | items: every row ties on skiprate=100 (effectively unordered): different rows on the page |
| S-EN-streaming-label-completerate-desc | TIE-UNORDERED | items: every row ties on completerate=100 (effectively unordered): different rows on the page |
| S-EN-streaming-label-eventdate-asc | TIE-UNORDERED | items: every row ties on eventdate=None (effectively unordered): different rows on the page |
| S-EN-streaming-label-eventdate-desc | TIE-UNORDERED | items: every row ties on eventdate=None (effectively unordered): different rows on the page |
| S-EN-streaming-label-labelname-asc | TIE | items: same rows, different order |
| S-EN-streaming-label-labelname-desc | TIE | items: same rows, different order |
| S-EN-ugc-track-eventdate-asc | TIE-UNORDERED | items: every row ties on eventdate=None (effectively unordered): different rows on the page |
| S-EN-ugc-track-eventdate-desc | TIE-UNORDERED | items: every row ties on eventdate=None (effectively unordered): different rows on the page |
| S-EN-ugc-track-trackname-desc | TIE | items: same rows, different order |
| S-EN-ugc-track-trackversion-asc | TIE | items: same rows, different order |
| S-EN-ugc-release-watchtime-desc | TIE-UNORDERED | items: every row ties on watchtime=0 (effectively unordered): different rows on the page |
| S-EN-ugc-release-eventdate-asc | TIE-UNORDERED | items: every row ties on eventdate=None (effectively unordered): different rows on the page |
| S-EN-ugc-release-eventdate-desc | TIE-UNORDERED | items: every row ties on eventdate=None (effectively unordered): different rows on the page |
| S-EN-ugc-artist-eventdate-asc | TIE-UNORDERED | items: every row ties on eventdate=None (effectively unordered): different rows on the page |
| S-EN-ugc-artist-eventdate-desc | TIE-UNORDERED | items: every row ties on eventdate=None (effectively unordered): different rows on the page |
| S-EN-ugc-label-eventdate-asc | TIE-UNORDERED | items: every row ties on eventdate=None (effectively unordered): different rows on the page |
| S-EN-ugc-label-eventdate-desc | TIE-UNORDERED | items: every row ties on eventdate=None (effectively unordered): different rows on the page |
| S-EN-ugc-label-labelname-asc | TIE | items: same rows, different order |
| S-RV-track-releaseid-asc | TIE-BOUNDARY | items: page-boundary swap at releaseid=546092: prod-only=2 stage-only=2 |
| S-RV-track-releaseid-desc | TIE-BOUNDARY | items: page-boundary swap at releaseid=3485494: prod-only=1 stage-only=1 |
| S-RV-track-releasename-asc | TIE-UNORDERED | items: every row ties on releasename=!nt3rn3t l0s3r (effectively unordered): different rows on the page |
| S-RV-track-releasename-desc | TIE | items: same rows, different order |
| S-RV-track-releaseversion-asc | TIE-BOUNDARY | items: page-boundary swap at releaseversion=11th Anniversary Edition: prod-only=3 stage-only=3 |
| S-RV-track-releaseversion-desc | TIE | items: same rows, different order |
| S-RV-track-artistid-asc | TIE | items: same rows, different order |
| S-RV-track-artistid-desc | TIE-BOUNDARY | items: page-boundary swap at artistid=2156902: prod-only=2 stage-only=2 |
| S-RV-track-artistname-desc | TIE-UNORDERED | items: every row ties on artistname=예수 그리스도 후기 성도 교회 (effectively unordered): different rows on the page |
| S-RV-track-downloadscount-desc | TIE-BOUNDARY | items: page-boundary swap at downloadscount=3: prod-only=1 stage-only=1 |
| S-RV-track-downloadsrevenue-desc | TIE | items: same rows, different order |
| S-RV-track-assettypeid-asc | TIE-UNORDERED | items: every row ties on assettypeid=None (effectively unordered): different rows on the page |
| S-RV-track-assettypeid-desc | TIE-UNORDERED | items: every row ties on assettypeid=None (effectively unordered): different rows on the page |
| S-RV-release-artistid-asc | TIE | items: same rows, different order |
| S-RV-release-artistid-desc | TIE-BOUNDARY | items: page-boundary swap at artistid=2156902: prod-only=1 stage-only=1 |
| S-RV-release-artistname-desc | TIE-UNORDERED | items: every row ties on artistname=예수 그리스도 후기 성도 교회 (effectively unordered): different rows on the page |
| S-RV-artist-artistname-asc | TIE-BOUNDARY | items: page-boundary swap at artistname=$hadowflex: prod-only=1 stage-only=1 |
| S-RV-label-downloadscount-desc | TIE | items: same rows, different order |
| S-AS-track-flaggedassetscount-desc | TIE-UNORDERED | items: every row ties on flaggedassetscount=1 (effectively unordered): different rows on the page |
| S-AS-track-finesamount-desc | TIE-UNORDERED | items: every row ties on finesamount=0 (effectively unordered): different rows on the page |
| S-AS-track-eventdate-asc | TIE-UNORDERED | items: every row ties on eventdate=None (effectively unordered): different rows on the page |
| S-AS-track-eventdate-desc | TIE-UNORDERED | items: every row ties on eventdate=None (effectively unordered): different rows on the page |
| S-AS-track-trackversion-asc | TIE-UNORDERED | items: every row ties on trackversion=None (effectively unordered): different rows on the page |
| S-AS-track-trackversion-desc | TIE-UNORDERED | items: every row ties on trackversion=None (effectively unordered): different rows on the page |
| S-PL-streams-playlist-playlistscount-desc | TIE-UNORDERED | items: every row ties on playlistscount=1 (effectively unordered): different rows on the page |
| S-PL-streams-playlist-followerscount-desc | TIE-UNORDERED | items: every row ties on followerscount=None (effectively unordered): different rows on the page |
| S-PL-streams-playlist-trackscount-desc | TIE-UNORDERED | items: every row ties on trackscount=None (effectively unordered): different rows on the page |
| S-PL-streams-playlist-playlisttypeid-asc | TIE-UNORDERED | items: every row ties on playlisttypeid=None (effectively unordered): different rows on the page |
| S-PL-streams-playlist-playlisttypeid-desc | TIE-UNORDERED | items: every row ties on playlisttypeid=None (effectively unordered): different rows on the page |
| S-PL-streams-playlist-playlisttypename-asc | TIE-UNORDERED | items: every row ties on playlisttypename=None (effectively unordered): different rows on the page |
| S-PL-streams-playlist-playlisttypename-desc | TIE-UNORDERED | items: every row ties on playlisttypename=None (effectively unordered): different rows on the page |
| S-PL-streams-playlist-distributorid-asc | TIE-UNORDERED | items: every row ties on distributorid=10 (effectively unordered): different rows on the page |
| S-PL-streams-playlist-distributorid-desc | TIE-UNORDERED | items: every row ties on distributorid=61 (effectively unordered): different rows on the page |
| S-PL-streams-playlist-distributorname-asc | TIE-UNORDERED | items: every row ties on distributorname=Apple Music (effectively unordered): different rows on the page |
| S-PL-streams-playlist-distributorname-desc | TIE-UNORDERED | items: every row ties on distributorname=Spotify (effectively unordered): different rows on the page |
| S-PL-streams-playlist-countryid-asc | TIE-UNORDERED | items: every row ties on countryid=None (effectively unordered): different rows on the page |
| S-PL-streams-playlist-countryid-desc | TIE-UNORDERED | items: every row ties on countryid=None (effectively unordered): different rows on the page |
| S-PL-streams-playlist-iso2code-asc | TIE-UNORDERED | items: every row ties on iso2code=None (effectively unordered): different rows on the page |
| S-PL-streams-playlist-iso2code-desc | TIE-UNORDERED | items: every row ties on iso2code=None (effectively unordered): different rows on the page |
| S-PL-streams-playlist-eventdate-asc | TIE-UNORDERED | items: every row ties on eventdate=None (effectively unordered): different rows on the page |
| S-PL-streams-playlist-eventdate-desc | TIE-UNORDERED | items: every row ties on eventdate=None (effectively unordered): different rows on the page |
| S-PL-streams-playlistType-followerscount-desc | TIE | items: same rows, different order |
| S-PL-streams-playlistType-trackscount-desc | TIE | items: same rows, different order |
| S-PL-streams-playlistType-playlistid-asc | TIE | items: same rows, different order |
| S-PL-streams-playlistType-playlistid-desc | TIE | items: same rows, different order |
| S-PL-streams-playlistType-playlistname-desc | TIE | items: same rows, different order |
| S-PL-streams-playlistType-playlistowner-desc | TIE | items: same rows, different order |
| S-PL-streams-playlistType-playlistimageurl-desc | TIE | items: same rows, different order |
| S-PL-streams-playlistType-distributorid-desc | TIE | items: same rows, different order |
| S-PL-streams-playlistType-distributorname-desc | TIE | items: same rows, different order |
| S-PL-streams-playlistType-countryid-asc | TIE | items: same rows, different order |
| S-PL-streams-playlistType-iso2code-desc | TIE | items: same rows, different order |
| S-PL-streams-playlistType-eventdate-asc | TIE | items: same rows, different order |
| S-PL-streams-distributor-trackscount-desc | TIE | items: same rows, different order |
| S-PL-streams-distributor-playlistid-asc | TIE | items: same rows, different order |
| S-PL-streams-distributor-playlistid-desc | TIE | items: same rows, different order |
| S-PL-streams-distributor-playlistname-asc | TIE | items: same rows, different order |
| S-PL-streams-distributor-playlistname-desc | TIE | items: same rows, different order |
| S-PL-streams-distributor-playlistowner-asc | TIE | items: same rows, different order |
| S-PL-streams-distributor-playlistimageurl-desc | TIE | items: same rows, different order |
| S-PL-streams-distributor-playlisttypeid-desc | TIE | items: same rows, different order |
| S-PL-streams-distributor-playlisttypename-asc | TIE | items: same rows, different order |
| S-PL-streams-distributor-playlisttypename-desc | TIE | items: same rows, different order |
| S-PL-streams-distributor-countryid-desc | TIE | items: same rows, different order |
| S-PL-streams-distributor-iso2code-asc | TIE | items: same rows, different order |
| S-PL-streams-distributor-eventdate-asc | TIE | items: same rows, different order |
| S-PL-streams-country-followerscount-desc | TIE-UNORDERED | items: every row ties on followerscount=None (effectively unordered): different rows on the page |
| S-PL-streams-country-trackscount-desc | TIE-UNORDERED | items: every row ties on trackscount=None (effectively unordered): different rows on the page |
| S-PL-streams-country-playlistid-asc | TIE-UNORDERED | items: every row ties on playlistid=None (effectively unordered): different rows on the page |
| S-PL-streams-country-playlistid-desc | TIE-UNORDERED | items: every row ties on playlistid=None (effectively unordered): different rows on the page |
| S-PL-streams-country-playlistname-asc | TIE-UNORDERED | items: every row ties on playlistname=None (effectively unordered): different rows on the page |
| S-PL-streams-country-playlistname-desc | TIE-UNORDERED | items: every row ties on playlistname=None (effectively unordered): different rows on the page |
| S-PL-streams-country-playlistowner-asc | TIE-UNORDERED | items: every row ties on playlistowner=None (effectively unordered): different rows on the page |
| S-PL-streams-country-playlistowner-desc | TIE-UNORDERED | items: every row ties on playlistowner=None (effectively unordered): different rows on the page |
| S-PL-streams-country-playlistimageurl-asc | TIE-UNORDERED | items: every row ties on playlistimageurl=None (effectively unordered): different rows on the page |
| S-PL-streams-country-playlistimageurl-desc | TIE-UNORDERED | items: every row ties on playlistimageurl=None (effectively unordered): different rows on the page |
| S-PL-streams-country-playlisttypeid-desc | TIE-UNORDERED | items: every row ties on playlisttypeid=None (effectively unordered): different rows on the page |
| S-PL-streams-country-playlisttypename-asc | TIE-UNORDERED | items: every row ties on playlisttypename=None (effectively unordered): different rows on the page |
| S-PL-streams-country-playlisttypename-desc | TIE-UNORDERED | items: every row ties on playlisttypename=None (effectively unordered): different rows on the page |
| S-PL-streams-country-distributorid-asc | TIE-UNORDERED | items: every row ties on distributorid=None (effectively unordered): different rows on the page |
| S-PL-streams-country-distributorname-asc | TIE-UNORDERED | items: every row ties on distributorname=None (effectively unordered): different rows on the page |
| S-PL-streams-country-distributorname-desc | TIE-UNORDERED | items: every row ties on distributorname=None (effectively unordered): different rows on the page |
| S-PL-streams-country-eventdate-asc | TIE-UNORDERED | items: every row ties on eventdate=None (effectively unordered): different rows on the page |
| S-PL-country-playlistname-crossdim | TIE-UNORDERED | items: every row ties on playlistname=None (effectively unordered): different rows on the page |
| S-PM-active-playlistid-asc | TIE | items: same rows, different order |
| S-MV-track-trackversion-asc | TIE-UNORDERED | items: every row ties on trackversion=None (effectively unordered): different rows on the page |
| E-MV-track-streamChangePercent-daily-falling | TIE-UNORDERED | items: every row ties on absolutechange=-1 (effectively unordered): different rows on the page |
| E-MV-track-streamChangePercent-weekly-falling | TIE-UNORDERED | items: every row ties on absolutechange=-1 (effectively unordered): different rows on the page |
| E-MV-track-streamChangePercent-monthly-falling | TIE-UNORDERED | items: every row ties on absolutechange=-1 (effectively unordered): different rows on the page |
| E-MV-track-streamChangeAbsolute-daily-falling | TIE-UNORDERED | items: every row ties on absolutechange=-1 (effectively unordered): different rows on the page |
| E-MV-track-streamChangeAbsolute-weekly-falling | TIE-UNORDERED | items: every row ties on absolutechange=-1 (effectively unordered): different rows on the page |
| E-MV-track-streamChangeAbsolute-monthly-falling | TIE-UNORDERED | items: every row ties on absolutechange=-1 (effectively unordered): different rows on the page |
| E-MV-release-falling | TIE-UNORDERED | items: every row ties on absolutechange=-1 (effectively unordered): different rows on the page |
| G-PL-followers-playlistType-DAILY | TIE-UNORDERED | items: every row ties on playliststreamscount=None (effectively unordered): different rows on the page |
| G-PL-followers-distributor-DAILY | TIE-UNORDERED | items: every row ties on playliststreamscount=None (effectively unordered): different rows on the page |
| G-PL-followers-distributor-MONTHLY | TIE | items: same rows, different order |
| G-PL-followers-date-DAILY | TIE-UNORDERED | items: every row ties on playliststreamscount=None (effectively unordered): different rows on the page |
| G-PL-followers-date-MONTHLY | TIE | items: same rows, different order |
| P-CS-track-lastpage14184 | TIE-BOUNDARY | items: tie group streamscount=1 sliced differently: 3 rows differ |
| P-EN-track-lastpage15037 | TIE-UNORDERED | items: every row ties on sub30scount=0 (effectively unordered): different rows on the page |
| P-RV-track-lastpage14411 | TIE-UNORDERED | items: every row ties on netrevenue=0 (effectively unordered): different rows on the page |
| P-AS-track-lastpage20 | TIE-UNORDERED | items: every row ties on artificialstreamscount=1 (effectively unordered): different rows on the page |
| F-CS-streaming-track-onlyNew | TIE-BOUNDARY | items: page-boundary swap at streamscount=1: prod-only=2 stage-only=2 |
| F-EN-streaming-track-onlyNew | TIE | items: same rows, different order |
| F-EN-ugc-track-releaseIds | TIE-BOUNDARY | items: page-boundary swap at creationscount=1: prod-only=1 stage-only=1 |
| F-AS-track-onlyNew | TIE-BOUNDARY | items: page-boundary swap at artificialstreamscount=17: prod-only=3 stage-only=3 |

## Not covered

- Parked as too heavy for prod (timed out): 56 tests. Parent YEARLY granularity, DAILY on track/release/artist/label/country/sourceOfStream/playlist, the Movement deep last page. Their stage-only versions passed on the child.
- Parent yearly Movement (`periodType=yearly`): stage-only contract check (prod timed out).
- AS `organizationIds` for the parent (403 on the organization dimension, so no seeds). YouTube dimensions/filters and `targetEnterpriseId` (user decisions). Revenue `metricsByDate` (out of scope).
- Parent filters on the main dimensions only (user decision); the child ran the full filter x dimension matrix.
- NOT RUN at report time: 2 tests (C-S-EN-country-trackname-crossdim, C-S-RV-country-trackname-crossdim)
