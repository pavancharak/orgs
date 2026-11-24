# Org A database

This page is the database that Org A's Parmana service uses: what it is, where it runs, which schema version it has, and the rules for changing it.

The schema code is shared Parmana software, and it lives in `parmana-core`, under `supabase/migrations/`. This page does not copy that code. The database instance, its data and its connection settings are Org A's, and they are described here.

Nothing in this folder is a secret. The connection string is set in the environment of the service that uses the database, and its value is never written down here.

## What it is

| Item | Value |
| --- | --- |
| Engine | PostgreSQL |
| Host | Supabase, region `ap-south-1` (Mumbai) |
| Connection setting | `DATABASE_URL`, in the environment of the Parmana service |
| Test database | Separate from production. Set as `TEST_DATABASE_URL` only for running database tests, never for production. |
| Owner | Org A's operator |

## Schema state

Last checked on 2026-11-24, against production:

- **60 migrations applied** before the GitHub work, and **2 more applied that day**, bringing it to **62 applied, 0 pending**.
- The last migration applied is `20261124120000_enable_rls_refusal_record_outbox.sql`. It turns on row-level security for `refusal_record_outbox`, and removes table access from the `anon` and `authenticated` roles.
- Before that, `20261123120000_add_refusal_record_outbox.sql` created the refusal outbox table.

Re-check the state before every change, because it can move.

## The tables that matter for policies

| Table | What it holds | Who writes it |
| --- | --- | --- |
| `pending_policy_changes` | Every policy proposal, its status, and who proposed and resolved it | Parmana, through the maker and checker flow |
| `policy_change_approval_records` | Signed, chained approval records. The latest per policy name decides the version in effect. | Parmana only |
| `policies` | The approved policy content | Parmana, when a change is approved |
| `consumed_policy_change_step_up_nonces` | Step-up authorizations already used | Parmana |
| `consumed_approval_nonces` | Action approvals already used | Parmana |
| `refusal_records` | Every refused request, signed | Parmana |
| `refusal_record_outbox` | Refusal records waiting to be saved | Parmana |

**Nobody writes these rows by hand.** The approval table is the evidence of who approved what. A row written by hand cannot be told apart from a real approval afterwards.

## Checking the schema

From a checkout of `parmana-core`, with `DATABASE_URL` set in the shell (not written to any file in this repository):

```bash
npx tsx scripts/migrate-database.ts status
npx tsx scripts/migrate-database.ts apply --dry-run
```

`status` and `--dry-run` do not write. Read the output before any apply.

## Applying a migration

1. Take a backup of the database first, and confirm it can be restored.
2. Run `status`, then `apply --dry-run`, and read what would change.
3. Run `apply`, then `status` again. It should show zero pending.
4. Record the result in this file: the date, the migrations applied, and the commit of `parmana-core` they came from.

Apply migrations before deploying the code that needs them. The deployment order is written in `parmana-core`'s production runbook.

## Connecting

- The Parmana service reads `DATABASE_URL` from its own environment. Set it there, not in this repository.
- Use the pooled connection string that Supabase gives for the service, not the direct one, unless the operator says otherwise.
- Never paste a connection string into a chat, a commit, a log, or a file in this repository.

## Rules

- **Back up before any change.** Test the restore once, so a backup you cannot use is found before you need it.
- **Never edit the approval tables by hand.**
- **Keep the test database separate.** Tests that write to a database must never point at production.
- **Record every change here.** The date, the migrations, the operator, and the result.

## Status

| Item | Status |
| --- | --- |
| Schema on production | 62 migrations applied, 0 pending, as of 2026-11-24. |
| Row-level security on `refusal_record_outbox` | Enabled, as of 2026-11-24. |
| Backup and restore test | **Not recorded.** Record it before the first real use. |
| Test database | Not recorded here. Set up separately. |
