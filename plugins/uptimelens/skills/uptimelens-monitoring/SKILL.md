---
name: uptimelens-monitoring
description: Sets up and manages UptimeLens uptime monitoring through the UptimeLens MCP server — create HTTP/HTTPS monitors, recommend what to monitor after a deploy, pause or resume checks, schedule maintenance windows to mute deploy noise, model monitor dependencies, and open, update or resolve incidents, then write up the postmortem and its follow-up actions. Use when the user mentions adding or creating a monitor, monitoring a URL or a newly deployed service, what to monitor after deploying, pausing or resuming a monitor, scheduling a maintenance window, checking what is down or what the uptime was, or opening, updating, resolving or writing a postmortem for an incident.
---

# UptimeLens Monitoring

Set up and run UptimeLens uptime monitoring on the customer's own resources, through the UptimeLens MCP server. This skill turns a monitoring request ("monitor this URL", "what should I watch after this deploy?", "what's down?", "open an incident") into the correct sequence of MCP tool calls.

## When to use this skill

Use it for the monitoring-setup and incident-bookkeeping side of the product:

- Creating HTTP/HTTPS monitors, and recommending coverage after a deploy or for a new service.
- Checking what is currently down and doing a light investigation (status history, per-probe root cause, response-time/availability).
- Scheduling a maintenance window so a deploy or planned work does not page anyone.
- Modeling monitor dependencies so a downstream check does not alert when its upstream is already down.
- Opening, updating, or resolving incidents.

Hand off when the request is about responding to a page rather than monitoring: for acknowledging or snoozing an active escalation, covering a shift, or inspecting escalation runs, use the **`uptimelens-on-call`** skill. For a deep, guided root-cause investigation of an incident, prefer the read-only **`triage_incident`** MCP prompt (and the `get_root_cause` tool) rather than scripting the whole analysis here.

## Prerequisites

1. The UptimeLens MCP server is connected to this client. Read tools work on every plan.
2. **Write actions** (`create_monitor`, `update_monitor`, `pause_monitor`, `resume_monitor`, `register_monitor_dependency`, `delete_monitor_dependency`, `create_incident`, `post_incident_update`, `resolve_incident`, `cancel_incident`, `reopen_incident`, `save_postmortem`, `add_incident_action_item`, `create_maintenance`, `delete_maintenance`) require a **read-write** API token and a **Pro plan or higher**. With a read-only token the write tools are not even listed; on the Free plan they return `FeatureNotAvailable`.
3. Some read tools are gated by plan. `get_root_cause` (paid) and `get_incident_analytics` (Pro+) are **hard-gated** — below the plan they return a `FeatureNotAvailable` error; handle it like the write-tool prerequisites below (tell the user the plan is unmet, do not retry). `query_metrics` percentiles are **soft-gated** — the percentile fields come back null below Team, not an error.

If a write tool is missing or returns `FeatureNotAvailable`, tell the user which prerequisite is unmet instead of retrying.

## Tools

These tools come from the UptimeLens MCP server. Reference them fully qualified as `UptimeLens:<tool>` — replace `UptimeLens` with whatever name the connector was given in this client, otherwise the tool may not be found.

Read (discovery and context):

