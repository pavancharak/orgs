# github-pr-comment 1.0.0

Authorizes one comment on a GitHub pull request, owner/repo#number, through the GitHub connector (github:comment). A comment is approved only with a signed human approval for that exact pull request, and a body of 1 to 4,000 characters. Any other case is refused.

| | |
|---|---|
| Capability | `github:comment` |
| Policy | `github-pr-comment`, version `1.0.0` |
| Schema | `1.0.0` |
| Status | **Draft.** Not in effect until the maker and checker flow approves it. |

## What it covers

Posts one comment on a GitHub pull request, with a body of 1 to 4,000 characters. The connector adds a marker with the transaction id, so the comment can be found later. A comment cannot be removed through the connector.

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
| `body` | string | Bound | the comment text, from the request's `parameters.body`. Its length is checked. |
| `commentApproved` | boolean | Approved | a signed approval for that pull request. |

## Rules

Checked in this order. The first match decides.

| # | Rule | Outcome | What it means |
|---|---|---|---|
| 1 | `reject-no-approval` | reject | Refused: no signed approval covers commenting on this pull request. |
| 2 | `reject-bad-body` | reject | Refused: the comment body must be 1 to 4,000 characters. |
| 3 | `approve-approved-comment` | approve | Comment approved: a signed approval covers this pull request. |
| 4 | `reject-otherwise` | reject | Refused: the target is not a pull request reference. |

## What the approval covers, and what it does not

A signed approval must name the pull request (`target`) and match it exactly. **It does not cover the comment text.** The approval says which pull request may be commented on. It does not say what the comment says. An approver who approves a pull request has not approved a particular body.

## Known gaps

- The comment text is not covered by the approval. A caller can post a different body under the same approval. The approver should expect that, or the policy should be changed to bind the body, for example with a hash of the approved text.
- A comment cannot be removed through the connector, so a wrong comment must be deleted by hand on GitHub.

## Changing this policy

Do not edit `policy.json` in place once it is approved. Add a new version folder, and run the maker and checker flow for it. The approval record goes in `org-a/governance/policy-approvals.jsonl`. Parmana writes that record. Never write it by hand.
