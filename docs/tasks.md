# Tasks

## Part 1 — Ingestion and storage

Breaks `docs/design.md` into small, reviewable steps. Each task lists the
requirements it covers (`docs/requirements.md`) and when it is done.

Tasks marked **AWS** create or change resources in the shared account: per
`AGENTS.md`, ask the team before running them, and add each resource to
`docs/resources.md` and a delete step to `scripts/teardown.sh` in the same
change. Everything else runs locally and costs nothing.

Order: T0 first (it can change the design), then T1–T4 locally, then the
AWS tasks T5–T9.

### T0 — Answer the open questions (no code)

T0.1 — Account and IAM
  Do:      Refresh credentials and run
           `aws sts get-caller-identity --region ca-central-1`. Ask the
           instructor whether we can create IAM users and roles in this
           account (design Open question 1). Do not test by creating one.
  Blocks:  T5, T6, T8.
  Done:    We know whether R4 uses IAM users or the presigned-URL fallback.

T0.2 — Sample trip file
  Do:      Get one real or agreed sample: field names, JSON array vs JSON
           lines, timestamp format and time zone (Open question 2).
           Commit it as `tests/data/sample_trip.json`.
  Blocks:  T3.1.
  Covers:  R2.

T0.3 — Operations' trip list
  Do:      Agree its format and how it reaches `ops/service_day=D/`
           (Open question 3).
  Blocks:  T3.3.
  Covers:  R13.

T0.4 — Retention
  Do:      The brief keeps records 2 years; decide how long to keep `raw/`
           (Open question 4). Record the decision in `design.md`.
  Blocks:  T2 (lifecycle rule).
  Covers:  R7, R8.

T0.5 — Time zone for the schedule
  Do:      Confirm the agency's IANA time zone (Open question 5).
  Blocks:  T8.

T0.6 — Fill in AGENTS.md
  Do:      Replace the naming and tagging TODOs. Use `team007-` as the
           prefix: S3 bucket names cannot contain `_`, so `team_007-` would
           fail. Fill in the test and lint commands (T1).
  Covers:  R15.

T0.7 — Alert recipients
  Do:      Confirm the team accepts SNS as a new service, and who receives
           the failure email (design Open question 6).
  Blocks:  T6a.

### T1 — Repository setup (local)

  Do:      Create `scripts/`, `src/`, `tests/` and `docs/resources.md`.
           Add `scripts/common.sh` with the shared values every script
           sources: region `ca-central-1`, prefix `team007`, bucket name,
           and the `Project`/`Team`/`Owner` tags. Add an empty
           `scripts/teardown.sh` with `set -euo pipefail`. Set up pytest and
           a linter, and put their commands in `AGENTS.md`.
  Covers:  R15, R16.
  Done:    `pytest` runs (no tests yet); every script starts with
           `set -euo pipefail` and sources `common.sh`.

### T2 — Bucket (AWS)

  Do:      `scripts/s3-create.sh` creates `team007-transit-739563957294`
           only if it does not exist, then (re)applies: Block Public Access,
           default SSE-S3 encryption, tags, and the lifecycle rule from
           T0.4. No versioning (design: re-upload must overwrite, R6).
           Teardown step: empty the bucket, then delete it.
  Covers:  R7–R10, R14, R15, R16.
  Done:    Running the script twice changes nothing the second time;
           `get-public-access-block` shows all four settings on.

### T3 — Nightly job code (local)

Code in `src/nightly_convert/`, tests in `tests/`. Uses pyarrow. No AWS
calls in T3.1–T3.3; S3 access is isolated in T3.4.

T3.1 — Key and record validation
  Do:      Parse `raw/service_day=D/D_bus-N_trip-M.json`; reject a name that
           doesn't match or whose folder date differs from its filename
           date. Check each record has the seven fields, and that its bus
           and trip match the filename.
  Covers:  R2, R3, R11.
  Done:    Tests cover a good file, a bad name, a date mismatch, a missing
           field, and a trip ending after midnight (stays in day D).

T3.2 — Parquet writer
  Do:      Build one Arrow table per day with the design's schema
           (`bus_id, trip_id, route_id, lat, lon, next_stop, ts,
           source_key`), sorted by `bus_id, trip_id, ts`; write
           `trips.parquet`. Bad files produce a quarantine note instead.
  Covers:  R2, R9.
  Done:    Round-trip test: records in equal records out; quarantined file
           absent from the Parquet; output ~1/10 the size of the JSON.

T3.3 — Completeness check
  Do:      Count distinct trips per bus and compare with operations' list;
           write `completeness.csv` (`bus_id, trips_received,
           trips_ops_list, difference`) plus the quarantine count.
  Covers:  R6, R13.
  Done:    Tests: one withheld file → that bus is one short; all present →
           no bus reported; same file uploaded twice → counted once.

