# orgs

Follow read me strictly

This repository holds each organization's own Parmana decisions: the policies that say what it allows, the signed records of who approved them, and the setup of the connectors it uses.

It is separate from the Parmana software and from the connector code, on purpose.

## Why it is built this way

Parmana is shared software. The same code runs for every organization. What an organization allows is different for each one, so it cannot live in the shared code.

| Repository | Holds | Who changes it |
| --- | --- | --- |
| `parmana-core` | The Parmana service: policy evaluation, approval, release | The Parmana maintainers, through reviewed pull requests |
| `parmana-connectors` | Connector code: how Parmana talks to GitHub, Razorpay and the others | The Parmana maintainers, through reviewed pull requests |
| `orgs` (this repository) | Each organization's policies, approval records and connector setup | That organization's owners |

Three reasons this split matters:

1. **A policy is a decision, not code.** Org A's rule for merging a pull request is Org A's decision. Putting it in `parmana-core` would make it look like a rule for everyone.
2. **Approval records are evidence.** Parmana writes a signed record each time a checker approves a policy. That evidence should sit with the organization it covers, so an audit of Org A looks at one place.
3. **Secrets never go here.** Connector credentials, API keys and private keys live in the environment of the service that runs the connector. This repository only records which connectors are set up, and what each needs before it can release.

## Layout

```
orgs/
  <org>/
    README.md               what the organization uses, and its status
    policies/
      <policy name>/
        <version>/policy.json
    governance/
      policy-approvals.jsonl   signed approval log, written by Parmana, never by hand
    connectors/
      README.md               connector setup and status for this organization
```

Each organization has its own folder, and nothing is shared between folders.

## How to use it

### Running Parmana with an organization's policies

Set `PARMANA_POLICY_DIR` to a checkout of this repository's `<org>/policies` folder. The runbook's policy-loading step (`scripts/apply-policies-table-and-backfill.ts`) reads from that folder into the database.

### Changing a policy

1. Edit or add `policies/<name>/<version>/policy.json`. Never edit a version that has been approved. Add a new version instead.
2. Validate it with the Parmana policy validator, from `parmana-core`.
3. The maker proposes the change with their own key. The file is sent unchanged as `proposedContent`.
4. A different person, the checker, reviews it and approves it on their own machine with their step-up key.
5. Parmana appends the signed record to `governance/policy-approvals.jsonl`. Commit that file here, with the policy change.

The maker and the checker must be different people, each with a `USER` key. See the maker and checker guide in `parmana-core`.

### Adding a connector

1. The connector's code stays in `parmana-connectors`. Do not copy it here.
2. Add a section to `<org>/connectors/README.md`: the capabilities used, the policies they need, the endpoint URL, and the status.
3. Keep the secret out of the repository. Name the environment variable that holds it, and say where that variable is set.

## Why not use one shared policy repository for everyone

Organizations can have different rules, and one organization's approvers should not be able to change another's policies. Separate folders make that boundary visible. A reviewer can see at a glance which organization a change affects.

## What is not here yet

- The Org A policies are drafts. None is approved, and `org-a/governance/` has no records.
- The GitHub endpoint is not deployed or registered.
- Org A's other connectors are not set up.