- `UptimeLens:get_overview` — **start here for "what is broken right now?"**: monitor counts by status, 24-hour uptime, open incidents, and the distinct failures grouped together — forty monitors that broke the same way at the same target come back as ONE outage instead of forty rows. Use `list_monitors` instead when you need the inventory itself (names, tags, ids).
- `UptimeLens:list_monitors` — monitors with current health; filter by `status` (up/degraded/down/unconfirmed-down/unknown/not-monitored/in-maintenance), name, or tags. The source of `monitorId` for almost everything else, and the answer to "what's down?".
- `UptimeLens:get_monitor` — one monitor's configuration and current health (no credentials).
- `UptimeLens:list_status_changes` — a monitor's up/down/degraded history with durations; the source of the `statusChangeId` that `get_root_cause` needs.
- `UptimeLens:get_status_change` — one status change with its context (previous and next change, the incident and escalation runs covering it) and a per-location summary of the checks around it: how many failed, `firstDownAt`, and the most frequent `errorCategory`. Use it to tell where an outage started and how it was confirmed; available on every plan.
- `UptimeLens:get_root_cause` — per-probe network diagnostics for one status change: DNS/connect/TLS timing, traceroute, TLS version, SSL days remaining, error category. The richest "why was it down" source (paid plan).
- `UptimeLens:query_metrics` — response-time and uptime statistics for one monitor over a period (period summary, not a full series; percentiles are Team+).
- `UptimeLens:get_monitor_availability` — SLA view for one monitor: uptime %, downtime, incident count, MTTR, MTBF, response-time stats. Reading it: an absent `uptimePct` means the period had NO measurements — it is NOT 100, so say the monitor was not measured; `longestIncidentSec` is an incident's wall-clock time and CAN exceed the measured `totalDowntimeSec` — that is expected, not a data error; `uptimePct` is rounded to two decimals, so a very short outage can read 100.00 next to a positive downtime; an absent `mtbfSec`/`mttrSec` is NOT zero.
- `UptimeLens:get_uptime_report` — the same question for the **whole account** over a period: time-weighted uptime, downtime, error budget against an SLA target, plus what broke (`causes`) and where it was confirmed from (`locations`). Pick a period with a preset (`last_month`, `last_30d`, …) or explicit dates. Use this for "how did last month go?"; `get_monitor_availability` for one monitor.
- `UptimeLens:list_monitor_dependencies` — the parent monitors a monitor depends on (what must be up for its alerts to fire).
- `UptimeLens:list_monitor_dependents` — the reverse: monitors that depend on this one — its **blast radius**, for telling root cause from symptom.
- `UptimeLens:list_incidents` — incidents, newest first; **defaults to the last 7 days** unless you pass `startDate`/`endDate`. Filter by `status`, `monitorId`, or `source`.
- `UptimeLens:get_incident` — one incident's detail plus its update timeline and affected monitors.
- `UptimeLens:get_incident_analytics` — fleet-level incident metrics over a date range (MTTR, volume/trend, severity/source breakdown, top monitors) — Pro+.
- `UptimeLens:list_maintenance` — maintenance windows; `status=active` answers "is anything in maintenance right now?".
- `UptimeLens:list_status_pages` / `UptimeLens:get_status_page` — what is published to customers and which monitors appear on a page (useful when deciding coverage).

Write (change state — see "Confirm before writing"):

- `UptimeLens:create_monitor` — create an HTTP/HTTPS monitor from a URL. See "What create_monitor can and cannot do".
- `UptimeLens:update_monitor` — edit an existing monitor's **name and/or check interval** (`id` + `name` and/or `intervalSeconds`, at least one). All other settings (URL, type, auth, headers, thresholds, regions) are preserved untouched and can't be changed via MCP. Reversible and idempotent.
- `UptimeLens:pause_monitor` / `UptimeLens:resume_monitor` — stop or restart active monitoring for one monitor. Input: `id`. Reversible and idempotent.
- `UptimeLens:register_monitor_dependency` — declare a child→parent dependency: while the parent is down, the child's down alerts are suppressed. Input: `monitorId` (child), `parentId`. Direct parent only; cycles/self rejected.
- `UptimeLens:delete_monitor_dependency` — remove a dependency edge by its record `id` (from `list_monitor_dependencies`); does not delete any monitor.
- `UptimeLens:create_incident` — open an incident. See "Incidents open as drafts".
- `UptimeLens:post_incident_update` — add a timeline update (investigating/identified/monitoring) with a message. Subscribers are **not** emailed unless you pass `notifySubscribers=true` (and only if the incident is already published).
- `UptimeLens:resolve_incident` — mark an incident resolved. Subscribers are **not** emailed unless you pass `notifySubscribers=true` (and only if published). Reversible via `reopen_incident`.
- `UptimeLens:cancel_incident` — cancel as false_alarm/duplicate/test; **terminal** (cannot be reopened), excluded from analytics. Silent by default.
- `UptimeLens:reopen_incident` — reopen a previously **resolved** incident (a canceled one cannot be reopened).
- `UptimeLens:save_postmortem` — write (or overwrite) an incident's postmortem body as markdown; optional `status` of draft or in_review. This is where the `postmortem_draft` prompt's output goes — without it the draft only exists in the chat. **Publishing stays a human step in the dashboard.**
- `UptimeLens:add_incident_action_item` — add a follow-up action item to an incident (title, plus optional type, priority and due date). The half of a postmortem that actually changes anything.
- `UptimeLens:create_maintenance` / `UptimeLens:delete_maintenance` — schedule or cancel a maintenance window that suppresses alerts for its monitors during the window.

