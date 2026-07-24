# Revelator Wrap-Up — Session Index

Summaries of dev sessions across Revelator repos.  
Each entry links to a session file with deliverables, decisions, and follow-ups.

---

## netcore-analytics

- [2026-06-29 — WMG Download Orchestrator](sessions/netcore-analytics/2026-06-29-wmg-download-orchestrator/summary.md) — New CronJob orchestrator for Warner Music Group Spotify data via Analytics DAL microservice ([plan](sessions/netcore-analytics/2026-06-29-wmg-download-orchestrator/plan.md))
- [2026-07-24 — ArtificialStreams API DAL/SingleStore migration](sessions/netcore-analytics/2026-07-24-artificial-streams-dal-migration.md) — Migrated all 11 ArtificialStreams endpoints off direct Snowflake stored procs onto the Consumption/Engagement DAL pattern; verified tenant-isolation filter against real Snowflake data; unified dashboard endpoints onto the combined-call pattern, fixing a Totals/Items inconsistency
- [2026-07-24 — ArtificialStreams API staging vs prod testing](sessions/netcore-analytics/2026-07-24-artificial-streams-testing.md) — Postman collection + 160-test regression catalog for the ArtificialStreams DAL migration; found a real `dateGranularity=MONTHLY`+filter item-ordering bug (pre-existing on prod, mostly fixed by the migration) across all 7 entity-dimension endpoints

## netcore-report-ingestion

- [2026-07-01 — WMG Downloader + move to ReportIngestion](sessions/netcore-report-ingestion/2026-07-01-wmg-downloader-reportingestion/summary.md) — Built WMG Spotify downloader (stage→landing via DAL), migrated orchestrator+downloader into the ReportIngestion stack, switched load to COPY INTO, long-running ack handling, configurable 2h timeouts ([move plan](sessions/netcore-report-ingestion/2026-07-01-wmg-downloader-reportingestion/plan.md), [copy-into plan](sessions/netcore-report-ingestion/2026-07-01-wmg-downloader-reportingestion/copy-into-plan.md))
- [2026-07-06 — Apple + SoundCloud WMG downloads](sessions/netcore-report-ingestion/2026-07-06-apple-soundcloud-wmg-download/summary.md) — Generalized the WMG downloader to multi-source (Spotify+Apple+SoundCloud); per-report COPY-as-embedded-SQL, per-report FilePattern for shared dirs, tracking completeness+idempotency guards; diagnosed Apple 6/30–7/3 gap as upstream S3 non-delivery ([plan](sessions/netcore-report-ingestion/2026-07-06-apple-soundcloud-wmg-download/plan.md))
