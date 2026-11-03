# Compass queue flow

Reviewed against the current workspace on **2026-10-29**. These are source-verified behaviors, not confirmation that a scheduled job is deployed or that database lookup data matches seed data.

[Membership architecture](membership-architecture.md)

## What a queue record represents

A queue item is a `QueueMemberMaintenance` record attached to a `Membership` and a `MembershipTerm`. It is follow-up work, not a scheduler job. Management commands may generate or resolve these records.

There is no queue status enum such as Pending/In Progress/Failed. The principal state is `is_completed`, with independent `deleted`, notes, resolution notes, priority, due date, and optional assignment fields. A billing installment becoming FAILED or a membership becoming EXPIRED does not itself define the queue's state.

| Record state | Meaning |
| --- | --- |
| `is_completed=False`, `deleted=False` | Open follow-up item |
| `is_completed=True`, `deleted=False` | Completed item; can remain visible when completed items are requested |
| `deleted=True` | Soft-deleted item; omitted from the main queue and WINBACK report regardless of completion |

The shared creator defaults priority to NORMAL when omitted, sets the due date to now, creates the item incomplete, and creates a linked WINBACK history event for WINBACK items. It does not enforce one open item per term; repeated calls can create duplicates. Admin creation bypasses this helper.

Sources: [queue model](/home/projectds/projects/compass/usac/usac/models/queue.py), [base audit fields](/home/projectds/projects/compass/usac/usac/models/base.py), [DataLayerQueue](/home/projectds/projects/compass/usac/usac/code/data/queue.py).

## Queue types

The following types are defined in `default_data.py`. Actual lookup records and their `show`/`deleted` flags are database configuration. The queue type dropdown selects only types with `show=True`, `deleted=False`; a hidden type can still have records.

| Code | Seed description | Seed shown? | Verified standard creation flow |
| --- | --- | --- | --- |
| `MAINTENANCE` | Maintenance | Yes | Member → Remarks → Add new Special Request → Save |
| `WINBACK` | Winback | Yes | Membership maintenance, periodic billing, payment-response processing; detailed below |
| `CASEFEEDBACK` | Case Feedback | Yes | Case → Feedback → Add Case Feedback → submit |
| `PAYMENT` | Payment | No | CHECK/INVOICE refund reminder, or refund processing/transaction-record errors |
| `LOCATEADDR` | Locate Address | No | No dedicated current application creator found |
| `MEMBERNOTE` | Member Note | No | No dedicated current application creator found |
| `MEMBERNOTFOUND` | Member Not Found | No | No dedicated current application creator found |
| `OCALL` | Outbound Call | No | No dedicated current application creator found |

All types can also be selected through permitted Django admin operations; this is separate from the standard member/case UI.

Sources: [load_data_for_TypeActivity](/home/projectds/projects/compass/usac/usac/default_data.py:3607), [queue filters](/home/projectds/projects/compass/usac/usac/views.py:7810), [admin registration](/home/projectds/projects/compass/usac/usac/admin.py:5392).

## UI entry points

| UI location | Route / behavior |
| --- | --- |
| Queues | `/queue/`; filter/search/export and follow a member link. No direct create, edit, or complete controls in this list template. |
| WINBACK report | `/view-report/WINBACK_QUEUE`; reports existing WINBACK records, with report-specific joins and completion/date filters. It does not create them. |
| Membership page | `/view/<membership_id>/<membership_term_id>/`; choose the exact term, then Remarks. |
| Remarks → Outcome / Notes → Save | `/ajax/winback_notes/`; records an outcome and updates an existing open WINBACK item, if found. |
| Remarks → Add new Special Request | POST `/ajax/create_special_request/`; always creates MAINTENANCE. |
| Remarks → Pending Requests → pencil | `/ajax/update_special_req/`; edits the selected queue item's notes. |
| Remarks → Pending Requests → checkmark → resolution → Complete | `/ajax/special_req_completed/`; completes the selected queue record. |
| Case → Feedback → Add Case Feedback | `/ajax/create_case_feedback/`; creates CASEFEEDBACK and a linked CaseFeedback record. |
| Membership → Change/Cancel → Cancel → refund selection | Refund processing can generate PAYMENT reminders/errors; cancellation itself does not create WINBACK. |
| Django admin → Member Maintenance Queue | `/admin/usac/queuemembermaintenance/`; add/edit records, including completion, type, priority, due date, assignment, and soft-delete fields, subject to admin permissions. |