## Core workflow: set up monitoring (especially after a deploy)

Discovery comes first: almost every write needs an ID, or a decision that only the current inventory can inform. Copy this checklist and work through it:

```
Set up monitoring:
- [ ] 1. See current coverage (list_monitors) — what exists, what's missing, what's already down
- [ ] 2. For a new URL/service: check list_monitors for a duplicate (create_monitor is NOT idempotent)
- [ ] 3. Confirm the URL(s) with the user, then create_monitor (HTTP/HTTPS only)
- [ ] 4. Wire topology if relevant (register_monitor_dependency: the new check under its upstream)
- [ ] 5. If a deploy/maintenance is happening, mute noise (create_maintenance for the window)
```

**Step 1 — inventory.** Call `list_monitors`. To answer "what's down?", pass `status=["down"]` (add `degraded`/`unconfirmed-down` if the user means "unhealthy"). This is also how you spot gaps: a service the user just deployed with no matching monitor.

**Step 2-3 — create, carefully.** `create_monitor` is **not idempotent** — calling it twice for the same URL creates two monitors. Check `list_monitors` for an existing one first, and confirm the exact URL with the user before creating. See the constraints below.

**Step 4 — topology (optional).** If the new monitor sits downstream of something you already monitor (an app behind a gateway, a service behind its database), `register_monitor_dependency` with the new monitor as `monitorId` and the upstream as `parentId` so the child does not page when the upstream is the real problem.

**Step 5 — mute deploy noise (optional).** If the user is deploying or doing planned work, `create_maintenance` for the affected monitors over the window so the work itself does not page.

## What create_monitor can and cannot do

`create_monitor` is a quick-add: give it a URL, it creates an **HTTP/HTTPS** monitor (https is assumed when no scheme is given), the name defaults to the host, and the check interval and alert group use the tenant defaults.

It **cannot**, via MCP: create non-HTTP monitors (TCP/UDP, DNS, ping/ICMP, SSH, SMTP, etc.), or set authentication, custom headers, keyword/body assertions, a specific interval, or alert groups. For any of those, tell the user to finish the setup in the dashboard — do not pretend the MCP can configure them. After creation, `update_monitor` can change a monitor's **name and check interval** (only those two — everything else stays in the dashboard). There is **no `delete_monitor`** tool: to remove a monitor, point the user to the dashboard.

## Incidents open as drafts

`create_incident` always opens an **unpublished draft**: it is not shown on any status page and subscribers are not notified. A human publishes it from the dashboard. Say this when you create one, so the user knows it is not yet public. `post_incident_update`, `resolve_incident`, `cancel_incident` and `reopen_incident` email subscribers **only when you explicitly pass `notifySubscribers=true`** — the MCP default is silent, because mail to the customer's own customers cannot be taken back. (The dashboard behaves the opposite way: there a human is pressing the button.) `cancel_incident` is terminal (false_alarm/duplicate/test) and cannot be reopened; `resolve_incident` is reversible via `reopen_incident`. Use `cancel` for "this was not real", `resolve` for "this is over".

## Confirm before writing

Read and recommend freely. Before any write, state what you are about to do and get the user's go-ahead — these change what is monitored, what pages, or what is recorded:

- Create monitor: name the exact URL. Because it is not idempotent, confirm there is not already a monitor for it (you checked `list_monitors`).
- Pause/resume: pausing stops monitoring, so a real outage will **not** page while paused — confirm the user means to silence it.
- Maintenance: name the monitors and the exact window (RFC3339; default UTC, or pass an IANA `timezone` such as `Europe/Istanbul` — recurring windows are DST-aware); during it, alerts are suppressed.
- Incident open/update/resolve/cancel: name the incident and, for resolve/cancel, say whether you are asking for a subscriber email (`notifySubscribers=true`) — by default no mail is sent.

