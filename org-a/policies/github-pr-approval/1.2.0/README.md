# github-pr-approval 1.2.0

Authorizes the merge of one GitHub pull request, owner/repo#number, through the GitHub connector (github:pr-merge). A merge needs a signed human approval for that exact pull request and that exact head commit (the expectedHeadSha). If the head has moved since the approval, the merge is refused. The branch protection, repository authorization, required reviews and status checks are read from GitHub by a signal source, and each must hold. The risk score is declared by the caller and can refuse a merge, never authorize one. Only the merge method is allowed. A merge with any other method is refused. A merge is refused when no head commit is given, when the branch is not protected, the repository is not authorized, required reviews are not complete, a status check is not passing, or the risk score is above 3.

| | |
|---|---|
| Capability | `github:pr-merge` |
| Policy | `github-pr-approval`, version `1.2.0` |
| Schema | `1.0.0` |
| Status | **Draft.** Not in effect until the maker and checker flow approves it. Its signal source must also be registered first. |

## What it covers

Merges one GitHub pull request through the GitHub connector, using the `merge` method only. A merge cannot be undone through the connector.

The approval covers the pull request and the head commit (`expectedHeadSha`). A merge at any other head is refused. The branch protection, repository authorization, reviews and checks are read from GitHub by a signal source, at the same head, and each must hold.

## How a decision is made

1. Parmana reads each signal's value from the request, from the approval, or from a signal source.
2. The rules are checked in order, and the first match decides.
3. If no rule matches, the outcome is reject.

Each signal's trust is in the table below. A sourced fact counts only when the source signs its answer. An untrusted fact can refuse, never authorize.

## Signals

| Signal | Type | Trust | Meaning |
|---|---|---|---|
| `targetRef` | string | Bound | The pull request, `owner/repo#number`. |
| `headSha` | string | Bound | The head commit the merge names (`expectedHeadSha`). The approval covers this exact value. |
| `mergeApproved` | boolean | Approved | A signed approval for that pull request, at that head commit. |
| `repositoryAuthorized` | boolean | Sourced | GitHub signal source: the repository is on Org A's allowed list. |
| `branchProtected` | boolean | Sourced | GitHub signal source: the base branch has branch protection. |
| `requiredReviewsCompleted` | boolean | Sourced | GitHub signal source: the required approving reviews exist at the expected head, and none request changes. |
| `statusChecksPassed` | boolean | Sourced | GitHub signal source: every check run at the expected head passed, and every commit status is success. |
| `riskScore` | number | Untrusted | Declared by the caller. Can only refuse above 3. |

## Rules

Checked in this order. The first match decides.

| # | Rule | Outcome | What it means |
|---|---|---|---|
| 1 | `reject-no-approval` | reject | Refused: no signed approval covers merging this pull request at this head commit. |
| 2 | `reject-no-head-sha` | reject | Refused: the merge does not name the head commit the approver reviewed (expectedHeadSha). |
| 3 | `reject-not-merge-method` | reject | Refused: only the merge method may be used for this policy. |
| 4 | `reject-unsafe-state` | reject | Refused: the branch, reviews, status checks or repository do not meet the merge requirements. |
| 5 | `reject-high-risk` | reject | Refused: the risk score is above 3. |
| 6 | `approve-approved-merge` | approve | Merge approved: a signed approval covers this pull request at this head commit, and GitHub confirms the branch, repository, reviews and checks. |
| 7 | `reject-otherwise` | reject | Refused: the target is not a pull request reference. |

## What the approval covers, and what it does not

- **Covered:** the pull request and the head commit. A merge at any other head is refused.
- **Restricted by a rule:** the merge method. Only `merge` is accepted.
- **Read from GitHub, not from the approval:** branch protection, repository authorization, reviews and checks.
- **Not checked:** the risk score. It is the caller's word, and it can only refuse.

## Known gaps

- The GitHub signal source reads branch protection through the branch protection API. Branch rulesets, which GitHub also offers, are not read. A repository that protects its branch only with rulesets would be reported as unprotected, so the merge is refused.
- The risk score is declared by the caller. Until a real source exists, it can only refuse.
- A merge is irreversible through the connector.

## Decisions

1. **Merge method:** `merge` only. Decided.
2. **Branch protection, repository authorization, reviews and checks:** read from GitHub by a signal source, and each must hold. Decided.
3. **Risk score:** declared by the caller, refuse-only, or dropped. Still to decide. Dropping it is simpler, unless Org A has a real source for it.

## Changing this policy

Do not edit `policy.json` in place once it is approved. Add a new version folder, and run the maker and checker flow for it. Parmana records the approval in its database. To add it to this repository's log, export and copy the files as described in `org-a/governance/README.md`.