Pending/Completed Requests on Remarks are filtered by the current term and completion flag, **not activity type**. They can contain WINBACK, PAYMENT, or CASEFEEDBACK as well as MAINTENANCE. These member-page querysets do not consistently exclude soft-deleted records; the main queue does.

Sources: [routes](/home/projectds/projects/compass/usac/usac/urls.py), [member context](/home/projectds/projects/compass/usac/usac/views.py:3432), [Remarks layout](/home/projectds/projects/compass/usac/usac/templates/usac/mem-view/main-tab.html:224), [queue list template](/home/projectds/projects/compass/usac/usac/templates/usac/mat-admin/queue-maintenance.html), [special request UI](/home/projectds/projects/compass/usac/usac/templates/usac/mem-view/special_request.html), [case feedback UI](/home/projectds/projects/compass/usac/usac/templates/usac/case/case-feedback.html).

## Creation conditions

### Standard member and case actions

| Type | Exact triggering condition | UI / processing path |
| --- | --- | --- |
| MAINTENANCE | A member's special request is submitted successfully | Membership → Remarks → Add new Special Request; the handler hardcodes MAINTENANCE |
| CASEFEEDBACK | Feedback passes handler validation, including phone/email and classification lookups | Case → Feedback → Add Case Feedback; uses supplied membership and term links |
| PAYMENT | Refund by CHECK or INVOICE, unless this is a chargeback | A refund operation, including Change/Cancel refund processing, calls `refund_money_for_member`; creates a reminder to send the refund |
| PAYMENT | Error creating the refund transaction record, or an exception in refund processing | Same refund service; queue notes carry the error; not every ordinary gateway decline necessarily creates a queue record |

Sources: [special request creator](/home/projectds/projects/compass/usac/usac/views.py:14437), [feedback creator](/home/projectds/projects/compass/usac/usac/views.py:14650), [refund service](/home/projectds/projects/compass/usac/usac/code/business/payment.py:863).

### WINBACK creation

| Process | Conditions / branch | UI connection |
| --- | --- | --- |
| CHECK follow-up during term renewal | Current term ACTIVE, not deleted, end date on/before job run date, program renewal enabled, member `is_auto_renewal=True`, latest payment method CHECK, and no qualifying new active term | Enrollment / member payment setup → CHECK; member details → AutoRenews at end of the Term on; processed by `run_maintenance_membership`. CHECK enrollment initially sets member auto-renew off. |
| CHECK follow-up during periodic billing | Term/installment returned by the billing eligibility query, pricing present, latest payment method CHECK; helper creates WINBACK unless a qualifying new active term exists | Enrollment/payment setup → CHECK; leave a later installment pending; processed by `run_membership_billing` |
| Activation payment failure | Not-deleted NOT-YET-ACTIVE term for the client; today's date within term range; RETAIL program; no approved RENEW/ENROLLMENT/AUTORENEW payment; payment attempt has failed processing/transaction result | Future-dated enrollment or a future term prepares the data; maintenance activation processing creates WINBACK later |
| Activation transaction status FAILED | Separate explicit FAILED branch attempts to cancel and queue the term | Same activation flow; current code references undefined `new_pricing.amount` before queue creation, so this branch can error before creating the item |
| Retail renewal payment failure | Renewal attempts payment and enters explicit FAILED status or payment-processing error branch | Automatic renewal; shared renewal service is also called by Financials → Pre-Pay Next Term |
| Pricing failure | Periodic billing encounters `mterm.pricing is None`; retail renewal contains a missing-new-pricing queue branch | Background processing / configuration failure. Some renewal lookup failures return early or raise before that queue branch; missing pricing does not guarantee WINBACK creation. |
| Citi response failure | Hard decline, or a soft decline that reaches retry limit | Payment response file processing; no immediate ordinary UI creation action |
| Citi transfer exception | SPECIAL_CASE_PT_TRANSFER and retry limit exceeded or the code's `payment_changed` condition | Response processing. Current `payment_changed` expression compares method equality, despite the variable name; do not infer behavior from its label alone. |
| Synchrony response failure | Hard decline, or soft decline limit reached | Response file processing |
| Exxon suspended follow-up | `run_misc.send_exxon_suspended_to_winback`: selected SUSPENDED terms for configured client, EXXON payment, and transaction-date condition | Maintenance utility converts term to CANCEL-PAYMENT-DECLINED and creates WINBACK; no direct member UI trigger |

