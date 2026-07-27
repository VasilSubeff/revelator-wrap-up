# Playlists API — Staging vs Prod Regression Testing — 2026-07-27

**Repo:** Netcore.Analytics (testing target; branch `vasil/playlists-dal`, DAL/SingleStore migration in progress, code not touched this session)
**Branch:** n/a — testing session only, no code changes made by the assistant

## What we did

- Verified the `PlaylistsAnalytics` DAL scaffolding exists (uncommitted) in `RepositoryDataAccessor.Analytics` on `vasil/playlists-dal`, mirroring the ArtificialStreams migration pattern.
- Explored `PlaylistsController.cs` (2 endpoints: `byMetadata`, `movements`), their filter DTOs, allowed-value lists, orderBy field maps, and query-service SQL builders in `Netcore.Analytics` to build an exhaustive test matrix.
- Built a Postman collection (`C:\Users\vasil\playlists.postman_collection.json`, 277 prod/stage request pairs) and a 274-test + 6-equivalence-check regression catalog (`C:\Users\vasil\playlists-test-cases.md`): full metric×metadata cross (5×5), movementType×period cross (3×3), all dateGranularity values, every ID filter with ≥3 distinct value sets (single-top/multi/mid-seed) per user's exhaustiveness requirement, full orderByProperty sweep (16 byMetadata + 28 movements fields ×ASC/DESC), pagination incl. the 2000 cap, and validation/negative paths.
- Wrote Python regression scripts (scratchpad: `playlists_common.py`, `playlists_regression.py`, `playlists_sort_suite.py`, `playlists_reverify.py`, `playlists_seed.py`) and ran them against staging (`http://127.0.0.1:8050`) vs prod (`https://api.revelator.com`).
- Manually root-caused all 108 raw ❌ from the first run rather than reporting them as-is — found six distinct findings (F1–F6), documented with stage/prod curl pairs in `C:\Users\vasil\playlists-findings-curls.md`:
  - **F1** — staging DAL serializes datetimes with a `Z` suffix (`Kind.Utc` round-trip through the DAL's GraphQL hop, no Kind normalization anywhere), legacy path doesn't. Same standing behavior as every other DAL-migrated domain (Consumption S2 dims already do this in prod). **User decision: accept as-is**, note as PR behavior change, and always compare dates as parsed values in regression tooling — not raw strings.
  - **F2** — staging returned a phantom all-null item (`totalItemsCount: 1`) instead of an empty list whenever the filtered/family result set was empty. **Fixed and verified same day** — no side effects on re-run.
  - **F3** — `onlyNew=true` rendered `AND EventDate = ''` in the daily-family snapshot filter (`SnapshotDateFilter`), causing a live-reproduced SingleStore `Invalid DATE/TIME in type conversion` leaf error, swallowed into zeroed totals or (for the Followers-primary-query case) a multi-minute stall then an empty 200. **Fixed same day** by anchoring the null-ToDate case to the table-wide `MAX(EventDate)`. **Accepted caveat (user decision):** since streams data loads roughly a day ahead of the followers/tracks snapshot, that anchor date usually has no daily rows, so `onlyNew` still returns zero followers/tracks most days — left as-is intentionally rather than anchoring per-family.
  - **F4** — `SnowflakePlaylistsProvider`'s three DB-call methods caught all exceptions and returned empty/zero results, silently masking real DB failures as "no data" (caused several flaky-looking transient failures during the run). **Fixed same day** — all three now `throw;` after logging; verified both endpoints' happy paths still return normal 200s.
  - **F5** — `onlyNew=true` returns HTTP 500 on prod for any metric — a pre-existing legacy Snowflake-path bug, out of scope for this migration (staging incidentally "fixes" the streams family, per F3's caveat).
  - **F6** — movements `orderByProperty=playlistscount` was whitelisted in `OrderByMovementsFieldMapping` with no matching column in the query — prod 500s, staging previously swallowed to an empty 200 (F4). **Fixed same day** by removing it from the whitelist — staging now returns a clean 400; prod unchanged (legacy bug, out of scope).
- After F2/F3/F4/F6 landed on staging, reran the **entire** 274+test suite + sort sweep end-to-end (not just targeted retests) specifically to catch side effects. Result: one transient (`releaseIds` filter briefly 401'd on stage mid-run, reproduced clean on immediate retest — unrelated to the fixes), otherwise **zero unexplained failures** and no regressions introduced by any of the four fixes.
- Updated the `revelator-analytics-testing` skill with a full Playlists section (endpoints, allowed values, known-behavior notes, both pre-fix and post-fix results-history rows) so future sessions have the baseline.

## Key decisions

- **F1 (Z-suffix dates) accepted as-is** — root cause is systemic to the DAL migration pattern (HotChocolate DateTime scalar + Kind.Utc round-trip), not a playlists-specific bug; fixing it here wouldn't fix the other already-shipped DAL domains. Regression tooling must parse dates rather than string-compare — codified in the skill as a standing rule for *all* analytics suites, not just playlists.
- **F3's anchor-date caveat accepted as-is** — the user chose the simpler table-wide `MAX(EventDate)` fix over per-family anchoring, accepting that `onlyNew` won't reliably return followers/tracks data given the normal 1-day ingestion lag between streams and the daily snapshot.
- **F5 and F6's prod side left out of scope** — both are pre-existing legacy bugs that predate this migration; not the migration's responsibility to fix unless the user separately decides to patch prod.
- **Full end-to-end rerun requested and performed after the fixes**, rather than trusting targeted retests alone — user explicitly asked "is there anything else left for fixing" and opted for the full rerun to catch any side effects. None were found.

## Files changed

| File | Change |
|---|---|
| `C:\Users\vasil\playlists.postman_collection.json` | New — Postman collection, prod/stage pairs for both endpoints, 277 request groups |
| `C:\Users\vasil\playlists-test-cases.md` | New — 274-test + 6-equivalence catalog, pre-fix and post-fix results narratives |
| `C:\Users\vasil\playlists-findings-curls.md` | New — F1–F6 findings with stage/prod curl pairs and resolution status |
| `C:\Users\vasil\.claude\skills\revelator-analytics-testing\SKILL.md` | Added Playlists API section (endpoints, allowed values, known bugs/fixes, results history) |
| scratchpad: `playlists_common.py`, `playlists_regression.py`, `playlists_sort_suite.py`, `playlists_reverify.py`, `playlists_seed.py`, `playlists_gen_artifacts.py` | New — seeding, comparator, suite runners, normalizing re-verify pass |
| `Netcore.Analytics` / `RepositoryDataAccessor.Analytics` source | **Not modified by the assistant** — F2/F3/F4/F6 fixes were applied by the user directly; this session only diagnosed, root-caused, and verified them via live HTTP/SQL probes |

## Still to do / follow-up

- F5 (prod `onlyNew` 500) and F6's prod side (`playlistscount` sort 500) remain unpatched on prod — pre-existing legacy bugs, decide separately whether to fix.
- No PR opened yet for `vasil/playlists-dal` (still in progress as of this session).
- Consider whether F1's Z-suffix behavior should eventually be normalized across *all* DAL-migrated domains (Consumption, Playlists, ArtificialStreams) in one pass, rather than accepted piecemeal per migration.
