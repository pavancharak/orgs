# github-pr-read 1.2.0

Authorizes a read of one GitHub pull request, owner/repo#number, through the GitHub connector (github:pr-fetch). Reads never change anything. Every read needs a signed human approval for that exact pull request. The target must name a pull request, and anything else is refused.

| | |
|---|---|
| Capability | `github:pr-fetch` |
| Policy | `github-pr-read` version `1.2.0` |
| Schema | `1.0.0` |
| Status | Draft. Not in effect until the maker and checker flow approves it. |

## What it does

Reads one GitHub pull request through the GitHub connector: its head SHA, whether it is mergeable, and its base branch.

## Signals

| Signal | Type | Source |
|---|---|---|
| `targetRef` | string | Intent path `target` |
| `readApproved` | boolean | Signed approval for the target |

## Rules

Rules are checked in order. The first one that matches decides.

| Rule | Outcome | Summary |
|---|---|---|
| `reject-no-approval` | reject | Refused: no signed approval covers reading this pull request. |
| `approve-approved-pull-request` | approve | Read approved: a signed approval covers this pull request. |
| `reject-otherwise` | reject | Refused: the target is not a pull request reference. |

## Known gaps

None that the policy itself must close. A read changes nothing on GitHub. Its safety rests on the signed approval for the exact pull request.

## Changing this policy

Do not edit `policy.json` in place once it is approved. Add a new version folder instead, and run the maker and checker flow for it. The approval record goes in `org-a/governance/policy-approvals.jsonl`.
