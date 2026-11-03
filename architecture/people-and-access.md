# People, login accounts, and access

[Architecture index](../membership-architecture.md) · Reviewed 2026-10-20

## Covered people and identity across enrollments

`MemberPerson` stores the person's details directly; it does not inherit from
`Person` and has no foreign key to it. Its direct membership and term links are
both nullable. When both are populated, the models do not enforce that
`member_person.membership` matches `member_person.membership_term.membership`.

One term can have several covered people. `authorized_user_type` describes a
coverage role such as `PRIMARY` or `SPOUSE`; “user” here does not refer to an
authentication account. There is no uniqueness constraint enforcing exactly one
primary person per term. The `primary_member_person` helper warns when multiple
active primary people exist and has fallback logic when no primary is found.

A single `MemberPerson` row can directly reference at most one membership and
one term. Representing the same human under several terms or clients therefore
requires separate person records for those direct links. These models contain
no shared identity key connecting those records. Matching names or birth dates
does not establish a database relationship.

`MembershipTermPerson` is explicitly described in its model docstring as a
staging table for migration from Quasar. It stores links to membership, term,
and member person; it should not be treated as evidence of a canonical shared
person identity across live enrollments.

Sources: [MemberPerson](/home/projectds/projects/compass/usac/usac/models/member.py#L1911),
[primary person selection](/home/projectds/projects/compass/usac/usac/models/member.py#L630), and
[MembershipTermPerson](/home/projectds/projects/compass/usac/usac/models/member.py#L2697).

## Login accounts and client associations

`User.clients` associates an account with zero or more clients through
`UserClient`. `UserProfile.client` provides a separate optional client on the
profile. These are independent relationships: the models do not synchronize them
or require the profile client to appear in `User.clients`. In
`User.get_list_of_agencies`, an explicit client code takes precedence, followed
by the profile client when available; agency selection also depends on user type.
Client associations alone are therefore not a complete description of access
control.

`Membership`, `MembershipTerm`, `MemberPerson`, `Person`, `Client`, and `Program`
inherit `BaseModel` audit fields: required `created_by`, optional
`last_updated_by` and `deleted_by`, timestamps, and a `deleted` flag. One staff
or integration account can create many member records. Those audit links do not
make that account the member or establish a member login.

The creation helpers reinforce this distinction: `create_new_membership`,
`create_new_membershipterm`, and `create_new_memberperson` accept an acting
`user` and assign it to `created_by`; they do not create a login account for
the covered person.

Sources: [User and agency selection](/home/projectds/projects/compass/usac/usac/models/user.py#L9),
[UserProfile](/home/projectds/projects/compass/usac/usac/models/user.py#L203),
[UserClient](/home/projectds/projects/compass/usac/usac/models/user.py#L253),
[BaseModel](/home/projectds/projects/compass/usac/usac/models/base.py#L15), and
[member creation helpers](/home/projectds/projects/compass/usac/usac/code/data/member.py#L449).

## System feature permissions

`SystemModuleFeature` declares a named capability. `UserSystemModuleFeature` and
`UserTypeSystemModuleFeature` assign it to a user or user type.
`User.has_system_module_feature(feature_key)` currently resolves:

1. Find the feature by its unique key; missing means false.
2. Find the newest enabled, non-deleted user assignment; if found, return true.
3. Otherwise find the newest enabled, non-deleted user-type assignment; if found,
   return true.
4. Otherwise return false.

Disabled user assignments are filtered out; they do **not** explicitly deny an
otherwise enabled user-type grant. The feature lookup itself does not filter the
feature's inherited `deleted` flag. These describe current implementation details,
not recommendations for changing the permission system.

Creating a feature does not grant it. Payment reuse uses the current key
`ENROLLMENT_CLONE_PAYMENT`; the earlier discussed name
`OFFLINE_ENROLLMENT_CLONE_PAYMENT` is not the key checked by the implementation.

Sources: [feature declarations](/home/projectds/projects/compass/usac/usac/models/system.py),
[user and user-type assignments and resolver](/home/projectds/projects/compass/usac/usac/models/user.py),
[payment reuse](enrollment-payment-reuse.md).


## WinBack queue cleanup

The CS-3413 enhancement is implemented in the current working tree; deployment
and live feature/event-type configuration have not been verified.

`WINBACK_QUEUE_CLEANIUP` (spelling intentional) controls the **WinBack Queue
Cleanup** action on `/queue/`. Creating the feature does not grant it. Assign it
through the existing user or user-type feature administration. The setup script
only creates the feature if missing; it does not grant access or change No Sale
classification.

The action is visible only for an authenticated, active user with that feature,
with valid applied search filters selecting `WINBACK` and `is_completed=False`.
The queue results and POST processing enforce these conditions on the server.
The new optional Created On date range has no fixed lookback restriction and is
applied to the normal queue search and cleanup requests.

The `/queue/` search results display checkboxes and inline resolution notes and
Mark No Sale controls, without a modal. Entries on the displayed page are selected
by default. Select all clears or restores the selection for that page only;
individual entries can also be unselected. Checkboxes explicitly retain native
appearance because the global theme otherwise hides them. Changing any filter
hides the controls until the applied filters are restored or Search is run again.
After processing, results refresh to the first page, with selection cleared so
another action requires explicit selection. The WinBack report is unchanged.

Selected IDs are submitted in sequential batches of at most 500, each with the
required resolution notes; only checked IDs on the displayed page are sent. The server reapplies all
search filters and the existing queue client/program exclusions, locks and
rechecks each selected entry, and skips entries that no longer match. It sets
`is_completed`, `completed_datetime`, resolution notes and the acting user, then
records both a `WINBACK` queue-history event and a `WBFD_NOSALE` outcome event.
Completion and both events are atomic per entry. Records are retained without
soft deletion; completed entries disappear from the open queue search.

Processing requires nondeleted `WINBACK` and `WBFD_NOSALE` event types, with
No Sale assigned to `WINBACK_COMPLETED`, matching the existing individual
`winback_notes` completion condition. Seed data uses the legacy `WINBACK_FDWR`
category, so the bulk action reports a configuration error before changes if
that category is still in use. No automatic reclassification is performed.

Sources: [queue filtering and cleanup view](/home/projectds/projects/compass/usac/usac/views.py),
[inline cleanup controls](/home/projectds/projects/compass/usac/usac/templates/usac/queues/winback-cleanup-controls.html),
[feature setup](/home/projectds/projects/compass/usac/scripts/system_settings/CS_3413_winback_queue_cleanup_system_module_setup.py).


### Reviewed CSV Apply and JSON Rollback lists

The one-time CS-3413 script uses `CS-3413-no_sale_apply_list.csv` for reviewed
input and `CS-3413-no_sale_rollback_list.json` for generated undo data. The former
Python dictionary list files have been replaced. Empty templates are included
in the queue scripts directory. `generate_apply_list` exports aged eligible
records with a batch ID, frozen cutoff, queue/account/term identity, creation
timestamp, and review-only account/program columns. Reviewers remove whole CSV
rows to select test records, preserving the header and identity metadata.
Duplicate IDs, mixed batches, invalid identifiers/timestamps and incomplete
headers are rejected before processing.

`apply_no_sale` rechecks the selected records and saves their prior completion
fields, resolution notes and applied state in each No Sale event's snapshot,
alongside the specific queue-history event ID. The snapshot and both events are
committed atomically with completion. Successful changes populate the Rollback
JSON with nested prior/applied state and both event IDs, keyed by queue ID.
Existing rollback records for the same batch are retained. Another batch cannot
silently overwrite either list. The files are parsed as CSV and JSON data.
If JSON export fails after a database commit, `generate_rollback_list` reconstructs
it from authoritative committed snapshots.

`revert_no_sale` defaults to the Rollback JSON. It verifies exact event IDs and
saved states against database history, refuses later queue changes or later
queue/account-completion events, restores prior business fields, soft-deletes
only the two Apply events, and records a reversal. The reverting user is recorded
as the actor; original audit values remain in the snapshot. Repeated requests
skip already processed entries; the same Apply CSV can be applied again after
reversal. Both operations default to preview mode. Older UI/script actions
without an undo snapshot cannot be reversed this way. No schema changes are
required.

Source: [CS-3413 cleanup script](/home/projectds/projects/compass/usac/scripts/queue/CS-3413-mark_aged_winback_no_sale.py).