The shared CHECK helper uses notes: “Payment mode is Check. Please contact Member to renew Term.” For periodic billing, the job also marks the installment FAILED and expires the term/associated membership entities. For CHECK term renewal, it expires the old term without creating a new term automatically.

Sources: [CHECK helper](/home/projectds/projects/compass/usac/usac/code/business/member.py:64), [activation](/home/projectds/projects/compass/usac/usac/code/business/member.py:454), [renewal selection](/home/projectds/projects/compass/usac/usac/code/business/member.py:801), [retail renewal](/home/projectds/projects/compass/usac/usac/code/business/member.py:1093), [periodic billing](/home/projectds/projects/compass/usac/usac/code/business/member.py:2107), [billing query loader](/home/projectds/projects/compass/usac/usac/code/data/member.py:218), [Citi processing](/home/projectds/projects/compass/usac/usac/code/business/citi.py:704), [Synchrony processing](/home/projectds/projects/compass/usac/usac/code/business/synchrony.py:1553), [Exxon utility](/home/projectds/projects/compass/usac/usac/management/commands/run_misc.py:111).

### Actions that do not directly create WINBACK

- Saving ordinary enrollment, including a successful initial CHECK payment. CHECK enrollment records payment received and marks the initial installment Collected; this is not confirmation of bank clearing.
- Manual Change/Cancel cancellation, retention, or renewal override. Refund-related PAYMENT items are a separate possibility.
- Selecting an ordinary noncompletion Remarks outcome or callback. This records an event/updates existing WINBACK notes; it does not create a queue item.
- Opening `/queue/` or the WINBACK report.

Financials → Pre-Pay Next Term is an indirect UI call into the shared retail renewal service, not a dedicated “create WINBACK” control. Its normal CHECK flow records prepayment; error paths require inspecting the actual failed branch.

Sources: [enrollment payment/transaction handling](/home/projectds/projects/compass/usac/usac/views.py:1820), [prepaid handler](/home/projectds/projects/compass/usac/usac/views.py:5567), [cancel handler](/home/projectds/projects/compass/usac/usac/views.py:8247), [cancel service](/home/projectds/projects/compass/usac/usac/code/data/member.py:1020).

## Changes to existing items

| Trigger | Record changes | Completion effect / UI |
| --- | --- | --- |
| Remarks → Pending Requests → Edit → Save | Changes `notes`, updater, update time; WINBACK gets linked history event | Remains in its existing completion state; does not complete it despite a misleading handler log message |
| Remarks → noncompletion Outcome → Save | First incomplete WINBACK item for this term gets notes/updater changes and linked WINBACK history; selected outcome also gets a membership event | Remains open; ordinary callback does not change `due_datetime` or create a new item; callback date/time are embedded in notes |
| Remarks → WINBACK_EMAIL_ADMIN outcome → Save | Existing item's notes/history updated, and configured email sent | UI asks about completion, but backend does **not** set `is_completed=True` for this category |
| Admin edit | Can change queue type, notes/resolution, priority, dates, assignment, completion and deletion fields | Direct model edit; no shared WINBACK history helper or automatic outcome event |
| Chevron ERS precedence inside shared creator | For client code 10242, existing open non-ERS WINBACK records with same account number are soft-deleted when creating for an ERS term; a new non-ERS item is created soft-deleted if an open ERS WINBACK exists | Hidden rather than completed. ERS means the program has service benefit amount records. This special handling is not guarded by the incoming activity type, so other queue creation calls for that client can invoke it too. |
| Configured list exclusion | Main queue/report exclude configured client/program codes | Visibility changes only; database item is not completed or deleted |

