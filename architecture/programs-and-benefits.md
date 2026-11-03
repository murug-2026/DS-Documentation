# Programs, term variants, and benefit configuration

[Architecture index](../membership-architecture.md) · Reviewed 2026-10-23

## A term's different dimensions

`MembershipTerm` is one model, not a separate model for each product category.
A term's program, benefit settings, pricing, renewal settings, status, and payment
records describe independent dimensions of the enrollment.

| Dimension | Representation | Meaning |
| --- | --- | --- |
| Commercial arrangement | `Program.program_type` | Retail or wholesale; this field does not identify Roadside versus ancillary coverage. |
| Coverage | Program's service and ancillary benefit records | Services/products and limits configured for the offering. |
| Member's coverage selection | `MembershipTerm.benefit_group` and retail `pricing` | Selects applicable coverage groups/product classes in benefit lookup. |
| Retail price and billing | `ProgramRetailPricing` | Price, program group, product class, benefit group, and billing frequency. |
| Duration and renewal | `ProgramTerm`, term dates, and `is_auto_renewal` | Program duration/renewal policy and the specific member's enrollment period/renewal choice. Billing frequency and term duration are different concepts. |
| Lifecycle | `MembershipTerm.status` | Active, scheduled, cancelled, and other configured statuses; not a coverage category. |
| Payment | Associated `MemberPaymentInfo` records | Payment method, card/token metadata, active/preferred flags, and expiry; independent of coverage classification. |

Sources: [Program and program configuration](/home/projectds/projects/compass/usac/usac/models/programs.py),
[MembershipTerm and service_benefit_limits](/home/projectds/projects/compass/usac/usac/models/member.py).

## What “benefit configuration” means

Benefits are configured against the **Program**, not copied into each term.
The term's coverage selection is used to resolve applicable benefit records.
The phrase “has benefits” is too broad to describe a particular relationship:
Roadside and ancillary benefits use different models.

| Model | Relationship from Program | Configures |
| --- | --- | --- |
| `ServiceBenefitAmount` | `program_servicebenefitamount` | Roadside service type, benefit group/product class, limit type, dollar limit, and mileage limit. |
| `ServiceAncillaryBenefitAmount` | `program_serviceancillarybenefitamount` | Ancillary service, optional product benefit, benefit group/product class, benefit limit, dollar amount, and incident limits. |
| `ServiceAncillaryBenefitLimit` | Separate program foreign key | Ancillary service incident limits by benefit group/product class. |

`ServiceBenefitAmount` is explicitly labelled **Road Side Service Benefit Limits**
in its model. Its service examples include tow, jump-start, fuel/fluid delivery,
lockout, tire change, and winch. `program_servicebenefitamount` is a reverse
relationship name, not a model named `ProgramServiceBenefitAmount`.

Source: [service benefit models](/home/projectds/projects/compass/usac/usac/models/service.py).

## Identifying Roadside programs

The existing `MembershipTerm.has_ers_program` helper identifies associated Chevron
programs through the presence of `ServiceBenefitAmount` records. The new payment
reuse helper `_active_roadside_terms` follows that convention and additionally
requires the benefit record to be non-deleted.

This is a configuration-based classification. It does not inspect program names,
program codes, retail/wholesale type, or payment methods. It also does not prove
that every service is covered: a configured limit can mean `NOT_COVERED`, and the
payment-reuse lookup does not resolve benefits for the particular covered person.

| Program configuration | Included by payment-reuse source-term lookup? |
| --- | --- |
| At least one non-deleted Roadside benefit record | Yes, if the other source-term conditions pass |
| Ancillary benefits only | No |
| Both Roadside and ancillary benefits | Yes, if the other conditions pass |
| No non-deleted Roadside benefit record | No |

The table describes what the lookup does, not which combinations exist in every
client's data. A term with no reusable card can still be listed with an
unavailability explanation; Roadside classification does not establish card
eligibility.

Sources: [has_ers_program](/home/projectds/projects/compass/usac/usac/models/member.py),
[payment-reuse source selection](/home/projectds/projects/compass/usac/usac/code/business/enrollment_payment.py).

## CS-3398 program configuration maintenance

Reviewed against the workspace on **2026-10-23**. These are maintenance scripts,
not an automatically deployed change to enrollment or runtime benefit lookup.

- [CS-3398_confie_program_cloning.py](/home/projectds/projects/compass/usac/scripts/programs/CS-3398_confie_program_cloning.py)
  exposes one atomic call, `clone_program(program_id, new_program_code, user)`,
  returning `{table_name: {old_primary_key: new_primary_key}}`. It creates a new
  program within the same client and copies active, explicitly
  owned configuration. It copies program-specific phones, services, agency
  mappings and direct-to-consumer fees in addition to term, pricing, benefit and
  fulfillment settings. Fulfillment schedules and source-owned foreign keys
  are remapped to copied parents. A self-renewal target becomes the new program;
  other renewal targets remain unchanged. Incoming renewal references owned by
  other programs are not traversed. Customer, enrollment, payment, case, claim,
  incident and registration records are excluded.
