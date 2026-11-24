# Org A connectors

This folder holds Org A's connector setup: which connectors Org A uses, the settings for each, and their status. The connector code is shared software, in `parmana-connectors`. Nothing here is secret. Secrets are set in the environment of the service that runs the connector, never in this repository.

## GitHub

| Item | Value |
| --- | --- |
| Code | `parmana-connectors`, `src/connectors/github/` |
| Docs | `src/connectors/github/README.md`, `CAPABILITIES.md`, `LIVE.md`, `EXTERNAL.md` |
| Capabilities | `github:pr-fetch`, `github:pr-merge`, `github:comment` |
| Policies | `org-a/policies/github-pr-read/1.2.0`, `github-pr-approval/1.2.0`, `github-pr-comment/1.0.0`. Drafts, not yet approved. |
| Run as | External endpoint: `src/connectors/github/external-endpoint.ts` and `external-server.ts` |
| Endpoint URL | Not set. Needs a public HTTPS host. |
| Secret | `GITHUB_APP_CREDENTIAL`, set in the endpoint's environment. Never committed. |
| Registration with Parmana | Not registered. Needs a registration change approved by a checker. |
| Live tests | Passed 2026-11-24 on `pavancharak/parmana-github-test`: 5 of 5. See `src/connectors/github/LIVE-RUN-2026-11-24.md`. |

## Before this connector can release to Org A

1. The endpoint is deployed at a public HTTPS address, and its URL is recorded above.
2. The three policies are approved through the maker and checker flow, and recorded in `org-a/governance/`.
3. The endpoint uses a durable answer store, so a restart cannot repeat a release.
4. The endpoint is registered with Parmana, and the registration is approved by a checker.

## Other connectors

Razorpay, Paytm, Slack, HubSpot, Zoho and Zerodha are not set up for Org A yet. Add a section here when one is.
