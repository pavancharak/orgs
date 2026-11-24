# Org A connectors

This page is Org A's connector setup: which connectors are used, what each one can do, how each one runs, and what is still needed before it can release. It is written so that an operator can work from this repository alone.

The connector code is shared software. It lives in the `parmana-connectors` repository, and it is not copied here. Nothing in this folder is a secret. Secrets are set in the environment of the service that runs the connector, and their values are never written down here.

## How a connector runs

A connector is not called directly by an agent. Every action goes through Parmana:

1. An agent sends a request for a capability, such as `github:pr-merge`.
2. Parmana evaluates the policy in effect for that capability. The request is approved or rejected. ([policies README](../policies/README.md))
3. If approved, Parmana releases the action to the connector.
4. The connector performs the action with its own credential, and reports the result.

Two ways a connector can be run:

- **Built in.** The connector runs inside Parmana's own service.
- **External endpoint.** The connector runs as a separate service, at an HTTPS address that Parmana releases to. The endpoint checks Parmana's signature on each release before it acts. Org A's GitHub connector is planned as an external endpoint, so the credential stays with the endpoint and not with Parmana.

## GitHub

### Capabilities

| Capability | What it does | Changes GitHub? | Policy (draft, not yet approved) |
| --- | --- | --- | --- |
| `github:pr-fetch` | Reads a pull request: its head SHA, whether it is mergeable, and its base branch. | No | `github-pr-read` 1.2.0 |
| `github:pr-merge` | Merges a pull request with `merge`, `squash` or `rebase`, optionally only if the head SHA is the one the approver saw. | **Yes, and irreversible through the connector.** | `github-pr-approval` 1.2.0 |
| `github:comment` | Posts one comment (1 to 4,000 characters), marked with the transaction id so it can be found later. | **Yes.** A comment cannot be removed through the connector. | `github-pr-comment` 1.0.0 |

Each policy is in `../policies/`, with a README that explains its rules and gaps.

### Target and parameters

- **Target:** `owner/repo#number`, for example `acme/payments#17`. Any other target is refused before GitHub is called.
- **`github:pr-fetch`:** takes no parameters.
- **`github:pr-merge`:** `mergeMethod` (required: `merge`, `squash` or `rebase`) and `expectedHeadSha` (optional: a 40 character lowercase commit SHA). If `expectedHeadSha` is given, GitHub refuses the merge when the head has moved.
- **`github:comment`:** `body` (required: 1 to 4,000 characters).

Any other parameter is refused.

### Credential

The connector authenticates as a GitHub App. Its credential is one value:

`<app id>:<installation id>:<base64 of the App's PEM private key>`

- Set it in the environment of the service that runs the connector, as `GITHUB_APP_CREDENTIAL`.
- Each call signs a short-lived JWT, exchanges it for an installation token, and uses that token once. The token is never stored.
- No error message shows the key or the token.

**The App needs these permissions, and nothing more:**

- Pull requests: read and write
- Contents: read and write (a merge needs it)
- Metadata: read (GitHub sets it)

Install the App only on the repositories Org A allows. Turn off webhooks. The connector does not receive events.

### Running as an external endpoint

The endpoint serves all three capabilities. Its settings:

| Variable | Value |
| --- | --- |
| `PARMANA_PUBLIC_KEY_PEM` | Parmana's public key, PEM. Get it from Parmana's `publicKey("default")`. |
| `ENDPOINT_URL` | The endpoint's URL, exactly as it is registered with Parmana. |
| `GITHUB_APP_CREDENTIAL` | The credential above. |
| `PORT` | Optional. Defaults to 8080. |

Serve it over HTTPS, behind a proxy that terminates TLS. Parmana releases only to HTTPS.

Each release is refused, before GitHub is called, if its signature does not verify, its audience is not `ENDPOINT_URL`, it has expired, or its capability is not one of the three. A replayed release gets the first answer back and is never acted on twice.

**Known limit:** the endpoint keeps its first answers in memory. A restart forgets them, so a replayed release could act again. A durable store is needed before production.

### Status and steps before release

| Step | Status |
| --- | --- |
| Connector code tested | Done. The connector's test suite passed. |
| Live tests against GitHub | Passed on 2026-11-24, using a test repository. 5 of 5. Record in `connectors/` history. |
| External endpoint code | Merged into `parmana-connectors`. |
| Durable answer store | **Not done.** Required before production. |
| Policies approved | **Not done.** The three policies are drafts. Each needs the maker and checker flow. |
| Endpoint deployed at a public HTTPS address | **Not done.** |
| Endpoint registered with Parmana | **Not done.** Registration is a change that a checker approves. |
| App installed on Org A's repositories | **Not done.** Install only on the repositories Org A chooses. |

### Testing

The connector's tests use a fake GitHub, so they never touch a real repository. The live tests call GitHub's real API, and they run only when the operator sets `ALLOW_LIVE_GITHUB=1`. They need a disposable pull request for each write test, and they must not be pointed at a repository that matters.

Run the live tests on a machine that holds the App's key, with the settings in a git-ignored `.env.live` file.

## Other connectors

Razorpay, Paytm, Slack, HubSpot, Zoho and Zerodha are not set up for Org A. Add a section here when one is.
