# Compass membership architecture

This is the entry point for the membership architecture documentation. Reviewed
against the current workspace on **2026-10-17**; some described enrollment changes
are still working-tree changes rather than deployed behavior.

Compass separates client organizations, program definitions, membership accounts,
enrollment periods, covered people, and application login accounts. A covered
person does not require a login. One membership can have several terms, each
selecting its own program.

## Documentation map

| Document | Covers |
| --- | --- |
| [Core membership model](architecture/core-model.md) | Entity definitions, relationships, original assumptions, client/member identity, and active status |
| [People and access](architecture/people-and-access.md) | Covered people versus contacts and login users; client associations and feature permission resolution |
| [Programs and benefits](architecture/programs-and-benefits.md) | Roadside and ancillary configuration, retail/wholesale, pricing, and renewal |
| [Enrollment payment reuse](architecture/enrollment-payment-reuse.md) | Current eligibility, source terms, card validation, UI, backend flow, and future direction |
| [Queue flow](queue-flow.md) | Queue types, UI and job creation triggers, status changes, completion, visibility, and local testing |

## Reading and maintenance rules

- Distinguish a database relationship from a business workflow rule. What the
  schema permits is not necessarily supported by enrollment.
- Use explicit names: `Membership` is an account, `MembershipTerm` is an enrollment
  period, and `MemberPerson` is a covered person. There is no `Member` model here.
- Distinguish `ProgramTerm` configuration from a member's `MembershipTerm`.
- Treat source code as authoritative. Verify relevant implementations before
  changing behavior and update the affected document when architecture changes.
- Keep confirmed current behavior separate from proposed future changes. Do not
  turn a documentation statement into a new restriction without authorization.
- Use synthetic examples; do not store real card credentials or customer details.

## Scope

The collection covers clients, programs, memberships, terms, people, users,
benefit configuration, and the enrollment payment-reuse feature discussed during
implementation. Detailed MembershipTerm extension models, Cases, and Claims are
outside this collection's current scope.

Local documentation guidance is recorded in [AGENTS.md](AGENTS.md) so future agents
working in this repository consult and maintain these notes.

## Related operational documentation

[Queue flow](queue-flow.md) was reviewed against current source on **2026-10-29**.
It covers queue lifecycle and the case-feedback link; it does not extend this
collection into detailed Cases or Claims architecture.
