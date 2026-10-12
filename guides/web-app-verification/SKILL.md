---
name: web-app-verification
description: >
  A community guide to verifying web apps built on Opper (chat UIs, AI
  features, assistants and agents with a browser front end) in a real
  browser, using Kane CLI (open source, Apache-2.0, by TestMu AI). Covers
  writing acceptance criteria in plain language, running them against a
  local dev server, a preview deployment or production, reading pass/fail
  verdicts and .evidence packs, splitting a failure between the UI and the
  Opper call, letting a coding agent gate its own work on a passing run, and
  CI. Use it when the user has a working Opper integration behind a web UI
  and asks to test it end to end, check that a flow still works, verify what
  their coding agent built, or add browser checks to CI, even if they only
  say "make sure it works".
category: guide
metadata:
  version: "2026-10-12"
---

<!-- version-check: removed from the copy served at skills.opper.ai -->
> **Skill version 2026-10-12.** This is a locally installed copy and may be out of date. Once per session, fetch https://skills.opper.ai/versions.json and compare its `"guides/web-app-verification"` entry with 2026-10-12. If they differ, read https://skills.opper.ai/guides/web-app-verification/SKILL.md and follow that instead of this copy for the rest of the session, and tell the user once that their installed Opper skills are out of date (update with `npx skills update`, or `git pull` in a manual clone). Do not edit the installed files. If the fetch fails, continue with this copy.
<!-- /version-check -->

