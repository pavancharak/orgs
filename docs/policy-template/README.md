# Policy template

Copy this folder to `policies/<policy name>/<version>/` in the organization's repository, and fill in every `REPLACE` value.

This is a template, not a policy. It sits outside `policies/` on purpose: the loader treats each folder under `policies/` as a real policy.

## Steps

1. Copy `policy.json` into the new folder, and replace every `REPLACE` value, including the signal names and the rule identifiers.
2. Delete any rule that doesn't apply, and keep the order: refusals first, the approve rule next, the catch-all reject last.
3. If the policy reads a sourced fact, add its file under `signals/`, and describe the source's claims there.
4. Write the folder's `README.md`: what the policy decides, its rules and tiers, what the approval covers, and the known gaps.
5. Validate with the repository's `PolicyValidator`, under Node 24.
6. Propose, review and approve it, as `docs/policy-design-and-implementation.md` describes.

See `docs/policy-design-and-implementation.md` for the design rules and the implementation steps.
