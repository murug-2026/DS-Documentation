# Local documentation guidance

## Membership architecture reference

At the start of each task in this repository, consult
`/mnt/c/ProjectDS/DSDocumentation/membership-architecture.md` for architectural context and scope.
Read the linked modules relevant to the task before changing membership,
enrollment, client/program relationships, benefits, users/access, or payment reuse.
For unrelated work, the index is sufficient; do not expand task scope merely to
change architecture documentation.

Treat these documents as maintained context, not a replacement for inspecting
current source. Distinguish schema allowances, enforced workflow rules, and future
proposals. When authorized work changes those behaviors, update the relevant
module and review date in the same change. If code and documentation disagree,
verify the implementation and correct the affected document; do not introduce a
behavior change solely to match prose.

Keep detailed MembershipTerm extensions, Cases, and Claims outside this collection
unless the user expands its scope. Preserve existing user changes. Do not record
real customer details, card credentials, tokens, or secrets in documentation.
