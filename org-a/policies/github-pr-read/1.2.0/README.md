# github-pr-read 1.2.0

Authorizes a read of one GitHub pull request, owner/repo#number, through the GitHub connector (github:pr-fetch). Reads never change anything. Every read needs a signed human approval for that exact pull request. The target must name a pull request, and anything else is refused.

| | |
|---|---|
| Capability | `github:pr-fetch` |
| Policy | `github-pr-read`, version `1.2.0` |
| Schema | `1.0.0` |
| Status | **Draft.** Not in effect until the maker and checker flow approves it. |

## What it covers

Reads one GitHub pull request through the GitHub connector: its head SHA, whether it is mergeable, and its base branch. A read changes nothing on GitHub.

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
| `readApproved` | boolean | Approved | a signed approval for that pull request. |

## Rules

Checked in this order. The first match decides.

| # | Rule | Outcome | What it means |
|---|---|---|---|
| 1 | `reject-no-approval` | reject | Refused: no signed approval covers reading this pull request. |
| 2 | `approve-approved-pull-request` | approve | Read approved: a signed approval covers this pull request. |
| 3 | `reject-otherwise` | reject | Refused: the target is not a pull request reference. |

## What the approval covers, and what it does not

A signed approval must name the pull request (`target`) and match it exactly. The approval covers reading that one pull request. It covers nothing else, because a read has no other parameters.

## Known gaps

- A read can still reveal information. The approval is what decides who may see a pull request's state, so the approver must know which pull requests are sensitive.
- The target must look like `owner/repo#number`. A target that does not is rejected by the last rule, and the rule does not check that the repository exists.

## Changing this policy

Do not edit `policy.json` in place once it is approved. Add a new version folder, and run the maker and checker flow for it. The approval record goes in `org-a/governance/policy-approvals.jsonl`. Parmana writes that record. Never write it by hand.
