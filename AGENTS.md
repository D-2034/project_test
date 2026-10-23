# AGENTS.md

## Project

4th-year cloud computing course project: an AWS pipeline built by a student team.

A mid-sized city’s transit agency runs about 400 buses on 80 routes, from 05:00 until 01:00 the next night. Every bus has an on-board unit that records its position every 30 seconds. Today those records are thrown away at the end of each day.

That has become a problem. Complaints about late and skipped buses have grown this year, and customer service cannot check them. The planning team is starting a review of its routes and wants to see how long each route really takes. Finance has approved a small pilot, and the agency’s IT team, two people who look after the website and the office network, has asked you to design it: a pipeline in the agency’s AWS account that keeps the buses’ records and lets these people answer their questions.

- Each bus can upload one file per trip when it reaches the trip’s final stop: JSON, one record per 30 seconds, about 150 bytes each, with the bus’s id, its trip, its route, its position (latitude and longitude), its next stop and a timestamp. The upload can go to an S3 bucket the pipeline provides; IT gives each on-board unit a credential that can only upload. The unit names each file by service day, bus and trip, in that order (2026-10-04_bus-117_trip-09.json), and you choose the folders in front of that name.
- A bus makes about 20 trips a day, each about an hour long, so a trip file holds about 120 records.
- At a few stops there is no signal. A bus that cannot upload keeps its files and sends them all when it returns to the depot, between 01:00 and 01:30.
- By 01:30 each night, operations has a short list of how many trips each bus ran that day.
- Stored as Parquet, records like these take about a tenth of the space of the JSON.
- Below, a day means one service day, from 05:00 to 01:00, and yesterday means the last one.

AWS services in use: S3, Lambda, EC2, EMR, DynamoDB, API Gateway, and more potentially. 

## AWS environment

- University-managed AWS account shared with other students. Treat it as shared and limited.
- Region: ca-central-1. Always pass `--region` explicitly in CLI commands.
- Our IAM permissions may be restricted by the university. If a command fails with `AccessDenied`, stop and tell us. Do not try to work around it with other roles or policies.
- Budget: $50.00. Prefer serverless and on-demand options. Never create NAT gateways, large EC2/RDS instances, or anything with an hourly cost without asking first.
- account_id: 739563957294
- team_name: team_007

## Repository layout

- `scripts/` – AWS CLI scripts that create, update, and delete resources. One script per service.
- `src/` – application code (Lambda handlers, etc.)
- `tests/` – tests for `src/`
- `docs/` – architecture notes
  
## Commands

- Check who/where you are: `aws sts get-caller-identity`
- Run tests: `<test command>`
- Lint: `<lint command>`
- Create resources: `./scripts/<service>-create.sh`
- Tear down resources: `./scripts/teardown.sh`

## How we manage infrastructure

We use AWS CLI scripts and the console, not infrastructure-as-code.
- Every resource must be created by a script in `scripts/`, never by a one-off command. If something was made in the console, add the equivalent CLI command to a script and note it in `docs/resources.md`.
- Scripts must be safe to re-run: check whether a resource exists before creating it.
- Every create script needs a matching delete step in `scripts/teardown.sh`.
- Use `set -euo pipefail` at the top of bash scripts.

## Naming and tagging
- Prefix every resource name with `<!-- TODO: team prefix, e.g. team07 -->-` so we can find ours in a shared account.
- Tag every resource: `Project=<!-- TODO -->`, `Team=<!-- TODO -->`, `Owner=<student name>`.

## Boundaries
- **Always:** run tests after changing code; use least-privilege IAM policies scoped to our resources; read secrets from environment variables or AWS Secrets Manager / SSM Parameter Store.
- **Ask first:** before running any command that creates, modifies, or deletes AWS resources; before changing IAM policies or roles; before adding a new AWS service.
- **Never:** commit AWS keys, `.env` files, or credentials; make an S3 bucket or other resource public; use `"Action": "*"` or `"Resource": "*"` in IAM policies; touch resources without our team prefix; delete anything not listed in `docs/resources.md`.

## Working with our team
This is graded coursework, and every team member must be able to explain the code.
- Explain *why* when you choose an AWS service or design, in a short comment or in your reply.
- Keep changes small and focused so they are easy to review.
- When you add or change a resource, update `docs/resources.md`.