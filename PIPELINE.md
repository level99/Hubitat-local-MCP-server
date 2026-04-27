# PIPELINE.md — multi-agent code-work protocol

This file documents the dispatch protocol for AI-assisted code work in this repo. It's loaded conditionally — only when an AI session is about to write, modify, refactor, fix, test, deploy, or PR code (per the trigger in [`CLAUDE.md`](CLAUDE.md)).

For project conventions, architecture, safety gates, and the substantive contribution rules, see [`SKILL.md`](SKILL.md) — that's the canonical contributor guide. This file is purely about HOW agents collaborate to make changes; SKILL.md is about WHAT correct changes look like.

---

## The pipeline (HARD rule)

**Every code change goes through this pipeline. Do not skip steps.**

```
   ┌──────────────────────┐
   │  Main session        │
   │  (you, the orch.)    │
   └──────────┬───────────┘
              │ 1. Agent({ subagent_type: 'mcp-server-developer',
              │           name: 'dev', prompt: '<task>' })
              ▼
   ┌──────────────────────┐
   │ mcp-server-developer │ ← writes code, returns diff summary.
   │ (Sonnet)             │   No tests, no deploy.
   └──────────┬───────────┘
              │ 2. CHANGE SUMMARY returned to main
              ▼
   ┌──────────────────────┐
   │ Main session         │
   │ summarizes diff      │ ← Brief human-readable recap,
   │ briefly to user      │   no blocking on user response.
   └──────────┬───────────┘
              │ 3. Agent({ subagent_type: 'mcp-server-qa',
              │           name: 'qa', prompt: 'Review diff: ...' })
              ▼
   ┌──────────────────────┐
   │ mcp-server-qa        │ ← reads diff, audits against
   │ (Opus)               │   SKILL.md / TOOL_GUIDE.md / project conventions.
   │                      │   Returns APPROVE / WARN / FAIL with file:line refs.
   └──────────┬───────────┘
              │ 4. Verdict returned to main
              ▼
        ┌─────┴──────┐
   FAIL/WARN       APPROVE
        │             │
        │             ▼
        │      ┌──────────────────────┐
        │      │ mcp-server-tester    │ ← lint first (~5s).
        │      │ (Haiku)              │   If lint FAIL: short-circuit, skip Spock.
        │      │                      │   If lint PASS/WARN: ./gradlew test (~3-5min).
        │      │                      │   Returns PASS / FAIL / UNCERTAIN.
        │      └──────────┬───────────┘
        │                 │ 5. Verdict returned to main
        │                 ▼
        │           ┌─────┴─────┐
        │      FAIL/UNCERTAIN  PASS
        │           │           │
        │           │           ▼
        │           │    ┌──────────────────────┐
        │           │    │ With Hubitat MCP:    │
        │           │    │ mcp-server-operations│ ← deploy + verify on hub
        │           │    │ (Haiku)              │   logs, return PASS/FAIL/UNCERTAIN.
        │           │    │                      │
        │           │    │ Without MCP:         │
        │           │    │ orchestrator commits │
        │           │    │ + returns manual     │
        │           │    │ deploy steps to user │
        │           │    └──────────────────────┘
        │           │
        │           └─→ same loop as FAIL/WARN below
        │
        │ 6. Main relays issues briefly to chat:
        │    "QA found N issues:
        │     - BLOCKING: [...]
        │     - WARN: [...]"
        │
        │ 7. SendMessage({ to: '<dev-agent-id>',
        │                  message: 'QA feedback: ...' })
        │    (agent IDs come from the Agent dispatch return — see Rule 1)
        ▼
   ┌──────────────────────┐
   │ mcp-server-developer │ ← RESUMED via SendMessage with agent ID.
   │ (Sonnet)             │   Cache stays warm across rounds.
   │                      │   Don't dispatch fresh — that wastes ~60-70%
   │                      │   input tokens.
   └──────────┬───────────┘
              │ 8. Updated diff → return to step 3
              ▼
        (loop until APPROVE → tester PASS → optional deploy)
```

---

## Agent roles + cost rationale

Four agents, each tuned to a model tier matching its work shape. Anchored cost ratios (per-token, normalized to Haiku input = 1):

| Model | Input | Output |
|---|---|---|
| Haiku | 1 | 5 |
| Sonnet | 3 | 15 |
| Opus | 5 | 25 |

