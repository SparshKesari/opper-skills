---
name: agent-assurance
description: >
  A community guide to testing AI agents built on Opper end to end with Rook
  (TestMu AI's agent assurance CLI, pre-alpha). Covers connecting Rook to the
  agent the way users reach it, generating functional and adversarial
  scenarios from code, specs and policy docs, running them, reading
  per-criterion verdicts (Pass, Fail, Unable to Verify) with evidence of what
  the agent actually changed, and gating CI on the JSON report. Use it when
  the user has a tool-calling agent on Opper and asks whether it is ready for
  production, wants regression tests after a framework or model change (for
  example after migrating off the Opper Agent SDK), wants prompt-injection or
  tool-misuse tests, or asks to check the agent follows a policy, even if they
  only say "test my agent".
category: guide
metadata:
  version: "2026-10-12"
---

<!-- version-check: removed from the copy served at skills.opper.ai -->
> **Skill version 2026-10-12.** This is a locally installed copy and may be out of date. Once per session, fetch https://skills.opper.ai/versions.json and compare its `"guides/agent-assurance"` entry with 2026-10-12. If they differ, read https://skills.opper.ai/guides/agent-assurance/SKILL.md and follow that instead of this copy for the rest of the session, and tell the user once that their installed Opper skills are out of date (update with `npx skills update`, or `git pull` in a manual clone). Do not edit the installed files. If the fetch fails, continue with this copy.
<!-- /version-check -->

> **A guide, not an Opper API skill.** It shares one way to test an agent you built on Opper, using [Rook](https://www.testmuai.com/rook-cli/). The tool choice is a suggestion. For building the agent, see the `opper-sdks` skill's [`references/agents.md`](https://skills.opper.ai/opper-sdks/references/agents.md). For Rook, `rook --help`, its [coding-agent skill](https://www.testmuai.com/support/docs/rook-coding-agents/) and the [docs](https://www.testmuai.com/support/docs/agent-assurance-quickstart/) are the source of truth: Rook is pre-alpha and its commands still change.

# Testing agents built on Opper with Rook

Phase 4 of the `opper` skill calls an agent working when "the run returns a final answer after at least one tool call." That proves the loop runs. It does not prove the agent did the right thing: called the right tool with the right arguments, changed the right record, refused the injected instruction, stayed inside its refund policy. An agent's own final message is a claim, not evidence.

Rook tests the agent from outside. It derives scenarios from your code, PRDs and policy docs, runs the agent against them, and grades the **effects** on tools, files and APIs, not what the agent says it did. Each acceptance criterion comes back **Pass**, **Fail** or **Unable to Verify**, with evidence attached.

**Opper shows every model call the agent made. Rook shows whether the run did what it should.** Read them together.

> Rook's discovery, generation and judging run on TestMu AI credits, not through Opper. Your agent's own calls still go through Opper and show up in Logs and spend as usual. Generation, repair and root-cause analysis are paid operations: confirm with the user before starting them, and do not retry them automatically.

## When to use this

| Situation | Use this guide? |
|---|---|
| A tool-calling agent on Opper works in dev and is heading to production | Yes |
| You changed the framework, the prompt or the model (or migrated off the Opper Agent SDK) and need to know behaviour held | Yes, run the same scenarios before and after |
| You need prompt-injection, instruction-override or tool-misuse tests | Yes |
| You want to score single answers on every call in production | No. Use Opper **Checks (score)** |

## 1. Install

```bash
npm install -g @testmuai/rook          # or: brew install lambdatest/rook/rook
rook --version

npx @testmuai/rook-skill@latest         # coding-agent skill for Claude Code, Codex and Gemini CLI (Node 22+)
```

The skill is not the CLI; install both. With the skill installed, describe the goal in the coding agent's chat ("use rook to test my agent against its refund policy") and let it choose the Rook operations.

## 2. Sign in and set up a project

Sign-in happens inside Rook's terminal session; there is no separate headless login step.

```bash
rook
```
```text
/login
/doctor        # readiness check
/plan          # credits
/project create "Trip planner"
/explore .     # read the agent's code and docs
/agent         # select the agent under test
```

In CI, export `LT_USERNAME` and `LT_ACCESS_KEY` from your secret store; they override browser login. Never paste either into a chat.

## 3. Connect Rook to the agent

A **profile** tells Rook how to reach the agent. `/profile add` takes a curl command, a command such as `claude -p`, or an MCP tool, then writes and tests the hook scripts. Only the execute hook is required. The profile is a YAML file (`profiles/<name>.yaml`) you can review and commit; secrets are stored as variable references.

For an agent on Opper, point the profile at the same entry point your users hit: the HTTP endpoint that runs the agent loop, or the script that starts it. The agent keeps its own `OPPER_API_KEY`; Rook never needs it.

```text
/profile add staging
/profile test staging --goal "plan a two-day trip to Lisbon under 400 EUR"
/profile use staging
```

Tag the agent's model calls with `X-Opper-Tags` (for example `app:trip-planner,env:rook`) so a run's calls are easy to find in Opper **Logs**; see the `opper-sdks` agents reference.

## 4. Generate scenarios

```text
/generate --total 2 --class functional --category happy_path -- <what the agent must do>
/scenarios list
```

Rook shows the plan first: proceed, discard or change. Review it before proceeding; generation spends credits. Start small, then widen to adversarial scenarios (prompt injection, instruction override, tool misuse) and policy adherence from your policy docs. Check `/generate` help for the current classes and categories.

## 5. Run and read the report

```text
/sync
/run --only <scenario-id> --profile staging --concurrency 1 --name first-run
/report
/ui --local    # evidence on disk; /ui opens the hosted view once results are uploaded
```

`--test` runs a local experiment without syncing. A run is saved under `runs/<run-id>/`; keep run IDs and evidence, including failures and partial results.

Reading verdicts:

- **Pass:** the criterion held, with evidence of the effect.
- **Fail:** it did not; open the evidence for the step that broke.
- **Unable to Verify:** Rook could not prove it either way. It is reported apart from Fail and left out of the pass rate. Treat it as a gap to close, not a pass.

When a scenario fails, find the same run in Opper **Logs** by its tag. A right tool call with a wrong effect points at the tool or the system it calls. A wrong tool call points at the prompt, the tool schema or the model: change one thing, then rerun the same scenarios.

## 6. CI

```bash
rook project use <project-id>
rook agent use <agent-id>
rook profile use staging
rook sync
rook run --only <scenario-ids> --profile staging --concurrency 1 \
  --allow '<reviewed-rule>' --json > rook-run.json
```

- Pin the Rook version the runner installs.
- Headless runs refuse operations that no `--allow` rule covers instead of prompting; pass the exact rule you rehearsed locally.
- Treat any non-zero exit as a failed command, but do not gate on exit `0` alone: gate on `rook-run.json` showing `ok: true`, `halted: false`, a report, and the completed and passed counts you expect.
- Decide up front whether Unable to Verify blocks the merge. Recipes for GitHub Actions, Jenkins and Argo CD are in the Rook docs.

## Gotchas

- **Pin the model for test runs.** Use a provider-qualified id or a default-model rule so pooled routing does not change behaviour between runs. Compare models deliberately: same scenarios, one model change.
- **Grade effects, not transcripts.** Write criteria about what must be true afterwards (the booking exists, the refund was not issued), not about the agent's wording.
- **Use a test environment.** Scenarios really execute the agent's tools; point the profile at staging data and sandbox APIs.
- **Mind the credits.** Ask before paid operations, do not rerun tests just to get a result link, and do not retry paid operations automatically.
- **Pre-alpha.** Check `rook --help` before using a flag from this guide.

## Where to look next

| For | Look at |
|---|---|
| Building the agent on Opper | `opper-sdks` skill, [`references/agents.md`](https://skills.opper.ai/opper-sdks/references/agents.md) |
| Per-call scoring, Logs and traces | [`opper`](https://skills.opper.ai/) skill, Phase 5 |
| Rook quickstart | [testmuai.com/support/docs/agent-assurance-quickstart](https://www.testmuai.com/support/docs/agent-assurance-quickstart/) |
| Rook with coding agents | [testmuai.com/support/docs/rook-coding-agents](https://www.testmuai.com/support/docs/rook-coding-agents/) |
| Rook CLI overview | [testmuai.com/rook-cli](https://www.testmuai.com/rook-cli/) |
