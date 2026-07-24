# ArtificialStreams API — DAL/SingleStore Migration — 2026-07-24

**Repo:** Netcore.Analytics + RepositoryDataAccessor.Analytics
**Branch:** `vasil/artificial-streams-dal` (both repos, pushed to origin)

## What we did

- Analyzed how Consumption and Engagement already work (DAL/SingleStore pattern: SQL built in C# from `.sqlt` templates → GraphQL to the Analytics DAL microservice → `FromSqlRaw` + 24h EF cache) and found ArtificialStreams was the one domain still on direct Snowflake stored procs (`SP_GET_ARTIFICIAL_STREAMS_*`), even though its SingleStore target table (`ARTIFICIAL_STREAMS_BY_TRACK_COUNTRY_DSP`) already existed.
- Migrated all 11 endpoints (9 dimensions + 2 dashboards) to the DAL pattern: new `ArtificialStreamsQueryService` (SQL builder), rewritten `SnowflakeArtificialStreamsProvider` (S2 methods over the DAL client), 11 `.sqlt` templates + shared filters fragment, 11 `.graphql` operation files, 11 DAL-side `Response*` models + resolvers in RepositoryDataAccessor.Analytics.
- Solved the one hard SQL problem: SingleStore has no `COUNT(DISTINCT) OVER ()`, so the grand-total `FlaggedAssetsCount` (ported from the SP's `DISTINCT_ASSETS` CTE) rides in on a separate one-row `_assets` CTE cross-joined into the combined query.
- Deleted the legacy Snowflake path per user decision (not kept behind a TODO): `ReadArtificialStreamsService`, `ArtificialStreamsQueryBuilderService`, their interfaces and DI registrations.
- Verified the tenant-isolation filter assumption (`SuperTenantId = 1` for the super tenant, `TenantEnterpriseId = {id}` for regular tenants) directly against Snowflake, using a higher-privileged credential the user supplied mid-session — confirmed `TENANT_<id>.ARTIFICIAL_STREAMS_BY_TRACK_COUNTRY_DSP` are physical per-tenant tables (not views, despite the SP naming), `TENANT_1` is the union of all tenants (182 distinct `TenantEnterpriseId` values in PROD), and a regular tenant's table is scoped to exactly one `TenantEnterpriseId` with `SuperTenantId` always 1. No code change needed — design was correct as built.
- Unified the 2 dashboard endpoints onto the same single-combined-DAL-call pattern as the other 9 dimensions (`GetS2DashboardDataWithTotals`, window-function totals) after explaining the tradeoff and getting explicit confirmation — this fixed a pre-existing prod inconsistency where, when a caller passed both `dateGranularity=MONTHLY` and `previousFromDate`/`previousToDate`, the Totals object suppressed the previous-period comparison (nulls) while Items showed it, and `TotalItemsCount` was inflated (client×month count instead of distinct client count).
- Built both repos clean (0 errors) and ran the full test suite (349 tests passing) after each change; verified generated SQL for ~10 representative filter scenarios via a scratchpad console app before and after the dashboard unification.

## Key decisions

- **Delete legacy code, not keep-with-TODO** (user's explicit choice) — no Snowflake fallback remains; rollback is git revert.
- **Migrate all 11 endpoints in one pass**, including dashboards, rather than leaving them on Snowflake (user's explicit choice).
- **Unify dashboards to the combined-call pattern**, accepting the resulting behavior change (previous-period comparison + `TotalItemsCount` now consistent between Totals and Items) as a fix rather than preserving the legacy quirk byte-for-byte — user confirmed after the tradeoff was explained.
- Branched from `main` (not `deployed-staging`, which was the checked-out branch) after user correction.

## Files changed

| File | Change |
|---|---|
| `Netcore.Analytics/src/Revelator.Analytics.Infrastructure/Services/ArtificialStreamsQueryService.cs` | New — SQL builder (templates dict, tenant/date/id filters, `_assets` CTE, dashboard combined query) |
| `Netcore.Analytics/src/Revelator.Analytics.Infrastructure/Providers/SnowflakeArtificialStreamsProvider.cs` | Rewritten — S2 methods over `IAnalyticsDataAccessLayerService`, legacy Snowflake CALL methods removed |
| `Netcore.Analytics/src/Revelator.Analytics.Application/QueryHandlers/ArtificialStreams/*.cs` (11 files) | Switched to `GetS2Data*`/`GetS2DashboardDataWithTotals` |
| `Netcore.Analytics/.../DAL/Queries/Sql/streaming/artificialStreams/*.sqlt` (11 + fragment) | New |
| `Netcore.Analytics/.../DAL/Queries/Graphql/ArtificialStreams*.graphql` (11) | New |
| `Netcore.Analytics/src/Revelator.Analytics.Domain/Abstract/ArtificialStreams/{ArtificialStreamsBase,ArtificialStreamsDashboardBase}.cs` | Added `PrimaryKey`/`ItemsCount`/`Tot*` columns |
| `Netcore.Analytics/src/Revelator.Analytics.Domain/Entities/ArtificialStreams/ArtificialStreamsRootResponse.cs` | New |
| `Netcore.Analytics/.../Services/ReadArtificialStreamsService.cs`, `.../Application/Services/ArtificialStreamsQueryBuilderService.cs` + interfaces | Deleted |
| `RepositoryDataAccessor.Analytics/.../ArtificialStreamsAnalytics/Models/Response*.cs` (11) | New — DAL response models |
| `RepositoryDataAccessor.Analytics/.../ArtificialStreamsAnalytics/GraphQLOperations/AnalyticsQuery.cs` | New — 11 resolvers |

## Still to do / follow-up

- A separate same-day testing session ran staging-vs-prod regression against this migration — see [2026-07-24 — ArtificialStreams API staging vs prod testing](2026-07-24-artificial-streams-testing.md). Found one real, mostly pre-existing prod bug (`dateGranularity=MONTHLY` + entity-ID filter produces inconsistent item ordering) that needs a bug ticket; today's dashboard consistency fix was confirmed holding on staging.
- No PR opened yet for either branch (both pushed to origin, not merged).
- Prod cutover still depends on the `PIPE_ARTIFICIAL_STREAMS_BY_TRACK_COUNTRY_DSP_A/_B` + `FLIP_VIEWS` pipeline being live (unverified in this session).
- Playlists remains the one analytics domain still on direct Snowflake — flagged as a candidate for the same migration later, not started.
