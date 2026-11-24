# Signals for github-pr-approval 1.2.0

Each file here defines one signal source that this policy reads. The policy's `signalSources` says which claim each fact uses, and each file says how the source answers that claim.

| File | Source | Facts it provides |
| --- | --- | --- |
| `github-pull-requests.json` | `github-pull-requests` | `repositoryAuthorized`, `branchProtected`, `requiredReviewsCompleted`, `statusChecksPassed` |

The `audience` must be replaced with the signal service's real `https://` URL before the source is registered. Registration is approved through the maker and checker flow.