Prefer the least-surprising action, recommend rather than create at scale, and never open or resolve incidents in bulk without per-incident confirmation.

## Common scenarios

**"Monitor https://api.example.com."**
`list_monitors` (check it isn't already monitored) → confirm the URL → `create_monitor(url)`. If the user wanted auth, a keyword check, or a non-HTTP protocol, create the basic HTTP monitor and tell them to finish those settings in the dashboard.

**"I just deployed the billing service — what should I monitor?"**
`list_monitors` to see current coverage → identify the gaps (the new public endpoints / health checks with no monitor) and **recommend** them to the user, rather than creating silently → on confirmation, `create_monitor` for each → if the new monitor sits behind something you already watch, `register_monitor_dependency` to put it under that upstream. Note what can't be auto-created (non-HTTP, auth, keywords) and hand those to the dashboard.

**"What's down right now?"**
`get_overview` first — it answers the whole question in one call and, crucially, groups monitors that failed the same way so you report *one* outage rather than forty. Then, for a specific monitor, `list_status_changes(monitorId, statuses=["down"])` to see the current outage and `get_root_cause(statusChangeId)` for the DNS/TLS/traceroute diagnostics. Fall back to `list_monitors(status=["down"])` only when you need the raw per-monitor list. Read-only; don't change anything.

**"We're deploying tonight 22:00–23:00, don't page us."**
Pick the affected monitors (`list_monitors`) → set the `timezone` (IANA, e.g. the user's zone like `Europe/Istanbul`; recurring windows are DST-aware) or leave it UTC → confirm → `create_maintenance(monitorIds, scheduleType="once", startTime, durationMinutes=60, timezone)` → `list_maintenance(status=active)` (or `upcoming`) to confirm it is in place.
Two hard limits on this tool: **`monitorIds` is required** (there is no account-wide option here — list the monitors explicitly), and each occurrence may last **at most 240 minutes**. For a longer window, or one covering every monitor, hand the user to the dashboard instead of retrying.

**"Pause the staging monitor."**
`list_monitors` to find it → confirm (pausing means a real outage won't page) → `pause_monitor(id)`. `resume_monitor(id)` to restart.

**"Write up a postmortem for yesterday's outage."**
Use the read-only `postmortem_draft` prompt to produce the narrative from the incident's own timeline, show it to the user, and only then persist it with `save_postmortem(incidentId, body, status="draft")` — the tool overwrites whatever is there, so confirm before saving over an existing write-up. Follow-ups become `add_incident_action_item` calls, one per item, so they are tracked rather than buried in prose. Publishing the postmortem is a dashboard action; say so rather than implying it is now public.

**"Open an incident for the API outage."**
`create_incident(title, impact, message, affectedMonitorIds)` → tell the user it is a **draft** (not public, no subscriber notification) and that they publish it from the dashboard → optionally `post_incident_update` as things develop, and `resolve_incident` when it's over.

## Notes

- All timestamps are UTC RFC3339 (e.g. `2026-06-14T22:00:00Z`). Maintenance windows carry an IANA `timezone`; users usually mean local time, so convert to UTC and confirm the window back to them.
- List tools return at most 100 items with `total` and `hasMore`; page with `offset` when `hasMore` is true rather than assuming the first page is everything.
- Tier gates: `get_root_cause` (paid) and `get_incident_analytics` (Pro+) return a `FeatureNotAvailable` **error** below their plan — surface the plan prerequisite, don't retry. `query_metrics` percentiles are softer: the percentile fields come back **null** below Team, not an error.
- For multi-step write-ups, prefer the read-only MCP prompts over assembling the output by hand: `uptime_report` (a monitor's uptime/SLA report — for an account-wide period report call the `get_uptime_report` tool directly), `postmortem_draft` (a blameless postmortem for a resolved incident), `summarize_incident` (a short stakeholder/status summary), and `triage_incident` (guided root-cause). Each one orchestrates the right read tools for you.
- Every tool is scoped to the token owner's account; you cannot see or change another customer's monitoring setup.
