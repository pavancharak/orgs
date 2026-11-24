# Org A connectors

This page is Org A's connector setup: which connectors are used, what each can do, how each runs, the exact settings, and what must be true before it releases anything. An operator can deploy from this page alone.

The connector code is shared software, in the `parmana-connectors` repository, pinned by commit (see below). Nothing in this folder is a secret. Secrets are set in the environment of the service that runs the connector, and their values are never written here.

## How a connector runs

1. An agent sends a request for a capability, such as `github:pr-merge`.
2. Parmana evaluates the policy in effect for that capability, and approves or rejects the request. ([policies README](../policies/README.md))
3. An approved action is **released** to the connector. Parmana signs the release with its Ed25519 key.
4. The connector checks the release, performs the action with its own credential, and returns the result.

A connector runs in one of two ways:

- **Built in:** inside Parmana's own service.
- **External endpoint:** as its own HTTPS service, which Parmana releases to. The endpoint checks Parmana's signature and its own audience before it acts, so its credential never reaches Parmana. Org A's GitHub connector runs this way.

## Software pin

| Item | Value |
| --- | --- |
| Repository | `pavancharak/parmana-connectors` |
| Pinned commit | The merge commit of PR #89 (`efa7403`) and later commits on `main` |
| Checked | 2026-11-24 |

Record the commit you deploy in the table at the end of this page.

## GitHub

### Capabilities

| Capability | Does | Changes GitHub | Policy (draft, not approved) |
| --- | --- | --- | --- |
| `github:pr-fetch` | Reads a pull request: head SHA, mergeable state, base branch | No | `github-pr-read` 1.2.0 |
| `github:pr-merge` | Merges a pull request with `merge`, `squash` or `rebase` | **Yes. Irreversible through the connector.** | `github-pr-approval` 1.2.0 |
| `github:comment` | Posts one comment, marked with the transaction id | **Yes. Cannot be removed through the connector.** | `github-pr-comment` 1.0.0 |

Each policy and its rules are in `../policies/`.

### Target and parameters

- **Target:** `owner/repo#number`, for example `acme/payments#17`. Anything else is refused before GitHub is called.
- **`github:pr-fetch`:** no parameters.
- **`github:pr-merge`:** `mergeMethod` (required: `merge`, `squash`, `rebase`); `expectedHeadSha` (optional: 40 lowercase hex characters). When given, GitHub refuses the merge if the head has moved.
- **`github:comment`:** `body` (required: 1 to 4,000 characters).

Any other parameter is refused.

### The GitHub App

Create one GitHub App for Org A, owned by Org A's GitHub account.

- **Webhook:** off (Active unticked). The connector receives no events.
- **Permissions:** Pull requests: read and write. Contents: read and write (needed for a merge). Metadata: read (set by GitHub).
- **Where it can be installed:** only on this account.
- **Installation:** install it only on the repositories Org A allows. Record the list of repositories here, not the installation id.

The App's private key is a `.pem` file. Store it in a secret store or on the deployment host, and never in a repository.

### Settings for the endpoint

These names match the connectors' Razorpay service, so one operator runs all of them the same way.

| Variable | Value | Notes |
| --- | --- | --- |
| `PARMANA_PUBLIC_KEY` | Parmana's public key, as PEM text | Set this **or** `PARMANA_PUBLIC_KEY_FILE`, never both. Literal `\n` is accepted. |
| `PARMANA_PUBLIC_KEY_FILE` | A path to the same PEM file | As above. |
| `PARMANA_RELEASE_AUDIENCE` | This endpoint's URL, exactly as registered in Parmana | Must be `https://`. Must match the registration exactly. |
| `GITHUB_APP_CREDENTIAL` | `<app id>:<installation id>:<base64 of the App's PEM key>` | Secret. Set only on the endpoint's service. |
| `PORT` | Port the service listens on | Optional. Default `8080`. |

