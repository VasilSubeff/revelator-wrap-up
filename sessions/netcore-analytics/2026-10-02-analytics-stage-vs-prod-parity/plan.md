# Analytics stage vs prod parity test plan (via MCP)

**Date:** 2026-09-29
**Goal:** every stage response matches its prod counterpart 100%, after normalising only the contract changes the API-standards specs made on purpose (§3). Any other difference is a finding.
**Tools:** `Revelator Analytics Stage MCP Server` (stage, new contracts) vs `Revelator Analytics MCP Server` (prod 0.0.96, old contracts). No direct HTTP.
**Status:** DRAFT rev 3 (2026-09-30).
- rev 2: §5.F, a full filter × dimension matrix seeded from each filter's own dimension.
- rev 3: every accepted property (sorting, granularity, paging, dates, enums), and `metricsByDate` dropped.
- rev 4: user answers (§6), self-imposed rate limits (§2), stage child run (§5.C). Waiting for approval. Nothing runs until the user says go and provides tokens.

## 1. What is being compared

Stage has the standards code (combined endpoints). Prod still has the old routes. Each test is one **pair**: a prod call and the stage call it maps to.

| Domain | Prod MCP tool(s) | Stage MCP tool | Param mapping prod → stage |
|---|---|---|---|
| Consumption | `Consumption_GetConsumption` (`/v1/consumption`) | `Consumption_GetConsumption` | identical |
| Revenue | `Revenue_GetRevenueBy<X>` (`/v1/revenue/byX`), 11 dims | `Revenue_GetRevenue` | `byX` → `dimension=x` |
| Revenue legacy route | `Revenue_GetRevenueBy<X>2` (`/revenue/byX`) | `Revenue_GetRevenue` | smoke only (same handler as v1) |
| Engagement | `Engagement_GetEngagementBy<X>` × 12 | `Engagement_GetEngagement` | `byX` → `dimension=x`; `isUgc=true/false` → `mediaType=ugc/streaming` |
| Artificial Streams | `ArtificialStreams_GetArtificialStreamsBy<X>` × 9 | `ArtificialStreams_GetArtificialStreams` | `byX` → `dimension=x` |
| AS dashboard | `…DashboardByClient`, `…DashboardByOrganiz…` | `ArtificialStreams_GetArtificialStreamsDashboard` | → `dimension=client/organization` |
| Playlists | `Playlists_GetPlaylistsByMetadata` | `Playlists_GetPlaylists` | `metadata=X` → `dimension=X`; metadata omitted → `dimension=date` |
| Playlist movements | `Playlists_GetPlaylistMovements` | `Playlists_GetPlaylistMovements` | identical, except that stage has no `countryIds` |
| Movement | `Movement_GetMovement` | `Movement_GetMovement` | identical (`client` dim: see E-list) |

**Special cases:**
- Revenue `metricsByDate` is **out of scope** (user decision, 2026-09-30).
- Revenue `dimension=date` exists on stage only: its totals are compared with prod `byTrack` totals for the same filter.

## 2. Mechanics

- **Fixed historical windows**, never "today" defaults, so ingestion lag can't cause false diffs:
  - W1 = `2026-06-01..2026-06-30`
  - W2 = `2026-04-01..2026-06-30`
  - W3 = `2026-08-01..2026-08-31` (fresh-data check: base pairs of Consumption, Revenue and Engagement track are re-run on W3, +3 pairs)
  - Movement and playlist movements have no date params. Their windows follow each environment's latest loaded data, so check the returned `fromDate/toDate` first. If they differ, the pair is a **data** diff, not a code diff.
- **Small pages:** `pageSize=5` unless the test is about paging. That still exercises ordering and keeps MCP payloads small, which reduces transcription risk.
- **Step 0, pilot: DONE 2026-09-30.**
  - A subagent loaded and called the stage MCP healthcheck (OK).
  - `python time.sleep(3)` works in a subagent's Bash, so `throttle.py` can enforce real gaps.
  - Result: `scratchpad/parity/pilot.json`.
- **Step 1, gate:** stage is on 2.0.7 (user, 2026-09-30). The stage healthcheck returned OK with the parent token, but the healthcheck reports no version.
- **Execution:**
  - One subagent per domain, run **sequentially** (no parallel domains) because of rate limits.
  - For each pair, the agent calls prod then stage, saves both raw results to `scratchpad/parity/<domain>/<testId>.{prod,stage}.json`, and runs `parity_compare.py` (normalisation + deep diff, deterministic).
  - The agent reports only the summary and the diffs. This keeps the main context small.
