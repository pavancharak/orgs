# github-pr-comment 1.0.0

Authorizes one comment on a GitHub pull request, owner/repo#number, through the GitHub connector (github:comment). A comment is approved only with a signed human approval for that exact pull request, and a body of 1 to 4,000 characters. Any other case is refused.

| | |
|---|---|
| Capability | `github:comment` |
| Policy | `github-pr-comment` version `1.0.0` |
| Schema | `1.0.0` |
| Status | Draft. Not in effect until the maker and checker flow approves it. |

## What it does

Posts one comment on a GitHub pull request, with a body of 1 to 4,000 characters. The connector adds a marker with the transaction id, so the comment can be found later.

## Signals

| Signal | Type | Source |
|---|---|---|
| `targetRef` | string | Intent path `target` |
| `body` | string | Intent path `parameters.body` |
| `commentApproved` | boolean | Signed approval for the target |

## Rules

Rules are checked in order. The first one that matches decides.

| Rule | Outcome | Summary |
|---|---|---|
| `reject-no-approval` | reject | Refused: no signed approval covers commenting on this pull request. |
| `reject-bad-body` | reject | Refused: the comment body must be 1 to 4,000 characters. |
| `approve-approved-comment` | approve | Comment approved: a signed approval covers this pull request. |
| `reject-otherwise` | reject | Refused: the target is not a pull request reference. |

## Known gaps

A comment cannot be removed through the connector. Its safety rests on the signed approval and the body limit.

## Changing this policy

Do not edit `policy.json` in place once it is approved. Add a new version folder instead, and run the maker and checker flow for it. The approval record goes in `org-a/governance/policy-approvals.jsonl`.
