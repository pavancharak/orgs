# Org A governance

This folder holds the signed record of every policy change Org A has approved. It is the evidence for who proposed each change, who approved it, and exactly what text was approved.

Parmana writes the records. A person does not write them by hand, and nothing in this folder should be edited.

## The file

`policy-approvals.jsonl` is one JSON object per line. Each line is one approved policy change, in the order it was approved.

| Field | Meaning |
| --- | --- |
| `seq` | The position of the entry in the log, starting at 1. |
| `prevEntryHash` | The `entryHash` of the entry before it. `null` for the first entry. |
| `entryHash` | A hash of this entry, computed over its record and `prevEntryHash`. |
| `record.policyChangeApprovalRecordId` | The id of this approval record. |
| `record.pendingPolicyChangeId` | The id of the proposal that was approved. |
| `record.policyName`, `record.policyVersion` | The policy and the version approved. |
| `record.proposedBy` | The maker: the key holder who proposed the change. |
| `record.approvedBy` | The checker: the key holder who approved it. Always a different person from `proposedBy`. |
| `record.proposedAt`, `record.approvedAt` | When the change was proposed, and when it was approved. |
| `record.contentHashBefore` | The hash of the policy content in effect before this change. |
| `record.contentHashAfter` | The hash of the policy content this approval makes effective. |
| `record.signature` | The checker's ed25519 signature over the record. `keyId` says which public key to check it with. |

The `contentHashAfter` value is the link between the record and the policy file. The hash is computed over the policy's content exactly as proposed. If the file changes by even one character, its hash no longer matches, and Parmana refuses requests that name it.

## What the chain proves

- **Order.** Each entry names the entry before it. Removing or reordering an entry breaks the chain.
- **Authorship.** Each record is signed by the checker's key. Changing a record breaks its signature.
- **Content.** The content hash ties each approval to exact policy text.

It does not prove that the checker read the policy carefully, or that the approval was wise. Those are the checker's responsibility.

## How records are written

1. A maker proposes a policy change with their own key, sending the policy file unchanged.
2. A checker reviews it, then approves it on their own machine, using their step-up private key. That key never leaves their machine.
3. Parmana appends the signed record to this file, and to the database.

If the approval was recorded in the database but is missing here, export the log again from `parmana-core` (its `policy-log:export` script) rather than copying entries by hand. A missing entry is not an approval.

## Committing the log

Commit this file together with the policy change it approves. Then the policy and its evidence are in the same commit. Never rewrite or delete history for this file. If an entry is wrong, record a correction as a new change, and keep the wrong entry.

## Verifying the log

The verifier is a Parmana script, `scripts/verify-policy-changes-approved.ts`, in a `parmana-core` checkout. It checks the chain and every signature against the public keys, then checks that each live policy file matches its latest approval. It fails if the log does not verify, or if a live policy has no matching approval.

Run it from `parmana-core`, against this repository's policy folder. Check the script's options for pointing it at a log file outside its own repository before you rely on it for Org A.

## Who holds what

| Holds | Why |
| --- | --- |
| The maker's API key (human) | Proposes changes. It cannot approve them. |
| The checker's API key (human) | Lets the checker approve. Kept by the checker. |
| The checker's step-up private key | Signs each approval. Kept on the checker's own machine, never committed or shared. |
| The public keys | Used to check the signatures. Published by Parmana, and held by the verifier. |

## Rules for this folder

- Never edit `policy-approvals.jsonl` by hand.
- Never delete or reorder entries.
- Commit the log with the policy change it records.
- Keep a backup of this repository, because the log is evidence.
