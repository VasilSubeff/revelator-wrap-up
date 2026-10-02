# Findings with stage and prod curls

Each case shows the exact request the parity run sent (parent 127038 unless marked child 222139).
Set the tokens first; they are not stored here:

```bash
export STAGE_TOKEN=...        # parent 127038, stage
export PROD_TOKEN=...         # parent 127038, prod
export STAGE_CHILD_TOKEN=...  # child 222139, stage
```

Prod (0.0.96) still serves the old routes (`/v1/revenue/byX`, `/Engagement/byX`, `/ArtificialStreams/byX`, `/Playlists/byMetadata`),
stage serves the combined `/v1/*?dimension=` routes, so the two curls of a pair differ in path but query the same data.
The MCP client used in the run timed out at 5 s; plain curl has no such limit (add `--max-time 60`).


## F-01: Default sort has no real order without dateGranularity (prod + stage)

**`CS-ugc-track-base`**


Stage:
```bash
curl -s "https://platform.stage.revelator.com/analytics/v1/consumption?dimension=track&mediaType=ugc&fromDate=2026-06-01&toDate=2026-06-30&pageSize=5" \
  -H "accept: application/json" \
  -H "authorization: Bearer $STAGE_TOKEN" \
  -H "x-client-source: revelator-pro-app" \
  -H "origin: https://app.revelator.com" \
  -H "referer: https://app.revelator.com/"
```

Prod:
```bash
curl -s "https://platform.revelator.com/analytics/v1/consumption?dimension=track&mediaType=ugc&fromDate=2026-06-01&toDate=2026-06-30&pageSize=5" \
  -H "accept: application/json" \
  -H "authorization: Bearer $PROD_TOKEN" \
  -H "x-client-source: revelator-pro-app" \
  -H "origin: https://app.revelator.com" \
  -H "referer: https://app.revelator.com/"
```

Run result: **TIE-UNORDERED**: items: every row ties on eventdate=None (effectively unordered): different rows on the page


## F-02: Engagement streaming deliveryType sort validated against the distributor mapping (prod + stage)

**`S-EN-streaming-deliveryType-distributorid-asc`** - accepted, returns an empty 200


Stage:
```bash
curl -s "https://platform.stage.revelator.com/analytics/v1/engagement?dimension=deliveryType&mediaType=streaming&fromDate=2026-06-01&toDate=2026-06-30&orderByProperty=distributorid&orderByDescending=false&pageSize=5" \
  -H "accept: application/json" \
  -H "authorization: Bearer $STAGE_TOKEN" \
  -H "x-client-source: revelator-pro-app" \
  -H "origin: https://app.revelator.com" \
  -H "referer: https://app.revelator.com/"
```

Prod:
```bash
curl -s "https://platform.revelator.com/analytics/Engagement/byDeliveryType?isUgc=false&fromDate=2026-06-01&toDate=2026-06-30&orderByProperty=distributorid&orderByDescending=false&pageSize=5" \
  -H "accept: application/json" \
  -H "authorization: Bearer $PROD_TOKEN" \
  -H "x-client-source: revelator-pro-app" \
  -H "origin: https://app.revelator.com" \
  -H "referer: https://app.revelator.com/"
```

Run result: **FAIL**: prod: totalItemsCount 0 != base 3 (EN-streaming-deliveryType-base): sort changed the result set; prod: totals.streamsCount 0 != base 442008978 (EN-streaming-deliveryType-base); prod: totals.sub30sCount 0 != base 52034020 (EN-streaming-deliveryType-base)

**`S-EN-streaming-deliveryType-deliverytypeid-asc-DOCUMENTED`** - documented column, rejected with 400


Stage:
```bash
curl -s "https://platform.stage.revelator.com/analytics/v1/engagement?dimension=deliveryType&mediaType=streaming&fromDate=2026-06-01&toDate=2026-06-30&orderByProperty=deliverytypeid&orderByDescending=false&pageSize=5" \
  -H "accept: application/json" \
  -H "authorization: Bearer $STAGE_TOKEN" \
  -H "x-client-source: revelator-pro-app" \
  -H "origin: https://app.revelator.com" \
  -H "referer: https://app.revelator.com/"
```