| Agent | Model | Why this tier | Spec |
|---|---|---|---|
| `mcp-server-developer` | Sonnet | Code-writing workhorse — high output volume; ~5× cheaper output than Opus while easily handling Groovy + project conventions when primed by spec | [`.claude/agents/mcp-server-developer.md`](.claude/agents/mcp-server-developer.md) |
| `mcp-server-qa` | Opus | Input-heavy review work, low output (structured findings); reasoning-quality matters more than throughput; reading diffs ≠ reading whole files so per-round cost is bounded | [`.claude/agents/mcp-server-qa.md`](.claude/agents/mcp-server-qa.md) |
| `mcp-server-tester` | Haiku | Mechanical: invoke `./gradlew test` + `sandbox_lint.py`, parse output, return ~1KB structured summary. Test-output-dominated work (Gradle logs, Spock failures, JUnit XML) is where Haiku saves most cost | [`.claude/agents/mcp-server-tester.md`](.claude/agents/mcp-server-tester.md) |
| `mcp-server-operations` | Haiku | Mechanical: backup → upload → `update_app_code` → fetch logs → parse → structured PASS/FAIL. Same input-heavy / mechanical-output shape as tester. Only used in the with-MCP context | [`.claude/agents/mcp-server-operations.md`](.claude/agents/mcp-server-operations.md) |

**Total**: iterative work through this pipeline runs at roughly **30-40% of the cost of doing everything in the main Opus session**, primarily because:
- Sonnet for code-writing rather than Opus
- Haiku for log-dominated test/deploy work
- QA reads diffs (small) not whole files
- Resume-by-ID preserves agent caches across rounds (see Rule 1)

---

## Rules

### 1. Always try resume first; fall back to fresh Agent only on failure

For every handoff after the very first dispatch of a role, the protocol is:

a. **Capture the agent ID** from each `Agent({...})` dispatch result — the response includes `agentId: <id>` (e.g., `a745afcdc2e7112d6`). Store it for the duration of the round.