- Shared lookup definitions, PDF sets and generic terms PDFs are reused. A
  program's status, visibility, dispatch/external codes and document contents
  retain the source values and must be reviewed for the intended offering.
  Audit/deletion metadata is reset on newly created records. Existing targets
  are rejected. Source QBE mappings block cloning because the current integration
  resolves by state/company without a program or soft-delete filter; unchanged
  copies would make that lookup ambiguous.
- [CS-3398_confie_program_changes.py](/home/projectds/projects/compass/usac/scripts/programs/CS-3398_confie_program_changes.py)
  keeps the business rules in explicit if/elif branches at the top, followed by
  a numbered workflow: validate rows, clone affected programs once, update each
  benefit, carry incident limits and retire sources. CSV/database helpers follow
  below. It reads the supplied client/program/benefit CSV as a source snapshot. Duplicate
  program headers are handled by column position: the first program column
  (position 3 in the supplied layout) is the code; position 4 defaults to the
  program name. Both positions can be overridden. Client/program names and all benefit values are checked
  against current data. CSV Status is the program status and every row must be
  ACTIVE (case-insensitive, surrounding whitespace ignored). Non-ACTIVE or blank
  statuses reject the batch rather than silently skipping rows. The program must
  still have an undeleted ACTIVE status definition in the database. The former
  `--row-status` selector has been removed.
- The confirmed rule is category-driven. A program with selected category 2
  (Cosmetic Services) or category 3 (Vehicle Maintenance - EGV) replacements is
  cloned once, all selected changes are applied to its clone, and the source
  Program is then soft-deleted with audit fields in the same transaction. The CSV
  `new_program_code` column supplies the new codes; `--clone-map` is removed.
  Filling one row per source program is sufficient. Repeated values must agree,
  different sources cannot share a new code, and programs with only category 1
  changes must leave the column blank. The cleaned CSV includes this empty field
  for the user to populate. Category 1 alone
  (BENEFIT HUB to MemberDeals) updates the existing program in place. On a mixed
  program category 1 changes go to the clone too, regardless of CSV row order.
  The former `--mode` option has been removed; CSV Status does not trigger cloning.
  Already soft-deleted source programs are rejected to prevent repeat cloning.
- The changes script processes all clients listed in the CSV by default. Optional
  `--client-code` filters to one client's rows before validation, so the combined
  file need not be split. Every row's program is checked against that row's client.
- BENEFIT HUB rows are mapped to existing MemberDeals service/product definitions.
  Selected Cosmetic Services and Vehicle Maintenance - EGV rows are soft-deleted
  and replaced with DS rows. Other fields are retained, except Paintless Dent
  Repair - DS becomes Dollar Limit/$500 and Windshield Repair - DS becomes
  Dollar Limit/$150. These row replacements occur on the clone; the source's
  benefit rows are retained for existing members. Global lookup names are never
  renamed. Missing or ambiguous
  target definitions block the batch instead of creating shared definitions.
- Matching service-level ancillary incident limits are carried to the new
  service. Old limits remain active when untargeted products still use the old
  service. Conflicting limits or duplicate coverage block the batch.
- Both scripts preview by default and write only with `--commit`. The master
  validates the selected batch before writing, and all writes run in one atomic
  transaction, including source program soft deletion after successful changes.
  Repeated category 1 updates in place are recognized; an existing clone target
  or already retired source is rejected rather than cloned again. No live database
  migration has been executed as part of creating these scripts.

- The changes script now writes a durable `<input-stem>_rollback.csv` before
  committing Apply. `--rollback-csv PATH` selects another backup path; existing
  backups are never overwritten. This is a generated before/after snapshot,
  not a duplicate of the source CSV. It includes database/batch identity, source
  program IDs, every scoped configuration row (including deleted rows), generated
  IDs, Program M2M through rows, row counts and checksums. A backup write failure
  rolls back the database changes. A prepared backup may remain after a failed
  database commit; rollback compares original/applied states before proceeding.
- [CS-3398_confie_program_rollback.py](/home/projectds/projects/compass/usac/scripts/programs/CS-3398_confie_program_rollback.py)
  implements `--rollback --csv <backup>` preview and `--commit` reversal. It restores
  exact original fields/audit timestamps, un-retires source programs, and hard-deletes
  only this batch's unused created configuration, all in one transaction. It refuses
  later configuration changes and external incoming references, including deleted
  enrollments and renewal references owned by other programs. It also checks Django's
  deletion graph and verifies the final restored scope. Repeat rollback is a no-op;
  another apply run requires a new backup path. Original customer/history links and
  shared lookups/PDFs are retained. This does not roll back untracked historical runs.
- An already converted category 1 row on a subsequently cloned mixed program now
  reports its copied benefit ID, matching the actual cloned configuration. Incident
  limit selection computes the selected benefit IDs once per batch.

## Simpler CS-3398 batch entry point

