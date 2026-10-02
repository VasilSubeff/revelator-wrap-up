# Parity run findings (live log)

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
