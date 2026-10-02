# Veilus plugin for Claude Code

Ask Claude for a job on a website, and it does the whole thing in Veilus over MCP:

- imports your proxies;
- creates profiles whose timezone and language match each proxy;
- learns the site in a real profile;
- writes and tests a Playwright script;
- plans a schedule;
- runs it, and reports how it went.

You stay in control at three points:

1. You **approve the script** in the Veilus app before it can run unattended.
2. You **agree to the schedule plan** before it is created.
3. You **re-approve** the script whenever it has to change.

Vietnamese: [README.vi.md](README.vi.md).

## What you need

- **Veilus** desktop app **newer than 0.2.1** (0.2.1 does not have the campaign tools these skills use), running, on a **paid plan or the free trial**. The Free plan has no API access.
- **Claude Code**: https://claude.com/claude-code.
- Proxies, if the job needs them. Use static or sticky proxies pinned to a **state or city**; a whole-country pool is not enough (see [Proxies and timezones](#proxies-and-timezones)).

## Setup

### 1. Turn on the API in Veilus

1. Open Veilus → sidebar **API & MCP**.
2. Click **Turn on port**.
3. Under **Tokens**, name a token (for example `claude-code`) and click **Create token**. **Copy it now**: it is shown only once.

### 2. Connect Claude Code to Veilus (MCP)

On the same page, under **Connect an LLM → Claude Code**, click **Copy**. Run it in a terminal **with `--scope user` added right after `add`**, so Veilus is available in every folder you open Claude Code in (without it, Claude Code only sees Veilus in the folder where you ran the command):

```bash
claude mcp add --scope user veilus -e VEILUS_API_TOKEN=<your token> -- "<path to Veilus>" mcp
```

If it says `veilus` already exists, run `claude mcp remove veilus` first.

Check it:

```bash
claude mcp list
```

`veilus` should be listed as connected. Veilus must be running whenever Claude uses it.

> Using Cursor or Claude Desktop instead? Copy the JSON block under **Cursor / Claude Desktop (JSON)** into that app's MCP settings. The skills below are for Claude Code.

### 3. Install the plugin

```bash
claude plugin marketplace add veilus/claude-plugin
claude plugin install veilus@veilus
```

Or inside Claude Code: `/plugin marketplace add veilus/claude-plugin`, then `/plugin install veilus@veilus`.

To update later: `claude plugin marketplace update veilus`.

Restart Claude Code, then type `/veilus:` and you should see the four skills.

### 4. Try it

In Claude Code:

```
/veilus:campaign
```

Or just describe the job. Claude picks the skill:

```
Using my "Singapore" proxy pool, create 3 profiles and every morning at 9:00 open
https://example.com and print the page title. Report how the first run went.
```

## The skills

| Skill | What it does |
|---|---|
| `/veilus:campaign` | The whole job: asks what it needs, prepares proxies and profiles, then uses the three skills below. |
| `/veilus:script` | Learns the site in a real profile, writes a script, trial-runs it on up to 3 profiles, fixes it, then asks you to approve it. |
| `/veilus:schedule` | Sizes a schedule to your machine, shows you the plan as a table, and creates it after you agree. |
| `/veilus:run` | Runs the script now, watches the run, explains failures per profile, fixes and asks for re-approval if needed, and reports. |

Each skill also works on its own. For example, "check how last night's runs went" uses `run`.

## Approving a script

When Claude says a script is ready:

1. Open Veilus → sidebar **Veilus Flow** and open the script. Scripts written by Claude are marked as coming from MCP.
2. Read the source.
3. Click **Approve this script**.
4. Tell Claude it is done.

Why this step exists: an approved script runs on its schedule without anyone watching, using your profiles and their logins. Claude can trial-run an unapproved script on at most 3 profiles, but only you can approve it. Saving a new version always needs a new approval.

## Example requests

- "Import these proxies, test them, and tell me which ones are dead:" followed by the proxy lines.
- "Create 10 Windows profiles on my US pool, tagged `spring-sale`."
- "Write a script that logs in with each profile's `EMAIL` and `PASSWORD` variables and checks the dashboard loads. Don't schedule it yet."
- "I have accounts.csv with email and password. Put it in a dataset and assign one account to each profile tagged `spring-sale`."
- "Schedule the approved script `Daily check` for all `spring-sale` profiles every day at 08:30, 2 at a time."
- "Run `Daily check` now and tell me which profiles failed and why."

## Proxies and timezones

Each profile gets the timezone of its proxy's exit IP when it is created. When the profile is launched, Veilus checks that its proxy still exits in the same timezone. If not, the launch is blocked, because a mismatched timezone is an easy tell for websites.

Proxies pinned only to a country can jump between cities with different timezones: New York and Chicago, or Sydney and Perth. Veilus warns you when a pool spans several timezones. Fix it by pinning a **state or city** at your proxy provider, so every proxy in the pool shares one timezone.

## Safety

- Claude **cannot delete** anything in Veilus over MCP.
- Replacing an existing profile's proxy needs explicit confirmation, and Claude asks you first.
- Claude never prints proxy passwords or your token.
- Page scripting (`evaluate_js`) runs with the page's full rights, like a script you would run in the browser console. Claude uses it to read pages, not to act on your accounts.
- The API is limited to 30 expensive calls per minute per token.
- Revoke a token any time: **API & MCP → Tokens → Revoke**.

## Troubleshooting

| Problem | Fix |
|---|---|
| `veilus` missing from `claude mcp list`, or tools are not found | It was added for another folder only. Re-add it with `--scope user` (setup step 2), then restart Claude Code. |
| Connection refused / cannot reach Veilus | Veilus is not running, or the port is off: **API & MCP → Turn on port**. |
| `401` / token rejected | The token was revoked or mistyped. Create a new token and re-run the connect command with it. |
| "API access requires a paid plan" | The API needs a paid plan or the trial. |
| "needs your approval" / "not approved" | Approve the script in Veilus Flow (see [Approving a script](#approving-a-script)). |
| Launch blocked: timezone mismatch | The profile's proxy moved to another timezone. Pin a state or city at your provider, or use a single-timezone pool. |
| A profile was not created: "geo lookup failed" | Claude could not tell where that proxy exits. Ask Claude to test the pool again; replace dead proxies. |
| `409` pool full when launching | Too many browsers at once. Lower the schedule's concurrency or close some profiles. |
| `429` | Rate limit. Claude waits and continues. |
| Schedule did not run | The computer must be on and Veilus running at that time; check the schedule is enabled. |
