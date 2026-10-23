# Design

## Part 1 — Ingestion and storage

Covers R1–R16 in `docs/requirements.md`. Items marked **Open** need an answer
before we build; they are collected at the end.

### Sketch

```
 400 on-board units                        team007-transit-739563957294 (S3, ca-central-1)
 (upload-only credential)                  ┌─────────────────────────────────────────────┐
   │  PUT one file per trip                │ raw/service_day=D/<name>.json   (as sent)   │
   └──────────────────────────────────────►│ ops/service_day=D/trip_counts.csv (from ops)│
     05:00–01:00 at final stop,            │                                             │
     01:00–01:30 from depot                │ curated/trips/service_day=D/trips.parquet   │
                                           │ quarantine/service_day=D/<name>.json.txt    │
                                           │ reports/service_day=D/completeness.csv      │
                                           └──────────────▲──────────────┬───────────────┘
                                                          │ write        │ read raw/ + ops/
                                                          │              ▼
   EventBridge Scheduler ── 01:45 daily ──────►  Lambda team007-nightly-convert
                                                  (Python + pyarrow layer)
                                                  for D = yesterday and D-1:
                                                    1. list raw/service_day=D/
                                                    2. validate name + 7 fields
                                                    3. bad file → quarantine note
                                                    4. good files → one Parquet
                                                    5. trips per bus vs ops list
                                                  on failure, after Lambda's 2 retries:
                                                    └──► SNS team007-pipeline-alerts
                                                           └──► email to the team
```

One bucket, one Lambda function, one schedule, one alert topic. Nothing runs
between nightly jobs, so nothing costs anything while idle.

**Why a nightly batch, not a Lambda per upload.** Converting each file as it
arrives would give 8,000 tiny Parquet files a day (about 2 KB each) that later
queries would have to open one by one, and we'd still need a nightly step to
compact them and run the completeness check. One job after the upload window
closes does both, writes one ~14 MB file per day, and is one function to
explain and debug. The cost is that a day's Parquet exists only after 01:45;
every question in the brief so far is about "yesterday", so that is enough.

**Why 01:45.** The depot window closes at 01:30 (R5); 15 minutes of margin for
uploads still in flight.

**Why also re-run D-1.** A file that arrives after 01:45 (a slow depot upload,
a unit retrying) would otherwise never be converted. Re-processing the
previous day each night picks up stragglers one day late, for about one extra
minute of Lambda time. Any older day can be re-run by hand
(`scripts/lambda-run-day.sh <service_day>`).