- **Seeds (phase 0):** every filter value comes from the dimension that returns those ids. The base pair for that dimension doubles as the seed call, so seeding costs no extra calls. The full source table is in §5.F.1. Only ids present in **both** environments' results are used, which keeps filtered pairs from diverging just because the seed differs.
- **Rate limits (self-imposed; user: "don't put strain on prod"; the real limit is unknown):**
  - **One call in flight, globally.** No parallel domains, no parallel subagents, and never a prod and a stage call at the same time.
  - **Prod:** ≥ 3 s between calls (≤ 20/min; user decision 2026-09-30, below the ~20–30/min the testing skill's own suites ran at), ≤ 700 calls per sitting (~35 min of prod traffic), `pageSize` ≤ 5 except the three §5.P page-size cases. `pageSize=2000` only on the smallest dimension.
  - **Stage:** ≥ 2 s between calls (≤ 30/min).
  - **How the gap is enforced:** `throttle.py` records the last call time per environment in `scratchpad/parity/throttle.json` and waits out the remainder before the next call. If waiting isn't possible in the agent, the natural MCP round-trip is the floor, and the prod cap per sitting is halved to 350.
  - **Prod response cache:** prod calls are keyed by their exact normalised params. An identical prod request (e.g. every ignored-check, every `NO-PROD-PARAM` pair and the stage-only assertions) reuses the saved prod file instead of calling again. Stage-only checks (sort monotonicity, page stitching, granularity periods) never call prod.
  - **Back-off:** on a 429, "Could not acquire server resources", a timeout or two 5xx in a row on prod, stop prod calls for 5 minutes and then retry once. On a second hit, stop the chunk and report where it stopped. Never retry a prod call more than once.
  - **Resumable:** a test whose result files are already saved is skipped.
- **Retry before classifying:** stage providers swallow DAL errors into empty 200s. Any "stage empty / prod has data" pair is re-run once before it counts.

## 3. Normalisation rules (intentional contract changes only)

| # | Rule | Source |
|---|---|---|
| N1 | Dates compared as parsed values (`…T00:00:00Z` == `…T00:00:00`) | playlists F1, all DAL domains |
| N2 | `pageSize`: prod = rows returned, stage = requested. Check `prod.pageSize == len(items)` and `stage.pageSize == requested`, then drop the field | Revenue §4.5, all standards specs |
| N3 | `totalPagesCount` exists on stage only. Check `== ceil(totalItemsCount / pageSize)`, then drop it | same |
| N4 | Metadata: stage uses one union model with null keys for other dimensions' fields. Drop null-valued metadata keys on both sides before comparing | Revenue / Engagement / AS specs |
| N5 | `items: null` (prod) == `items: []` (stage) | standards specs |
| N6 | Error bodies: compare status codes only, not the ProblemDetails text | Movement spec (ProblemDetails) |
| N7 | Top-level envelope field names are compared after N2–N5. Anything left over is a finding | — |

Numbers are compared **exactly**: no epsilon, no rounding. If a float differs only in the last digits, it is reported as its own class (`FLOAT`) and not hidden.

**Ordering:** items are compared in order. If the order differs **only inside a group of equal sort-key values**, the pair is re-compared as a multiset and classed `TIE` (content equal, DB order non-deterministic). Anything else is `FAIL`. You decide whether `TIE` counts toward 100% (§6 Q4).

## 4. Expected divergences (documented behaviour changes, asserted, not failures)

| # | Case | Expected |
|---|---|---|
| E1 | Engagement / Consumption `uniqueKeys` on sourceOfStream | stage applies the filter; prod ignores it (the SingleStore path never applied it) → stage ⊆ prod |
| E2 | Engagement ugc × sourceOfStream / discoveryType | stage 400; prod returns streaming data |
| E3 | Consumption ugc × assetType / sourceOfStream / discoveryType | stage 400; prod 500 or streaming data |
| E4 | Movement `dimension=client` | stage 400; prod returns track rows |
| E5 | Playlist movements `countryIds` | not on stage (dropped). Prod ignored it anyway, so one pair checks prod-with-countryIds == stage-without |
| E6 | Playlists `onlyNew` | prod 500 (F5); stage 200 |
| E7 | Playlist movements `orderByProperty=playlistscount` | prod 500, stage 400 (F6) |
| E8 | Revenue sort `labelname` on track/release/artist; `deliverytypenname` | prod 500. Stage: record the result, and flag it if it isn't 400 or a valid 200 |
| E9 | Engagement `discoveryTypeIds` on sourceOfStream | ignored on both sides → equal |

## 5. Test catalogue

**Scope rule (user, 2026-09-30):** every property each endpoint accepts is exercised, with every allowed value, on every dimension where the value changes the SQL. The inventory below is the stage MCP tool schemas as of 2026-09-30. Revenue `metricsByDate` is **out of scope** (user decision).

W1 is the default window (W2 for AS). "seed" = an id from §5.F.1. Every sub-matrix uses `pageSize=5` unless it is testing paging.

| Section | What | Pairs |
|---|---|---|
| 5.0 | Smoke + base (every dimension, default params) | ≈ 105 |
| 5.S | Sorting: every `orderByProperty` × every dimension it applies to (metric DESC; text/id/date ASC+DESC) | ≈ 890 |
| 5.G | `dateGranularity`: every allowed value × every dimension | ≈ 410 |
| 5.P | Paging: `pageNumber` / `pageSize` values and bounds | ≈ 160 |
| 5.D | Dates: `fromDate` / `toDate` defaults, edges and invalid values; `onlyNew` × dates | ≈ 42 |
| 5.E | Enum / required params: `dimension`, `mediaType`, `metric`, `movementType`, `period`, `metricType`, `periodType`, `previousFromDate/ToDate`, case-insensitivity, invalid values | ≈ 165 |
| 5.F | Filters: every filter × every dimension (YouTube and off-dimension organizationIds dropped) | ≈ 820 |
| **Total parent parity** | | **≈ 2,560 pairs**, ≈ 2,560 stage calls + ≤ 2,560 prod calls (fewer after the prod cache, §2) |
| 5.C | Stage child 222139 functional run (no prod) | ≈ 2,370 stage calls |
| **Grand total** | | **≈ 7,530 calls: ≈ 4,960 stage + ≈ 2,565 prod** |

### 5.0 Smoke + base (≈ 105)
- Healthcheck on both environments (record versions); one Revenue v1 vs legacy-route pair.
- **Base (default sort, no filters, W1):**
  - Consumption: 13 streaming + 10 ugc dims
  - Engagement: 12 + 10
  - Revenue: 11
  - AS: 9 + dashboard 2
  - Playlists: 4 dims × 4 metrics + date × 2 metrics + metric=all (19)
  - movements: 3 types × 3 periods (9)
  - Movement: 6 dims (rising, monthly, streamChangeAbsolute)
- **W2 and W3 re-runs:** track + country base on Consumption, Revenue and Engagement (12).
- **Kept special pairs:**
  - combos artistIds+countryIds and labelIds+distributorIds (Consumption)
  - stage Revenue `dimension=date` totals vs prod byTrack totals (1)
  - empty-set parity with a non-matching id (Playlists)
- **E-list pairs** from §4 (E2–E8).

### 5.S Sorting (≈ 890)

For each endpoint and dimension: **every** accepted `orderByProperty`. Direction rule (user, Q9): **metric (numeric) columns DESC only; text, id and date columns (`eventdate`, `dateadded`, `dateremoved`) both ASC and DESC**. The accepted list per dimension = the domain's metric columns + that dimension's own columns, taken verbatim from the stage tool description. Before execution, the list is generated into `scratchpad/parity/sorts.json` so the count is exact.

| Endpoint | Metric sort columns (DESC) | Own-dimension columns (ASC + DESC) | Pairs |
|---|---|---|---|
| Consumption streaming (13 dims) | streamscount, trackdownloadscount, releasedownloadscount, eventdate | e.g. track: trackid/trackname/trackversion; country: countryid/iso2code; sourceOfStream: sourceofstream* + discoverytype* | 39 + (13 + 28) × 2 = 121 |
| Consumption ugc (10 dims) | viewscount, eventdate | same own columns | 10 + (10 + 20) × 2 = 70 |
| Engagement streaming (12) | sub30scount, savescount, skiprate, completerate, eventdate | as Consumption; sourceOfStream has 6 | 48 + (12 + 28) × 2 = 128 |
| Engagement ugc (10) | creationscount, likescount, sharescount, commentscount, watchtime, eventdate | as above | 50 + (10 + 20) × 2 = 110 |
| Revenue (9, YouTube excluded) | streamscount, downloadscount, streamsrevenue, downloadsrevenue, netrevenue | id/name columns of the dimension (track also carries release/artist names) | 45 + ~23 × 2 ≈ 91 |
| AS list (9) | artificialstreamscount, flaggedassetscount, finesamount, eventdate | per dimension (client/organization included) | 27 + (9 + 18) × 2 = 81 |
| AS dashboard (2) | artificialstreamscount, previousartificialstreamscount, artificialstreamschange, artificialstreamschangepct, eventdate (run with previous dates) | clientid/clientname or organizationid/organizationname | 8 + (2 + 4) × 2 = 20 |
| Playlists (5 dims, metric=streams) | 5 metric columns DESC + 11 text/id/date columns ASC+DESC, on every dimension | — | (5 + 22) × 5 = 135 |
| Playlists (5 dims, metric=followers) | followerscount, trackscount | — | 10 |
| Playlist movements (active, 28) | 7 metric DESC + 20 text/id/date ASC+DESC | — | 47 |
| Movement (6 dims) | currentperiodvalue, previousperiodvalue, absolutechange, percentagechange | per dimension id/name(/version) | 24 + 14 × 2 = 52 |

Plus, per endpoint (8 endpoints, ≈ 24 pairs):
- `orderByDescending` omitted must equal `false` (8).
- A column from another dimension, e.g. `trackname` on `dimension=country` (8, status-code parity).
- An unknown column (8, status-code parity).

**Sort-specific classification:**
- **Ties.**
  - Numeric ASC sorts and low-cardinality text columns tie heavily, and `pageSize=5` makes page-boundary membership swaps likely. The `TIE` rule in §3 applies.
  - Additionally, a pair where both sides are internally sorted by the key and differ only in *which* equal-key rows fill the page is classed `TIE-BOUNDARY`.
- **Monotonic check (per side, no prod needed).** Every returned page must actually be ordered by the requested key in the requested direction. A side that returns 200 but is unordered is a `FAIL` even if the other side matches it.
- **NULL ordering.** SingleStore sorts ASC NULLS FIRST / DESC NULLS LAST. If prod is Snowflake-backed for that domain, a pair that differs only in NULL placement is classed `NULLS`, not `FAIL` (you decide in §6 Q4).
- **Known 500s on prod** (E8: `labelname` on Revenue track/release/artist; `deliverytypenname`) and E7 stay expected divergences.

### 5.G dateGranularity (≈ 410)

Every allowed value on every dimension, with `orderByProperty` = the main metric desc (Revenue: `netrevenue`, the default-sort + granularity workaround). Windows: DAILY on W1; WEEKLY / MONTHLY on W2; QUARTERLY / YEARLY on 2026-01-01..2026-08-31.

| Endpoint | Values | Dims | Pairs |
|---|---|---|---|
| Consumption | DAILY, WEEKLY, MONTHLY, QUARTERLY, YEARLY, ALL | 13 streaming + 10 ugc | 138 |
| Engagement | same 6 | 12 + 10 | 132 |
| Revenue | DAILY, MONTHLY, YEARLY, ALL | 9 (YouTube excluded) | 36 |
| AS list / dashboard | MONTHLY, ALL | 9 / 2 | 18 + 4 |
| Playlists | DAILY, WEEKLY, MONTHLY, QUARTERLY, YEARLY, ALL | 5 dims × 2 families (streams, followers) | 60 |
| Playlist movements | 6 values (validated, not applied → must equal no granularity) | active/28 | 6 |
| All 7 endpoints that take it | an unknown value, and lowercase `monthly` (case-insensitivity) | — | 8 + 7 |

Assertion on top of parity: DAILY..YEARLY rows carry one metrics entry per period in range (zero-filled), as the stage contract states. A missing period on either side is a `FAIL`.

### 5.P Paging (≈ 160)

**Per endpoint** (Consumption, Engagement, Revenue, AS, AS dashboard, Playlists, movements, Movement = 8):

| Case | Params | Expectation |
|---|---|---|
| default page size | `pageSize` omitted, on a small dimension (deliveryType / distributor / playlistType) | equal; stage `pageSize` field per N2 |
| `pageSize=1` | | equal |
| `pageSize=2000` | on the same small dimension | equal |
| page 2 | `pageNumber=2&pageSize=5` | equal |
| last page | `pageNumber=ceil(total/5)` | equal, partial page |
| past the end | `pageNumber=last+1` | equal: empty items, same totals and `totalItemsCount` |
| `pageSize=0`, `pageSize=2001` | | status-code parity (stage schema says 1..2000) |
| `pageNumber=0` | | status-code parity |

That's 9 × 8 = 72 pairs.

**Page 2 on every dimension:** Consumption 23, Engagement 22, Revenue 11, AS 11, Playlists 5, movements 3, Movement 6 = 81 pairs.

**Page stitching (per side, 8):** page 1 + page 2 at `pageSize=5` must equal page 1 at `pageSize=10`, on each endpoint's track (or playlist) dimension. This is the one check that catches unstable ordering across pages, which a single-page comparison can't see.

`totals` and `totalItemsCount` must be identical on every page of the same query (asserted per side).

### 5.D Dates (≈ 42)

Per date-taking endpoint (Consumption, Engagement, Revenue, AS, AS dashboard, Playlists = 6), on the track (playlist) dimension:

| Case | Expectation |
|---|---|
| both dates omitted (defaults to today − 3 months .. today) | equal, or `DATA` if the ingestion dates differ (checked against the max date in the rows) |
| only `fromDate` (default = +3 months) | same |
| only `toDate` (default = −3 months) | equal |
| `fromDate == toDate` (single day, 2026-06-15) | equal |
| `fromDate > toDate` | status-code parity |
| malformed date (`2026-13-01`) | status-code parity |
| `onlyNew=true` + `fromDate` | stage 400 by contract; prod result recorded (divergence expected, logged) |

That's 7 × 6 = 42 pairs.

### 5.E Enum and required params (≈ 165)

`targetEnterpriseId` is **not tested** (user decision).

| Param | Endpoints | Cases | Pairs |
|---|---|---|---|
| `dimension` | 7 endpoints | missing, unknown value, upper-case value (`TRACK`) | 21 |
| `mediaType` | Consumption, Engagement | missing, unknown value (both ugc/streaming are in base) | 4 |
| `metric` | Playlists | every value × every dimension (base covers 19 of 25: fill the remaining 6) + missing + unknown | 8 |
| `movementType` × `period` | movements | 9 in base; + unknown type, `period=30`, `period` missing | 3 |
| `metricType` × `periodType` × `movementType` | Movement | full 4 × 5 × 2 cross product on track (40); on each other dimension, every value of each param with the others at base (5 dims × 11 = 55); + missing / unknown for each of the 3 (6) | 101 |
| `previousFromDate` / `previousToDate` | AS dashboard (client, organization) | both set (base comparison), neither, only one (× 2), previous period not before current | 10 |
| case-insensitivity | 8 endpoints | `orderByProperty` in mixed case (`TrackName`) | 8 |

### 5.F Filter matrix (every filter property × every dimension that accepts it)

Filter list = the stage MCP tool schemas (2026-09-30), checked against the prod tool schemas. Rationale for the full matrix: filters are applied in per-dimension SQL, so a filter can work on one dimension and be silently dropped on another. PR #99 was exactly this: `assetTypes` worked on track and was ignored on discoveryType/sourceOfStream.

#### 5.F.1 Seed sources: each filter's values come from its own dimension

Rule: take the W1 base pair of the source dimension (W2 for AS), sorted by the domain's main metric desc, `pageSize=5`. Keep the rows whose id is present on **both** sides. The filter value is `[rank1, rank3]` (two values, so array binding is exercised); use `[rank1]` if fewer than 3 rows qualify. Seeds are written to `scratchpad/parity/seeds.json` and reused by every later chunk.

| Filter | Seed source (domain / dimension → field) |
|---|---|
| trackIds | same domain, `dimension=track` → `metadata.id` (Playlists: movements `active/28` → `trackId`) |
| releaseIds | same domain, `dimension=release` → `metadata.id` (Playlists: movements → `releaseId`) |
| artistIds | same domain, `dimension=artist` → `metadata.id` (Playlists: Consumption `dimension=artist`, since Playlists has no artist dimension) |
| labelIds | same domain, `dimension=label` → `metadata.id` (Playlists: Consumption `dimension=label`) |
| countryIds | same domain, `dimension=country` → `metadata.id` / `countryId` |
| distributorIds | same domain, `dimension=distributor` → `metadata.id` / `distributorId` |
| excludeDistributorIds | same domain, `dimension=distributor` → rank-1 id only, so the exclusion moves the totals |
| excludeTikTok | boolean. Precondition: TikTok is present in the domain's distributor seed, otherwise the pair is marked `VACUOUS` |
| assetTypes (Consumption) | `dimension=assetType` → `track`, `video` (each value tested separately) |
| assetTypes (Revenue) | `dimension=assetType` names: track, video, release, musicalWork, physicalRelease (youTubeChannel / youTubeVideo skipped, user Q6) |
| consumptionTypeIds | `dimension=consumptionType` → `metadata.id` |
| deliveryTypeIds | `dimension=deliveryType` → `metadata.id` |
| recordingVersionTypeIds | `dimension=recordingVersion` → `metadata.id` |
| sourceOfStreamIds | `dimension=sourceOfStream` → `metadata.sourceOfStreamId` |
| discoveryTypeIds | `dimension=discoveryType` → `metadata.discoveryTypeId` |
| uniqueKeys | `dimension=sourceOfStream` → `"{sourceOfStreamId}/{distributorId}"` of rows 1 and 3 |
| channelIds, videoIds | **skipped** (YouTube out of scope, user Q6) |
| payeeIds | **No payee dimension exists.** Read-only SingleStore query (user decision): top 2 `PayeeId` by `SUM(StreamsRevenue + DownloadsRevenue)` for the test enterprise in W1, on the revenue table the DAL resolves for `{{TABLE_NAME}}` (active side). Uses the same pymysql / DAL-connection-string route as the revenue-unknown investigation. The query is run against **both** stage and prod SingleStore, and only payees present in both are used |
| clientIds (AS) | AS `dimension=client` → `metadata.id` |
| organizationIds (AS) | AS `dimension=organization` → `metadata.id`. Tested **only** on `dimension=organization` (user Q7: skip the super-tenant off-dimension case) |
| clientIds (Movement) | Movement `dimension=track` → `metadata.clientId` |
| playlistIds / playlistTypeIds | Playlists `dimension=playlist` → `playlistId`; `dimension=playlistType` → `playlistTypeId` |
| onlyNew | boolean, no seed. Cannot be combined with dates, so it runs undated. Both sides read "today": a diff is re-run once and, if the ingestion clocks differ, classed `DATA` |

UGC seeds are taken separately from the `mediaType=ugc` base rows, because the UGC track/artist population differs from streaming.

#### 5.F.2 Matrix

"General" = the filters every dimension of the domain accepts. Each general filter runs on **every** dimension. Dimension-specific filters run on the dimension(s) they apply to, plus **one ignored-check on track** that asserts both sides return the unfiltered track result.

| Domain | Dimensions | General filters | Dimension-specific filters | Pairs |
|---|---|---|---|---|
| Consumption streaming | 13 | trackIds, releaseIds, artistIds, labelIds, countryIds, distributorIds, excludeDistributorIds, excludeTikTok, assetTypes=track, assetTypes=video (10) | consumptionTypeIds, deliveryTypeIds, recordingVersionTypeIds (own dim); sourceOfStreamIds, discoveryTypeIds (sourceOfStream + discoveryType); uniqueKeys (sourceOfStream, E1) + 6 ignored-checks | 130 + 8 + 6 = 144 |
| Consumption ugc | 10 | same minus assetTypes (8) | consumptionTypeIds, deliveryTypeIds, recordingVersionTypeIds (own dim); assetTypes ignored-check (1) | 80 + 3 + 1 = 84 |
| Consumption onlyNew | track × streaming/ugc | — | — | 2 |
| Engagement streaming | 12 | the 8 id/exclude filters + isSkipRate (9) | consumptionTypeIds, deliveryTypeIds, recordingVersionTypeIds (own dim); sourceOfStreamIds, uniqueKeys (sourceOfStream); discoveryTypeIds (discoveryType, plus sourceOfStream as E9) + 6 ignored-checks | 108 + 7 + 6 = 121 |
| Engagement ugc | 10 | the 8 id/exclude filters | the 3 own-dim filters; isSkipRate ignored-check (1) | 80 + 3 + 1 = 84 |
| Engagement onlyNew | track × streaming/ugc | — | — | 2 |
| Revenue | 9 (prod has no `date`; youTubeChannel / youTubeVideo skipped) | trackIds, releaseIds, artistIds, labelIds, countryIds, distributorIds, excludeDistributorIds, excludeTikTok, payeeIds (9) | assetTypes: 5 values on assetType + track/video on track, release, artist, label, country, distributor, deliveryType (14) + ignored-check on format (1); deliveryTypeIds (deliveryType) + 1 ignored-check; onlyNew on track (1) | 81 + 20 + 2 + 1 = 104 |
| Revenue `dimension=date` | stage only | the 9 general filters | — | 9 (stage `date` totals vs prod `byTrack` totals, same filter) |
| AS list | 9 | trackIds, releaseIds, artistIds, labelIds, countryIds, distributorIds, excludeDistributorIds, excludeTikTok, clientIds (9) | organizationIds on organization only (1); onlyNew on track (1) | 81 + 2 = 83 |
| AS dashboard | 2 (client, organization) | same 9 | organizationIds on organization (1) | 19 |
| Playlists | 5 (playlist, playlistType, distributor, country, date) × 2 SQL families (metric=streams range, metric=followers snapshot) | playlistIds, playlistTypeIds, countryIds, distributorIds, excludeDistributorIds, excludeTikTok, trackIds, releaseIds, artistIds, labelIds (10) | metric=all × 10 filters | 100 + 10 = 110 |
| Playlist movements | 3 movementTypes (period=28) | trackIds, releaseIds, artistIds, labelIds, playlistIds, playlistTypeIds, distributorIds (7) | — | 21 |
| Movement | 6 (rising, monthly, streamChangeAbsolute) | trackIds, releaseIds, artistIds, labelIds, countryIds, distributorIds, excludeDistributorIds, clientIds (8) | — | 48 |
| **Total** | | | | **≈ 830** |

Every §5.F pair uses W1 (W2 for AS), `pageSize=5`, and the base sort for its dimension (Revenue: `orderByProperty=netrevenue desc`).

#### 5.F.3 Extra assertions on filter pairs (on top of stage == prod)

Two sides that both ignore a filter would still pass stage == prod, so each filter pair also checks that the filter actually did something:

| Class | Check |
|---|---|
| `APPLIED` | Include filter on its own dimension: every returned row's id ∈ filter values. On other dimensions: `totals` ≠ the unfiltered base totals, **or** the seed's share is ≥ 99% of the total |
| `EXCLUDED` | Exclude filters: no row carries the excluded id (distributor dim), and totals are strictly below the base totals |
| `IGNORED-BOTH` | Filter should apply but both sides equal the unfiltered base → reported as a finding (a shared bug, not parity) |
| `IGNORED-ONE` | One side equals the unfiltered base, the other is filtered → `FAIL` |
| `VACUOUS` | Both sides empty. Retry with seed ranks 2, 4, 5 before accepting; if all are empty, report it as not covered, **not** a pass |
| `NO-PROD-PARAM` | Stage accepts the filter but the prod tool for that dimension has no such param (resolved in chunk 0 from the prod tool schemas). The prod call is sent without it. The pair is expected equal only for ignored-checks; otherwise it's listed as a contract addition, not a failure |

### 5.X Execution order (chunked, resumable, whole catalogue)

At ~5,900 calls, the catalogue runs as separate chunks, one subagent per chunk. Each chunk is resumable from the saved result files.

0. **Offline setup (no API calls):**
   - prod tool schema dump → the per-dimension `NO-PROD-PARAM` table
   - generate `sorts.json` (§5.S) and the full test list with exact counts
1. **Smoke + base (§5.0):** these double as the seeds. Plus the payee SingleStore query.
2. **Per domain, in this order:** Consumption → Engagement → Revenue → AS → Playlists + movements → Movement.
   - Each domain runs §5.F filters → §5.S sorts → §5.G granularity → §5.P paging → §5.D dates → §5.E enums.
   - Each (domain, section) is its own chunk: 36 chunks, ~50–250 pairs each.

3. **Stage child run (§5.C, 222139):** runs after the parent chunks, or in between them, since it puts no load on prod.

Each chunk reports its counts and non-pass diffs before the next one starts, so you can stop or reprioritise after any chunk.

### 5.C Stage child functional run: 222139 Royal Oakie Records LLC, stage only (≈ 2,370 calls, 0 prod)

**User decision 2026-09-30: no prod calls for any child.** The stage child token `c3cafe0c…` belongs to enterprise 222139, confirmed by the user and by SingleStore: its top tracks have `EnterpriseId` 222139 under tenant 127038. The child run is **stage only**:
- The same catalogue (§5.0, 5.S, 5.G, 5.P, 5.D, 5.E, 5.F) runs with the stage child token.
- Seeds come from the child's own dimension calls.
- Nothing is compared with prod. Every pair becomes a single stage call judged against the **contract**:

| Check | Pass rule |
|---|---|
| Status | 200 for valid params. 400 exactly where the stage contract says so (unknown enum / sort column / granularity, `pageSize` 0 or 2001, one-sided previous dates, `onlyNew` + dates, ugc × sourceOfStream/discoveryType/assetType). Any 5xx = `FAIL` |
| Filters | `APPLIED` / `EXCLUDED` / ignored-check as in §5.F.3, measured against the child's own unfiltered base call. `IGNORED` = `FAIL` |
| Sorts | the page is monotonic in the requested key and direction |
| Granularity | one metrics entry per period in range, zero-filled |
| Paging | page stitching; `totals` / `totalItemsCount` constant across pages; `totalPagesCount == ceil(total/pageSize)` |
| Scope | no row belongs to another enterprise (checked via the child's seeds: every id seen must be in the child's own base lists) |
| Movement | 403 for a client session (contract), on every Movement call. Movement is therefore reduced to 1 call |

The prod token received for 311209 (Lo-fi Records) is **not used**: it has no stage pair, and child prod runs are out of scope.

**Reference: top 10 stage children of parent 127038 with data.** Source: read-only stage SingleStore query, 2026-09-30. Streams from `TRANSACTIONS_BY_TRACK_COUNTRY_DSP`, Jun–Aug 2026; revenue from `USER_STATEMENTS_BY_TRACK_COUNTRY_DSP`, Jan–Aug 2026. All ten have data through 2026-08-31.

| # | EnterpriseId | Name | Streams Jun–Aug | Tracks | DSPs | Revenue Jan–Aug |
|---|---|---|---|---|---|---|
| 1 | 311209 | Lo-fi Records | 328,119,802 | 8,054 | 9 | 2,648,109.00 |
| 2 | 620152 | 1914861 ONTARIO LTD. | 61,492,977 | 2,472 | 9 | 304,301.49 |
| 3 | 284087 | Echoes Blue Music Limited | 52,492,210 | 1,975 | 9 | 277,577.29 |
| 4 | 346189 | Ex Habit | 39,183,950 | 156 | 9 | 129,559.80 |
| 5 | 576827 | Silent Sonic AB | 33,418,358 | 806 | 9 | 205,488.12 |
| 6 | 293694 | Church of Jesus Christ | 20,781,377 | 466 | 9 | 164,493.48 |
| 7 | 222067 | Sinnr | 20,364,329 | 456 | 9 | 155,091.42 |
| 8 | 359226 | Saliva Grey | 20,160,865 | 207 | 8 | 53,850.57 |
| 9 | 829526 | Crunkz | 18,266,553 | 17 | 8 | 55,271.69 |
| 10 | 835529 | Tim Vogt | 16,906,167 | 192 | 9 | 53,120.24 |

## 6. Decisions (user, 2026-09-30)

| Q | Decision |
|---|---|
| 0 Stage version | stage is on 2.0.7 |
| 1 Accounts | Parent 127038: prod `c45031d1…`, stage `f32799a8…` → parity run. Child 222139: stage `c3cafe0c…` → stage-only contract run (§5.C), **no prod calls for children**. The prod 311209 token `5b4e5565…` is saved but unused. Full tokens are kept in `scratchpad/parity/tokens.json`, never in this file |
| 2 Rate limit | unknown; the self-imposed limits in §2, strictest on prod |
| 3 Transcription tie-breaker | OK: every `FAIL`/`FLOAT` pair gets one direct-HTTP re-fetch of both sides (it counts against the prod cap) |
| 4 TIE | does **not** count as a match. Every `TIE`, `TIE-BOUNDARY` and `NULLS` pair is listed individually, with both orderings, for your review |
| 5 E-list | accepted as-is |
| 6 YouTube | skipped: no youTubeChannel / youTubeVideo dimensions, no channelIds / videoIds, no YouTube assetTypes values |
| 7 AS organizationIds | only on `dimension=organization` |
| 8 Throughput | MCP, strictly throttled (§2), chunked (§5.X) |
| 9 Sort directions | metric columns DESC only; text / id / date columns ASC + DESC |
| 10 targetEnterpriseId | not tested |
| — metricsByDate | out of scope |

## 7. Output

- Per domain: pass / TIE / TIE-BOUNDARY / NULLS / FLOAT / FAIL / EXPECTED / VACUOUS counts, plus a diff for every non-pass. A separate **tie review list** holds every TIE-class pair with both orderings side by side.
- Child run: contract-check results per check class (§5.C).
- Prod call count per sitting, and every back-off event.
- `C:\Users\vasil\analytics-parity-2026-09-29.md`: the full results table, findings with stage + prod repro params, and the tokens' first 8 characters.
- Skill "Results history" updated only if you say so.
