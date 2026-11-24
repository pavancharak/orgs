# Org A governance

This folder holds the signed record of every policy change Org A has approved. It is the evidence for who proposed each change, who approved it, and the exact text that was approved.

Parmana writes these records. Nobody writes them by hand, and nothing in this folder is edited.

## Files

| File | What it holds | Written by |
| --- | --- | --- |
| `policy-approvals.jsonl` | One JSON object per line, one per approved policy change, in approval order | Parmana's export script |
| `policy-approval-keys.json` | The public keys that check each record's signature, keyed by key id | Parmana's export script, which fetches them from Parmana |

Both are needed to verify the log. Keep them together, and commit them together.

## What a record contains

| Field | Meaning |
| --- | --- |
| `seq` | The entry's position, starting at 1 |
| `prevEntryHash` | The `entryHash` of the previous entry. `null` for the first. |
| `entryHash` | A hash of this entry, computed over its record and `prevEntryHash` |
| `record.policyChangeApprovalRecordId` | The id of this approval |
| `record.pendingPolicyChangeId` | The proposal that was approved |
| `record.policyName`, `record.policyVersion` | The policy and version approved |
| `record.proposedBy` | The maker: the human key holder who proposed the change |
| `record.approvedBy` | The checker: the human key holder who approved it. Always a different person from the maker. |
| `record.proposedAt`, `record.approvedAt` | When the change was proposed and approved |
| `record.contentHashBefore` | The hash of the policy content in effect before this change |
| `record.contentHashAfter` | The hash of the policy content this approval makes effective |
| `record.signature` | The checker's Ed25519 signature over the record. `keyId` names the public key that checks it. |

`contentHashAfter` ties the record to exact policy text. If a policy file differs by one character from the proposed text, its hash differs, and Parmana refuses requests that name that version.

## What the log proves, and what it does not

**Proves:**

- **Order:** each entry names the one before it. Removing or reordering an entry breaks the chain.
- **Authorship:** each record is signed by the checker's key. Changing a record breaks its signature.
- **Content:** each approval is tied to an exact policy text.

**Does not prove:** that the checker read the change carefully, or that the approval was a good decision. Those are the checker's responsibility, and the policy's own rules are the control on what gets approved.

## How a record gets here

1. The maker proposes a change, with their own key, sending the policy file unchanged. The proposal is stored in Parmana's database.
2. The checker reviews it, and approves it on their own machine, signing with their step-up private key. That private key never leaves their machine.
3. Parmana stores the signed record in its database. The export script then writes the same records to `policy-approvals.jsonl`, and the public keys to `policy-approval-keys.json`.

**A record that exists in the database but is missing from this folder is not a problem to fix by hand.** Run the export again, and commit what it writes. Never copy an entry in or out by hand.

## Exporting the log

**Where the scripts run.** The export and verify scripts are Parmana's. They run from a checkout of `parmana-core`, at the commit pinned in `org-a/database/README.md`. They read and write that checkout's own `governance/` folder, not this folder. So, for Org A, the export is a copy step:

```bash
# In a clean checkout of parmana-core, at the pinned commit,
# with DATABASE_URL set in the shell (never in a file):
npm ci
npm run policy-log:export
```

Then copy the two files it wrote, `governance/policy-approvals.jsonl` and `governance/policy-approval-keys.json`, into `org-a/governance/` in this repository, and commit them with the policy change they record.

Check the script's options before you rely on it: `scripts/policy-approval-log.ts` has `verify`, `export`, `--base` and `--require-coverage`. Its header comment, in `parmana-core`, is the reference for them.

## Verifying the log

The verify command checks the chain, the signatures against the public keys, and that each policy in effect has an approval. It has the same limit as the export: it reads its own checkout's `governance/` folder. For Org A, copy the two files into that checkout's `governance/` folder, run the check there, and remove the copies afterwards. Do not commit the copies into `parmana-core`.

```bash
# In a clean checkout of parmana-core at the pinned commit,
# after copying org-a's two governance files into its governance/ folder:
PARMANA_POLICY_DIR=/path/to/orgs/org-a/policies npx tsx scripts/verify-policy-changes-approved.ts --full-scan
```

It fails if the log does not verify, or if a live policy has no matching approval.

## Who holds what

| Holds | Why |
| --- | --- |
| The maker's API key (a human key) | Proposes changes. It cannot approve them. |
| The checker's API key (a human key) | Used by the checker to approve |
| The checker's step-up private key | Signs each approval. Kept on the checker's machine, never committed or shared. |
| The public keys in `policy-approval-keys.json` | Check each signature. They are not secret. |

## Rules for this folder

- Never edit `policy-approvals.jsonl` or `policy-approval-keys.json` by hand.
- Never delete or reorder entries.
- Commit the two files with the policy change they record.
- Keep a backup of this repository. The log is evidence.

## The permanent fix, not yet done

The scripts assume they run from their own checkout. The permanent fix is a small change in `parmana-core`: the export and verify scripts should take the governance folder and the policy folder as arguments. Then Org A's repository can be the root of its own governance, with no copy step. Until that change is made, follow the procedure above exactly.