The endpoint refuses to start if its public key is missing, both key settings are set, the key is not Ed25519, the audience is missing or not `https`, or the credential is missing.

**Where to get Parmana's public key:** from the Parmana deployment for Org A, through its public key endpoint for the `default` key id. Check the key's fingerprint with the operator before you set it. Do not copy it from another organization.

### Running the endpoint

From a checkout of `parmana-connectors` at the pinned commit:

```bash
npm ci
npm run build
npm run start:github-endpoint
```

The script loads `.env.live` if it exists, then starts the endpoint, which logs `github release endpoint listening on port <PORT>`. It reads the settings from the process environment, so on a deployment host set the variables there and do not rely on a file. Serve it over HTTPS, behind a proxy that terminates TLS. Parmana releases only to HTTPS.

**Known limit:** the endpoint keeps its first answers in memory. A restart forgets them, and a replayed release could then act again. A durable answer store is required before production. It is not built yet.

## Registering the endpoint with Parmana

A registration makes Parmana release a capability to the endpoint. It is a change that a checker approves, through the external connector change API.

1. **Propose.** `POST` to Parmana's `/external-connectors/changes` with the caller's key, and a body of:
   - `action`: `"register"`
   - `capability`: the capability, such as `github:pr-merge`
   - `endpointUrl`: the endpoint's `https://` URL, exactly as in `PARMANA_RELEASE_AUDIENCE`
   - `policy`: the policy name, such as `github-pr-approval`, matching `^[a-z0-9][a-z0-9-]{0,127}$`
   - `allowedParameters`: the parameter names the endpoint accepts, such as `["mergeMethod","expectedHeadSha"]`
   - `timeoutMs` (optional)
   - `reason`: why, at most the length the API allows
2. **Approve.** A checker, who is a different human caller from the maker, calls `POST /external-connectors/changes/<id>/approve` with their own key and a step-up authorization for that change id and action `approve`. Parmana checks the endpoint address again at approval, and applies the change in the same step.
3. **Verify.** The registration is active only after approval. Confirm it before releasing anything.

**What each call returns (from the Parmana code):**

- **Propose:** `201 Created`, with the change record in the body. Its `status` is `PENDING_APPROVAL`, and its `changeId` is the id for the approval.
- **Approve:** `200 OK`, with the resolved change record. Its `status` is `APPROVED`, and `resolvedBy` names the checker.
- **Refused:** an `error` message with a `code` field. Common codes are the ones for authentication (`401`), a missing step-up (`403`), and the same person proposing and approving.
- **List:** `GET /external-connectors/changes?status=PENDING_APPROVAL` returns `{ "changes": [...] }`.

**Each capability is one registration.** Register `github:pr-fetch`, `github:pr-merge` and `github:comment` separately, each with its own policy.

**Not confirmed here:** the exact checks Parmana makes on the endpoint address at approval. The register route checks it again at approval, and the rules are in `parmana-core`'s external connector code.

## Status

| Item | Status |
| --- | --- |
| Connector code tested | Passing (`parmana-connectors` test suite) |
| Live tests against GitHub | Passed 2026-11-24 on a test repository, 5 of 5 |
| External endpoint merged | Yes (PR #89) |
| Endpoint configuration names aligned with the connectors | Done in `parmana-connectors` commit `dd27c1f` on branch `fix/github-endpoint-config`. Not yet merged. |
| Durable answer store | **Not built. Required before production.** |
| Policies approved | **Not done. All three are drafts.** |
| Endpoint deployed at a public HTTPS address | **Not done** |
| Endpoint registered with Parmana, per capability | **Not done** |
| GitHub App created for Org A and installed on its repositories | **Not done** |

## Deployment record

| Date | Connector commit | Endpoint URL | Operator | Result |
| --- | --- | --- | --- | --- |
| (none yet) | | | | |

## Other connectors

Razorpay, Paytm, Slack, HubSpot, Zoho and Zerodha are not set up for Org A.
