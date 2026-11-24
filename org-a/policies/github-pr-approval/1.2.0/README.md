# github-pr-approval 1.2.0

Authorizes the merge of one GitHub pull request, owner/repo#number, through the GitHub connector (github:pr-merge). A merge needs a signed human approval for that exact pull request and that exact head commit (the expectedHeadSha). If the head has moved since the approval, the merge is refused. The caller-declared safety facts can refuse a merge, never authorize one. A merge is refused when no head commit is given, when the branch is not protected, the repository is not authorized, required reviews are not complete, a status check is not passing, or the risk score is above 3.

| | |
|---|---|
| Capability | `github:pr-merge` |
| Policy | `github-pr-approval`, version `1.2.0` |
| Schema | `1.0.0` |
| Status | **Draft.** Not in effect until the maker and checker flow approves it. |

## What it covers

Merges one GitHub pull request through the GitHub connector, with the merge method `merge`, `squash` or `rebase`. A merge cannot be undone through the connector.

The approval covers the pull request **and the head commit** (`expectedHeadSha`). If the branch has moved since the approver reviewed it, the merge is refused. This is what stops a merge of code the approver never saw.

The approval does **not** cover the merge method. The rules decide which methods are allowed, and the drafts allow all three. Decide this before approval; see the open decisions below.

## How a decision is made

1. Parmana reads the request and sets each signal's value. Each value comes from one of three places, shown in the signals table below.
2. The rules are checked in the order listed. The first rule whose condition matches decides the outcome, and the rules after it are not checked.
3. The outcome is `approve` or `reject`. Anything that doesn't match an approve rule is rejected, and the last rule catches every case the others didn't.

This draft uses the rule format where each rule approves or rejects. Parmana's newer format (policy version 2) checks rules in four named tiers, in this order: refuse, review, approve, reject. A rule in that format must name its tier. These drafts don't use it yet.

## How each signal is trusted

Parmana gives every value a provenance, which says where it may come from:

| Provenance | Meaning | Can it authorize? |
| --- | --- | --- |
| **approved** | A person signed an approval. Parmana checks that the signature verifies and that the approval matches the resource exactly. | Yes. This is the only way a policy here authorizes anything. |
| **bound** (`boundSignals`) | The value is read from the request itself, such as the target. The request is the same one Parmana executes, so the value cannot change between the decision and the action. | Only as a check on what was asked. It proves nothing about the world. |
| **sourced** | A registered business system established the value, and signed it. | Not used here. No GitHub signal source exists yet. |
| **untrusted** (caller-declared) | The caller says the value is true or false. Nothing checks it. | It can refuse. It can never authorize. Parmana's rule is that a caller-declared fact can refuse, never authorize, and every such fact must have a reason in `unboundSignalReasons`. |

Trust comes from the signature, not from who sent the value. An unsigned value is untrusted, whoever sent it.

## Signals

| Signal | Type | Trust | Meaning |
|---|---|---|---|
| `targetRef` | string | Bound | The target of the request, `owner/repo#number`. |
| `headSha` | string | Bound | The head commit the merge names (`expectedHeadSha`). The approval covers this exact value. |
| `mergeApproved` | boolean | Approved | A signed approval for that pull request, at that head commit. |
| `branchProtected` | boolean | Untrusted | Declared by the caller. Can only refuse. |
| `repositoryAuthorized` | boolean | Untrusted | Declared by the caller. Can only refuse. |
| `requiredReviewsCompleted` | boolean | Untrusted | Declared by the caller. Can only refuse. |
| `statusChecksPassed` | boolean | Untrusted | Declared by the caller. Can only refuse. |
| `riskScore` | number | Untrusted | Declared by the caller. Can only refuse above 3. |

## Rules

Checked in this order. The first match decides.

| # | Rule | Outcome | What it means |
|---|---|---|---|
| 1 | `reject-no-approval` | reject | Refused: no signed approval covers merging this pull request at this head commit. |
| 2 | `reject-no-head-sha` | reject | Refused: the merge does not name the head commit the approver reviewed (expectedHeadSha). |
| 3 | `reject-unsafe-state` | reject | Refused: the branch, reviews, status checks or repository do not meet the merge requirements. |
| 4 | `reject-high-risk` | reject | Refused: the risk score is above 3. |
| 5 | `approve-approved-merge` | approve | Merge approved: a signed approval covers this pull request at this head commit. |
| 6 | `reject-otherwise` | reject | Refused: the target is not a pull request reference. |

## What the approval covers, and what it does not

- **Covered:** the pull request (`target`) and the head commit (`headSha`). A merge at any other head is refused.
- **Not covered:** the merge method. Any allowed method is accepted under the approval.
- **Not covered:** the caller-declared safety facts. They can refuse a merge, but a true value does not prove anything about GitHub.

## Known gaps

- The five caller-declared facts (`branchProtected`, `repositoryAuthorized`, `requiredReviewsCompleted`, `statusChecksPassed`, `riskScore`) are not checked against GitHub. They only refuse. Until a GitHub signal source exists, a caller can declare these facts true to avoid a refusal it would otherwise get. The facts can never produce an approval on their own, so the approver's signed review of the head commit remains the control that authorizes a merge.
- A merge is irreversible through the connector.

## Open decisions before approval

1. **Which merge methods are allowed.** The drafts allow `merge`, `squash` and `rebase`. Decide whether to allow only `squash`, for example.
2. **Whether to require a GitHub signal source** for the five facts before approval, or to accept them as refuse-only.

## Changing this policy

Do not edit `policy.json` in place once it is approved. Add a new version folder, and run the maker and checker flow for it. Parmana records the approval in its database. To add it to this repository's log, export and copy the files as described in `org-a/governance/README.md`.