No automatic assignment workflow or standard “reopen completed item” control was found in the application UI. Assignment fields exist in the model/admin. Generic admin edits can reopen an item by clearing completion; this does not automatically reconcile dates, notes, or events.

Sources: [update_special_req](/home/projectds/projects/compass/usac/usac/views.py:14510), [winback_notes](/home/projectds/projects/compass/usac/usac/views.py:15739), [Remarks JavaScript](/home/projectds/projects/compass/usac/usac/templates/usac/mem-view/member_winback.html), [Chevron rules](/home/projectds/projects/compass/usac/usac/code/data/queue.py:58), [queue exclusion](/home/projectds/projects/compass/usac/usac/views.py:12248), [report](/home/projectds/projects/compass/usac/usac/code/business/reports.py:15805).

## All verified standard completion paths

| Trigger | Which queue records are completed? | Fields / events |
| --- | --- | --- |
| Remarks → Pending Requests → checkmark → resolution → Complete | Selected record ID, any queue type | `is_completed=True`, resolution notes, completion time, updater; linked WINBACK history if type is WINBACK; no mandatory No Sale outcome |
| Remarks → Outcome from WINBACK_COMPLETED → Save | First incomplete WINBACK for the selected term | Completion flag, resolution notes, updater; linked WINBACK history plus chosen outcome event. This handler does **not** set `completed_datetime`. |
| Successful membership activation through `DataLayerMember.activate_membership` | All incomplete WINBACK entries whose term membership has the same `client_member_id` as the activated term | Completion flag/time, resolution “Member activated. Removing item from Queue.” and audit fields; linked history. Can resolve records on older terms. |
| Successful enrollment using the reopen flow | Same shared account-number match, for WINBACK | Completion flag/time and audit fields; “Member enrolled. Removing item from Queue.”; linked history |
| Admin edit → mark completed → Save | Selected record | Direct field save. Completion time and outcome/history are not automatically populated by the queue model/admin. |
| CS-3413 apply script | Non-deleted, incomplete WINBACK records with `created_on < now - 90 days`, bounded to IDs present when the run starts | Completion flag/time, fixed resolution, updater/time; creates linked WINBACK and WBFD_NOSALE events; saves rollback CSV |

Activation completion applies to callers of the activation service, including membership maintenance and applicable approved payment-response paths. Changing a status directly to ACTIVE through admin or a different updater is not equivalent to calling this service and does not guarantee queue completion.

The shared completion helper matches **account number only**, not exact membership ID, term ID, or client, and does not exclude `deleted=True`. It may therefore complete same-number entries beyond the viewed membership/client, including soft-deleted entries. Its completion events use the activated/reopened term, which may differ from the queue's original term.

The Remarks completion handler also omits `deleted=False` when selecting its first incomplete WINBACK, although dropdown eligibility checks exclude deleted items. With multiple items, one Save does not necessarily resolve every open item.

Sources: [generic completion](/home/projectds/projects/compass/usac/usac/views.py:14476), [outcome completion](/home/projectds/projects/compass/usac/usac/views.py:15739), [activation hook](/home/projectds/projects/compass/usac/usac/code/data/member.py:966), [shared completion helper](/home/projectds/projects/compass/usac/usac/code/data/queue.py:119), [reopen hook](/home/projectds/projects/compass/usac/usac/views.py:2320), [CS-3413 apply](/home/projectds/projects/compass/usac/scripts/queue/CS-3413-winback_batch_update_mark-no-sale.py).

## No Sale dropdown eligibility

The current term must have at least one WINBACK with `is_completed=False` and `deleted=False` for `WINBACK_COMPLETED` outcomes to appear. The membership itself need not be ACTIVE. Outcome records must also have `show=True`, `deleted=False` and category code beginning WINBACK.

