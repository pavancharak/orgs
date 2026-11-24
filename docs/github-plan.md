# GitHub plan

This is the plan for Parmana's GitHub App: what it does, what has been decided, what is built, and what is still needed before anyone uses it. It is written to stand alone, so it can be read without the other repositories.

Status is as of 2026-11-24.

## What the App does

Parmana governs one action on GitHub: **merging pull requests that an AI agent proposes.** A merge goes ahead only when a person has approved that exact change, and every decision is recorded.

Reads and comments are out of scope. Routine merges that don't touch production paths will, once path rules are built, follow GitHub's own rules without a Parmana approval.

## Decisions

| Decision | Choice | Why |
| --- | --- | --- |
| Actions governed | Merges only | Reads and comments add approval work without adding protection |
| Merge method | `merge` only | The approved commit stays in the history as a parent of the merge commit |
| Approval scope | The pull request and its exact head commit | A merge at any other commit is refused |
| Branch protection, repository authorization, reviews, checks | Read from GitHub by a signal source, each must hold | GitHub knows these facts, so the caller doesn't need to be trusted for them |
| Risk score | Dropped | It was the caller's word, and it could only refuse |
| Policy distribution | One standard policy, the same text for every organization | One set of rules for everyone, with no organization-specific files in the product |
| Approval model | Per-organization approvals stay the default | Each organization keeps its own evidence and checkers |
| Path-based approvals | Approved in principle. Only changes to production paths need a Parmana approval. | Approvals go where the risk is |
| Public install | Approved in principle. One deployment serves many organizations, with tenants. | Organizations can install without running their own deployment |
| Standing approvals | Approved in principle, for non-production merges within a repository | Routine merges stay in the audit trail without needing an approval each |
| Billing | Not decided. A flat monthly fee per organization, tiered by repository count, is the recommendation. | Usage is recorded for audit, and doesn't set the price |

## What is built, and where

### `parmana-core` (the Parmana service)

| Item | Where | Status |
| --- | --- | --- |
| Standard merge policy `github-pr-approval` 1.2.0 | Branch `feat/standard-github-policies` (PR #776) | Open, not yet approved by a maintainer |
| Governance folder settings for the approval log scripts | PR #775 | Open |
| Migration bundle conflict fix on the outbox work | PR #771 | Conflict fixed. Blocked on review and CI. |
| Commit sign-off and signature on the gateway test | PR #773 | Sign-off pushed. Unsigned, and the branch needs rebasing onto `main`. Blocked. |
| Design for standing approvals and tenants | Branch `feat/rfc-0031-standing-approvals-tenants` (RFC-0031) | Draft, no code |

### `parmana-connectors` (the GitHub connector)

| Item | Where | Status |
| --- | --- | --- |
| External endpoint, merge only | `feat/merge-only-endpoint` (in PR #92) | Open. Tests pass. |
| Endpoint settings aligned with the connectors | PR #91 | Open |
| GitHub signal source for the four GitHub facts | `feat/github-signal-source` (PR #92) | Built and tested. Not deployed. |
| Live tests against GitHub | `tests/connectors/github/github-live.test.ts` | Passed, 5 of 5, on 2026-11-24 |

### `orgs` (Org A's records and this plan)

| Item | Where | Status |
| --- | --- | --- |
| Org A's merge policy, with its `signals/` folder | `org-a/policies/github-pr-approval/1.2.0/` | Pushed. Drafts, not approved. |
| Org A's connector setup and status | `org-a/connectors/README.md` | Pushed |
| Org A's database and governance records | `org-a/database/`, `org-a/governance/` | Pushed. The approval log is empty until the first approval. |
| Multi-organization setup | `docs/multi-org-setup.md` | Pushed |

## What is needed before anyone can use the App

Nothing is approved, deployed or registered. Parmana refuses merges until all of these are done.

1. **Review and merge the open pull requests:** #775, #776, #91, #92, and #771 and #773 once they're unblocked.
2. **Sign and rebase PR #773.** Its signature check fails until the commit is signed on a machine that holds the key. The rebase onto `main` is needed first, to clear a typecheck failure inherited from an older base.
3. **Clear the release gate for `parmana-core`.** Its release check needs owner-signed records (deployment verification and backup and recovery drill), and a fourth check that hasn't been read yet. These records must be on the base branch, not in a pull request. The open question is whether a release gate should run on pull requests at all.
4. **Get the maintainer approval** for the standard merge policy, through the maker and checker flow.
5. **Deploy the signal source and the endpoint** at HTTPS addresses.
6. **Register both** with Parmana, through the maker and checker flow. The signal source is registered as a business signal source, and the endpoint as an external connector.
7. **Pilot with one organization** before opening the App to others.

## Building next

In order:

1. **Path-based approvals.** The signal source gets a `pr.touches-production` claim that reads `.parmana/production-paths.yml` from the base branch and matches the changed files. A missing or unreadable list counts as production.
2. **Standing approvals, in `parmana-core`.** A checker approves a rule for a repository's non-production paths once. This is the core change in RFC-0031.
3. **Multi-installation signal source.** One signal service serves many installations, with a registry and a GitHub check that each installation belongs to its installer.
4. **Tenants, in `parmana-core`.** Tenant columns and separation for every governed table, per-tenant checkers and keys. This is the largest step, and it follows RFC-0031.
5. **Standard approval path.** Maintainers approve the standard policy once, and each deployment accepts it.

Steps 2 and 4 are core changes, and both follow RFC-0031 as a single design.

## Billing

Not built. The recommendation:

- **Flat monthly fee per organization**, tiered by the number of repositories the App covers.
- **Usage is recorded for audit**, not used to set the price.
- **GitHub bills the customer's own plan**, and Marketplace billing only applies if the App is listed and charged for there.

The tier boundaries are a business decision. The billing system itself is built after a paid pilot, not before.

## Risks

- **A signal service compromise** would let false facts reach the policy. Each answer is signed, and the service holds the App's key, so the key is the thing to protect.
- **A shared service** makes every organization depend on it, so it needs its own availability plan before a public install.
- **The endpoint's first answers are in memory.** A restart would forget them, so a replayed merge could act again. A durable store is needed before production.
- **Branch rulesets are not read.** A repository protected only by rulesets reads as unprotected, so its merges are refused. That's safe, but it may surprise customers.
- **Owner records gate releases.** Until they exist, the release gate fails.

## Open decisions

1. **Whether a release gate runs on pull requests.** Today it does, so pull requests fail for things only a release needs.
2. **The tier boundaries and price**, once billing is designed.
3. **Standing approval length and path-list changes**, from RFC-0031's open questions.
4. **Which maintainer keys are trusted** for the standard policy, and how they're rotated.
