---
name: uptimelens-on-call
description: Runs UptimeLens on-call and escalation operations through the UptimeLens MCP server — find who is currently on call, acknowledge or snooze active escalation pages, cover a shift by overriding an on-call rotation, and inspect escalation runs and policies. Use when the user mentions on-call, who is on call, acknowledging, resolving, closing, taking or silencing an alert or page, snoozing a page, escalation runs, covering or swapping a shift, or overriding a rotation.
---

# UptimeLens On-Call

Operate UptimeLens on-call and escalation workflows on the customer's own resources, through the UptimeLens MCP server. This skill turns an alerting question ("who's on call?", "ack that page", "cover my shift") into the correct sequence of MCP tool calls.

## When to use this skill

Use it for the operational side of incident alerting:

- Finding who is currently on call (or why nobody is).
- Acknowledging or snoozing an active escalation that is paging someone.
- Covering a shift by overriding an on-call rotation for a time window.
- Inspecting escalation runs (active pages + history) and escalation policies.

For root-cause analysis of an *incident* (as opposed to managing the page), prefer the read-only `triage_incident` MCP prompt, or the read tools `get_incident`, `get_root_cause`, `list_status_changes`, and `query_metrics`. This skill stays on the on-call/escalation layer and links to incident context only when it helps decide who should act.

## Prerequisites

1. The UptimeLens MCP server is connected to this client. Read tools work on every plan.
2. **Write actions** (`acknowledge_escalation`, `bulk_acknowledge_escalation`, `resolve_escalation`, `snooze_escalation`, `escalate_escalation`, `create_on_call_override`, `update_on_call_override`, `delete_on_call_override`, `create_cover_request`, `accept_cover_request`, `decline_cover_request`) require a **read-write** API token and a **Pro plan or higher**. With a read-only token the write tools are not even listed; on the Free plan they return `FeatureNotAvailable`.
3. Some of these carry a **second, per-feature gate** on top of the plan gate, and they are not all the same one:
   - `create_on_call_override`, `update_on_call_override` and **`accept_cover_request`** need the **on-call override** feature. Accepting counts because it *produces* an active override — the very thing the feature sells. (A pending request created on a paid plan and accepted after a downgrade would otherwise create an override the plan no longer includes; that gap was closed deliberately.)
   - `create_cover_request` needs a **different** feature — on-call **cover requests** — so a plan can include one and not the other.
   - `delete_on_call_override` and `decline_cover_request` are **deliberately ungated**: undoing something must stay possible after a downgrade, otherwise a shift swap you can no longer edit is also one you can no longer cancel.
   Any of these can return `FeatureNotAvailable` even with a read-write token.

If a write tool is missing or returns `FeatureNotAvailable`, tell the user which prerequisite is unmet instead of retrying.

## Tools

These tools come from the UptimeLens MCP server. Reference them fully qualified as `UptimeLens:<tool>` — replace `UptimeLens` with whatever name the connector was given in this client, otherwise the tool may not be found.

Read (discovery and context):

- `UptimeLens:list_on_call_schedules` — schedules; source of the `scheduleId` that `who_is_on_call` and `create_on_call_override` need.
- `UptimeLens:who_is_on_call` — who is on call for one schedule right now; returns `gapReason` when nobody is.
- `UptimeLens:get_on_call_schedule` — the upcoming roster for one schedule over the next N days (default 7, max 90): each shift (who, when, rotation vs override), coverage gaps with their reason, and a coverage summary. Use this for *future* coverage; `who_is_on_call` is just *now*.
- `UptimeLens:list_on_call_overrides` — the active and upcoming overrides for one schedule; the source of the `id` that `update_on_call_override` and `delete_on_call_override` need. Each entry carries the user (who takes the shift), the requester (`requestedBy`, set only for a pending cover request — who asked for the cover), window, reason, and state. Only an `active` override is editable; a `pending` entry is a cover request still awaiting the target user's accept, and `update_on_call_override` will reject it. Past overrides are not listed.
- `UptimeLens:list_escalation_runs` — escalation runs (active pages + history); source of the `runId` that `acknowledge_escalation`, `snooze_escalation` and `escalate_escalation` need. Progress is 1-based: `currentStep` (the step the escalation has reached), `nextStep` (fires next), `totalSteps`.
- `UptimeLens:get_escalation_run` — one run's full state: which step the escalation has reached and which fires next (`currentStep`/`nextStep` of `totalSteps`), repeat progress, urgency, ack/resolve method (WHY it ended), snooze & ack-timeout deadlines, `nextActionAt`, triggering monitor/incident.
- `UptimeLens:get_escalation_run_timeline` — one run's chronological event log: notifications, delivery outcomes, ack timeouts, skipped targets, repeats, resolution — the richest "what happened / why it escalated" source.
- `UptimeLens:list_escalation_policies` — the on-call alerting rules.
- `UptimeLens:get_escalation_policy` — one policy's detail: ordered steps numbered 1..N (the numbers `currentStep`/`nextStep` in `list_escalation_runs` refer to), each step's delay, target strategy and targets. Resolve schedule targets via `who_is_on_call`, user targets via `list_team_members`.
- `UptimeLens:list_team_members` — people on the team; source of the `userId` that `create_on_call_override` needs.