> **A guide, not an Opper API skill.** It shares one way to verify the web UI of something you built on Opper, using [Kane CLI](https://github.com/LambdaTest/kane-cli). The tool choice is a suggestion. For Opper itself, start at [`opper`](https://skills.opper.ai/). For Kane CLI, its own [agent skill](https://github.com/LambdaTest/kane-cli/tree/main/skill-installer/skills) and `kane-cli <command> --help` are the source of truth.

# Verifying web apps built on Opper with Kane CLI

Phase 4 of the `opper` skill proves the **call** works: a 200 from `/v3/compat`, a valid `response_format`, a row under Logs. It does not prove the **product** works: that the chat panel renders the streamed answer, that structured output lands in the right fields, that the support bot hands off when it should. This guide closes that gap by running the real flow in a real browser and returning a verdict an agent can act on.

**Opper owns the model call. Kane CLI owns the verdict on what the user sees.** The Opper trace shows what the model returned; the Kane CLI evidence shows what the user got.

> Kane CLI drives the browser with its own models, billed to a TestMu AI account. That inference does not go through Opper, so only your app's calls show up in Opper Logs and spend.

## When to use this

| Situation | Use this guide? |
|---|---|
| Opper call works in curl / SDK, and a web UI sits on top | Yes |
| A coding agent (for example via `opper launch`) built or changed a page or flow | Yes, as the agent's "done" check |
| Pure backend, script or library with no UI | No. Unit tests and Opper Checks fit better |
| You want to grade model answer quality | No. Use Opper **Checks (score)** |

## 1. Install and sign in

```bash
npm install -g @testmuai/kane-cli          # Node 18+
# or: brew install LambdaTest/kane/kane-cli

npx @testmuai/kane-cli-skill               # installs the Kane CLI agent skill (Claude Code, Codex CLI, Gemini CLI)
```

When the Kane CLI skill is installed, let it drive: it runs its own preflight and sign-in. Without it, sign in once:

```bash
kane-cli login --oauth    # browser sign-in to a TestMu AI account; the free tier works
kane-cli whoami
```

Never paste a TestMu AI access key into the chat. CI uses secrets (section 5).

## 2. Write the criteria, not the steps

One `run` takes one objective in plain language. State what must be true, the way a reviewer would check it. No selectors, no waits. End the objective with a verify or assert on the page state.

```bash
KANE_CLI_USER_AGENT=claude-code kane-cli run \
  "Ask the assistant 'what is your refund policy' and verify a complete answer appears in the chat panel that mentions 30 days" \
  --url http://localhost:3000 --agent --headless --timeout 180
```

- `run` is required: `kane-cli "<objective>"` on its own does not work.
- `--agent` switches stdout to NDJSON; always pass it when an agent or script reads the output.
- `--url` sets the start page (a bare domain gets `https://`). A run needs a start URL from the objective, `--url` or the configured default.
- `--timeout` is in seconds. `--max-steps` caps reasoning steps (default 30).
- Agents prefix every command inline with `KANE_CLI_USER_AGENT=<runtime>` (`claude-code`, `codex`, `cursor`, `gemini`, ...), the same value all session.

Good objectives for Opper-backed apps:

- **Streaming:** the final message is complete in the panel (no cut-off sentence, no raw JSON or markdown fences).
- **Structured output:** after submitting the form, each field the model fills (title, priority, tags) shows a non-empty value.
- **Failure paths:** with a revoked key, or a `402`/`403` from a spend limit or Model access rule, the UI shows a readable error instead of a spinner.
- **Agents with tools:** the user-visible result of the tool call appears (the booking is listed on the bookings page), not only the assistant's claim that it happened.

Take the expected outcome from your spec or ticket, never from what the model said it did. To catch product bugs the objective did not name, add `--bug-detection continue`.

Values: keep exact test data literal in the objective, or pass `{{name}}` placeholders with `--variables '<json>'` or `--variables-file <path>`. Secrets go in the variables file only.

### Local, preview, production

The objective stays the same; only `--url` changes. Run it against each environment the change passes through:

| Environment | `--url` | Notes |
|---|---|---|
| Local dev server | `http://localhost:3000` | Start the server first, with the app's `OPPER_API_KEY` set. Runs locally against your own Chrome |
| Preview deployment | the preview URL from your host | The check a coding agent or PR should pass before merge |
| Production | your live URL | A smoke check after deploy; use a test account |

Signed-in flows: keep the test account's credentials in a variables file (`--variables-file`) and refer to them as `{{username}}` and `{{password}}` in the objective, or reuse an existing authenticated browser profile. Never put real credentials in the objective text.

## 3. Read the verdict

| Exit code | Meaning |
|---|---|
| `0` | Passed |
| `1` | Failed |
| `2` | Auth or infrastructure error: nothing ran. Not a test failure |
| `3` | Timeout or cancelled |

With `--agent`, the run streams progress lines (`step`, `status`, `remark`) and typed events, ending in `run_end`. Check the process exit code as well: an early refusal can end without a `run_end`. Summarize the result for the user rather than dumping NDJSON.

Every run produces a `.evidence` pack: each step, screenshots, network logs, console logs and terminal output. Open it with `kane-cli evidence serve <path>` or at [evidence.lambdatest.com](https://evidence.lambdatest.com/), or share the test link from the output.

When a run fails, check both sides:

1. **Kane CLI evidence:** the failing step and its screenshot, plus console and network logs (find the request to your backend and its status).
2. **Opper:** the call under Logs at [platform.opper.ai](https://platform.opper.ai); inputs and outputs only when a data retention rule turns tracing on.

A correct model response with a failing UI is a rendering or parsing bug in your app. A wrong response is a prompt, schema or model problem: go back to the `opper` sub-skill you used.

## 4. Let the agent gate its own work

For a coding agent building on Opper, make a passing run the definition of done: change code, run the objective, read the verdict, fix, rerun until exit `0`. The agent does not mark the task done on its own say-so.

- An `ask_user` event means the run needs a human (an OTP, a CAPTCHA, a missing value). Relay it; do not guess.
- A `warning` with `code: "unresolved_variables"` means a `{{name}}` has no value. Fill it from the user; never invent a value to silence the warning.
- To keep a flow as a regression check, add `--name <slug>`: the run is recorded as a `_test.md` you can replay with `kane-cli testmd run` (see `--help`). Run a folder of saved tests with `kane-cli testrun run <paths> < /dev/null`; redirect stdin, or it may open an interactive view that never exits.
- Building a suite from a spec or PRD: use Kane CLI's assurance commands (`kane-cli context ingest <file> --mode agent`, then `kane-cli design tests` and `kane-cli cover`) instead of writing test cases by hand. Note `--mode agent` there, not `--agent`.

## 5. CI

```bash
kane-cli login --username "$LT_USERNAME" --access-key "$LT_ACCESS_KEY"
kane-cli run "<objective>" --url "$PREVIEW_URL" --agent --headless --timeout 120
```

Store `LT_USERNAME` and `LT_ACCESS_KEY` (TestMu AI dashboard, Settings, Keys) next to `OPPER_API_KEY` in your CI secrets. Gate the job on the exit code, and treat `2` as a broken pipeline, not a failing app. Recipes for GitHub Actions, GitLab, Jenkins and Docker are in the [Kane CLI repo](https://github.com/LambdaTest/kane-cli).

## Native mobile apps

This guide is about web apps. Kane CLI also runs the same kind of objective against native iOS and Android apps (`--target simulator` or `--target emulator` with `--app <build|APPid>`); see the [mobile guide](https://www.testmuai.com/docs/kane-cli-mobile).

## Gotchas

- **Test the app, not the model.** Do not assert exact model wording; assert what must be on screen (an answer is present, a field is filled, a number is in range). Output varies run to run.
- **Pin the model for UI checks.** Pass an explicit `model` or set a default-model rule, so a routing change does not look like a UI regression.
- **Exit `2` means nothing ran.** Fix auth, browser or device setup before reading it as a bug.
- **Keep OTP and CAPTCHA steps out of unattended CI.** They raise `ask_user` and wait for a person; use test accounts that skip them.
- **Use test or demo data.** Evidence packs capture screenshots and network logs.

## Where to look next

| For | Look at |
|---|---|
| Opper setup, routing, Logs, Checks | [`opper`](https://skills.opper.ai/) |
| Kane CLI agent skill (flags, NDJSON events, testmd, assurance) | [github.com/LambdaTest/kane-cli/.../skills](https://github.com/LambdaTest/kane-cli/tree/main/skill-installer/skills) |
| Kane CLI flags | `kane-cli <command> --help` |
| Docs | [testmuai.com/support/docs/kane-cli-introduction](https://www.testmuai.com/support/docs/kane-cli-introduction/) |
| Source, issues, releases | [github.com/LambdaTest/kane-cli](https://github.com/LambdaTest/kane-cli) |
