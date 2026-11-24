# Org A connectors

This page is Org A's connector setup. Org A uses the GitHub connector for one action only: merging pull requests that an AI agent proposes. It covers the settings, how the connector runs, and what must be true before it releases anything. An operator can deploy from this page alone.

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

### Capability

| Capability | Does | Changes GitHub | Policy |
| --- | --- | --- | --- |
| `github:pr-merge` | Merges a pull request with the `merge` method only, at the head commit the approver reviewed | **Yes. Irreversible through the connector.** | `github-pr-approval` 1.2.0 (draft, not approved) |

The read and comment capabilities are not used by Org A. The connector still has them, but the endpoint does not serve them, and Org A does not register them.

### Target and parameters

- **Target:** `owner/repo#number`, for example `acme/payments#17`. Anything else is refused before GitHub is called.
- **`github:pr-merge`:** `mergeMethod` (required: `merge`; the policy refuses `squash` and `rebase`) and `expectedHeadSha` (required by the policy: 40 lowercase hex characters, the head commit the approver reviewed). GitHub refuses the merge if the head has moved.

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

**One registration.** Register `github:pr-merge` only, with the policy `github-pr-approval`.

**The endpoint address is checked twice, and the checks are the same both times.** Parmana refuses the address unless every one of these holds:

- The URL is an absolute `https://` URL, at most 2,048 characters.
- It has no user name, no password and no fragment.
- Its host is a name, not an IP address.
- The host is not `localhost`, not `*.localhost`, and has at least one dot.
- The host resolves in DNS, and **every** address it resolves to is public. An address in any of these ranges is refused: `0.0.0.0/8`, `10.0.0.0/8`, `100.64.0.0/10` (carrier-grade NAT), `127.0.0.0/8` (loopback), `169.254.0.0/16` (link-local), `172.16.0.0/12`, `192.0.0.0/24`, `192.0.2.0/24`, `192.88.99.0/24`, `192.168.0.0/16`, `198.18.0.0/15`, `198.51.100.0/24`, `203.0.113.0/24`, `224.0.0.0/4` (multicast), `240.0.0.0/4`, and the IPv6 equivalents (`::/128`, `::1/128`, `64:ff9b::/96`, `100::/64`, `2001::/23`, `2001:db8::/32`, `2002::/16`, `fc00::/7`, `fe80::/10`, `fec0::/10`, `ff00::/8`).

The checks run when a registration is **proposed**, and again when it is **approved** (`packages/api/src/routes/external-connectors.ts`, the approve route). A refused check returns an error and the change is not stored, or not approved. The checker should expect the address check to run at approval too.

**Checked again on every release.** Each release resolves the endpoint's host again and refuses the release if any address it resolves to is not public. It then connects only to the addresses it has just checked, and it verifies the TLS certificate against the host name. So a DNS change after approval does not let a release reach a private address. This is in `packages/execution-gateway/src/connector-execution/GatewayExternalAdapter.ts`: `execute` calls `resolvePublicEndpointAddresses`, and `createPinnedHttpsTransport` connects to the checked addresses.

**What this does not cover:** the endpoint itself. If the endpoint's own host is under someone else's control, a release goes to whatever answers at a public address that the host name resolves to. Org A should control the endpoint's DNS and certificate, and that is part of the deployment steps above.

## Status

| Item | Status |
| --- | --- |
| Connector code tested | Passing (`parmana-connectors` test suite) |
| Live tests against GitHub | Passed 2026-11-24 on a test repository, 5 of 5 |
| External endpoint merged | Yes (PR #89) |
| Endpoint configuration names aligned with the connectors | Done in `parmana-connectors` commit `dd27c1f` on branch `fix/github-endpoint-config`. Not yet merged. |
| Durable answer store | **Not built. Required before production.** |
| Policies approved | **Not done. All three are drafts.** |
| Endpoint deployed at a public HTTPS address | **Not done.** Read the address checks above first. |
| Endpoint registered with Parmana, per capability | **Not done** |
| GitHub App created for Org A and installed on its repositories | **Not done** |

## Deployment record

| Date | Connector commit | Endpoint URL | Operator | Result |
| --- | --- | --- | --- | --- |
| (none yet) | | | | |

## Other connectors

Razorpay, Paytm, Slack, HubSpot, Zoho and Zerodha are not set up for Org A.
