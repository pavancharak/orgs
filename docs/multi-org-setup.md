# Setting up Parmana for more than one organization

This document explains how to run Parmana for each organization: with its own database, keys, policies, approvals and connectors, kept apart from every other organization. It is written so that an operator can set up an organization from this repository, and from the pinned Parmana commit it names.

Each step names its source. Where a detail is not confirmed, the document says so, and lists it under "Not confirmed" at the end.

## Principle

**One organization, one deployment, one database.** Each organization runs its own Parmana service, pointed at its own database, with its own keys and its own policy and approval records. Two organizations never share a database, a deployment, a key, an approval log or a policy directory.

Why: a Parmana approval is evidence for one organization. If two organizations shared a database, one organization's records could be read or changed in the other's context.

### Tenant ids are not a boundary between organizations

Parmana has a `metadata.tenantId` field, and each API key lists the tenant ids it may act for (`allowedTenantIds`). A request for another tenant is refused with `403 TENANT_NOT_ALLOWED`. Tenants can also have their own signing keys, as an opt-in. ([authentication](https://github.com/pavancharak/parmana-core/blob/main/docs/site/api-reference/authentication.mdx), [cryptography overview](https://github.com/pavancharak/parmana-core/blob/main/docs/site/cryptography/overview.mdx))

Tenant ids separate requests within one deployment. They do not separate one organization's database rows from another's, and this document does not rely on them for that.

## What is separate, and what is shared

| Item | Per organization? | Where |
| --- | --- | --- |
| Deployment (a Vercel project or a container) | **Yes** | The hosting platform |
| Database (PostgreSQL) | **Yes** | `DATABASE_URL` in that deployment's environment |
| API keys (`PARMANA_API_KEYS`) | **Yes** | That deployment's environment |
| Signing keys (`PARMANA_KEY_DIR` or `PARMANA_KEY_MATERIAL_JSON`) | **Yes** | That deployment's environment or secret store |
| Policy directory (`PARMANA_POLICY_DIR`) | **Yes** | `orgs/<org>/policies` |
| Approval log | **Yes** | `orgs/<org>/governance/` |
| Checker list (`POLICY_CHANGE_CHECKERS`) | **Yes** | That deployment's environment |
| External connector endpoint | **Yes** | Its own URL, key and credential |
| GitHub App | **Yes** | Installed only on that organization's repositories |
| Connector code (`parmana-connectors`) | No, shared | Pinned by commit per organization |
| Parmana software (`parmana-core`) | No, shared | Pinned by commit per organization |

## Repository layout

```
orgs/
  README.md                   how this repository is organized
  docs/multi-org-setup.md     this document
  org-a/
    README.md                 what Org A uses, and its status
    policies/                 Org A's policies, one version per folder, each with a README
    governance/               Org A's approval log and public keys
    connectors/README.md      Org A's connector setup
    database/README.md        Org A's database: instance, schema state, history
  org-b/                      the same structure, for Org B
```

Each organization's folder is self-contained. Nothing in `org-b/` refers to `org-a/`, or the reverse.

## Steps for a new organization

### 1. Create the folder

Choose a short lowercase name, such as `org-b`. Copy the layout of `org-a/`, and replace each README with that organization's details. Do not copy `governance/` from another organization. Each organization starts with its own empty log.

### 2. Create the database

- Create a new PostgreSQL database for the organization, separate from every other. For Supabase, that is a new project.
- Record the instance in `org-b/database/README.md`: the engine, region and owner. Never record the connection string.
- Set `PARMANA_STORAGE=postgres` and `DATABASE_URL` in the deployment (step 4).

### 3. Pin the Parmana software, and apply the schema

- Choose a commit of `parmana-core` on `main`. Record it in `org-b/database/README.md`, under the software pin.
- From a checkout at that commit, run `npm ci`.
- Set `DATABASE_URL` in the shell, not in a file. Run `npx tsx scripts/migrate-database.ts status`, then `apply --dry-run`, and read the output. Then run `apply`, and `status` again.
- The command is `npm run db:migrate -- status` and so on, as in `org-a/database/README.md`.

**Confirm** that `status` shows zero pending before the next step.

### 4. Create the deployment and set its environment

Create a separate deployment. Do not reuse one, and do not copy another organization's environment.

Set these variables. Names are taken from `docs/site/deployment/environment-variables.mdx` in `parmana-core` at the pinned commit. Read that file for any variable not listed here.

| Variable | Value | Notes |
| --- | --- | --- |
| `NODE_ENV` | `production` | |
| `PARMANA_STORAGE` | `postgres` | |
| `DATABASE_URL` | This organization's database | Secret |
| `PARMANA_API_KEYS` | A JSON array of this organization's callers and approvers | Required. The server refuses to start if it is empty. Each entry has a 64-character `keyHash`. |
| `PARMANA_POLICY_DIR` | A checkout of `orgs/<org>/policies` | The policies are loaded from here |
| `POLICY_CHANGE_CHECKERS` | A JSON object of policy name to checker caller ids | **Required before any policy can be approved.** Unset means no one may approve. Example: `{"github-pr-approval":["checker-1"]}`. |
| `KEY_PROVIDER` | `local` or `aws-kms` | Default `local` |
| `PARMANA_KEY_DIR` | The directory with `default.private.pem` and `default.public.pem` | For `local`. Or use `PARMANA_KEY_MATERIAL_JSON`. |
| `PARMANA_SIGNING_KEY_ID` | The key id, such as `default` | |
| `PARMANA_AUTH_DISABLED` | **Unset, or `false`** | Must not be `true` in production |

Two more settings are needed only in specific cases:

- `PARMANA_PUBLIC_KEY` and `PARMANA_PUBLIC_KEY_FILE` are the connectors' settings for checking Parmana's releases. They are set on the connector service, not on Parmana (step 8).
- `PARMANA_DB_SCHEMA` is unset, unless the database is shared with another Parmana system on purpose.

**Record** the variable names and where each is set, in the organization's README. Never record a value.

### 5. Generate the keys

For each person, and each caller that will send requests, generate a key. The generator is `npm run generate:api-key`, from the pinned checkout.

- **A caller (service):** `npm run generate:api-key -- --caller-id <name> --allowed-capabilities <capability>`. Each capability it lists is one it may request.
- **A maker or a checker (a person):** `npm run generate:api-key -- --caller-id <name> --credential-holder-type USER`. The key is registered as a human, which the maker and checker rules require.
- **A checker's step-up key pair:** add `--generate-step-up-key`. The step-up private key is shown once. The checker keeps it on their own machine.

The generator prints the raw key once, and an entry for `PARMANA_API_KEYS`. Give the raw key to its holder. Put the entry into `PARMANA_API_KEYS` in the deployment. Never put a raw key in this repository, and never reuse a key across organizations.

For `POLICY_CHANGE_CHECKERS`, list the checker's caller id under each policy that checker may approve.

### 6. Load and approve the policies

1. Set `PARMANA_POLICY_DIR` to the organization's policy folder, and run the loader once from the pinned checkout: `npx tsx scripts/apply-policies-table-and-backfill.ts`. It copies each policy in that folder into the database.
2. Each policy version then needs an approval before it takes effect. Use the maker and checker flow in `org-a/policies/README.md`, which gives the exact steps. Every version starts as a draft.

A policy that is not approved does not take effect, and the check at the start of each request refuses it.

### 7. Confirm the deployment

- `GET /ready` returns `{"status":"READY","authDisabled":false}`. Anything else names the problem in its `reason`.
- `GET /policies/in-effect?capability=<capability>`, with a caller's key, returns the policy in effect. Before a policy is approved, it returns `CAPABILITY_NOT_BOUND`.
- Send one request the policy refuses, and one it approves, and check each result.

### 8. Set up any external connector endpoint

For a connector that runs as an external endpoint, such as the GitHub endpoint:

- Set its settings with the same names the connectors use: `PARMANA_PUBLIC_KEY` or `PARMANA_PUBLIC_KEY_FILE` (never both), `PARMANA_RELEASE_AUDIENCE` (an https URL), and its own credential. See `org-a/connectors/README.md`.
- Deploy the endpoint at a public HTTPS address, and use that address as `PARMANA_RELEASE_AUDIENCE`.
- Register the endpoint with Parmana, one capability at a time. The request, the approval, and the step-up needed are in `org-a/connectors/README.md`. Do not register an endpoint without the checker's approval.

### 9. Record the organization as ready

Update the organization's `README.md` and `database/README.md` with the date, the software commit, the deployment, and the results of step 7. Commit to `orgs`.

## Day-to-day rules

- **Never copy an environment between deployments.** Set each one by hand.
- **Never share a checker's step-up private key,** with another organization or with anyone other than that checker.
- **Check the database before running anything.** The migrate command prints the database it will use. Read that line.
- **Keep one approval log per organization,** in its own folder, committed there.
- **Rotate keys per deployment.** A rotation in Org A does not touch Org B.

## Upgrading the software for one organization

1. Choose the new `parmana-core` commit, and read its changes.
2. Back up that organization's database. Check the migrations between the old pin and the new one. Apply them to that database only.
3. Update the software pin in that organization's `database/README.md`.
4. Redeploy that organization's deployment at the new commit.
5. Do each organization separately, on its own schedule.

## Backups

- Back up each organization's database on its own schedule.
- Test a restore for each organization once, into a separate database, before the first real use. Record the test in that organization's `database/README.md`.

## What the code shows

These were checked against `parmana-core` on `main`:

- **The database does not separate tenants.** No migration in `supabase/migrations/` has a tenant column. Every organization's data must therefore be in its own database. This document depends on that.
- **Per-tenant signing keys are not a switch.** The runtime always has a tenant key resolver. A request for a tenant uses that tenant's own key only when one has been provisioned for it. Otherwise the default key signs. Provisioning is described in `packages/runtime/src/TenantKeyResolver.ts`. This document does not rely on tenant keys for separation between organizations.
- **Registration responses** are as described in `org-a/connectors/README.md`.

## Still to confirm before a deployment

- **The checks Parmana makes on an endpoint address at approval.** The rules are in the external connector code. Read them before registering an endpoint in production.
- **Whether one deployment can ever serve two organizations safely.** This document does not support it.

## Checklist for a new organization

- [ ] Folder created, with the layout of `org-a/`
- [ ] Database created, separate from every other, recorded without its connection string
- [ ] Software pinned to a `parmana-core` commit, and recorded
- [ ] Migrations applied, `status` shows zero pending
- [ ] Deployment created, with its own variables, including `PARMANA_STORAGE`, `PARMANA_API_KEYS` and `POLICY_CHANGE_CHECKERS`
- [ ] Keys generated: callers, maker, checker, and the checker's step-up key. Holders recorded by name, never the keys.
- [ ] Policies loaded, and each approved through the maker and checker flow
- [ ] `/ready` returns `READY`, and the in-effect check returns the expected policy
- [ ] One refused request and one approved request checked
- [ ] External endpoint deployed and registered per capability, if used
- [ ] Backup taken, and a restore tested
- [ ] Organization README and database README updated, and committed