Without that open queue, the backend excludes category exactly `WINBACK_COMPLETED`; it does not hide the entire Remarks form. After completing all qualifying entries, reload the page to refresh the choices. Another open item, including one younger than 90 days, keeps completion outcomes available after CS-3413.

This rule applies to No Sale only if `WBFD_NOSALE` actually belongs to `WINBACK_COMPLETED` in the database. Old seed data uses `WINBACK_FDWR`, while CS-3413 explicitly requires `WINBACK_COMPLETED`. Verify lookup configuration rather than assuming the seed matches the environment.

Sources: [term queue check](/home/projectds/projects/compass/usac/usac/models/member.py:1324), [outcome list](/home/projectds/projects/compass/usac/usac/views.py:3204), [legacy seed](/home/projectds/projects/compass/usac/usac/default_data.py:2945).

## One-time scripts and rollback

These are manual operations, not standard navigation or proof of currently scheduled work:

- CS-3413 rollback restores the saved queue completion flag, completion date, resolution and update audit values, and soft-deletes its two generated events. Previously open entries can become open again. It does not recreate memberships or reverse unrelated later actions.
- `scripts/payments/payix/reverse_mtd_2023_05_10.py` has reinstatement/reversal logic that completes all incomplete entries on the selected term. Despite its helper name, that queue update does not filter activity type or deletion state.
- `scripts/queue/queue1.py` and `scripts/memberships/union_privilege.py` contain targeted soft deletion of incomplete queue records; this is not completion.
- Legacy renewal/billing scripts, `scripts/batch-billing/citi/pending_transactions.py`, `scripts/payments/refunds_issue_ge.py`, and `scripts/memberships/create_new_payment_declient_membership_with_winback.py` contain explicit queue-creation calls. They require separate execution and are not normal UI entry points. Some legacy signatures are stale; do not treat these as supported test commands.

Sources: [CS-3413 rollback](/home/projectds/projects/compass/usac/scripts/queue/CS-3413-rollback_winback_batch_update_mark-no-sale.py), [Payix repair](/home/projectds/projects/compass/usac/scripts/payments/payix/reverse_mtd_2023_05_10.py:184), [queue cleanup](/home/projectds/projects/compass/usac/scripts/queue/queue1.py), [client cleanup](/home/projectds/projects/compass/usac/scripts/memberships/union_privilege.py).

## Local testing and limits

1. Enroll a synthetic retail CHECK member. Initial payment is recorded as received; the first installment is Collected. Use the Django application's local date for same-day testing; OS/browser date alone is not authoritative.
2. For periodic billing, choose a later pending installment and pass its due date as `--rundate` to `run_membership_billing`. Its CHECK branch can create WINBACK and expire the term without waiting a month.
3. For renewal, enable both program renewal and member auto-renew, avoid an activated prepaid next term, and run `run_maintenance_membership` with a date on/after the current term end date.
4. Both existing commands filter by client code, not exact membership/term ID. Counts mode previews selection; interactive mode stops on a non-Y answer rather than skipping a record. A single-term test needs a deliberately restricted harness/query or an explicit command enhancement; no built-in member/term CLI flags were found.
5. A newly generated WINBACK does not qualify for CS-3413 until older than 90 days. Apply/rollback scripts are separate manual operations.

Known enrollment limitation: for an ordinary future start date outside the special forced-payment activation path, enrollment disables immediate payment but leaves the proceed flag false and the message empty. It can render a blank error. Changing WSL timezone does not change an explicitly configured Django timezone. This is current behavior, not a proposed rule that future enrollments should fail.

Billing eligibility is loaded from the configured `QUERY_FOLDER/other/billing_job.sql`; that SQL was not present among tracked source files found in this review. The rows returned by that query determine exact due-date/status eligibility; this document does not invent additional predicates. Review the configured SQL in the target environment when diagnosing a term that is not selected.

Documentation-only review: no queues, memberships, payment transactions, scheduled jobs, or lookup configuration were modified. No proposed workflow changes are implemented by these notes.