Write (change state — see "Confirm before writing"):

- `UptimeLens:acknowledge_escalation` — stop further paging for one active run. Input: `runId`.
- `UptimeLens:resolve_escalation` — close the page **permanently**. Input: `runId`. Unlike acknowledge, a resolved run never re-pages, so use it when the underlying issue is confirmed fixed — not as a way to make a noisy alert stop.
- `UptimeLens:bulk_acknowledge_escalation` — acknowledge many active runs at once. Input: `runIds` (1–50). Partial-success: already-resolved/unknown runs are counted as skipped, not errors. Use after `list_escalation_runs(state=active)` to clear a batch.
- `UptimeLens:snooze_escalation` — pause paging for one run for a number of minutes. Input: `runId`, `durationMinutes` (1–240).
- `UptimeLens:escalate_escalation` — page the **next not-yet-paged step now** instead of waiting out its timer ("escalate now"). Input: `runId`. Nothing is skipped: the step shown as `nextStep` fires; after the last step the next repeat starts immediately, and with no repeat left the run is closed as exhausted (`nextStep` is 0 — tell the user there is nobody left to page). The run goes back to active and its urgency becomes `high`. Rejected for test runs and finished runs.
- `UptimeLens:create_on_call_override` — put a user on call for a schedule during a window. Input: `scheduleId`, `userId`, `endAt`; optional `startAt` (defaults to now), `reason`. Requires the on-call override feature (see prerequisites).
- `UptimeLens:update_on_call_override` — edit an existing override in place: change its `endAt` or `reason`, and (only before it starts) its `userId` or `startAt`. Input: `id` (from `create_on_call_override`'s result, or discovered via `list_on_call_overrides` when you didn't just create it) plus the fields to change; omit a field to leave it unchanged. This is how you shorten, extend, or reassign a cover; use `delete_on_call_override` when the swap should not happen at all. Requires the on-call override feature.
- `UptimeLens:delete_on_call_override` — delete an override outright: the swap is cancelled and the regular rotation applies again. Input: `id`. Use this rather than editing when the cover should not happen at all.
- `UptimeLens:create_cover_request` — **ask** a teammate to take a window instead of assigning it: creates a `pending` request and emails the target, who must accept before it becomes an active override. Input: `scheduleId`, `userId` (who is being asked), `startAt`, `endAt`; optional `reason`. Use `create_on_call_override` when you mean to assign the shift immediately without asking.
- `UptimeLens:accept_cover_request` — accept a `pending` cover request → it becomes an active override (the target user takes the shift). Input: `id` (a `pending` entry from `list_on_call_overrides`). Only the target user or an admin can accept.
- `UptimeLens:decline_cover_request` — decline a `pending` cover request (no override is created). Input: `id` (a `pending` entry from `list_on_call_overrides`). Only the target user or an admin can decline.

## Core workflow: handle an on-call alert

Most write tools need an ID that only a read tool can give you, so always discover first. Copy this checklist and work through it:

```
On-call response:
- [ ] 1. Find the active escalation runs (list_escalation_runs, state=active, isTest=false)
- [ ] 2. Pick the run in question (rank by startedAt; check urgency and currentStep/totalSteps)
- [ ] 3. Add context if useful (incidentId → get_incident; triggeringMonitorId → list_status_changes → get_root_cause)
- [ ] 4. Decide: acknowledge (own it, paging stops) or snooze (pause N minutes, paging resumes)
- [ ] 5. Confirm the action with the user, then call the write tool with the run's id
```

**Step 1 — find active runs.** Call `list_escalation_runs` with `state="active"`. Use `isTest=false` for real, incident-triggered pages; `isTest=true` are "page-yourself" test runs and should not be treated as live incidents.

**Step 2 — pick the run.** Each run has an `id` (this is the `runId` for write tools), `urgency` (high/low), `currentStep` of `totalSteps` (how far the escalation has climbed — 1-based; higher means more people have been paged; `nextStep` is who gets paged next), and `startedAt`. When several are active, rank by `startedAt` (oldest first) and surface `high` urgency first.