Reviewed against the workspace on **2026-10-26**. The original maintenance scripts
remain available. The alternative
[CS-3398_batch_update_confie_programs.py](/home/projectds/projects/compass/usac/scripts/programs/CS-3398_batch_update_confie_programs.py)
uses the original
[CS-3398_confie_program_changes_source_data_cleaned.csv](/home/projectds/projects/compass/usac/scripts/programs/CS-3398_confie_program_changes_source_data_cleaned.csv)
with three appended columns: `expected_action`, `action_note`, and `changes`.
All original cells and the order of all 2,011 rows were preserved for cross-checking.
The grouped JSON input has been removed in favor of explicit, repeated CSV rows.

- `expected_action` is `update` or `clone_and_replace`.
- `action_note` explains the expected action for reviewers; it is not executed.
- `changes` contains only new values, separated by `|`, for example:
  `ancillary_service=Cosmetic Services - DS | product_benefit=Paintless Dent Repair - DS | benefit_limit=Dollar Limit | dollar_amount=500.00`.
  Whitespace around separators is ignored. Unknown/duplicate fields, missing values
  or malformed pairs are rejected. Omitted fields remain unchanged; `NULL` clears
  a nullable field. Coverage group/class changes are not supported by this batch.

Program/client codes and service/product with optional coverage or `benefit_id`
identify the intended source benefit. Each selector must resolve to one row.
Original names, status, amounts and limits remain in the file for comparison;
the batch reads current business values and original rollback state from the DB.
It verifies the source program's client, undeleted state and ACTIVE status.
Shared lookup definitions must already exist. `--client-code` filters the batch.

The CSV covers 946 programs, including 593 requiring clones. Their original
`new_program_code` cells are still blank and must be populated with approved codes
before preview/apply of selected clone programs. One filled row per source is
sufficient; repeated codes must agree. Update-only programs must leave it blank.
If any row requests cloning, the program is cloned once and all its rows apply to
that clone. Clone rows replace copied benefits; update rows modify them in place.
After success the cloned source is retired; existing member/history links remain.
Service incident limits follow service changes; product/amount-only changes keep
current service limits.

[cs3398_program_cloning.py](/home/projectds/projects/compass/usac/scripts/programs/cs3398_program_cloning.py)
remains the separate cloning helper with explicit owned configuration and reference
remapping. The batch continues to use the existing rollback module for durable
before/after CSV snapshots and exact restoration. Every source scope is captured
before the first write. Backup publication precedes the single transaction's commit;
backup failure reverses the writes and existing backups are never overwritten.
Preview shows actual current/new values and the CSV line/action note for each change.

From the `usac` directory:

```bash
python scripts/programs/CS-3398_batch_update_confie_programs.py \
  --csv scripts/programs/CS-3398_confie_program_changes_source_data_cleaned.csv --user-id USER_ID
```

Preview is the default; add `--commit` to apply. The default backup is
`<input-stem>_rollback.csv`; `--rollback-csv PATH` overrides it.
Use `--rollback --csv BACKUP` to preview reversal, adding `--commit` to execute it.

The isolated SQLite runner
[CS-3398_batch_tests.py](/home/projectds/projects/compass/usac/scripts/programs/tests/CS-3398_batch_tests.py)
passes 24 checks, including CSV notes/pipe parsing, malformed cells, mixed-program
cloning, all ticket rules, current DB values, omitted/NULL fields, client filtering,
ambiguous selectors, service limits, exact/guarded rollback and backup failures.
Every generated CSV action was checked against the prior requested rules, and all
original cells/row order were compared with the unchanged input backup. No live
program data was changed during implementation or verification.

## Earlier CSV entry points

Example preview from the `usac` directory, assuming column 4 is the program name
and SOURCE_CODE has selected category 2/3 changes:

```bash
python scripts/programs/CS-3398_confie_program_changes.py \
  --csv /path/to/input.csv --user-id USER_ID
```

Populate approved new codes in the CSV before the database preview. The preview
must also validate source records, target definitions, unused codes and ancillary
incident limits; QBE routing mappings remain a blocker where present. Incoming
feed/enrollment mappings still need to route new enrollments to the new codes:
some current lookups resolve the old code without a deleted filter. PDFs remain
shared until separately revised. No live database preview has yet been performed.
CSV Status means program status;
all input rows must specify ACTIVE, and newly cloned programs retain that status.
Use `--help` for the standalone clone command. The standalone clone utility
creates a program only; source retirement is the changes script's responsibility.
The former `CS-3398_clone_programs.py` and `CS-3398_update_benefits.py` files have
been replaced by these two Confie-specific names.

Validation uses an isolated SQLite regression runner with actual configuration
model class definitions and fixtures for unrelated shared/transaction models:
[CS-3398_tests.py](/home/projectds/projects/compass/usac/scripts/programs/tests/CS-3398_tests.py).
It exercises CSV new-code validation, multi-client processing and client filtering,
category-driven clone decisions (including mixed and separate
programs), replacements, audit fields, preview, reruns, rollback of source retirement,
dependency remapping, renewal direction, document reuse and preservation of customer/history
links. Rollback tests cover exact restoration for both action sets and every declared
owned model, M2M/schedule cleanup, preview/idempotency, backup integrity/failure,
intervening edits, external dependencies and full transaction reversal on failure. This does not establish deployment status or verify production data.
