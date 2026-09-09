# Amazon Data Pulse → S3 → Snowflake (Snowpipe) setup — 2026-08-31

**Repo:** none (AWS console + Snowflake worksheets; DDL destined for `revelator-analytics-snowflake`)
**Branch:** n/a

## What we did

- Designed and partially executed the full pipeline: Lake Formation (Data Pulse share) → Athena `UNLOAD` (Parquet/Snappy) → own S3 buckets → storage integration → Snowpipe → Snowflake.
- **S3**: two buckets created — `amazon-data-pulse-stage` and `amazon-data-pulse-prod` (us-east-1, SSE-S3, Block Public Access all-on). Prefix convention: `<table>/report_date=YYYY-MM-DD/run=<timestamp>/` — `run=` segment because UNLOAD requires an empty target and Snowpipe dedupes by filename (reloads must produce new names).
- **Manual UNLOAD** executed for `daily_play_events`, `report_date=2026-08-10`, landed at `run=2026-08-31T15-00/` in the stage bucket.
- **Snowflake integration**: `AWS_DATAPULSE` storage integration; AWS IAM role `snowflake-datapulse-reader` with read-only policy on both buckets; trust policy narrowed from Snowflake's account `root` to the exact integration IAM user (from `DESC INTEGRATION` → `STORAGE_AWS_IAM_USER_ARN`) plus the `sts:ExternalId` condition.
- **Snowflake objects** in `REVELATOR_ALL.DEPLOYMENT`: `PARQUET_FF` file format, stages `AWS_DATA_PULSE_STAGE` / `AWS_DATA_PULSE_PROD` (one per bucket, shared integration).
- **Test load** into `REVELATOR_ALL.TEST.AMAZON_PULSE_PARQUET_LOAD` (36 all-VARCHAR Data Pulse columns + `LOAD_DATETIME`, `FILENAME`): both COPY variants written — `MATCH_BY_COLUMN_NAME=CASE_INSENSITIVE` + `INCLUDE_METADATA`, and positional transformation COPY (`$1:field::VARCHAR`, case-sensitive field names, Athena lowercases).
- **Phase 6 DDL delivered** (not yet confirmed executed): per-env tables `AMAZON_PULSE_DAILY_PLAY_EVENTS_STAGE/_PROD`, pipes `..._STAGE_PIPE`/`..._PROD_PIPE` with explicit `FILE_FORMAT = (FORMAT_NAME = 'REVELATOR_ALL.DEPLOYMENT.PARQUET_FF')`, S3 event notifications (prefix `daily_play_events/`, `s3:ObjectCreated:*`, both buckets → the single shared `notification_channel` SQS ARN from `SHOW PIPES`).
- Read the Music Central PDF (`Notification Music Central.pdf`): Amazon Music publishes table-update notifications via SNS topic `arn:aws:sns:us-east-1:767398096717:T…` to a subscriber SQS/HTTP/email — this is the authoritative "data ready" trigger for automation.

## Key decisions

- One storage integration + one IAM role covering both buckets; per-env separation at stage/pipe/table level. `ALTER` (never `CREATE OR REPLACE`) the integration — replace rotates the external ID and silently breaks the trust policy.
- SSE-S3 over SSE-KMS on buckets: KMS would require key grants for both Athena (write) and Snowflake (read).
- Pipes use `MATCH_BY_COLUMN_NAME` (schema-drift tolerant), not the positional transform; file format explicit in pipe DDL rather than inherited from stage.
- Manual reload procedure: re-UNLOAD to fresh `run=` prefix (never overwrite files — 14-day filename dedup skips them), delete old rows via `FILENAME` lineage column.
- Automation trigger = Music Central SQS notification (Standard queue, SSE-SQS, not FIFO/KMS) → Lambda → parameterized UNLOAD ×2 (stage + prod get identical data, keeps stage a parity env for regression testing).
- Completeness options ranked: delivery-log table in the share (if it exists) > Music Central notifications > count-stability polling; trailing-window re-pulls + dedup task handle restatements regardless.

## Files changed

None in tracked repos — all work in AWS console / Snowflake. Phase 5–6 DDL still needs committing to `revelator-analytics-snowflake` (suggested: `REVELATOR_ALL/DEPLOYMENT/AMAZON_DATAPULSE/`).

## Still to do / follow-up

- Confirm test COPY row count vs Athena `count(*)` for 2026-08-10.
- Execute Phase 6 (tables, pipes, event notifications) on both buckets; test via re-UNLOAD to new `run=` prefix; `SYSTEM$PIPE_STATUS` + `COPY_HISTORY` checks.
- Phase 7 automation: `amazon-datapulse-availability` SQS queue + DLQ per the Music Central PDF (get full SNS topic ARN — truncated in the PDF), send ARN to Amazon Music team, confirm subscription, inspect real payload schema, then Lambda `amazon-datapulse-export`.
- Dedup task keyed on `run=` parsed from `FILENAME`; lifecycle rule on buckets (~90d); commit all DDL to the snowflake repo.
- Side note from the PDF banner: Music Central now has self-service fraud reports (Royalty Manager role, Royalty tab) — possibly relevant to artificial-streams work.
