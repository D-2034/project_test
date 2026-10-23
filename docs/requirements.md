# Requirements

## Part 1 — Ingestion and storage

Scope: trip files arriving from the buses, how they are stored, and checking
them against the nightly trip list operations already keeps. The
customer-service complaint lookup and planning's route-duration analysis are
later parts.

Source: the project brief (not yet in the repo) and the constraints in
`AGENTS.md`. Where a line goes beyond what the brief says, its Why starts
with "Assumption".

### Ingestion

R1 — Per-trip upload
  Statement: The pipeline accepts one JSON file per bus trip, uploaded by the
             on-board unit to an S3 bucket the pipeline provides when the bus
             reaches the trip's final stop.
  Why:       The brief fixes this: the unit sends one file per trip when it
             reaches the final stop. Each file is small (~18 KB), and a failed
             upload loses only one trip.
  Accept:    A unit can PUT one trip file and the object appears in the bucket.

R2 — Record shape
  Statement: Each file holds one record per 30 seconds of the trip, about 150
             bytes each, with bus id, trip, route, latitude, longitude, next
             stop and timestamp.
  Why:       Fixes the schema every later stage reads. Assumption: the brief
             doesn't say what to do with a malformed file; we keep it but
             don't convert it, so a bad file is visible rather than lost.
  Accept:    A file with these seven fields is converted; one missing a field
             is kept raw and quarantined, not silently converted.

R3 — File naming
  Statement: The pipeline accepts files named by service day, bus and trip in
             that order (`2026-10-04_bus-117_trip-09.json`). The pipeline
             chooses the prefix in front of that name.
  Why:       The unit controls the filename; only the prefix is ours.
  Accept:    A correctly named file lands under the intended prefix.

R4 — Upload-only credentials
  Statement: Each on-board unit uses a credential that can only upload. It
             cannot read, list or delete any object.
  Why:       400 credentials live on vehicles we do not physically control; a
             stolen one must not expose the fleet's history.
  Accept:    With a unit credential, PUT succeeds and GET, LIST and DELETE are
             all denied.

R5 — Late uploads
  Statement: The pipeline accepts files for a service day that arrive between
             01:00 and 01:30, when buses that had no signal return to the
             depot and send everything they buffered.
  Why:       Some stops have no coverage, so a trip's file can arrive hours
             after the trip ended.
  Accept:    A file for a trip that ended at 19:00, uploaded at 01:20, is
             stored and counted in that service day.

R6 — Out-of-order and duplicate arrival
  Statement: Files for one service day may arrive in any order, and the same
             trip file may be uploaded more than once. A re-uploaded file must
             not double-count the trip.
  Why:       Out-of-order follows from R5. Assumption: the brief doesn't
             mention retries, but a unit that retries after a failed upload
             can send a file twice. The service day, bus and trip in the
             filename identify the trip.
  Accept:    Uploading the same trip file twice yields one trip in the counts.

### Storage

R7 — Records are retained
  Statement: Every uploaded trip record is kept. Nothing is discarded at the
             end of the service day.
  Why:       This is the whole point of the pilot — today's records are thrown
             away and nobody can answer questions about them.
  Accept:    Records from a past service day are still readable.

R8 — Raw files kept as uploaded
  Statement: The JSON files are kept as uploaded, separately from any
             converted copy.
  Why:       If conversion has a bug we can re-run it; the raw upload is the
             only original.
  Accept:    A raw file is byte-identical to what the unit sent.

R9 — Parquet copy
  Statement: Trip records are also stored as Parquet, which takes about a
             tenth of the space of the JSON.
  Why:       Columnar storage cuts both storage cost and the amount read per
             query, which matters against a $50 budget.
  Accept:    A service day's Parquet is roughly a tenth the size of its JSON.

R10 — Partitioning by service day
  Statement: Stored records are organised so that reading one service day does
             not read the others.
  Why:       The brief defines "a day" and "yesterday" as service days, and
             the trip list is per service day; scanning every past day to
             answer a question about one is wasted cost.
  Accept:    A query for one service day reads only that day's objects.

R11 — Service day boundary
  Statement: A service day runs 05:00 to 01:00 the next night. A trip
             belongs to the service day in its filename, not to the calendar
             date of its timestamps.
  Why:       Trips after midnight carry the previous service day's date, so
             calendar-date grouping would split a night in two.
  Accept:    A trip ending at 00:40 on 2026-10-05 counts toward service day
             2026-10-04.

R12 — Volume
  Statement: The pipeline handles about 8,000 files and 960,000 records per
             service day — 400 buses x 20 trips x 120 records — roughly 144 MB
             of JSON, about 14 MB as Parquet.
  Why:       Sets the sizing every design choice is measured against.
  Accept:    A full simulated service day is ingested without throttling or
             manual intervention.

### Completeness check

R13 — Check received trips against operations' list
  Statement: After 01:30 each night, the pipeline compares, for each bus, the
             number of distinct trip files received for that service day with
             the number of trips on operations' list, and reports every bus
             where they differ.
  Why:       The brief says operations already has a list of trips per bus
             by 01:30; the pipeline doesn't produce it. It is the only
             independent record of what should have arrived. Without it, a
             missing upload looks the same as a skipped trip.
  Accept:    For a test day with one trip file withheld, that bus is
             reported as one trip short; with all files present, no bus is
             reported.
  Open:      The brief doesn't say how operations' list reaches the
             pipeline (format, delivery) — ask before designing this.

### Constraints

R14 — Nothing public
  Statement: No bucket or other resource in the pipeline is publicly
             accessible, and no credentials are committed to this repository.
  Why:       The data is vehicle position history for a public agency.
  Accept:    Public access is blocked on the bucket; no key appears in git.

R15 — Shared account hygiene
  Statement: Every resource is named with the team prefix, tagged with
             Project, Team and Owner, created by a script in `scripts/`, and
             has a matching delete step in `scripts/teardown.sh`.
  Why:       The AWS account is shared with other students, and $50 has to
             cover the whole pilot.
  Accept:    `scripts/teardown.sh` removes everything the create scripts made.

R16 — Region and cost
  Statement: All resources are created in `ca-central-1` and are serverless or
             on-demand. Nothing with a standing hourly cost is created.
  Why:       Budget is $50 for the pilot; the agency's IT team is two people.
  Accept:    No resource outside `ca-central-1`; no NAT gateway, no
             always-on instance.