Prod:
```bash
curl -s "https://platform.revelator.com/analytics/Engagement/byDeliveryType?isUgc=false&fromDate=2026-06-01&toDate=2026-06-30&orderByProperty=deliverytypeid&orderByDescending=false&pageSize=5" \
  -H "accept: application/json" \
  -H "authorization: Bearer $PROD_TOKEN" \
  -H "x-client-source: revelator-pro-app" \
  -H "origin: https://app.revelator.com" \
  -H "referer: https://app.revelator.com/"
```

Run result: **EXPECTED**: prod 400, stage 400 (expected {'prod_status': 'any', 'stage_status': 'any', 'then': 'equal'}); HTTP error 400: Bad Request - {'type': 'https://tools.ietf.org/html/rfc9110#section-15.5.1', 'title': 'One or more validation errors occurred.', 'status': 400, 'errors': {'General.


## F-03: 403 message calls a child enterprise a 'Payee' (stage)

**`C-AS-client-base`** (child 222139, stage only)


Stage:
```bash
curl -s "https://platform.stage.revelator.com/analytics/v1/artificial-streams?dimension=client&fromDate=2026-04-01&toDate=2026-06-30&pageSize=5" \
  -H "accept: application/json" \
  -H "authorization: Bearer $STAGE_CHILD_TOKEN" \
  -H "x-client-source: revelator-pro-app" \
  -H "origin: https://app.revelator.com" \
  -H "referer: https://app.revelator.com/"
```

Run result: **EXPECTED**: stage status 403


## F-05: Revenue metadata renamed to id/name (intentional)

**`RV-track-base`**


Stage:
```bash
curl -s "https://platform.stage.revelator.com/analytics/v1/revenue?dimension=track&fromDate=2026-06-01&toDate=2026-06-30&pageSize=5" \
  -H "accept: application/json" \
  -H "authorization: Bearer $STAGE_TOKEN" \
  -H "x-client-source: revelator-pro-app" \
  -H "origin: https://app.revelator.com" \
  -H "referer: https://app.revelator.com/"
```

Prod:
```bash
curl -s "https://platform.revelator.com/analytics/v1/revenue/byTrack?fromDate=2026-06-01&toDate=2026-06-30&pageSize=5" \
  -H "accept: application/json" \
  -H "authorization: Bearer $PROD_TOKEN" \
  -H "x-client-source: revelator-pro-app" \
  -H "origin: https://app.revelator.com" \
  -H "referer: https://app.revelator.com/"
```

Run result: **PASS**


## F-06: Playlists onlyNew takes > 5 s for the parent (prod + stage)

**`PL-E6-onlyNew`**


Stage:
```bash
curl -s "https://platform.stage.revelator.com/analytics/v1/playlists?dimension=playlist&metric=streams&onlyNew=true&pageSize=5" \
  -H "accept: application/json" \
  -H "authorization: Bearer $STAGE_TOKEN" \
  -H "x-client-source: revelator-pro-app" \
  -H "origin: https://app.revelator.com" \
  -H "referer: https://app.revelator.com/"
```

Prod:
```bash
curl -s "https://platform.revelator.com/analytics/Playlists/byMetadata?metadata=playlist&metric=streams&onlyNew=true&pageSize=5" \
  -H "accept: application/json" \
  -H "authorization: Bearer $PROD_TOKEN" \
  -H "x-client-source: revelator-pro-app" \
  -H "origin: https://app.revelator.com" \
  -H "referer: https://app.revelator.com/"
```

Run result: **FAIL**: prod 599, stage 599 (expected {'prod_status': 'any', 'stage_status': 200}); HTTP request timed out (ReadTimeout)


## F-07: Country PrimaryKey collision: id 2000 has ZZ and Unknown (prod + stage)

**`S-CS-streaming-country-countryid-desc`** - same row twice on one page


Stage:
```bash
curl -s "https://platform.stage.revelator.com/analytics/v1/consumption?dimension=country&mediaType=streaming&fromDate=2026-06-01&toDate=2026-06-30&orderByProperty=countryid&orderByDescending=true&pageSize=5" \
  -H "accept: application/json" \
  -H "authorization: Bearer $STAGE_TOKEN" \
  -H "x-client-source: revelator-pro-app" \
  -H "origin: https://app.revelator.com" \
  -H "referer: https://app.revelator.com/"
```

Prod:
```bash
curl -s "https://platform.revelator.com/analytics/v1/consumption?dimension=country&mediaType=streaming&fromDate=2026-06-01&toDate=2026-06-30&orderByProperty=countryid&orderByDescending=true&pageSize=5" \
  -H "accept: application/json" \
  -H "authorization: Bearer $PROD_TOKEN" \
  -H "x-client-source: revelator-pro-app" \
  -H "origin: https://app.revelator.com" \
  -H "referer: https://app.revelator.com/"
```

Run result: **FAIL**: prod: the same row appears 2x on one page (duplicate row, see F-07); items[0].metadata.iso2Code: prod=ZZ stage=Unknown; items[0].metrics[0].streamsCount: prod=2609340 stage=2844

**`C-G-EN-ugc-country-QUARTERLY`** (child 222139, stage only) - 6 rows for pageSize=5, copied metrics


Stage:
```bash
curl -s "https://platform.stage.revelator.com/analytics/v1/engagement?dimension=country&mediaType=ugc&fromDate=2026-01-01&toDate=2026-08-31&orderByProperty=creationscount&orderByDescending=true&pageSize=5&dateGranularity=QUARTERLY" \
  -H "accept: application/json" \
  -H "authorization: Bearer $STAGE_CHILD_TOKEN" \
  -H "x-client-source: revelator-pro-app" \
  -H "origin: https://app.revelator.com" \
  -H "referer: https://app.revelator.com/"
```

Run result: **FAIL**: stage page not ordered by creationscount desc


## F-08: With dateGranularity, sort ranks items by their best single period

**`G-CS-streaming-track-MONTHLY`**


Stage:
```bash
curl -s "https://platform.stage.revelator.com/analytics/v1/consumption?dimension=track&mediaType=streaming&fromDate=2026-04-01&toDate=2026-06-30&orderByProperty=streamscount&orderByDescending=true&pageSize=5&dateGranularity=MONTHLY" \
  -H "accept: application/json" \
  -H "authorization: Bearer $STAGE_TOKEN" \
  -H "x-client-source: revelator-pro-app" \
  -H "origin: https://app.revelator.com" \
  -H "referer: https://app.revelator.com/"
```

Prod:
```bash
curl -s "https://platform.revelator.com/analytics/v1/consumption?dimension=track&mediaType=streaming&fromDate=2026-04-01&toDate=2026-06-30&orderByProperty=streamscount&orderByDescending=true&pageSize=5&dateGranularity=MONTHLY" \
  -H "accept: application/json" \
  -H "authorization: Bearer $PROD_TOKEN" \
  -H "x-client-source: revelator-pro-app" \
  -H "origin: https://app.revelator.com" \
  -H "referer: https://app.revelator.com/"
```

Run result: **PASS**


## F-09: Playlist movements paging validation (prod + stage)

**`P-PM-active-page0`** - 500 on both


Stage:
```bash
curl -s "https://platform.stage.revelator.com/analytics/v1/playlists/movements?movementType=active&period=28&orderByProperty=playliststreamscount&orderByDescending=true&pageSize=5&pageNumber=0" \
  -H "accept: application/json" \
  -H "authorization: Bearer $STAGE_TOKEN" \
  -H "x-client-source: revelator-pro-app" \
  -H "origin: https://app.revelator.com" \
  -H "referer: https://app.revelator.com/"
```

Prod:
```bash
curl -s "https://platform.revelator.com/analytics/Playlists/movements?movementType=active&period=28&orderByProperty=playliststreamscount&orderByDescending=true&pageSize=5&pageNumber=0" \
  -H "accept: application/json" \
  -H "authorization: Bearer $PROD_TOKEN" \
  -H "x-client-source: revelator-pro-app" \
  -H "origin: https://app.revelator.com" \
  -H "referer: https://app.revelator.com/"
```

Run result: **FAIL**: prod 500, stage 500 (expected {'prod_status': 'any', 'stage_status': 400}); HTTP error 500: Internal Server Error - {'type': 'https://tools.ietf.org/html/rfc9110#section-15.6.1', 'title': 'An error occurred while processing your request.', 'status': 500, '

**`P-PM-active-size0`** - empty 200 on both


Stage:
```bash
curl -s "https://platform.stage.revelator.com/analytics/v1/playlists/movements?movementType=active&period=28&orderByProperty=playliststreamscount&orderByDescending=true&pageSize=0" \
  -H "accept: application/json" \
  -H "authorization: Bearer $STAGE_TOKEN" \
  -H "x-client-source: revelator-pro-app" \
  -H "origin: https://app.revelator.com" \
  -H "referer: https://app.revelator.com/"
```

Prod:
```bash
curl -s "https://platform.revelator.com/analytics/Playlists/movements?movementType=active&period=28&orderByProperty=playliststreamscount&orderByDescending=true&pageSize=0" \
  -H "accept: application/json" \
  -H "authorization: Bearer $PROD_TOKEN" \
  -H "x-client-source: revelator-pro-app" \
  -H "origin: https://app.revelator.com" \
  -H "referer: https://app.revelator.com/"
```

Run result: **FAIL**: prod 200, stage 200 (expected {'prod_status': 'any', 'stage_status': 400})

**`P-PL-playlist-page0`** - byMetadata: prod 500, stage 400 (fixed on stage)


Stage:
```bash
curl -s "https://platform.stage.revelator.com/analytics/v1/playlists?dimension=playlist&metric=streams&fromDate=2026-06-01&toDate=2026-06-30&orderByProperty=playliststreamscount&orderByDescending=true&pageSize=5&pageNumber=0" \
  -H "accept: application/json" \
  -H "authorization: Bearer $STAGE_TOKEN" \
  -H "x-client-source: revelator-pro-app" \
  -H "origin: https://app.revelator.com" \
  -H "referer: https://app.revelator.com/"
```

Prod:
```bash
curl -s "https://platform.revelator.com/analytics/Playlists/byMetadata?metric=streams&fromDate=2026-06-01&toDate=2026-06-30&orderByProperty=playliststreamscount&orderByDescending=true&pageSize=5&metadata=playlist&pageNumber=0" \
  -H "accept: application/json" \
  -H "authorization: Bearer $PROD_TOKEN" \
  -H "x-client-source: revelator-pro-app" \
  -H "origin: https://app.revelator.com" \
  -H "referer: https://app.revelator.com/"
```

Run result: **EXPECTED**: prod 500, stage 400 (expected {'prod_status': 'any', 'stage_status': 400}); Error calling tool 'Playlists_GetPlaylists': HTTP error 400: Bad Request - {'type': 'https://tools.ietf.org/html/rfc9110#section-15.5.1', 'title': 'One or more validation errors oc


## F-10: Inconsistent error bodies

**`E-RV-dim-unknown`** - Revenue: different error shape


Stage:
```bash
curl -s "https://platform.stage.revelator.com/analytics/v1/revenue?dimension=zzdim&fromDate=2026-06-01&toDate=2026-06-30&orderByProperty=netrevenue&orderByDescending=true&pageSize=5" \
  -H "accept: application/json" \
  -H "authorization: Bearer $STAGE_TOKEN" \
  -H "x-client-source: revelator-pro-app" \
  -H "origin: https://app.revelator.com" \
  -H "referer: https://app.revelator.com/"
```

Prod:
```bash
curl -s "https://platform.revelator.com/analytics/v1/revenue/byTrack?fromDate=2026-06-01&toDate=2026-06-30&orderByProperty=netrevenue&orderByDescending=true&pageSize=5" \
  -H "accept: application/json" \
  -H "authorization: Bearer $PROD_TOKEN" \
  -H "x-client-source: revelator-pro-app" \
  -H "origin: https://app.revelator.com" \
  -H "referer: https://app.revelator.com/"
```

Run result: **EXPECTED**: prod 200, stage 400 (expected {'prod_status': 'any', 'stage_status': 400}); Error calling tool 'Revenue_GetRevenue': HTTP error 400: Bad Request - {'error': 'The provided dimension is not valid.'}

**`D-CS-malformed`** - prod message has Cyrillic 'ММ'


Stage:
```bash
curl -s "https://platform.stage.revelator.com/analytics/v1/consumption?dimension=track&mediaType=streaming&fromDate=2026-13-01&toDate=2026-06-30&orderByProperty=streamscount&orderByDescending=true&pageSize=5" \
  -H "accept: application/json" \
  -H "authorization: Bearer $STAGE_TOKEN" \
  -H "x-client-source: revelator-pro-app" \
  -H "origin: https://app.revelator.com" \
  -H "referer: https://app.revelator.com/"
```

Prod:
```bash
curl -s "https://platform.revelator.com/analytics/v1/consumption?dimension=track&mediaType=streaming&fromDate=2026-13-01&toDate=2026-06-30&orderByProperty=streamscount&orderByDescending=true&pageSize=5" \
  -H "accept: application/json" \
  -H "authorization: Bearer $PROD_TOKEN" \
  -H "x-client-source: revelator-pro-app" \
  -H "origin: https://app.revelator.com" \
  -H "referer: https://app.revelator.com/"
```

Run result: **EXPECTED**: prod 400, stage 400 (expected {'prod_status': 'any', 'stage_status': 'any'}); Error calling tool 'Consumption_GetConsumption': HTTP error 400: Bad Request - {'type': 'https://tools.ietf.org/html/rfc9110#section-15.5.1', 'title': 'One or more validation error


## F-11: Engagement consumptionTypeIds ignored (prod + stage)

**`F-EN-streaming-consumptionType-consumptionTypeIds`**


Stage:
```bash
curl -s "https://platform.stage.revelator.com/analytics/v1/engagement?dimension=consumptionType&mediaType=streaming&fromDate=2026-06-01&toDate=2026-06-30&orderByProperty=sub30scount&orderByDescending=true&pageSize=5&consumptionTypeIds=41&consumptionTypeIds=44" \
  -H "accept: application/json" \
  -H "authorization: Bearer $STAGE_TOKEN" \
  -H "x-client-source: revelator-pro-app" \
  -H "origin: https://app.revelator.com" \
  -H "referer: https://app.revelator.com/"
```

Prod:
```bash
curl -s "https://platform.revelator.com/analytics/Engagement/byConsumptionType?isUgc=false&fromDate=2026-06-01&toDate=2026-06-30&orderByProperty=sub30scount&orderByDescending=true&pageSize=5&consumptionTypeIds=41&consumptionTypeIds=44" \
  -H "accept: application/json" \
  -H "authorization: Bearer $PROD_TOKEN" \
  -H "x-client-source: revelator-pro-app" \
  -H "origin: https://app.revelator.com" \
  -H "referer: https://app.revelator.com/"
```

Run result: **FAIL**: prod: rows outside consumptionTypeIds=[41, 44]: [45, 47, 48]; stage: rows outside consumptionTypeIds=[41, 44]: [45, 47, 48]


## F-12: Playlists ignores excludeDistributorIds / excludeTikTok (prod + stage)

**`F-PL-streams-distributor-excludeDistributorIds`** - Spotify (10) still present


Stage:
```bash
curl -s "https://platform.stage.revelator.com/analytics/v1/playlists?dimension=distributor&metric=streams&fromDate=2026-06-01&toDate=2026-06-30&orderByProperty=playliststreamscount&orderByDescending=true&pageSize=5&excludeDistributorIds=10" \
  -H "accept: application/json" \
  -H "authorization: Bearer $STAGE_TOKEN" \
  -H "x-client-source: revelator-pro-app" \
  -H "origin: https://app.revelator.com" \
  -H "referer: https://app.revelator.com/"
```

Prod:
```bash
curl -s "https://platform.revelator.com/analytics/Playlists/byMetadata?metric=streams&fromDate=2026-06-01&toDate=2026-06-30&orderByProperty=playliststreamscount&orderByDescending=true&pageSize=5&metadata=distributor&excludeDistributorIds=10" \
  -H "accept: application/json" \
  -H "authorization: Bearer $PROD_TOKEN" \
  -H "x-client-source: revelator-pro-app" \
  -H "origin: https://app.revelator.com" \
  -H "referer: https://app.revelator.com/"
```

Run result: **FAIL**: prod: excluded ids still present: [10]; stage: excluded ids still present: [10]


## F-13: sourceOfStream + dateGranularity returns more rows than pageSize (prod + stage)

**`G-CS-streaming-sourceOfStream-DAILY`** - 8 rows for pageSize=5


Stage:
```bash
curl -s "https://platform.stage.revelator.com/analytics/v1/consumption?dimension=sourceOfStream&mediaType=streaming&fromDate=2026-06-01&toDate=2026-06-30&orderByProperty=streamscount&orderByDescending=true&pageSize=5&dateGranularity=DAILY" \
  -H "accept: application/json" \
  -H "authorization: Bearer $STAGE_TOKEN" \
  -H "x-client-source: revelator-pro-app" \
  -H "origin: https://app.revelator.com" \
  -H "referer: https://app.revelator.com/"
```

Prod:
```bash
curl -s "https://platform.revelator.com/analytics/v1/consumption?dimension=sourceOfStream&mediaType=streaming&fromDate=2026-06-01&toDate=2026-06-30&orderByProperty=streamscount&orderByDescending=true&pageSize=5&dateGranularity=DAILY" \
  -H "accept: application/json" \
  -H "authorization: Bearer $PROD_TOKEN" \
  -H "x-client-source: revelator-pro-app" \
  -H "origin: https://app.revelator.com" \
  -H "referer: https://app.revelator.com/"
```

Run result: **FAIL**: prod: page has 8 rows for pageSize=5 (see F-13); stage: page has 8 rows for pageSize=5 (see F-13)


## F-14: Movement accepts out-of-range paging (prod + stage)

**`P-MV-track-size0`**


Stage:
```bash
curl -s "https://platform.stage.revelator.com/analytics/v1/movement?dimension=track&metricType=streamChangeAbsolute&movementType=rising&periodType=monthly&orderByProperty=absolutechange&orderByDescending=true&pageSize=0" \
  -H "accept: application/json" \
  -H "authorization: Bearer $STAGE_TOKEN" \
  -H "x-client-source: revelator-pro-app" \
  -H "origin: https://app.revelator.com" \
  -H "referer: https://app.revelator.com/"
```

Prod:
```bash
curl -s "https://platform.revelator.com/analytics/v1/movement?dimension=track&metricType=streamChangeAbsolute&movementType=rising&periodType=monthly&orderByProperty=absolutechange&orderByDescending=true&pageSize=0" \
  -H "accept: application/json" \
  -H "authorization: Bearer $PROD_TOKEN" \
  -H "x-client-source: revelator-pro-app" \
  -H "origin: https://app.revelator.com" \
  -H "referer: https://app.revelator.com/"
```

Run result: **FAIL**: prod 200, stage 200 (expected {'prod_status': 'any', 'stage_status': 400})

**`P-MV-track-page0`**


Stage:
```bash
curl -s "https://platform.stage.revelator.com/analytics/v1/movement?dimension=track&metricType=streamChangeAbsolute&movementType=rising&periodType=monthly&orderByProperty=absolutechange&orderByDescending=true&pageSize=5&pageNumber=0" \
  -H "accept: application/json" \
  -H "authorization: Bearer $STAGE_TOKEN" \
  -H "x-client-source: revelator-pro-app" \
  -H "origin: https://app.revelator.com" \
  -H "referer: https://app.revelator.com/"
```

Prod:
```bash
curl -s "https://platform.revelator.com/analytics/v1/movement?dimension=track&metricType=streamChangeAbsolute&movementType=rising&periodType=monthly&orderByProperty=absolutechange&orderByDescending=true&pageSize=5&pageNumber=0" \
  -H "accept: application/json" \
  -H "authorization: Bearer $PROD_TOKEN" \
  -H "x-client-source: revelator-pro-app" \
  -H "origin: https://app.revelator.com" \
  -H "referer: https://app.revelator.com/"
```

Run result: **FAIL**: prod 200, stage 200 (expected {'prod_status': 'any', 'stage_status': 400})


## F-15: Stage Movement quarterly/yearly by track takes > 5 s; prod < 5 s

**`E-MV-track-revenueChangeAbsolute-quarterly-rising`**


Stage:
```bash
curl -s "https://platform.stage.revelator.com/analytics/v1/movement?dimension=track&metricType=revenueChangeAbsolute&movementType=rising&periodType=quarterly&orderByProperty=absolutechange&orderByDescending=true&pageSize=5" \
  -H "accept: application/json" \
  -H "authorization: Bearer $STAGE_TOKEN" \
  -H "x-client-source: revelator-pro-app" \
  -H "origin: https://app.revelator.com" \
  -H "referer: https://app.revelator.com/"
```

_Prod: stage-only test (prod call parked as too heavy, or no prod counterpart)._

Run result: **FAIL**: stage status 599, expected 200: Error calling tool 'Movement_GetMovement': HTTP request timed out (ReadTimeout)

**`E-MV-track-streamChangeAbsolute-yearly-rising`**


Stage:
```bash
curl -s "https://platform.stage.revelator.com/analytics/v1/movement?dimension=track&metricType=streamChangeAbsolute&movementType=rising&periodType=yearly&orderByProperty=absolutechange&orderByDescending=true&pageSize=5" \
  -H "accept: application/json" \
  -H "authorization: Bearer $STAGE_TOKEN" \
  -H "x-client-source: revelator-pro-app" \
  -H "origin: https://app.revelator.com" \
  -H "referer: https://app.revelator.com/"
```

_Prod: stage-only test (prod call parked as too heavy, or no prod counterpart)._

Run result: **FAIL**: stage status 599, expected 200: Error calling tool 'Movement_GetMovement': HTTP request timed out (ReadTimeout)


## F-16: Movement country ignores excludeDistributorIds (prod + stage)

**`F-MV-country-excludeDistributorIds`**


Stage:
```bash
curl -s "https://platform.stage.revelator.com/analytics/v1/movement?dimension=country&metricType=streamChangeAbsolute&movementType=rising&periodType=monthly&orderByProperty=absolutechange&orderByDescending=true&pageSize=5&excludeDistributorIds=85" \
  -H "accept: application/json" \
  -H "authorization: Bearer $STAGE_TOKEN" \
  -H "x-client-source: revelator-pro-app" \
  -H "origin: https://app.revelator.com" \
  -H "referer: https://app.revelator.com/"
```

Prod:
```bash
curl -s "https://platform.revelator.com/analytics/v1/movement?dimension=country&metricType=streamChangeAbsolute&movementType=rising&periodType=monthly&orderByProperty=absolutechange&orderByDescending=true&pageSize=5&excludeDistributorIds=85" \
  -H "accept: application/json" \
  -H "authorization: Bearer $PROD_TOKEN" \
  -H "x-client-source: revelator-pro-app" \
  -H "origin: https://app.revelator.com" \
  -H "referer: https://app.revelator.com/"
```

Run result: **IGNORED-BOTH**: prod: excludeDistributorIds=[85] returned exactly the unfiltered result (filter ignored?); stage: excludeDistributorIds=[85] returned exactly the unfiltered result (filter ignored?)


## F-17: AS dashboard MONTHLY totalItemsCount counts client x month (prod + stage)

**`G-ASD-client-MONTHLY`**


Stage:
```bash
curl -s "https://platform.stage.revelator.com/analytics/v1/artificial-streams/dashboard?dimension=client&fromDate=2026-04-01&toDate=2026-06-30&orderByProperty=artificialstreamscount&orderByDescending=true&pageSize=5&dateGranularity=MONTHLY" \
  -H "accept: application/json" \
  -H "authorization: Bearer $STAGE_TOKEN" \
  -H "x-client-source: revelator-pro-app" \
  -H "origin: https://app.revelator.com" \
  -H "referer: https://app.revelator.com/"
```

Prod:
```bash
curl -s "https://platform.revelator.com/analytics/ArtificialStreams/dashboard/byClient?fromDate=2026-04-01&toDate=2026-06-30&orderByProperty=artificialstreamscount&orderByDescending=true&pageSize=5&dateGranularity=MONTHLY" \
  -H "accept: application/json" \
  -H "authorization: Bearer $PROD_TOKEN" \
  -H "x-client-source: revelator-pro-app" \
  -H "origin: https://app.revelator.com" \
  -H "referer: https://app.revelator.com/"
```

Run result: **FAIL**: prod: totalItemsCount 31 != base 14 (ASD-client-artificialstreamscount-desc): sort changed the result set; stage: totalItemsCount 31 != base 14 (ASD-client-artificialstreamscount-desc): sort changed the result set


## E1: uniqueKeys applied on stage, ignored on prod (documented, PR #113)

**`F-CS-streaming-sourceOfStream-uniqueKeys`**


Stage:
```bash
curl -s "https://platform.stage.revelator.com/analytics/v1/consumption?dimension=sourceOfStream&mediaType=streaming&fromDate=2026-06-01&toDate=2026-06-30&orderByProperty=streamscount&orderByDescending=true&pageSize=5&uniqueKeys=4%2F10&uniqueKeys=7%2F10" \
  -H "accept: application/json" \
  -H "authorization: Bearer $STAGE_TOKEN" \
  -H "x-client-source: revelator-pro-app" \
  -H "origin: https://app.revelator.com" \
  -H "referer: https://app.revelator.com/"
```

Prod:
```bash
curl -s "https://platform.revelator.com/analytics/v1/consumption?dimension=sourceOfStream&mediaType=streaming&fromDate=2026-06-01&toDate=2026-06-30&orderByProperty=streamscount&orderByDescending=true&pageSize=5&uniqueKeys=4%2F10&uniqueKeys=7%2F10" \
  -H "accept: application/json" \
  -H "authorization: Bearer $PROD_TOKEN" \
  -H "x-client-source: revelator-pro-app" \
  -H "origin: https://app.revelator.com" \
  -H "referer: https://app.revelator.com/"
```

Run result: **EXPECTED**: prod 200, stage 200 (expected {'prod_status': 'any', 'stage_status': 'any', 'note': 'E1: stage applies uniqueKeys, prod ignores it (documented, PR #113)'})


## F-18: Playlists metric=All with trackIds over 3 months fails after 15 s (prod, real user)

Real user request (enterprise 971905, child of 960929), 2026-10-01 15:06 UTC, trace `6abe76f000000000ea73cd4e7e8322b4`. Needs a token that can see 971905 (`targetEnterpriseId`).

Prod:
```bash
curl -s "https://platform.revelator.com/analytics/Playlists/byMetadata?fromDate=2026-07-04&toDate=2026-10-01&dateGranularity=All&pageNumber=1&pageSize=100&metric=All&metadata=Distributor&orderByDescending=true&trackIds=8728903&targetEnterpriseId=971905" \
  -H "accept: application/json" \
  -H "authorization: Bearer $PROD_TOKEN" \
  -H "x-client-source: revelator-pro-app" \
  -H "origin: https://app.revelator.com" \
  -H "referer: https://app.revelator.com/" \
  --max-time 60
```

Stage (combined endpoint, same query):
```bash
curl -s "https://platform.stage.revelator.com/analytics/v1/playlists?fromDate=2026-07-04&toDate=2026-10-01&dateGranularity=All&pageNumber=1&pageSize=100&metric=All&orderByDescending=true&trackIds=8728903&targetEnterpriseId=971905&dimension=distributor" \
  -H "accept: application/json" \
  -H "authorization: Bearer $STAGE_TOKEN" \
  -H "x-client-source: revelator-pro-app" \
  -H "origin: https://app.revelator.com" \
  -H "referer: https://app.revelator.com/" \
  --max-time 60
```

