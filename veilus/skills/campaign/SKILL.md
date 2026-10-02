---
name: campaign
description: Run a whole Veilus browser-automation campaign end to end over the Veilus MCP server — proxies, profiles, a tested Playwright script, a schedule, a first run and a report. Use when the user asks Veilus to do a job on a website across many profiles (for example "create 10 profiles with my US proxies and post on this site every morning").
---

# Veilus campaign — from one request to a running schedule

You drive the Veilus desktop app through its MCP server (tools named `mcp__veilus__<tool>`; the server may have another name on this machine — use whichever Veilus server is connected). Answer the user in their language.

The work has four parts. Parts 2–4 each have their own skill in this plugin; follow them when you reach that part:

1. **Understand and prepare** — this skill.
2. **Write and test the script** — the `script` skill.
3. **Plan the schedule** — the `schedule` skill.
4. **Run, watch, report** — the `run` skill.

There are **three stops where you must wait for the user**. Never skip them, never try to work around them:

| Stop | Why |
|---|---|
| A. The user approves the script in the Veilus app (sidebar Veilus Flow → the script → "Approve this script") | An agent cannot approve its own script. Schedules and `run_batch` refuse unapproved scripts. |
| B. The user agrees to the schedule plan table | A schedule runs unattended with the user's accounts. |
| C. After every change to an approved script, the user approves it again | Saving a new version drops the approval. |

## 0. Check the connection

Call `list_profiles`. If the Veilus tools are missing or every call fails to connect, stop and tell the user:

- Veilus must be running, with **API & MCP** turned on (sidebar → API & MCP).
- Claude Code must be connected with the command shown on that page (`claude mcp add veilus … mcp`), including the token.

A `401`/token error means the token is missing, wrong or revoked: mint a new one on the same page.

## 1. Get the whole request before touching anything

Ask in **one** message for whatever is missing. Do not create anything until you have it:

- **Site and job**: URL, what to do there, and what counts as success (what should be printed or checked at the end).
- **Accounts or inputs per profile**, if the job needs them (logins, texts to post). They go into a dataset (section 3b) or profile variables; never paste secrets into script source.
- **How many profiles**, and which **OS** (default: same as this computer).
- **Proxies**: an existing pool (show `list_proxy_pools`) or a list of lines to import. Ask which country/city they exit in.
- **How often**: once, every N minutes/hours, daily at a time, weekly, or a cron expression; and for how long.

Repeat the request back as a short checklist and get a yes.

## 2. Proxies

- New lines → `import_proxies(name, lines)`. Report `imported` and the `rejected` line numbers; never echo proxy passwords.
- Then `test_proxy(pool_id)` (existing pools too — it refreshes the geo that profile creation uses). Report dead proxies.
- Call `list_proxy_pools` and read the pool's `timezones` and `timezone_warning`. **If `timezone_warning` is present, stop and tell the user**: the proxies exit in several timezones, so a profile can be blocked at launch when its proxy moves to another city. Suggest pinning a **state or city** at the proxy provider (not just the country). Continue only if the user accepts the risk or fixes the pool.

## 3. Profiles

`create_profiles(count, proxy_pool_id, os?, name_template?, tags?)`:

- Each profile gets a proxy slot and a timezone/language matching that slot's exit IP.
- A profile whose slot has no known geo is listed in `failed` and not created — report it, suggest `test_proxy`, retry only those.
- If the result has `timezone_warning`, relay it to the user.
- The whole call is refused if it would exceed the plan's profile limit — tell the user the limit instead of retrying smaller batches silently.
- Use a `name_template` like `Campaign {n}` and a tag for the campaign so the user can find them.

Existing profiles instead? Use them as they are. `assign_proxy_pool` refuses profiles that already have a proxy; pass `force: true` **only after the user says so**, and warn that the timezone is not regenerated (the geo check may then block launch).

## 3b. Data for scripts

Per-profile inputs reach a script as `process.env.VEILUS_VAR_<COLUMN>`. Each profile has two dataset slots; the slot follows the dataset's mode:

- **Identity** — a `fixed` dataset: exactly one row per profile, kept for good (accounts). `create_dataset(name, mode: "fixed", columns, rows)`, then `assign_dataset(dataset_id, profile_ids)` or `identity_dataset_id` in `create_profiles`. Columns arrive as `VEILUS_VAR_<COLUMN>`. Profiles left without a row are listed in `withoutRow`: tell the user, and never share one account between profiles.
- **Content** — a `consume` dataset (posts, keywords, links) with `rows_per_run` N (1-50). Each run takes N unused rows as `VEILUS_VAR_ROWS` (JSON array of `{COLUMN: value}`); with N = 1 each column also arrives on its own. `VEILUS_VAR_ROW_INDEX` is the row's number. A failed run gives its rows back; when none are left the profile fails early: offer `append_dataset_rows` or `reset_dataset_rows`. Use `content_dataset_id` in `create_profiles`, or `assign_dataset`.
- **A value shared by one run** → `variables` in `run_script` / `run_batch`; it wins over a dataset or stored variable of the same name.

Rules:

- Values must be strings. From a spreadsheet (`.xlsx`) read the file yourself and convert numbers and dates before `create_dataset`. Bad rows come back in `rejected` by row number.
- Mark password columns `secret: true`. Secret columns are given to approved scripts but no tool returns them (`get_dataset_rows`, `list_datasets`); do not try to read them back.
- Column names are UPPER_SNAKE_CASE; `ROWS`, `ROW_INDEX`, `PROFILE_ID`, `RUN_ID`, `DEBUG_PORT` are reserved, and the two slots cannot share a column name.
- Dataset data reaches **approved** scripts only. To trial an unapproved script, pass that profile's values in `run_script(variables)`: one call per profile (max 3 profiles).

## 4. Hand over to the other skills

- Part 2: follow the `script` skill. It ends at **stop A**.
- Part 3: follow the `schedule` skill. It ends at **stop B** and creates the schedule.
- Part 4: follow the `run` skill for the first run, the report and any fix (**stop C**).

## Rules for the whole campaign

- **Nothing can be deleted over MCP.** If the user wants something removed, tell them to do it in the app.
- **Rate limit:** at most 30 expensive calls per 60 seconds per token (creating profiles, launching, running, saving scripts…). On `429`, wait the `Retry-After` seconds, then continue; never loop fast on errors.
- **`evaluate_js` is not a sandbox**: it runs with the page's full rights (cookies, requests as the user). Use it to read, not to act on the user's accounts.
- **Profile variables and dataset rows reach only approved scripts.** For a trial of an unapproved script, pass values in the `run_script` call.
- **Never print secrets**: proxy passwords, tokens, account passwords.
- **Stop profiles you launched** (`stop_profile`) when you are done learning a site.
- If a call returns something you did not expect, say so and show the message; do not guess and continue.

## Final report

Close with a short summary:

- **Pool:** its name and countries, and any timezone warning.
- **Profiles:** how many were created, with their names or tag.
- **Script:** its name, version and approval state.
- **Schedule:** its name, timing, concurrency and stagger, and the next run.
- **First run:** how many profiles passed and failed, and the main error if any.
- **Your next steps:** what the user still has to do, if anything.
