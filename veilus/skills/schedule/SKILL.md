---
name: schedule
description: Plan a Veilus schedule for an approved script — timing, target profiles, concurrency and stagger sized to this machine — show the plan to the user as a table, and create it only after they agree. Use when a Veilus campaign reaches scheduling, or the user asks to schedule a Veilus script.
---

# Plan and create a Veilus schedule

Tools are on the Veilus MCP server (`mcp__veilus__<tool>`). Answer in the user's language.

## 1. Preconditions

- `get_script(script_id)` must return `approved: true` (`list_scripts` shows it for every script at once). If not, go back to stop A of the `script` skill: the user must approve it in Veilus Flow. `create_schedule` refuses unapproved scripts.
- You know the target profiles (ids) and how often the user wants it to run.

## 2. Size it to the machine

- `get_capacity()` returns `active`, `queued` and `max`, the browsers allowed at once on this computer.
- `list_schedules()` shows what else already runs, and when. Avoid stacking a new schedule on the same minutes as a heavy existing one.
- **concurrency** (1–16, default 1) is how many profiles run at once. Keep it at or below `max`, minus what other schedules use at the same time. Start low (1–3) for a new script.
- **stagger_ms** (0–600000, default 0) is the pause between launches. Use a few seconds or more (for example 5000–30000) so many profiles do not hit the site at the same second.
- A run takes roughly (profiles ÷ concurrency) × (script duration + stagger). Make sure it fits well inside the interval; the next run is refused while one is still running.

## 3. Timing

`schedule_type` is one of:

| Type | Fields | Notes |
|---|---|---|
| `interval` | `interval_minutes`, `interval_unit` (`minutes` or `seconds`) | at least 5 seconds |
| `daily` | `daily_hour`, `daily_minute` | this computer's local time |
| `weekly` | `daily_hour`, `daily_minute`, `weekly_day` (0 = Sunday) | local time |
| `cron` | `cron_expr` | 5 fields |
| `once` | `run_at` (RFC 3339 with offset, e.g. `2026-10-04T09:00:00+07:00`) | runs once, then turns itself off; must be in the future |

Use `once` for a single run later (for example "tomorrow at 9"), not a cron with a fixed date: that cron repeats next year. The computer must be on and Veilus running at those times.

## 4. Stop B: the plan table

Show the plan and wait for an explicit yes:

| | |
|---|---|
| Name | … |
| Script | NAME, version V, approved |
| Profiles | N (names or tag) |
| When | e.g. every day at 09:00 (this computer's time) |
| Concurrency / stagger | e.g. 2 at once, 10 s apart |
| One run takes about | … |
| Starts | enabled now, or created disabled |

Change it as the user asks and show it again. Do not create anything before a yes.

## 5. Create

`create_schedule(name, script_id, profile_ids, schedule_type, …timing fields…, concurrency, stagger_ms, enabled)`. It refuses unknown profile ids (404) and plan limits (the number of schedules the plan allows). Report the error as-is.

Report the returned `nextRunAt`. To pause or resume later, use `set_schedule_enabled(schedule_id, enabled)`. Turning a schedule on requires its script to still be approved.

Next, hand over to the `run` skill for a first run with `run_schedule_now`, so the user sees it work before the first timed run.
