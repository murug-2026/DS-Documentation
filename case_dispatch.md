**Dispatch options are selected for the case; Compass does not display every integration as a separate provider button.** Towbook and Atlas sit behind the Compass vendor-dispatch route.

| UI route | Meaning |
|---|---|
| **Dispatch via Honk** | Submit the case to Honk’s network |
| **Dispatch via Compass** | Use vendors selected by Compass; integrated vendors can receive requests through Towbook or Atlas |
| **Dispatch via Quest** | Separate dispatch route, when enabled and eligible |
| **Find OON Vendors** | Out-of-network route, subject to availability and rate checks |

**Why you may see only Honk**

The backend calculates eligibility and the template hides unavailable routes. Relevant conditions include:

- Global Honk, Quest, and OON feature settings.
- Client/program restrictions on Compass dispatch.
- Whether eligible case vendors exist and their current statuses.
- OON availability and estimated-cost eligibility.
- Existing Honk/Quest requests and dispatch history.
- Automatic dispatch already being in progress.

For your particular case, Honk-only visibility could reflect those settings or vendor availability. **I have not queried the case/configuration, so I cannot identify its exact exclusion reason yet.**

Start with these code paths:

| Order | Code | What to follow |
|---|---|---|
| 1 | [create-case.html:2935](/home/projectds/projects/compass/usac/usac/templates/usac/case/create-case.html:2935) | `load_dispatch_tab()` calls `vendor_form` with `action='honk_or_compass'`, then shows/hides tabs |
| 2 | [dispatch.py:1314](/home/projectds/projects/compass/usac/usac/code/business/dispatch.py:1314) | Returns eligibility flags and explanatory messages |
| 3 | [dispatch.py:741](/home/projectds/projects/compass/usac/usac/code/business/dispatch.py:741) | Read `able_to_oon()`, `able_to_quest()`, `able_to_honk()`, and `able_to_compass()` |
| 4 | [dispatch.py:839](/home/projectds/projects/compass/usac/usac/code/business/dispatch.py:839) | Client/program restrictions for Compass dispatch |

**There are three different kinds of “retry”**

**1. Manual Honk dispatch**

```text
Dispatch button
 → POST /api/honk/dispatch
 → DispatchToHonkAPIView.post()
 → HonkDispatchHandler.dispatch_to_honk()
 → _dispatch_to_honk()
 → external HTTP request
```

Read [Honk views.py:30](/home/projectds/projects/compass/usac/usac/apps/honk/views.py:30) and [Honk dispatcher.py:477](/home/projectds/projects/compass/usac/usac/apps/honk/dispatcher.py:477).

This direct path does **not** use the Celery retry decorator. Its attempt history is collected from stored `HonkAPI` CREATE_JOB records.

**2. Automatic selection of another vendor**

```text
Case creation, when auto-dispatch is enabled
 → task_dispatch_case
 → AutoDispatchHandler.run_dispatch_selection(initial=True)
 → vendor selection and availability checks
 → Towbook/Atlas dispatch
 → rejection, timeout, or qualifying cancellation
 → reattempt_auto_dispatch
 → run_dispatch_selection(initial=False)
 → another candidate, or Honk fallback when eligible
```

Read:

- [views.py:10892](/home/projectds/projects/compass/usac/usac/views.py:10892): starts automatic dispatch.
- [digital_dispatch_tasks.py:21](/home/projectds/projects/compass/usac/usac/tasks/digital_dispatch_tasks.py:21): initial task and reattempt task.
- [dispatch.py:1646](/home/projectds/projects/compass/usac/usac/code/business/dispatch.py:1646): `AutoDispatchHandler`.
- [towbook.py:39](/home/projectds/projects/compass/usac/usac/tasks/towbook.py:39): dispatch and response-timeout handling.
- [atlas.py:32](/home/projectds/projects/compass/usac/usac/tasks/atlas.py:32): corresponding Atlas flow.

Several failure paths schedule reselection with **`countdown=2`**. That is a two-second delay before reselection, **not the provider’s response timeout**. Response timeouts come from integration configuration.

Webhook handlers also trigger qualifying reattempts: [Towbook processing.py:700](/home/projectds/projects/compass/usac/usac/apps/digital_dispatch/processing.py:700) and [Atlas handler.py:651](/home/projectds/projects/compass/usac/usac/apps/atlas/handler.py:651).

**3. Retrying the background Honk task**

[Tasks honk.py:23](/home/projectds/projects/compass/usac/usac/tasks/honk.py:23) configures **up to three retries for escaping exceptions**, with backoff and jitter.

**An ordinary returned `success=False` does not trigger that retry.** The explicit retry-on-Honk-error code is commented out, and the handler catches many exceptions and returns failure instead. Therefore, a Honk API error is not automatically retried merely because the task has a retry decorator.

WebSockets deliver progress updates to the browser; reconnecting a WebSocket is separate from dispatching again.

For learning, follow the numbered visibility path first, then the automatic-selection path. All inspection here was local and read-only; no external requests or background tasks were started.

**Yes, there are potential gaps that could allow another dispatch after work has been completed.** This is a code-review finding, not evidence that it has happened.

The intended flow is to try another provider after rejection, cancellation, or no response. However:

| Path | Gap found |
|---|---|
| Automatic reattempt | Stops when it finds a vendor with status `DISPATCHED`, but does not explicitly check whether the case/service is already completed. |
| Towbook timeout task | Schedules a reattempt even when the request is no longer `REQUEST_SENT`. Normally the timeout is revoked on acceptance, but a late-running task could still reach this code. |
| Manual Honk API | The inspected endpoint and handler do not explicitly reject an already-completed case before submitting a new job. |
| Manual Compass dispatch | The `action='dispatch'` branch updates the vendor and case to dispatched without an explicit completed-case guard in that branch. |

The important distinction is **physical completion versus recorded status**. If Provider A finishes but its completion update is missing or delayed, Compass may still consider the request eligible for another attempt. Even when completion is recorded, these paths do not share one consistent final check before sending.

Read these locations:

- [dispatch.py:1839](/home/projectds/projects/compass/usac/usac/code/business/dispatch.py:1839): reattempt selection and the `DISPATCHED` check.
- [dispatch.py:1872](/home/projectds/projects/compass/usac/usac/code/business/dispatch.py:1872): automatic routing.
- [towbook.py:276](/home/projectds/projects/compass/usac/usac/tasks/towbook.py:276): timeout and unconditional reattempt scheduling.
- [Honk views.py:30](/home/projectds/projects/compass/usac/usac/apps/honk/views.py:30): manual Honk entry point.
- [Honk dispatcher.py:477](/home/projectds/projects/compass/usac/usac/apps/honk/dispatcher.py:477): outbound dispatch handling.

**So I would not describe the current implementation as guaranteeing “completed work can never be dispatched again.”** UI restrictions and task cancellation provide some protection, but they are not a consistent backend guarantee.

No dispatches, database updates, or code changes were made.