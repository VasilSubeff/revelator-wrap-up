# ArtificialStreams API — Staging vs Prod Regression Testing — 2026-07-24

**Repo:** Netcore.Analytics (testing target; branch `vasil/artificial-streams-dal`, uncommitted DAL migration from an earlier session)
**Branch:** n/a — this was a testing session, no code changes

## What we did

- Built a Postman collection (`C:\Users\vasil\artificial-streams.postman_collection.json`) from 18 supplied prod curls across the `/artificialStreams` endpoints, mirrored as prod/staging request pairs (38 requests total after adding the missing plain `byOrganization` endpoint, which the original curls didn't cover).
- Wrote a 160-test regression catalog (`C:\Users\vasil\artificial-streams-test-cases.md`): core sort/filter/pagination regression per dimension, validation/negative-path, dashboard-specific (incl. today's Totals/Items consistency fix), authorization boundaries, and filter edge cases.
- Wrote and ran a Python regression script (scratchpad: `artificial_streams_regression.py`) against staging (`http://127.0.0.1:8050`, local DAL-migration branch instance) vs prod (`https://api.revelator.com`) — 126 of 160 tests executed; org-token-dependent tests skipped at user's request.
- Manually investigated all 24 raw ❌ results rather than reporting them as-is: 16 were false positives from the script itself (missing `sort_field=eventdate` on default-sort tests, where `eventDate` is null on every item so any order is a valid tie), 1 was a mis-flagged tie-break (byRelease page-2 boundary, large block of items tied at count=1), and 7 are a real finding.
- **Real finding:** `dateGranularity=MONTHLY` + an entity-ID filter (trackIds/releaseIds/artistIds/labelIds/countryIds/distributorIds/clientIds) returns matching totals and per-item values but a *different item order* between environments, across all 7 entity-dimension endpoints. Deep-verified with fresh matched fetches: staging returns the correct descending order in 6/7 cases; prod is correct in only 1/7 (byLabel); byCountry is wrong identically on both sides. Prod's own ordering for this combo was also observed to change across separate identical calls (byRelease) — consistent with a missing `ORDER BY` on the underlying per-item monthly-breakdown query rather than a deliberate (if wrong) sort rule.
- Verified today's dashboard bug fix (Totals/Items consistency when `dateGranularity=MONTHLY` + `previousFromDate`/`previousToDate` are combined) holds on staging.
- Documented curl pairs + explanations for the ordering issue, one dimension at a time, in `C:\Users\vasil\artificial-streams-ordering-bug-curls.md` for manual review.

## Key decisions

- Deferred all execution until the user confirmed the staging data refresh was done and supplied a staging URL + token — did not run anything speculatively per the user's explicit "don't execute yet" instruction earlier in the session.
- Skipped organization/super-tenant-token-dependent tests (byOrganization, dashboard/byOrganization, related auth-boundary cases) at the user's explicit request rather than guessing at a suitable token.
- Treated the 37 raw ⚠️ tie-break results as expected non-determinism, matching the skill's already-documented Revenue API tie-break pattern, rather than regressions — verified by confirming the sort-field value is genuinely equal at every position where order diverged.
- Corrected course mid-session: an initial pass concluded "neither environment sorts this combo correctly"; a clean re-fetch with matched timing (after a token refresh) showed staging is actually correct in most cases and prod carries the pre-existing bug — reported the corrected finding rather than the first impression.

## Files changed

| File | Change |
|---|---|
| `C:\Users\vasil\artificial-streams.postman_collection.json` | New — Postman collection, prod/staging pairs for all 11 known endpoints |
| `C:\Users\vasil\artificial-streams-test-cases.md` | New — 160-test regression catalog + results narrative |
| `C:\Users\vasil\artificial-streams-ordering-bug-curls.md` | New — curl pairs + explanation for the MONTHLY+filter ordering issue, all 7 dimensions |
| scratchpad: `artificial_streams_regression.py` | New — regression runner script (known gap: default-sort tests don't pass `sort_field=eventdate`, causing false-positive FAILs) |
| scratchpad: `artificial_streams_results.json` | New — raw results from the 126-test run |

## Still to do / follow-up

- File a bug ticket for the `dateGranularity=MONTHLY` + entity-filter ordering issue; recommend an explicit `ORDER BY` on the per-item monthly-breakdown query, and investigate why `byCountry` diverges from the other 6 dimensions.
- Run the remaining ~34 tests once an org/super-tenant token pair is available for both environments (byOrganization, dashboard/byOrganization, the AZ auth-boundary suite, FX-03).
- VA-09/VA-10 (artist-scoped and label-scoped filter rejection) still need suitably restricted test tokens.
- DC-07 (`artificialStreamsChangePct` null when previous period = 0) was inconclusive — didn't find a qualifying item in the sample scanned; needs a larger page or a known seed entity.
- Fix the regression script's `sort_field` gap before reusing it, to avoid the 16 false-positive misclassifications seen this run.
- The underlying `vasil/artificial-streams-dal` migration (Netcore.Analytics + RepositoryDataAccessor.Analytics) is still uncommitted on its branch — this testing session didn't touch that.
