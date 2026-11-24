# Setting up Parmana for more than one organization

This document explains how to run Parmana for each organization that uses it, with its own database, keys, policies, approvals and connectors, and how to keep them apart. It is written so that someone with this repository, and access to the deployment, can do the setup without reading any other repository.

Where a detail depends on software in `parmana-core`, this document says so and names the file to check. Where a detail is not yet confirmed, it says that too.

## Principle

**One organization, one deployment, one database.** Each organization runs its own Parmana service, pointed at its own database, with its own keys and its own policy and approval records. Two organizations never share a database, a deployment, a key, an approval log or a policy directory.

The reason is isolation. Parmana's approvals and records are evidence for one organization. If two organizations shared a database, a row written for one could be read or changed in the context of the other, and an approval given by one organization's checker could be used by the other's requests.

### What the tenant feature is, and is not

`parmana-core` has a `metadata.tenantId` field and a per-key `allowedTenantIds` list. A key may act only for the tenant ids it lists, and a request for another tenant is refused with `403 TENANT_NOT_ALLOWED`. Tenants can also have their own signing keys, as an opt-in. ([authentication](https://github.com/pavancharak/parmana-core/blob/main/docs/site/api-reference/authentication.mdx), [cryptography overview](https://github.com/pavancharak/parmana-core/blob/main/docs/site/cryptography/overview.mdx))

Tenant ids separate requests **inside one deployment**. They are not a substitute for separate deployments and separate databases. This document does not rely on tenant ids for isolation between organizations, because the database itself is shared within a deployment.

> **Not confirmed:** whether any table in the Parmana schema is partitioned by tenant id. Check the `parmana-core` migrations before relying on tenant ids for any data separation at the database level. Until that is checked, the rule above stands: one database per organization.

## What is separate for each organization

| Item | Separate per organization? | Where it lives |
| --- | --- | --- |
| Deployment (Vercel project, or container) | **Yes** | The hosting platform |
| Database (PostgreSQL, Supabase project or database) | **Yes** | Its own connection string, `DATABASE_URL` |
| API keys and their `allowedTenantIds` | **Yes** | `PARMANA_API_KEYS`, in that deployment's environment |
| Signing keys for approvals and releases | **Yes** | That deployment's environment and key store |
| Policy directory | **Yes** | `orgs/<org>/policies`, checked out for that deployment |
| Approval log | **Yes** | `orgs/<org>/governance/policy-approvals.jsonl` |
| External connector endpoint | **Yes** | Its own URL and its own credential |
| GitHub App installation | **Yes** | Installed only on that organization's repositories |
| Connector code | **No, shared** | `parmana-connectors`, pinned by version |
| Parmana software | **No, shared** | `parmana-core`, pinned by commit per organization |

## The repository layout

```
orgs/
  README.md                   how this repository is organized, for every organization
  docs/
    multi-org-setup.md        this document
  org-a/
    README.md                 what Org A uses, and its status
    policies/                 Org A's policies, one version per folder
    governance/               Org A's approval log
    connectors/README.md      Org A's connector setup
    database/README.md        Org A's database: instance, schema state, history
  org-b/                      the same structure, for Org B
```

Each organization's folder is self-contained. Nothing in `org-b/` refers to `org-a/`, and the reverse.

## Steps for one new organization

These steps are done once per organization, in order.

### 1. Name the organization and create its folder

Choose a short, lowercase name, such as `org-b`. Copy the layout of an existing organization folder, and replace each README with that organization's details. Do not copy `governance/policy-approvals.jsonl` from another organization. Each organization starts with an empty log.

### 2. Create its database

- Create a new database for the organization, separate from every other organization's. For a Supabase project, that means a new project.
- Record the instance in `org-b/database/README.md`: the engine, the region, and the owner. Never record the connection string.
- Note the connection setting name, `DATABASE_URL`, and the place it is set: the deployment's environment.
- Use the pooled connection string, as the existing Org A database does.

### 3. Pin the Parmana software for the organization

- Choose a commit of `parmana-core` to run. Record it in `org-b/database/README.md`, under the software pin.
- Check out that commit. Install its dependencies with `npm ci`.
- Apply its migrations to the new database, with the commands in `org-a/database/README.md`, adapted for the new organization. Run `status` and `apply --dry-run` first, and read the output.

**Confirm** that `status` shows zero pending before you go on.

### 4. Create the deployment

Create a new deployment for the organization. For a Vercel project, create a separate project. For a container, run a separate container. Do not reuse a deployment or its environment.

Set the environment variables in the table below. Names are the ones used in `parmana-core`. Check the full reference before you rely on a name that is not listed: `docs/site/deployment/environment-variables.mdx` in `parmana-core`.

| Variable | Value for this organization | Notes |
| --- | --- | --- |
| `DATABASE_URL` | The organization's own database | Never shared with another organization |
| `PARMANA_POLICY_DIR` | A checkout of `orgs/<org>/policies` | The service reads it at startup |
| `PARMANA_API_KEYS` | This organization's callers and approvers | A JSON array. Never copy another organization's array. |
| `PARMANA_RELEASE_AUDIENCE` | This deployment's audience, for releases | Confirm the exact name and format in the reference |
| Signing key settings | This organization's own keys | Generate new keys for each organization |
| `GITHUB_APP_CREDENTIAL` | Only if the organization runs the GitHub endpoint | Set on the endpoint's service, not in this repository |
| `ENDPOINT_URL` | Only for an external endpoint | The URL exactly as registered |

Do not write any of these values into this repository. Record only the variable names and where they are set.

### 5. Generate the keys

For each organization:

- **Callers.** One key for each caller that will send requests. Each key is registered as a service caller, unless it is a person.
- **Makers and checkers.** Two human keys, one for the maker and one for the checker, registered as `USER`. The checker also needs a step-up key pair. Keep the checker's private key on the checker's own machine.
- **Signing keys.** Generated for this deployment. They are not shared with another organization.

The runbook steps for generating keys are in `parmana-core`, in `docs/site/deployment/production.mdx`, step 3. The generator is `npm run generate:api-key`.

Record each key holder by name and role in the organization's `README.md`. Never record a key.

### 6. Load the policies

Set `PARMANA_POLICY_DIR` to the organization's policy folder, then run the policy-loading step, `scripts/apply-policies-table-and-backfill.ts`, once, from the pinned checkout.

Each policy then needs an approval before it takes effect. Follow the maker and checker flow in `org-a/policies/README.md` for the drafts, adapted for the organization. Every version starts as a draft. A policy is in effect only after an approval is recorded.

### 7. Confirm the deployment

- `GET /ready` returns `READY` with `authDisabled: false`. If it returns `NOT_READY`, the `reason` names the problem. Check `DATABASE_URL` and the migrations.
- `GET /policies/in-effect?capability=<capability>` returns the policy in effect for that capability, once it is approved. Before approval, it returns `CAPABILITY_NOT_BOUND`.
- Send one request that the policy refuses, and one it approves, and check that each gives the expected answer.

### 8. Register any external endpoint

If the organization runs a connector as an external endpoint:

- Deploy the endpoint at a public HTTPS address, and record that address as `ENDPOINT_URL`.
- Propose the registration through the external connector change API, and have a checker approve it. Do not register an endpoint without that approval.
- The endpoint's own requirements are in `org-a/connectors/README.md`.

### 9. Record the organization as ready

Update the organization's `README.md` and `database/README.md` with the date, the software commit, the deployment URL, and the checks in step 7. Commit the update to `orgs`.

## Keeping organizations apart in day-to-day use

- **Never copy an environment between deployments.** Each deployment's variables are set by hand, for that deployment.
- **Never share a checker's private key** with another organization, or with anyone who is not that checker.
- **Never point a tool at the wrong database.** Before any migration or script, check the `DATABASE_URL` in the shell. The migrate command prints the database it will use. Read that line before you press enter.
- **Keep one approval log per organization.** It lives in that organization's folder, and it is committed there.
- **Rotate keys per deployment.** A rotation in Org A does not touch Org B.

## Upgrading the software for one organization

1. Choose the new `parmana-core` commit, and read its changes.
2. Check the migrations between the old pin and the new one, and apply them to that organization's database only, after a backup.
3. Update the software pin in that organization's `database/README.md`.
4. Redeploy that organization's deployment with the new commit.
5. Repeat for each other organization, separately, on its own schedule.

Do not upgrade every organization at once. A failure in one upgrade should not affect another organization.

## Backups and restores

- Back up each organization's database separately, on its own schedule.
- Test a restore for each organization once, into a separate database, before the first real use.
- Record each restore test in that organization's `database/README.md`.

## What is not confirmed

- **Tenant separation inside one database.** Not confirmed. This document does not rely on it.
- **Per-organization signing keys.** The tenant documentation describes per-tenant signing keys as opt-in. Confirm the setting before relying on it. Until then, use a separate deployment, with its own keys, for each organization.
- **The exact names of every environment variable.** The table above lists the ones this document depends on. The full list is in `parmana-core`, and it should be checked for each new deployment.
- **Whether a single deployment can ever serve two organizations safely.** Not confirmed. This document does not support it.

## Checklist for a new organization

- [ ] Folder created, with the same layout as `org-a/`
- [ ] Database created, separate from all others, and recorded without its connection string
- [ ] Software pinned to a `parmana-core` commit, and recorded
- [ ] Migrations applied, `status` shows zero pending
- [ ] Deployment created, with its own environment variables
- [ ] Keys generated: callers, maker, checker, and signing keys. Holders recorded by name, never the keys.
- [ ] Policies loaded, and each approved through the maker and checker flow
- [ ] `/ready` returns `READY`, and the in-effect check returns the expected policy
- [ ] Test requests checked: one refused, one approved
- [ ] External endpoint deployed and registered, if used
- [ ] Backup taken, and a restore tested
- [ ] Organization README and database README updated, and committed