**Step 3 — context (optional).** If the run has an `incidentId`, call `get_incident` for impact and timeline. If it has a `triggeringMonitorId`, call `list_status_changes` with that monitor id to see its recent up/down changes, then `get_root_cause` with a `statusChangeId` from that list for per-probe network diagnostics. Use this only to help the user decide; do not start a full investigation here.

**Step 4 / 5 — act.** Choose acknowledge vs snooze vs escalate (below), confirm, then write.

## Acknowledge, resolve, snooze, or escalate

- **Acknowledge** (`acknowledge_escalation`) means "I own this now" — paging stops while you work. This is the right answer for "take", "I've got it", or "silence" a page. Acknowledging a run that is no longer active returns an `EscalationRunAlreadyResolved` error — harmless (it was already acknowledged or resolved); report it as already-handled and do **not** retry.
- **Resolve** (`resolve_escalation`) means "this is over" — the run is closed permanently and **never re-pages**. Use it when the user says the underlying issue is fixed, not merely that they are on it. A run also resolves on its own when its incident resolves, so resolving by hand is for the case where the page outlived the problem. Because it cannot be undone, prefer acknowledge when in doubt.
- **Snooze** (`snooze_escalation`) means "quiet it for `durationMinutes`, then page again" — a deferral, not ownership. It is **not** idempotent: each call re-sets the snooze window to now + N minutes, so calling it again *extends* the silence. Use the smallest duration that fits (1–240), and never snooze instead of acknowledging when the alert actually needs an owner.

- **Escalate now** (`escalate_escalation`) means "this needs more people, now" — the run's `nextStep` is paged immediately instead of after its wait, urgency becomes `high` (quiet hours no longer apply), and an acknowledged run becomes active again. Use it when the user says the current responder cannot handle it or nobody is responding; it is the opposite of snooze, so never call both on the same run in one turn. Tell the user which step will be paged (`nextStep` from `get_escalation_run`).

Acknowledging the wrong run silences a real page and no one is alerted, so confirm the `runId` matches the alert the user means.

## Coverage gaps

`who_is_on_call` returns a `gapReason` when nobody is on call:

- `schedule_disabled` — the schedule is turned off; the policy is relying on other targets.
- `no_active_rotations` / `no_rotation_members` — the schedule has no one assigned to page right now.
- `outside_active_window` — it is outside this schedule's active hours.

When there is a gap and someone needs to take the page, `create_on_call_override` puts a specific user on call for a window. Discover `scheduleId` via `list_on_call_schedules` and `userId` via `list_team_members` (team members expose no email or phone — match by name/role).

`who_is_on_call` only reports the gap happening *now*. To find gaps **before** they page anyone, call `get_on_call_schedule` for the schedule — its `gaps` array lists every uncovered window over the next N days with the same reasons, and `coveragePercent` summarizes how well the period is covered. This is the proactive companion to `who_is_on_call`.

## Confirm before writing

Read and triage freely. For the write tools, state what you are about to do and get the user's go-ahead first — these change who gets paged:

- Acknowledge / snooze: name the run (urgency, what triggered it) and, for snooze, the minutes.
- **Resolve**: say plainly that the page is being closed for good and will not re-page. Ask whether the underlying issue is actually fixed — "make it stop" is a request for acknowledge or snooze, not resolve.
- Override: name the schedule, the user, and the exact window (`startAt`–`endAt`, UTC RFC3339). If the window crosses midnight, `endAt`'s date is the following day; `endAt` must be after `startAt`.
- **Delete override / decline cover request**: name whose shift goes back to the regular rotation. Cancelling a cover changes who gets paged just as much as creating one does.

Most of these are reversible — an override can be edited or shortened, an ack or snooze re-escalates — so the goal is a clear human decision, not a permission wall. Two are **not**: `resolve_escalation` closes a page permanently, and `delete_on_call_override` removes the swap rather than editing it. Prefer the least-surprising action and never batch-acknowledge multiple runs without explicit per-run confirmation.

## Common scenarios

**"Who's on call for <team/schedule>?"**
`list_on_call_schedules` → match the schedule by name → `who_is_on_call(scheduleId)`. If `gapReason` is set, explain it and offer an override.

**"Who's on call this week / show the upcoming roster (and any coverage gaps)?"**
`list_on_call_schedules` → pick the schedule by name → `get_on_call_schedule(scheduleId)` (defaults to the next 7 days; pass `days` up to 90 for a longer view). Summarize the shifts (who and when), then call out any `gaps` with their reason and the `coveragePercent`, and offer an override for an uncovered window. Use this instead of `who_is_on_call` whenever the question is about the upcoming period rather than the current moment.

