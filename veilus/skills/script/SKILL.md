---
name: script
description: Learn a website with Veilus page-driving tools, then write, save and test a Playwright (TypeScript) script for Veilus Flow until it passes, and hand it to the user for approval. Use when a Veilus campaign needs a script, or the user asks for a Veilus script for a site.
---

# Write and test a Veilus script

Tools are on the Veilus MCP server (`mcp__veilus__<tool>`). Answer in the user's language.

## 1. Learn the site on a real profile

1. Pick one profile of the campaign. `launch_profile(profile_id)`. On `409` (pool full) check `get_capacity`. On a geo/timezone mismatch, tell the user; the proxy's exit IP moved, so do not force it.
2. `navigate(profile_id, url)`. If the page has not finished loading after 15 seconds, the result says it is still loading: then `wait_for_element` for what you need before `snapshot`. A slow first load is normal on a fresh profile with an empty cache.
3. `snapshot(profile_id)` lists up to 150 elements as `role name [id=N]`. Ids are only valid until the page changes; take a new snapshot after every navigation or click. If `truncated: true`, use `wait_for_element(selector)` to reach elements that are not listed. Elements inside a same-origin iframe appear after a `--- iframe URL ---` line and work like any other; a cross-origin iframe shows only its URL, so navigate to that URL to work inside it. In a Playwright script, reach iframe content with `page.frameLocator(...)`.
4. Walk the job by hand with `click(node_id)`, `type(node_id, text)` (it replaces the field's content), `press_key(key)` and `scroll(direction, amount)`. After each step, `wait_for_element` for what should appear next. If a step opens a JavaScript alert/confirm/prompt, the tool answers it at once and says so: pass `dialog: "accept"` (and `prompt_text` for a prompt) to `click`, `press_key` or `evaluate_js` when the job needs OK rather than the default Cancel. In the script, answer the same dialog with `page.once("dialog", d => d.accept())` before the action. For a file upload field, use `set_file(node_id, path)` instead of clicking it (a click opens the system file picker). It only takes files inside `~/.veilus/data/uploads`, so ask the user to copy the file there first. In the script, use `page.setInputFiles(selector, path)`.
5. Write down, for the script, **stable selectors** for each step: prefer `getByRole`/`getByLabel`/`getByText` or ids and `data-*` attributes over nth-child chains. `evaluate_js` can read the DOM to find them (`document.querySelector(...)?.outerHTML`), but it is **not a sandbox**: never use it to act on the user's accounts.
6. Record what "success" looks like: a URL, a text, an element.
7. `stop_profile(profile_id)` when you are done learning.

## 2. Write the script to the contract

`save_script` rejects violations with line numbers. The contract:

1. Connect with `chromium.connectOverCDP(\`http://127.0.0.1:${process.env.VEILUS_DEBUG_PORT}\`)`.
2. Never `chromium.launch` / `launchPersistentContext`: the profile's browser is already running.
3. Use `browser.contexts()[0]`, never `newContext()`: the profile's cookies and fingerprint live in that context.
4. Never `setUserAgent` / `setViewportSize`: identity belongs to the profile.
5. Errors must exit non-zero: `main().catch((e) => { console.error(e); process.exit(1); });`
6. Import only `playwright` and Node built-ins.
7. End `main()` with `await browser.close()`. It only disconnects from the profile's browser and lets the script exit; without it the run hangs.

Per-profile inputs come from `process.env.VEILUS_VAR_<NAME>` (UPPER_SNAKE_CASE). Print what you need to check with `console.log`; stdout and stderr tails appear in `get_run_result`.

Dataset columns arrive the same way. A content dataset also sets `VEILUS_VAR_ROWS`: `JSON.parse` it (it may hold fewer rows than asked when the dataset is nearly empty), and `VEILUS_VAR_ROW_INDEX`: print it next to each result. Every value is a string.

Template:

```ts
import { chromium } from "playwright";

async function main() {
  const browser = await chromium.connectOverCDP(
    `http://127.0.0.1:${process.env.VEILUS_DEBUG_PORT}`,
  );
  const context = browser.contexts()[0];
  const page = context.pages()[0] ?? (await context.newPage());

  await page.goto("https://example.com", { waitUntil: "domcontentloaded" });
  // ... steps learned in part 1, with explicit waits:
  // await page.getByRole("button", { name: "Continue" }).click();
  // await page.waitForURL(/done/);

  console.log("RESULT", JSON.stringify({ ok: true, title: await page.title() }));
  await browser.close();
}

main().catch((e) => {
  console.error(e);
  process.exit(1);
});
```

Good practice:

- Wait for elements and URLs explicitly; no fixed sleeps longer than needed.
- Keep secrets out of the source: account passwords and texts go in variables.
- Fail loudly: throw when the success check does not hold, so the run is marked failed.

## 3. Save, trial, fix

0. Call `list_scripts` first. It lists every script with `name`, `mode`, `origin` and `approved`, without the source. If one with the same purpose exists and its `origin` is `mcp`, update it with `save_script(script_id, source)` instead of saving a duplicate. Scripts with `origin: app` belong to the user: read them, never overwrite them.
1. `save_script(name, source, description)` returns `scriptId`, `version` and `approved: false`. To change it later, call `save_script(script_id, source)` again. Only scripts saved by an agent can be updated, and updating does not rename them.
2. Trial it with `run_script(profile_ids, script_id, variables?)` on **at most 3** profiles. It returns a run immediately; its `id` is the run id.
3. Poll `get_run_result(run_id)` every few seconds until the run is no longer running. Read each profile's exit code and stdout/stderr tail.
4. On failure, read the error, fix the source (use `get_script` to reread it), save again, trial again. Stop after about 5 failed attempts and show the user the last error instead of looping.
5. Unapproved scripts get **only** the variables passed to `run_script`: not the profiles' stored variables, not their dataset rows. If each profile needs different values (its own account), make one `run_script` call per profile, each with that profile's values.

## 4. Stop A: approval

When the trial passes, stop and ask the user to approve it:

> The script "NAME" (version V) passed the trial on N profiles. Please open Veilus Flow, open "NAME", review the source, and click **Approve**. Tell me when it is done.

Then confirm with `get_script(script_id)` that `approved` is `true` before any schedule or batch run. If it is still `false`, ask again; do not try to work around it.

After approval, per-profile inputs go into a dataset (campaign skill, section 3b); `set_profile_variables(profile_id, variables)` remains for a one-off value. It **replaces all** of that profile's variables: omitted names are deleted. It returns names, never values.