b. `SendMessage({ to: '<agentId>', message: "..." })` — **addresses by agent ID, NOT by the descriptive `name:` parameter.** Per [code.claude.com/docs/en/sub-agents#resume-subagents](https://code.claude.com/docs/en/sub-agents#resume-subagents): *"When a subagent completes, Claude receives its agent ID. Claude uses the SendMessage tool with the agent's ID as the `to` field to resume it."*

c. If the recipient's prior run has already exited, the runtime **resumes the agent from its transcript in the background** (`SendMessage` returns `success: true` with message *"had no active task; resumed from transcript in the background"*). Same-session agents from many hours ago are typically still resumable; transcript-cached state replays cheaply. Only when the transcript itself has been evicted does `SendMessage` return `success: false`.

d. **Only then** dispatch a fresh `Agent({ name: ..., ... })` with full re-briefing context (file paths, task recap, all prior QA/tester findings the agent needs, the specific delta being asked for).

**Note on the `name:` parameter:** the `name:` field in `Agent({name: 'dev', ...})` is a human-readable label for the dispatch (visible in transcripts/UI), but it does NOT make the agent addressable by that name in `SendMessage`. The in-environment SendMessage tool description may say *"refer to teammates by name, never by UUID"* — that's correct for **agent teams** (TeamCreate-style coordination) but **not** for single-session subagent resume. For subagent resume, ALWAYS use the agent ID. Using the wrong addressing (name instead of ID) consistently fails — every name-based send returns `"No agent named '...' is currently addressable"` and forces a fresh dispatch, defeating the cache-preservation purpose entirely.

**Detection heuristic:** if you sent a SendMessage and want to know whether the agent is alive: if `SendMessage` itself returned `success: false`, the session is gone — fresh-dispatch immediately. If it returned success but no completion notification arrives in a reasonable window (~60s for trivial follow-ups, ~3min for code edits), assume the agent exited and re-dispatch fresh.

**Re-dispatch must re-supply context:** a fresh agent has no memory of prior rounds. The re-briefing prompt MUST include: the file paths, the task recap, all prior QA/tester findings the agent needs, and the specific delta being asked for. Copy-paste from the SendMessage body you were about to send, plus enough preamble to orient cold.

**Why this matters:** resume preserves ~60-70% input tokens via warm cache (the MCP server source is several thousand lines — reading it fresh every round is expensive). Fresh dispatches reload everything. **Never default to fresh just because it's easier to compose** — that silently burns the cache savings the pipeline was designed to capture.

### 2. Always run QA before tester or deploy

Even for "trivial" changes — many bugs in this codebase have looked trivial and shipped anyway. After QA APPROVE, run tester (Spock harness + sandbox lint) before deploy/commit. Tester catches regressions QA can't (compile errors, runtime sandbox-class-resolution issues, behavior assertions).

### 3. Brief human-readable summary between rounds, but don't block on user response

When QA or tester returns findings, output a brief human-readable summary to the main chat BEFORE forwarding to the developer. Then proceed directly to the SendMessage handoff. The user reads the summary and can interrupt if they want to redirect; otherwise the pipeline keeps flowing.

Example (dispatching dev resume immediately after):

> QA found 3 issues on the get_app_config diff:
> - BLOCKING: `findApp` missing `.isInteger()` validation on appId arg (line 6285)
> - BLOCKING: ASCII-punctuation rule violation — em-dash in IllegalArgumentException msg (line 6291)
> - WARN: `sendRmActionFallback` could inline into `sendRmAction` — caller's call
>
> Resuming developer with fixes.

### 4. Division of responsibility

- **Developer** writes code, returns CHANGE SUMMARY (files touched, intent, notable decisions, risk surface, self-check verification). Does NOT run tests, does NOT deploy.
- **QA** reviews diff. Returns APPROVE / WARN / FAIL with file:line refs and specific fixes. Does NOT touch the hub, does NOT run the test suite, does NOT write code.
- **Tester** runs `sandbox_lint.py` then `./gradlew test`. Returns PASS / FAIL / UNCERTAIN with verbatim failure quotes. Does NOT modify code, does NOT deploy.
- **Operations** (with-MCP context only) backs up the hub, uploads the source, calls `update_app_code`, fetches logs, returns PASS / FAIL / UNCERTAIN. Does NOT modify code, does NOT make architectural decisions.
- **Main (orchestrator, you)** dispatches, summarizes findings to user, makes calls on ambiguous cases (tester UNCERTAIN, dev-QA disagreement), handles commit/push/PR under the global preview-before-publish rule, owns the deploy decision.

### 5. Iteration cap: 3 rounds

If QA flags BLOCKING on 3 consecutive rounds OR tester returns FAIL after 2 rounds of dev fixes on the same specs, escalate to the user. Usually means the spec is wrong, the task is under-specified, or there's an infrastructure gap (e.g., a sandbox-classloader issue requiring harness work).

### 6. Honest pushback on disagreements

If the developer thinks QA's feedback would cause a regression (e.g., *"the suggested fix would break the fallback semantics QA itself flagged last round"*), surface the disagreement to the user for a decision. Don't rubber-stamp either side.

### 7. Tester UNCERTAIN handling

If the tester reports UNCERTAIN (output pattern it couldn't classify — flaky-looking spec, unfamiliar Spock error, transient environment issue), main reviews the verbatim output and decides: **benign** (tell tester to tag and continue), **concerning** (feed back to dev or escalate). Don't let UNCERTAIN sit — it blocks PASS. For benign patterns the tester confirmed once, those stay benign across rounds (cache discipline).

### 8. Tester runs lint first (fast static check), then Spock

Lint is ~5 seconds; Spock is ~3-5 minutes. If `sandbox_lint.py` returns errors (not warnings — warnings are tolerable), tester reports the lint findings and SKIPS the Spock run. Reason: structural bugs (missing trailing commas, em-dash in IllegalArgumentException, sandbox-forbidden calls inside GStrings) produce noisy spurious Spock failures stacked on top of the real issue, which obscures both. Fix the lint issue first, then re-run the full pipeline. Lint warnings still allow Spock to proceed; the warning is reported alongside the Spock verdict.

### 9. Pre-PR tester-only runs are fine

If the user asks "run tests on this branch" without a code change in play, dispatch tester directly. Don't spin up dev/QA just to babysit a test run.

### 10. No hub deploy from the dev/QA/tester pipeline

Deploying to the running hub is a separate step that the operations agent (with-MCP context) or the orchestrator manually (without-MCP context) handles. The pipeline's job ends at green tests. The orchestrator decides when to deploy based on the broader change context (PR state, user intent, etc.).

### 11. Outbound comments / PRs stay orchestrator-owned

Under the global preview-before-publish rule, any GitHub PR comment, review reply, or PR body goes through user approval in chat BEFORE posting. Dev/QA/tester/ops never post to GitHub directly. They produce the substance (findings, test results, etc.); the orchestrator drafts and posts under user approval.

---

## Two deployment contexts

### A. With Hubitat MCP server (maintainer + admin contributors)

After QA + tester PASS, dispatch `mcp-server-operations`. It handles the deploy + verify cycle and returns a structured PASS/FAIL/UNCERTAIN report. The agent needs:
- Source file path on disk
- Hub appId for the MCP Rule Server (typically a small integer; check `manage_apps_drivers list_hub_apps`)
- Test plan (which MCP tools to call post-deploy, what behavior to verify)

The orchestrator can also do the deploy itself for trivial changes:
1. Confirm a recent (`<24h`) backup exists, or call `create_hub_backup`
2. Upload source: `curl -F "uploadFile=@<file>" -F "folder=/" "http://<hub-IP>/hub/fileManager/upload"`
3. Deploy: `update_app_code` MCP tool with `appId`, `sourceFile`, `confirm: true`
4. Verify: invoke a sentinel tool (e.g., a known-affected `get_*` tool), inspect response shape

Use the operations agent when: you want structured PASS/FAIL evidence captured in the work item, the test plan is non-trivial (multiple tool calls + log inspection), or you want to keep the main session's context lean (operations is Haiku, much cheaper than Opus on log-heavy verification).

### B. Without Hubitat MCP server (typical contributor)

The pipeline still works — you just stop at the green-tests step. Push your changes and let the maintainer (or you, manually via the Hubitat web UI) deploy:

1. Run developer + QA + tester pipeline as above.
2. Once tester returns PASS, commit changes locally.
3. To verify in the Hubitat web UI:
   - **Apps Code** → find "MCP Rule Server" → **Edit** → paste new content → **Save**.
   - Use the MCP endpoint (configured in app settings) to invoke an affected tool from any MCP client.
   - Check **Logs** for the application instance.
4. Open a PR with the diff + test plan + manual verification notes.

You don't have to deploy to merge — code review + spec-conformance via the dev/QA/tester pipeline is sufficient gate. The maintainer will deploy on the live hub before/during merge.

---

## Example dispatch prompts

### Initial dev dispatch (small feature)

```
Agent({
  subagent_type: 'mcp-server-developer',
  name: 'dev',
  prompt: `
Add a small follow-up to existing tool foo: when args.bar is null, return
[success: false, error: "bar is required", note: "..."] instead of throwing
NPE inside the existing helper.

Requirements:
- Validation goes at the top of toolFoo(args)
- Error map shape matches the project's existing pattern (see SKILL.md)
- ASCII-only punctuation in the error string
- Add Spock spec covering null-bar + valid-bar paths
- BAT-v2 scenario if the tool is on the BAT list (check tests/BAT-v2.md)
- No version bump

Return CHANGE SUMMARY only. Do not run tests; do not deploy.
  `
})
```

### Resume dev with QA feedback

```
SendMessage({
  to: '<dev-agent-id-from-prior-dispatch>',
  message: `
QA returned 2 BLOCKING + 1 WARN:

1. BLOCKING line 234: ASCII-only rule violation -- em-dash in error string.
   Replace with " -- " (ASCII).
2. BLOCKING line 245: Spock spec asserts args.bar == null but the helper
   coerces "" to null upstream; assert on the empty-string path too.
3. WARN line 178: redundant null-guard; toolFoo's caller already validates.
   Optional cleanup.

Apply 1 + 2; 3 is your call. Return updated diff.
  `
})
```

### Tester re-dispatch after dev fixes

```
SendMessage({
  to: '<tester-agent-id-from-prior-dispatch>',
  message: `
Dev pushed fixes for the 2 BLOCKING items. Re-run lint + Spock and report
whether previously-failing specs now pass.
  `
})
```

---

## When this file is wrong

If reality and this file diverge (new agent role added, dispatch protocol shifts, model-tier ratios change as Anthropic updates pricing, the runtime's resume behavior changes), update this file in the same change. The agents read this file as part of their context priming; outdated PIPELINE.md silently misleads future work and silently burns the cost-discipline savings the pipeline was designed to capture.

Concrete signals that an update is overdue:
- A rule was followed exactly and still produced a wrong result
- A pattern documented here doesn't match what the agent specs say
- An outbound message had to be rewritten because the format described here is stale
- `SendMessage` keeps returning `success: false` despite the agent allegedly being live (typically: name vs. agent ID drift, or runtime resume-window changed)
- A new failure mode keeps recurring and the rules don't address it

Don't gather drift into a "documentation pass later" — fix in-session. Five-minute file edit beats hours of re-debugging the same wrong-pattern next time.
