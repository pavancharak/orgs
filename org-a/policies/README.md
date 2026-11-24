# Org A policies: how they work

This folder holds Org A's policies, one version per folder. This page explains the whole policy model as it applies here. Each policy folder also has its own README, which covers that policy's rules and gaps.

The full reference is in `parmana-core`, under `docs/site/`. Where this page describes a rule, the source is linked. If the two ever disagree, the `parmana-core` docs win, and this page should be corrected.

## Vocabulary

| Term | Meaning |
| --- | --- |
| **Capability** | A named action that Parmana can authorize, such as `github:pr-merge`. |
| **Policy** | The rules that decide whether a request for a capability is approved or rejected. Identified by a name and a version. |
| **Signal** | A value a rule reads, such as `targetRef` or `mergeApproved`. Each signal has a source that says where its value comes from. |
| **Target** | The thing the request acts on, here `owner/repo#number`. It is part of the request Parmana executes. |
| **Approval** | A signed statement by a person (the checker) that a specific action on a specific target is allowed. |
| **Approval record** | The stored, signed, chained record that a policy version was approved. Parmana writes it. |
| **Policy in effect** | The approved version of a policy that a request must name. Ask `GET /policies/in-effect?capability=...` which one it is. |
| **Decision** | Parmana's outcome for one request: approve or reject. |

## How a decision is made

Parmana decides one request at a time, in this order:

1. **The version must be in effect.** A request that names a version other than the one in effect is refused with `403 POLICY_DENIED`. A request cannot choose its own policy version. ([policies-and-the-decision](https://github.com/pavancharak/parmana-core/blob/main/docs/site/concepts/policies-and-the-decision.mdx))
2. **Rules are checked in order, and the first match decides.** Later rules are not checked. ([policies-and-the-decision](https://github.com/pavancharak/parmana-core/blob/main/docs/site/concepts/policies-and-the-decision.mdx))
3. **If no rule matches, the outcome is reject.** There is no path where an unmatched request is approved. A policy should say so explicitly with a final catch-all rule. The drafts here use `matches "^"` for that. The reference recommends `{ "always": true }`, and the drafts should switch to it.

Newer Parmana policies (version 2) check rules in four named tiers, in this order: **refuse**, then **review**, then **approve**, then **reject**. Every rule must name its tier. These drafts use the simpler format, where each rule approves or rejects. ([policy-version-2](https://github.com/pavancharak/parmana-core/blob/main/docs/site/guides/policy-version-2.mdx))

## Where each signal's value comes from

Every signal has a provenance, which says what makes its value trustworthy:

| Provenance | Where the value comes from | Can it authorize? |
| --- | --- | --- |
| **Bound** (`boundSignals`) | The request itself. For example, `targetRef` comes from `target`. | Only as a check on what was asked. It cannot prove anything about the world. |
| **Approved** | A person's signed approval. Parmana checks that the signature verifies and that the approval matches the resource exactly. | **Yes.** Every approve rule here requires it. |
| **Sourced** | A registered business system that signed its answer. Parmana checks the signature against the system's registered key. | Yes, once a source is connected. No GitHub source exists yet. |
| **Untrusted** (caller-declared) | The caller says what the value is. Nothing checks it. | **It can refuse. It can never authorize.** |

The rule that matters most: an unsigned value is untrusted, whoever sent it. A caller-declared fact can refuse a request, and it can never be the reason a request is approved. Every caller-declared fact needs a documented reason in `unboundSignalReasons`. ([policy-version-2](https://github.com/pavancharak/parmana-core/blob/main/docs/site/guides/policy-version-2.mdx), [connect-any-external-system](https://github.com/pavancharak/parmana-core/blob/main/docs/site/guides/connect-any-external-system.mdx))

`boundSignals` closes a different gap: it ties a signal to the request Parmana will execute, so the value cannot change between the decision and the action. Binding is per field, and it does not verify that the value is true. ([policies-and-the-decision](https://github.com/pavancharak/parmana-core/blob/main/docs/site/concepts/policies-and-the-decision.mdx))

## Writing a policy

A policy is one file, `policies/<name>/<version>/policy.json`. The folder names must match the file's `policyId` and `policyVersion`.

Required fields, in the drafts' format:

- `policyId`, `policyVersion`, `schemaVersion`, `description`
- `signalsSchema`: the type of each signal (`string`, `boolean`, `number`)
- `boundSignals`: which signals come from the request, and from which path
- `signalSources`: the sources of any sourced signals. Empty until a source exists.
- `approvalSignals`: the approval signal, and the resource it names (`target`, or a path into `parameters`)
- `rules`: each with an `id`, a `condition`, and an `outcome` with an `action` and a `reason`
- `unboundSignalReasons`: a reason for every signal that is neither bound nor sourced

Two rules the validator enforces:

- **Every approve rule must require the approval signal with `is_true`.** A policy that could approve without a signed approval is refused.
- **Every signal that is not bound needs a reason.**

Conditions use these operators: `eq`, `neq`, `gt`, `gte`, `lt`, `lte`, `between`, `in`, `not_in`, `contains`, `contains_any`, `contains_all`, `not_contains`, `starts_with`, `ends_with`, `matches` (RE2 regex, linear time), `is_true`, `is_false`, `is_null`, `is_not_null`, `exists`, `not_exists`, `length_eq`, `length_gt`, `length_gte`, `length_lt`, `length_lte`, `type_is`, `string`, `number`, `boolean`, `object`. Conditions combine with `all` and `any`.

Check a draft before proposing it. Run the repository's `PolicyValidator` from `parmana-core` on the file, with Node 24. See [policy-lifecycle-and-approvals](https://github.com/pavancharak/parmana-core/blob/main/docs/site/guides/policy-lifecycle-and-approvals.mdx), Part 1, step 1.

## What an approval covers

An approval covers exactly the action and the resource it names. It does not cover other parameters. An approval for a comment does not fix the comment text, and an approval for a merge does not fix the merge method. Each policy's README says what its approval covers and what it leaves open. ([integrate](https://github.com/pavancharak/parmana-core/blob/main/docs/site/agents/integrate.mdx))

## Who does what

| Role | What they do | What they hold |
| --- | --- | --- |
| **Author** | Writes the policy file | A clone of this repository |
| **Maker** (proposer) | Proposes the policy for approval | An API key registered as a human (`credentialHolderType USER`) |
| **Checker** (approver) | Reviews, then approves or rejects | Their own human API key, and their own step-up private key |
| **Operator** | Adds API keys, and deploys | Access to the deployment |
| **Agent** | Sends requests that name a policy | A service API key |

The maker and the checker must be different people. A checker cannot approve a change they proposed, and there is no override. ([policy-lifecycle-and-approvals](https://github.com/pavancharak/parmana-core/blob/main/docs/site/guides/policy-lifecycle-and-approvals.mdx))

## The lifecycle

1. **Write** the policy file in this repository, under the next version folder.
2. **Propose.** The maker sends the policy's content, unchanged, as `proposedContent`, with their own key.
3. **Review.** The checker reads the proposal.
4. **Approve or reject.** The checker signs the decision on their own machine, with their step-up private key. The private key never leaves that machine. ([policy-governance-maker-checker](https://github.com/pavancharak/parmana-core/blob/main/docs/site/guides/policy-governance-maker-checker.mdx))
5. **Take effect.** The approval is recorded. The latest approved version of a policy is the one in effect.

Parmana records each proposal, approval and rejection. Each approval record is signed and chained to the one before it, so a removed or altered record is detectable.

## What is checked on every request

- The version named is the version in effect.
- The loaded policy has an approval record, the record's signature verifies, and its content hash equals the live policy's. Anything else refuses the request. A policy edited in the database outside this flow cannot authorize anything.
- The gateway checks the approval record again just before it releases an action.
- At startup, the server checks every policy against its latest approval record and logs the result.

These checks run in every environment, including test and development. No environment variable turns them off. A policy that is still pending, or was rejected with no approved replacement, refuses every request. ([policy-lifecycle-and-approvals](https://github.com/pavancharak/parmana-core/blob/main/docs/site/guides/policy-lifecycle-and-approvals.mdx), step 7)

## Where the records are kept

| Table | What it holds |
| --- | --- |
| `pending_policy_changes` | Every proposal, its status, and who proposed and resolved it |
| `policy_change_approval_records` | Signed, chained approval records. The latest per policy name decides the version in effect. |
| `policies` | The approved policy content |
| `consumed_policy_change_step_up_nonces` | Step-up authorizations already used |
| `consumed_approval_nonces` | Action approvals already used |
| `refusal_records` | Every refused request, signed |

Parmana records each approval in its database. For Org A, the signed log is kept in this repository, under `org-a/governance/`, by the export procedure in `org-a/governance/README.md`. Never write it by hand.

## Changing a policy

- **Never edit an approved version in place.** Add a new version folder, and run the lifecycle again.
- **A new version is not in effect until it is approved.** The previous approved version stays in effect until then.
- **Deploy the policy before a deployment executes against it.** A policy still pending is refused on the first request.

## Known limits

From the `parmana-core` docs ([business-validation](https://github.com/pavancharak/parmana-core/blob/main/docs/site/concepts/business-validation.mdx)):

- Cumulative limits, such as a daily total, are not built. Limits apply per request.
- Parmana does not yet check that a grant's caller exists. A grant to an unknown key is accepted, and has no effect.
- Any registered source can answer any claim that a policy asks it. Claims are not limited per source.
- A request refused at release is left in an errored state until a person closes it.

For Org A, the GitHub policies have their own known gaps, listed in each policy's README.

## Troubleshooting

| You see | Usual cause | Fix |
| --- | --- | --- |
| `401 authentication required` | The key is missing, wrong or rotated, or the header is not `Authorization: Bearer <key>` | Check the key is set in this terminal |
| `403 POLICY_DENIED` | The request names a version that is not in effect | Ask `GET /policies/in-effect?capability=...` which version is |
| `403 SAME_ACTOR_CANNOT_APPROVE_OWN_CHANGE` | The approving key belongs to the maker | A different person approves, with their own key |
| `403 NON_HUMAN_CALLER_DENIED` | The key is not registered as a human | The operator registers it as `USER` |
| `403 STEP_UP_AUTHORIZATION_INVALID` | The step-up signature expired (over 120 seconds), was reused, or used a key that does not match the registered one | Sign again and send at once |
| `400` naming a field, on propose | The body is malformed, `policyId` does not match the URL, or the version has characters outside `A-Z a-z 0-9 . _ -` | Read the `error` message |
