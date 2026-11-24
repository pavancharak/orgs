# github-pr-approval 1.2.0

Authorizes a merge of one GitHub pull request, owner/repo#number, through the GitHub connector (github:pr-merge). A merge is approved only with a signed human approval for that exact pull request, the branch protected, the repository authorized, required reviews completed, status checks passing, and a risk score of 3 or less. Any other case is refused.

| | |
|---|---|
| Capability | `github:pr-merge` |
| Policy | `github-pr-approval` version `1.2.0` |
| Schema | `1.0.0` |
| Status | Draft. Not in effect until the maker and checker flow approves it. |

## What it does

Merges one GitHub pull request through the GitHub connector, with merge method `merge`, `squash` or `rebase`, and optionally the head SHA the approver saw.

## Signals

| Signal | Type | Source |
|---|---|---|
| `targetRef` | string | Intent path `target` |
| `mergeApproved` | boolean | Signed approval for the target |
| `branchProtected` | boolean | Declared by the caller (see note) |
| `repositoryAuthorized` | boolean | Declared by the caller (see note) |
| `requiredReviewsCompleted` | boolean | Declared by the caller (see note) |
| `statusChecksPassed` | boolean | Declared by the caller (see note) |
| `riskScore` | number | Declared by the caller (see note) |

## Rules

Rules are checked in order. The first one that matches decides.

| Rule | Outcome | Summary |
|---|---|---|
| `reject-no-approval` | reject | Refused: no signed approval covers merging this pull request. |
| `reject-unsafe-state` | reject | Refused: the branch, reviews, status checks or repository do not meet the merge requirements. |
| `reject-high-risk` | reject | Refused: the risk score is above 3. |
| `approve-approved-merge` | approve | Merge approved: a signed approval covers this pull request and every safety check passed. |
| `reject-otherwise` | reject | Refused: the target is not a pull request reference. |

## Known gaps

Five of the signals (`branchProtected`, `repositoryAuthorized`, `requiredReviewsCompleted`, `statusChecksPassed`, `riskScore`) are declared by the caller, not read from GitHub. A caller can declare them true when they are not. Until a GitHub signal source exists, the signed approval is the control that matters. A merge is irreversible through the connector.

## Changing this policy

Do not edit `policy.json` in place once it is approved. Add a new version folder instead, and run the maker and checker flow for it. The approval record goes in `org-a/governance/policy-approvals.jsonl`.
