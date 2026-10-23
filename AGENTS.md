# AGENTS.md

## Project

4th-year cloud computing course project: an AWS pipeline built by a student team.

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