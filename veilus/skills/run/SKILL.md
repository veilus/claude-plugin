---
name: run
description: Run an approved Veilus script now — through its schedule or as a one-off batch — watch the run, read per-profile results, fix and re-approve if it fails, and report. Use after a Veilus schedule is created, or when the user asks to run a Veilus script on profiles or check how runs went.
---

# Run, watch and report

Tools are on the Veilus MCP server (`mcp__veilus__<tool>`). Answer in the user's language.

## 1. Start a run

- **Scheduled campaign:** `run_schedule_now(schedule_id)` runs it once without waiting for the timer. It is refused while that schedule already has a run in progress, and when its script is no longer approved.
- **One-off, no schedule:** `run_batch(script_id, profile_ids, concurrency?, stagger_ms?, variables?)`. The script must be approved. Size `concurrency` with `get_capacity`. An unknown profile id counts as a failed profile; it is not rejected.
- Both return **immediately**. The returned `id` is the run id.

Variables passed to the run win over a profile's stored variables of the same name.

## 2. Watch

- Poll `get_run_result(run_id)` every 5–15 seconds (mind the rate limit). Stop polling when the status is no longer running.
- `list_runs(schedule_id?, limit?)` shows recent runs with counts only. Use it to find a run id or to review history. Without `schedule_id` it lists every run, including `run_script` and `run_batch`, and batch runs started from the app.
- A run stuck much longer than (profiles ÷ concurrency) × (script duration + stagger) deserves a look. `get_capacity` shows whether profiles are queued behind other work.

## 3. Read the result

For each profile, `get_run_result` gives an exit code and the tail of stdout and stderr. Group the failures by cause:

| Symptom | Likely cause | What to do |
|---|---|---|
| Timeout or selector not found on some profiles | Site changed, slow proxy, or a different page (consent banner, captcha) | `launch_profile` that profile, `navigate` and `snapshot` to see the page; fix the script |
| Same error on every profile | Script bug | Fix the script |
| Launch blocked by timezone/geo mismatch | That profile's proxy exit IP moved to another offset | Tell the user; suggest pinning a state/city at the provider |
| Pool full, `409` | More profiles than this machine allows at once | Lower concurrency |
| `429` | Rate limit | Wait `Retry-After`, continue |

## 4. Fix loop (stop C)

1. Fix the source and `save_script(script_id, source)`. The new version is **not approved**.
2. Trial it with `run_script` on at most 3 of the failing profiles, then read `get_run_result`.
3. When it passes, **stop and ask the user to approve the new version** in Veilus Flow. Confirm `approved: true` with `get_script`.
4. Rerun with `run_schedule_now` or `run_batch`.

Never keep a schedule running a script you know is broken. Offer `set_schedule_enabled(schedule_id, false)` while you fix it.

## 5. Report

- **Run:** its id, when it ran, and how many profiles passed and failed.
- **Failures:** each cause, the affected profiles, and what you did or recommend.
- **Schedule:** the next run time, and whether the schedule is enabled.
- **For the user:** anything left to do, such as approving a new version or fixing proxies.

Stop any profile you launched for debugging (`stop_profile`).