T3.4 — Handler
  Do:      `handler(event)` takes `{"service_days": [...]}` (default:
           yesterday and the day before). For each day: list `raw/`, fetch
           files **in parallel** (thread pool, ~32 workers), run T3.1–T3.3,
           write outputs once at the end, free memory before the next day.
           Log one summary line per day, not one per file (log cost).
  Covers:  R5, R6, R12.
  Done:    Test against a stubbed S3 (e.g. moto) passes; running twice on
           the same input gives identical outputs.

### T4 — Simulated service day (local)

  Do:      `src/simulate/make_day.py` writes a day of trip files (default
           the full fleet: 400 buses × 20 trips × 120 records) plus a
           matching ops list. Options to add: late files, a duplicate, a
           malformed file, and one withheld file.
  Covers:  R12, and the test data for T9.
  Done:    Generates ~8,000 files, ~144 MB, in under a minute.

### T5 — Lambda execution role (AWS, IAM — ask first)

  Do:      `scripts/iam-create.sh` creates `team007-nightly-convert-role`:
           `s3:ListBucket` on the bucket, `s3:GetObject` on `raw/*` and
           `ops/*`, `s3:PutObject` on `curated/*`, `quarantine/*`,
           `reports/*`, CloudWatch Logs for its own log group only, and
           `sns:Publish` on the alert topic only (T6a; Lambda's on-failure
           destination publishes with the function's role). No `*` actions
           or resources. Depends on T0.1.
  Covers:  R14, R15.
  Done:    Policy reviewed by the team before it is created.

### T6 — Upload-only unit credential (AWS, IAM — ask first)

  Do:      Per T0.1: either an IAM user/policy allowing only
           `s3:PutObject` on `raw/*`, or the presigned-URL fallback in the
           design. Store test credentials outside the repo.
  Covers:  R1, R4.
  Done:    With the unit credential, PUT succeeds and GET, LIST and DELETE
           are denied (R4's Accept line), shown in a script output.

### T6a — Failure alert topic (AWS, new service — ask first)

  Do:      `scripts/sns-create.sh` creates topic `team007-pipeline-alerts`
           if it does not exist, tags it, and subscribes the T0.7 email
           addresses (each must confirm). Teardown deletes the topic and
           its subscriptions. Depends on T0.7.
  Done:    Every subscription shows as confirmed; a test publish arrives
           by email.

### T7 — Deploy the function (AWS)

  Do:      `scripts/lambda-create.sh` zips `src/nightly_convert/`, creates
           `team007-nightly-convert` or updates its code if it exists:
           Python 3.12, 2048 MB, **timeout 600 s** (the Lambda maximum is
           900 s — a larger value is rejected), bucket name as an
           environment variable, tags. pyarrow from a layer (the
           AWS-managed AWS SDK for pandas layer includes it; check its
           current ARN for ca-central-1). Set the async invoke config:
           2 retries and an on-failure destination of the T6a topic.
           Add `scripts/lambda-run-day.sh
           <service_day>` to re-run one day by hand. Teardown deletes the
           function and its log group.
  Covers:  R12, R15, R16.
  Done:    `lambda-run-day.sh` on a small test day writes all outputs.

### T8 — Nightly schedule (AWS)

  Do:      `scripts/scheduler-create.sh` creates schedule
           `team007-nightly-convert` at 01:45 daily in the T0.5 time zone,
           targeting the function, plus the small role the scheduler needs
           to invoke it (IAM — ask first). Teardown deletes both.
  Covers:  R5, R13, R16.
  Done:    Schedule shows the right next run time; a one-off test fires.

### T9 — End-to-end acceptance (AWS)

  Do:      Upload a full simulated day (T4) with the unit credential,
           including late, duplicate, malformed and withheld files. Run the
           job; check every Accept line in R1–R16. Record run time, peak
           memory and the day's cost. Force one failure (e.g. invoke
           asynchronously with an invalid service day) and confirm the
           alert email arrives after the retries. Then run
           `scripts/teardown.sh` and
           confirm nothing with our prefix is left.
  Covers:  R1–R16.
  Done:    Results written to `docs/test-results.md`. If run time is over
           ~5 min, split the schedule into one invocation per day (design).

### T10 — Docs (ongoing)

  Do:      Update `docs/resources.md` with every resource as it is
           created. Replace the cost section in `design.md` with the
           monthly breakdown (it currently omits storage growth). Keep
           `design.html` in sync or add it to `.gitignore`.
  Covers:  R15.