**Rejected:** EMR or EC2 (hourly cost, far more than 144 MB a day needs —
R16); AWS Glue jobs (billed in 1-minute DPU blocks, more to configure, no gain
at this size); DynamoDB for trip counts (only needed if counts must be live
during the day, which the brief doesn't ask for).

### Data layout

Bucket: `team007-transit-739563957294` — team prefix for the shared account,
account id because bucket names are global.

| Prefix | Written by | Contents | Kept |
|---|---|---|---|
| `raw/service_day=YYYY-MM-DD/` | on-board units | Trip files exactly as uploaded | Forever (R7, R8) — **Open** |
| `ops/service_day=YYYY-MM-DD/` | operations | Their trips-per-bus list | Forever |
| `curated/trips/service_day=YYYY-MM-DD/` | Lambda | `trips.parquet`, one per day, overwritten on re-run | Forever |
| `quarantine/service_day=YYYY-MM-DD/` | Lambda | One `.txt` note per rejected file, saying why | Forever |
| `reports/service_day=YYYY-MM-DD/` | Lambda | `completeness.csv` | Forever |

**Raw keys.** A unit uploads to
`raw/service_day=2026-10-04/2026-10-04_bus-117_trip-09.json`. The unit
builds the folder from the date that is already the first part of its
filename, so it needs no extra knowledge. The job rejects a file whose folder
date and filename date differ.

**Why partition by service day.** Every question so far is about one day
(R10), and `service_day=…` is the Hive-style naming that Athena, DuckDB and
Spark all recognise as a partition, so Part 2 can query one day without
reading the rest. Partitioning raw the same way lets the job list one day with
one `ListObjectsV2` call, and lets us expire or re-run a day by prefix.

**Why not partition by bus or route too.** A day is ~14 MB of Parquet;
splitting it further only adds small files.

**Parquet schema** (`trips.parquet`), sorted by `bus_id, trip_id, ts`:

| Column | Type | From |
|---|---|---|
| `bus_id` | string | record (checked against filename) |
| `trip_id` | string | record (checked against filename) |
| `route_id` | string | record |
| `lat` | double | record |
| `lon` | double | record |
| `next_stop` | string | record |
| `ts` | timestamp (UTC) | record |
| `source_key` | string | S3 key of the raw file, for tracing a row back |

`service_day` is not a column; it comes from the folder (R11). Sorting keeps
each trip's rows together, which also compresses better.

**Open:** the brief doesn't give the JSON field names, whether a file is a
JSON array or one object per line, or the timestamp format and time zone. We
need one sample file.

**Completeness report** (`completeness.csv`): one row per bus —
`bus_id, trips_received, trips_ops_list, difference` — plus the number of
quarantined files. A bus with `difference ≠ 0` is a missing upload or an
uncounted trip (R13).

### How the requirements are met

| R | Property of the design |
|---|---|
| R1, R3 | Units PUT to `raw/service_day=D/<name>`; we fix the prefix, the unit fixes the name. |
| R2 | Job checks the name pattern, folder/filename date match, and the seven fields; a failure writes a quarantine note, the raw file stays where it is and is left out of the Parquet. |
| R4 | Unit credential allows only `s3:PutObject` on `arn:aws:s3:::team007-transit-739563957294/raw/*`. No Get, List or Delete; can't touch `curated/`, `ops/` or `reports/`. **Open** below. |
| R5 | Files can land at any time; the job doesn't run until 01:45. |
| R6 | One S3 key per (service day, bus, trip): a re-upload overwrites the same object, so it can't count twice. Order doesn't matter because the job reads the whole day at once. |
| R7, R8 | Nothing in `raw/` is changed, moved or deleted by the pipeline. |
| R9 | `curated/` holds the Parquet copy, ~14 MB per day vs ~144 MB of JSON. |
| R10 | Both `raw/` and `curated/` are partitioned by `service_day=`. |
| R11 | Service day is taken from the folder/filename, never from the timestamps. |
| R12 | 8,000 GETs and 960k rows per run: about a minute with parallel reads, with 2 GB memory and a 10-minute timeout (Lambda max is 15). |
| R13 | `reports/…/completeness.csv` compares received trips against `ops/`. |
| R14 | Block Public Access on; default SSE-S3 encryption; unit credentials kept out of the repo. |
| R15 | Every resource named `team007-…` and tagged, one create script per service, steps in `teardown.sh`. |
| R16 | ca-central-1; S3, Lambda, EventBridge Scheduler and SNS are all pay-per-use. |

**Other properties**

- **Re-runnable.** The job's output for a day depends only on what is in
  `raw/` and `ops/` for that day, and it overwrites its own outputs. Running
  it twice gives the same result, so fixing a bug means re-running, not
  repairing.
- **Failure handling.** EventBridge Scheduler invokes the function
  asynchronously, so once Lambda accepts the call the scheduler's job is
  done; a failure inside the function (an error or the 600 s timeout) is
  retried by Lambda itself, twice. If the last retry also fails, the
  function's on-failure destination publishes the event (including the
  error) to the SNS topic `team007-pipeline-alerts`, which emails the team.
  The raw files are untouched, so the fix is to correct the cause and run
  `scripts/lambda-run-day.sh <service_day>`; otherwise the next night's D-1
  re-run catches up a day late.

  **Why an alert.** The error is also in CloudWatch Logs, but nobody reads
  logs at 02:00; without an alert the operations director finds out at
  06:00, when yesterday's data is missing. **Why an on-failure destination,
  not a CloudWatch alarm.** It fires once per failed night, after the
  retries, and carries the error message; an alarm on the `Errors` metric
  would also fire on retries that later succeed, and costs $0.10 a month.
  **Not covered:** if the schedule itself never invokes the function (it
  is disabled, or its role is broken), no failure event exists, so no email
  is sent. The completeness report's date shows this; a scheduler
  dead-letter queue would close the gap but adds SQS for an unlikely case.
- **Cost at full fleet** (estimated, ca-central-1 prices):
  8,000 PUTs/day ≈ $0.045/day; storage ≈ 160 MB/day added ≈ $0.11/month
  more each month; one ~1-minute Lambda run per day ≈ free tier; SNS
  alerts free (first 1,000 emails a month are free, and we expect almost
  none).
  About **$1.50 for a 30-day month**. For the pilot we'll simulate a few days,
  costing cents.
- **No versioning.** A re-upload replaces the earlier copy (which is what
  makes R6 work). Turning versioning on would keep every retry and make
  teardown slower, for no requirement we have.

### Open questions

1. **Can we create IAM users and roles?** R4 needs an upload-only credential,
   and the Lambda function needs a role scoped to this bucket. The course
   KB says AWS Academy Learner Lab can't create users or roles (only the
   shared `LabRole`). If that's our account, R4 needs a different design:
   an API Gateway endpoint with one API key per unit, backed by a Lambda that
   returns a presigned PUT URL for that bus's file only. Ask the instructor
   before building; IAM has no dry-run, and trying to create a user to find
   out would itself be an IAM change.
2. **One sample trip file:** field names, array vs JSON lines, timestamp
   format and time zone.
3. **Operations' list:** format and how it reaches `ops/`.
4. **Retention:** how long to keep raw and curated data (R7).
5. **Agency time zone** for the 01:45 schedule (EventBridge Scheduler takes
   an IANA zone, e.g. `America/Toronto`).
6. **Who gets the failure email**, and is SNS allowed in our account?
   SNS is a new service for this project, so the team confirms it before
   it is created (`AGENTS.md`). Each address must click a confirmation
   link before it receives anything.
