# Revelator Wrap-Up — Session Index

Summaries of dev sessions across Revelator repos.  
Each entry links to a session file with deliverables, decisions, and follow-ups.

---

## netcore-analytics

- [2026-06-29 — WMG Download Orchestrator](sessions/netcore-analytics/2026-06-29-wmg-download-orchestrator/summary.md) — New CronJob orchestrator for Warner Music Group Spotify data via Analytics DAL microservice ([plan](sessions/netcore-analytics/2026-06-29-wmg-download-orchestrator/plan.md))
- [2026-07-24 — ArtificialStreams API DAL/SingleStore migration](sessions/netcore-analytics/2026-07-24-artificial-streams-dal-migration.md) — Migrated all 11 ArtificialStreams endpoints off direct Snowflake stored procs onto the Consumption/Engagement DAL pattern; verified tenant-isolation filter against real Snowflake data; unified dashboard endpoints onto the combined-call pattern, fixing a Totals/Items inconsistency
- [2026-07-24 — ArtificialStreams API staging vs prod testing](sessions/netcore-analytics/2026-07-24-artificial-streams-testing.md) — Postman collection + 160-test regression catalog for the ArtificialStreams DAL migration; found a real `dateGranularity=MONTHLY`+filter item-ordering bug (pre-existing on prod, mostly fixed by the migration) across all 7 entity-dimension endpoints
- [2026-07-27 — Playlists API staging vs prod testing](sessions/netcore-analytics/2026-07-27-playlists-testing.md) — Postman collection + 274-test regression catalog for the Playlists DAL migration; found 6 findings (F1–F6): Z-suffix dates accepted as-is, phantom-row/swallowed-error/broken-orderBy-whitelist bugs fixed same day and verified with a full end-to-end rerun, 2 pre-existing prod bugs left out of scope

## revelator-etl-datafactory

- [2026-08-07 — db-rev-prod → Snowflake sync inventory](sessions/revelator-etl-datafactory/2026-08-07-db-rev-prod-sync-inventory.md) — Full static inventory of both ADF factories: 25 distinct db-rev-prod base tables synced by 8 active trigger chains, FQ Snowflake targets, 3 ADF_ views resolved, 7-tab Excel deliverable + 6–10 week Airbyte/stored-proc migration estimate

## netcore-report-ingestion

- [2026-07-01 — WMG Downloader + move to ReportIngestion](sessions/netcore-report-ingestion/2026-07-01-wmg-downloader-reportingestion/summary.md) — Built WMG Spotify downloader (stage→landing via DAL), migrated orchestrator+downloader into the ReportIngestion stack, switched load to COPY INTO, long-running ack handling, configurable 2h timeouts ([move plan](sessions/netcore-report-ingestion/2026-07-01-wmg-downloader-reportingestion/plan.md), [copy-into plan](sessions/netcore-report-ingestion/2026-07-01-wmg-downloader-reportingestion/copy-into-plan.md))
- [2026-07-06 — Apple + SoundCloud WMG downloads](sessions/netcore-report-ingestion/2026-07-06-apple-soundcloud-wmg-download/summary.md) — Generalized the WMG downloader to multi-source (Spotify+Apple+SoundCloud); per-report COPY-as-embedded-SQL, per-report FilePattern for shared dirs, tracking completeness+idempotency guards; diagnosed Apple 6/30–7/3 gap as upstream S3 non-delivery ([plan](sessions/netcore-report-ingestion/2026-07-06-apple-soundcloud-wmg-download/plan.md))

## ibuprofen-formulation

- [2026-08-10 — Ibuprofen 200 mg/5 mL f₂ dissolution analysis](sessions/ibuprofen-formulation/2026-08-10-ibuprofen-f2-analysis.md) — Reconstructed the 45-formulation Excel dataset, decoded the f₂ convention (rows 42/43 = f₂ vs each reference batch, 3-pt), found recorded-vs-recalculated discrepancies (best run E3П54 = 49.9/57.3, not 47.8), recommended S-250 + xanthan 0.60 with a 5-run DoE; Word report + charts + dataset in Downloads\Ibuprofen_analysis
