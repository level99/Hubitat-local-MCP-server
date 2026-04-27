# CLAUDE.md — orientation for AI-assisted contributors

This file is auto-loaded by Claude Code (and similar AI coding agents that respect the convention) at the start of every session in this repo. It exists to point you at the right docs for what you're trying to do — not to duplicate them.

**Human contributors don't need this file.** It's purely AI-tooling orientation. If you're a human reading this for project understanding, start with [`SKILL.md`](SKILL.md) — that's the canonical contributor guide.

---

## What this project is

A Hubitat Elevation SmartApp that exposes a Hubitat hub's devices, rules, and admin surface via the [Model Context Protocol (MCP)](https://modelcontextprotocol.io/) so AI assistants can read and control a smart home through a single endpoint. The parent app implements the MCP protocol; a child app implements an MCP-native rule engine. Detailed architecture, tool conventions, safety-gate tiers, and the de-facto contribution rules are in [`SKILL.md`](SKILL.md).

---

## When to load `PIPELINE.md`

If the user asks you to **write, modify, refactor, fix, test, deploy, or PR code in this repo**, read [`PIPELINE.md`](PIPELINE.md) before editing. It documents the multi-agent dispatch protocol (developer / QA / tester / operations), iteration discipline, deployment context, and cost-conserving resume rules.

Trigger phrases that indicate you should load it: *"write a tool"*, *"add a feature"*, *"fix the bug"*, *"refactor"*, *"update the helper"*, *"run tests"*, *"deploy to hub"*, *"open a PR"*, *"address Gemini"*, *"apply the review"*.

`PIPELINE.md` is intentionally NOT auto-loaded — it's a few-tens-of-KB and only relevant when doing code work. Sessions that just answer questions don't need it.

---

## Doc routing for common AI-session questions

| User asks about | Read | When |
|---|---|---|
| Project architecture, conventions, contribution rules | [`SKILL.md`](SKILL.md) | Almost always — it's the substantive guide |
| What an MCP tool does, its arguments, examples | Call the tool with no args (returns full schema), or read [`TOOL_GUIDE.md`](TOOL_GUIDE.md) | Tool-specific question |
| Test harness setup, Spock conventions, mocks | [`docs/testing.md`](docs/testing.md) | Anything test-related |
| Release history, what shipped in v0.X.Y | [`CHANGELOG.md`](CHANGELOG.md) | Past releases |
| Roadmap, future direction, philosophy | [`futureplans.md`](futureplans.md) | Future direction |
| Install, supported features, user-facing overview | [`README.md`](README.md) | User-facing question |

---

## Source of truth for AI-introduced confusion

The project's safety-critical patterns (gate tiers, tool definition shape, sandbox-forbidden calls, version-string consistency rules, BAT scenario format, etc.) live in [`SKILL.md`](SKILL.md) — not here. If you're about to make a non-trivial code change and haven't read SKILL.md this session, do so before editing. The pipeline (PIPELINE.md) and the agent specs (`.claude/agents/*`) reference SKILL.md as canonical; updates to project conventions go there, not here.

---

## When this file is wrong

If reality and this file diverge (new doc added, tool surface restructured, agent spec changes its dispatch), update this file in the same change. Outdated CLAUDE.md silently misleads future AI sessions, which compounds. Five-minute fix beats hours of agent confusion later.
