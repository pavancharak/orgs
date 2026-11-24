# github-pr-approval 1.2.0

Authorizes a merge of one GitHub pull request, owner/repo#number, through the GitHub connector (github:pr-merge). Only a signed human approval for that exact pull request can authorize a merge. The caller-declared safety facts can refuse one, never authorize it. A merge is refused when the branch is not protected, the repository is not authorized, required reviews are not complete, a status check is not passing, or the risk score is above 3. Any other case is refused.

| | |
|---|---|
| Capability | `github:pr-merge` |
| Policy | `github-pr-approval`, version `1.2.0` |
| Schema | `1.0.0` |
| Status | **Draft.** Not in effect until the maker and checker flow approves it. |

**Changed in this version:** an earlier draft let the caller-declared safety facts authorize a merge. That broke the rule that such facts may only refuse, so the approve rule now depends on the signed approval alone.

## What it covers

Merges one GitHub pull request through the GitHub connector, with the merge method `merge`, `squash` or `rebase`, and optionally the head SHA the approver saw. A merge cannot be undone through the connector.

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
| `targetRef` | string | Bound | the target of the request (`target`). |
| `mergeApproved` | boolean | Approved | a signed approval for that pull request. |
| `branchProtected` | boolean | Untrusted | declared by the caller. Can only refuse. |
| `repositoryAuthorized` | boolean | Untrusted | declared by the caller. Can only refuse. |
| `requiredReviewsCompleted` | boolean | Untrusted | declared by the caller. Can only refuse. |
| `statusChecksPassed` | boolean | Untrusted | declared by the caller. Can only refuse. |
| `riskScore` | number | Untrusted | declared by the caller. Can only refuse. |

## Rules

Checked in this order. The first match decides.

| # | Rule | Outcome | What it means |
|---|---|---|---|
| 1 | `reject-no-approval` | reject | Refused: no signed approval covers merging this pull request. |
| 2 | `reject-unsafe-state` | reject | Refused: the branch, reviews, status checks or repository do not meet the merge requirements. |
| 3 | `reject-high-risk` | reject | Refused: the risk score is above 3. |
| 4 | `approve-approved-merge` | approve | Merge approved: a signed approval covers this pull request. The caller-declared checks can refuse a merge, but they cannot authorize one. |
| 5 | `reject-otherwise` | reject | Refused: the target is not a pull request reference. |

## What the approval covers, and what it does not

A signed approval must name the pull request (`target`) and match it exactly. The approval covers merging that pull request. **It does not cover the merge method or the head SHA.** A caller could send a different merge method, or a head SHA the approver never saw, under the same approval. The connector sends the head SHA to GitHub, which refuses the merge if the head has moved, but only when a SHA is given.

## Known gaps

- The five untrusted facts can refuse a merge, but they cannot prove the branch is protected, the reviews are done, or the checks pass. Only a GitHub signal source can prove those. Until one exists, the approver is relying on the caller's word for each of them when a merge is refused, and cannot rely on them when it is allowed.
- The approval does not bind the merge method. Either the approver names the method in the approval, or a rule must check it. The current draft checks neither.
- A merge is irreversible through the connector. Before the first real merge, the approver should confirm the merge method and the head SHA.

## Changing this policy

Do not edit `policy.json` in place once it is approved. Add a new version folder, and run the maker and checker flow for it. The approval record goes in `org-a/governance/policy-approvals.jsonl`. Parmana writes that record. Never write it by hand.