**"Acknowledge the database alert."**
`list_escalation_runs(state=active, isTest=false)` → identify the run (use `triggeringMonitorId`/`incidentId` to match "database") → confirm → `acknowledge_escalation(runId)`.

**"Snooze that page for 30 minutes."**
Find the active run as above → confirm → `snooze_escalation(runId, durationMinutes=30)`. Remind the user it will page again after the window.

**"Nobody is answering — escalate this now."**
Find the active run as above → `get_escalation_run(runId)` to name the step that will be paged (`nextStep`) → confirm → `escalate_escalation(runId)`. Report that the step is being paged and that the run is now high urgency.

**"Cover Sam's shift tonight, 18:00–06:00."**
`list_on_call_schedules` → pick the schedule (note its `timezone`) → `list_team_members` → find the covering user's `userId` → convert the user's local window to UTC → confirm the UTC window → `create_on_call_override(scheduleId, userId, startAt="2026-06-14T18:00:00Z", endAt="2026-06-15T06:00:00Z", reason="cover for Sam")`. The window crosses midnight, so `endAt` is the **next** calendar day — `endAt` must be after `startAt` or the call is rejected.

**"Ask Dana if she can take my shift on Friday night."**
The user wants to *ask*, not assign, so use the request flow: `list_on_call_schedules` → pick the schedule (note its `timezone`) → `list_team_members` → find Dana's `userId` → convert the window to UTC → confirm → `create_cover_request(scheduleId, userId, startAt, endAt, reason)`. This creates a **pending** request and emails Dana; nothing changes on the rotation until she accepts. Tell the user that — a request that is never accepted leaves the original person on call. Use `create_on_call_override` only when they mean "put Dana on call", not "ask Dana".

**"Cancel that cover — I'll keep the shift after all."**
`list_on_call_overrides(scheduleId)` → find the entry → confirm → `delete_on_call_override(id)` for an **active** override (the regular rotation applies again), or `decline_cover_request(id)` for one still `pending`. Deleting is the right call when the swap should not happen at all; shortening it with `update_on_call_override` is for when it should happen, differently.

**"The database issue is fixed — close that page."**
Find the run (`list_escalation_runs`, `state=active` or `acknowledged`) → check with the user that the *issue* is fixed, not just noisy → `resolve_escalation(runId)`. Say that it will not page again. If they only want quiet while they work, acknowledge or snooze instead.

**"Actually, extend that cover until 08:00."**
You already created the override, so edit it in place instead of making a new one: `update_on_call_override(id, endAt="2026-06-15T08:00:00Z")` — pass only the field you are changing. Confirm the new UTC window back to the user.

**"Extend the override on the database schedule to 08:00." (you didn't create it this session)**
You don't have the override's `id` in context, so discover it first: `list_on_call_schedules` → match the schedule → `list_on_call_overrides(scheduleId)` → pick the **active** override by its user/window (a `pending` entry is an unaccepted cover request and can't be edited) → `update_on_call_override(id, endAt="…Z")`. `list_on_call_overrides` is the only way to get an override `id` you didn't just create; never guess an id. The parameter is named `id`.

**"Accept the pending cover request on the database schedule."**
`list_on_call_schedules` → match the schedule → `list_on_call_overrides(scheduleId)` → find the `pending` entry (a cover request awaiting accept; its `userId` is who would take the shift and `requestedBy` is who asked — use them to pick the right one when several are pending) → confirm with the user → `accept_cover_request(id)`. Only the target user or an admin can accept; otherwise the tool returns a permission error. To reject it instead, use `decline_cover_request(id)`. (Accepting turns the pending request into an active override — it changes who gets paged, so confirm first.)

**"Anything paging right now?"**
`list_escalation_runs(state=active, isTest=false)` → summarize by urgency and age (`startedAt`), highest urgency and oldest first; offer to acknowledge or snooze each.

## Notes

- All timestamps are UTC RFC3339 (e.g. `2026-06-14T18:00:00Z`). Schedules carry their own IANA `timezone`; users usually mean schedule-local time, so convert to UTC before an override and confirm the UTC window back to them.
- Paginated list tools (`list_on_call_schedules`, `list_escalation_runs`, `list_team_members`) return at most 100 items with `total` and `hasMore`; page with `offset` when `hasMore` is true rather than assuming the first page is everything. A few naturally-bounded lists (e.g. `list_on_call_overrides` — one schedule's active+upcoming overrides) return the full set without paging.
- Every tool is scoped to the token owner's account; you cannot see or change another customer's on-call setup.
