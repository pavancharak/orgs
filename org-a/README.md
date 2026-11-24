# org-a policies

This repository holds Org A's Parmana policies and the signed approval records for them. It is kept apart from the Parmana software (`parmana-core`) and from the connectors (`parmana-connectors`), because the policies and approvals are Org A's own decisions.

## Layout

- `policies/<name>/<version>/policy.json`: one policy version per folder. Set `PARMANA_POLICY_DIR` to a checkout of this repository, and the runbook's policy-loading step reads from it.
- `governance/policy-approvals.jsonl`: the signed approval log. Each entry is written by Parmana when a checker approves a change. Do not write entries by hand.

## Policies

| Policy | Version | Capability | Notes |
| --- | --- | --- | --- |
| `github-pr-read` | 1.2.0 | `github:pr-fetch` | A read needs a signed approval for that pull request. |
| `github-pr-approval` | 1.2.0 | `github:pr-merge` | A merge needs a signed approval, and the safety checks below. Five of its facts are declared by the caller until a GitHub signal source exists. The file's `unboundSignalReasons` record this. |
| `github-pr-comment` | 1.0.0 | `github:comment` | A comment needs a signed approval, and a body of 1 to 4,000 characters. |

None of these is in effect until the maker and checker flow approves it.

## Changing a policy

1. The maker proposes the new version with their own key. The file is sent unchanged as `proposedContent`.
2. A different person, the checker, approves it on their own machine with their step-up key.
3. Parmana records the approval in `governance/policy-approvals.jsonl`.

See `docs/site/guides/policy-governance-maker-checker` in `parmana-core` for the full procedure.
