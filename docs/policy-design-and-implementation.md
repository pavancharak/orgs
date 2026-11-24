# Designing and implementing a Parmana policy

This guide explains how a policy is designed, how each part works, and how it is implemented and approved. It applies to every policy in this repository. Use it with the template in `docs/policy-template/`.

## What a policy decides

A policy decides whether one kind of request for a capability is approved or rejected. A capability is a named action, such as `github:pr-merge`. A policy is identified by a name and a version, for example `github-pr-approval` 1.2.0.

A policy never acts on its own. It only decides. The connector performs the action, and only after the policy approves it.

## The parts of a policy

| Part | What it holds | Why it matters |
| --- | --- | --- |
| Signals | The values a rule reads, such as `targetRef` or `headSha` | A rule can only check what a signal says, so every signal has to be defined and trusted |
| Signal sources | Where a signal's value comes from, when it isn't in the request | A sourced value is checked against the source's signature |
| Approval signal | The one signal that a person's signed approval sets | Every approve rule must require it |
| Rules | Conditions, checked in order, each with an outcome | The first rule that matches decides |
| Unbound reasons | A written reason for each signal not bound to the request or sourced | Keeps every weaker signal visible and justified |

## How a decision is made

1. **The version must be in effect.** A request naming another version is refused. A request cannot pick its own policy version.
2. **Each signal gets a value.** From the request (bound), from a signed approval (approved), from a signal source (sourced), or from the caller (untrusted).
3. **Rules are checked in order, and the first match decides.** The rest are not checked.
4. **If nothing matches, the outcome is reject.** A final catch-all rule makes this explicit.

## Trust levels

| Trust | Where the value comes from | Can it authorize? |
| --- | --- | --- |
| Bound | The request itself, such as the target | Only as a check on what was asked |
| Approved | A person's signed approval, matched to the exact resource | Yes. Every approve rule requires it. |
| Sourced | A registered source, which signs its answer | Yes, once the source is registered and its answer verifies |
| Untrusted | The caller's word | No. It can refuse, never authorize. |

A value's trust decides what it may do in a rule. An untrusted fact can appear only in rules that refuse or reject.

## Tiers

Parmana's newer policy format checks rules in four named tiers, in this order: refuse, review, approve, reject. The older format used here checks rules in order, and each rule approves or rejects. To design a policy, decide each rule's tier first. The rule then maps onto the older format directly.

- **Refuse:** a request that must never happen, whatever else is true. Example: a merge with no head commit named.
- **Review:** a request that needs a person. In the older format, the approve tier's requirement for a signed approval is the review.
- **Approve:** a request that is allowed, when a signed approval supports it and the sourced facts hold.
- **Reject:** the default for anything else.

## Designing a policy

Work through these steps in order.

1. **Name the decision.** Write one sentence: "This policy decides whether ___ may ___ for ___."
2. **List the facts the decision depends on.** For each one, say where the value should come from, using the trust table above. Prefer a sourced fact over an untrusted one wherever a source can exist.
3. **Write the refusals first.** Each refusal is a condition that must never be allowed. Put them before any approval.
4. **Decide what needs a person.** The approve rule requires the signed approval, and names the resource it covers, such as the exact pull request and head commit.
5. **Write the approve rule.** It requires the approval and every sourced fact that the decision depends on.
6. **Write the catch-all reject.** A final rule that rejects everything else.
7. **Write the reasons.** Every unbound signal needs a reason. Every outcome needs a message an operator can act on.
8. **Check the scope.** Say what the approval covers and what it does not. An approval covers the resource it names, not the other parameters. Say which ones are left open.
9. **Decide the version.** A new policy starts at a new version. An approved version is never edited.

## Implementing a policy

### Files

```
policies/<policy name>/<version>/
  policy.json       the policy
  README.md         what it decides, its rules, its gaps
  signals/          one file per signal source it reads (only if it reads any)
    README.md
    <source name>.json
```

Copy `docs/policy-template/` to the new folder, and fill in each placeholder. The template lives outside `policies/` on purpose: the loader treats every folder under `policies/` as a real policy.

### Fields in `policy.json`

- **`policyId`, `policyVersion`, `schemaVersion`, `description`.** The folder name must match `policyId` and `policyVersion`.
- **`signalsSchema`.** The type of each signal: `string`, `boolean` or `number`.
- **`boundSignals`.** Which signals come from the request, and from which path. Example: `"targetRef": "target"`.
- **`signalSources`.** For each sourced signal: the `source` name, the `claim` it asks for, the `subject`, `maxAgeSeconds`, and the `parameters` the source needs from the request.
- **`approvalSignals`.** The approval signal, its `resourceId` (`target`, or a path into `parameters`), and an optional `value`, which the approval must also match exactly.
- **`rules`.** Each has an `id`, a `condition` and an `outcome` with an `action` and a `reason`.
- **`unboundSignalReasons`.** A reason for each signal that is neither bound nor sourced.

### Conditions

A condition checks one fact with an operator, or combines conditions with `all` and `any`.

Operators: `eq`, `neq`, `gt`, `gte`, `lt`, `lte`, `between`, `in`, `not_in`, `contains`, `contains_any`, `contains_all`, `not_contains`, `starts_with`, `ends_with`, `matches` (a regular expression, checked in linear time), `is_true`, `is_false`, `is_null`, `is_not_null`, `exists`, `not_exists`, `length_eq`, `length_gt`, `length_gte`, `length_lt`, `length_lte`, `type_is`, `string`, `number`, `boolean`, `object`.

### Validation

Run the repository's `PolicyValidator` on the file before proposing it. It enforces:

- Every approve rule requires the approval signal with `is_true`.
- Every signal that is neither bound nor sourced has an `unboundSignalReasons` entry.
- The folder, `policyId` and `policyVersion` match.

Use Node 24 for the validator.

### Signal sources

A sourced signal needs a service that answers it. The service:

- checks each query's signature, audience, source name and freshness, against Parmana's public key
- answers from the system of record, not from the request
- signs each answer, and sets how long the answer is valid

The GitHub signal source in `parmana-connectors` (`src/signal/`) is the worked example. A source must be registered with Parmana through the maker and checker flow before a policy can use it.

### Approval and loading

1. **Propose.** The maker sends the file, unchanged, as `proposedContent`, with a reason, using their own human key.
2. **Review.** A different person, the checker, reads the proposal and its diff.
3. **Approve.** The checker signs with their step-up key, on their own machine, and approves.
4. **Record.** Parmana records the approval, and the policy is in effect for that version.
5. **Export.** The approval is exported into this repository's governance log, as `governance/README.md` describes.

Loading into a deployment is a separate step: `scripts/apply-policies-table-and-backfill.ts`, which reads the policy folder named by `PARMANA_POLICY_DIR`.

## Checklist

- [ ] The decision is one sentence.
- [ ] Every signal has a stated trust level and, if sourced, a source.
- [ ] Refusals come before approvals, and the catch-all reject is last.
- [ ] Every approve rule requires the approval signal.
- [ ] Every unbound signal has a reason.
- [ ] The README says what the approval covers, what it doesn't, and the known gaps.
- [ ] The validator passes.
- [ ] The version is new, and the folder names match the file.
- [ ] The maker and checker are different people.
