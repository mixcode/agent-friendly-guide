# agent-friendly-guide

[![llms.txt](https://img.shields.io/badge/llms.txt-compliant-blue)](llms.txt)

> [!NOTE]
> **AI agents**: start with [llms.txt](llms.txt); the full method/manual is
> [GUIDELINE.md](GUIDELINE.md).

A practical, reusable method for making codebases **agent-friendly** — repos that
LLM coding agents can use, extend, and not break, without a human in the loop.

It is distilled from real-world conversions of production libraries and CLI
tools, validated by having fresh agents build non-trivial things against the
results, and tested across eight language ecosystems.

## Contents

- **[GUIDELINE.md](GUIDELINE.md)** — the method: seven principles, an audit
  checklist, anti-patterns, and language/ecosystem notes. Start here.
- **[templates/](templates/)** — fill-in skeletons for `llms.txt`,
  `llms-full.txt`, `llms-full-cli.txt` (the CLI/tool-facing manual), and `AGENTS.txt`.
- **[evaluation/](evaluation/clean-agent-eval.md)** — the clean-agent evaluation
  harness: a reusable prompt and how to run it.
- **[languages/](languages/)** — per-ecosystem specifics (signals, doc surface,
  distribution model, traps) for Go, Python, JavaScript, C, Rust, JVM, Shell, Swift.
- **[skills/agent-ready/](skills/agent-ready/SKILL.md)** — an Agent Skill (this
  repo is also a Claude Code plugin) that runs the audit, scaffolds the
  artifacts, and offers the clean-agent evaluation.

## The skill: `agent-ready`

This repo is a Claude Code **plugin** that ships one skill, `agent-ready`. Run it
inside any repository to make it agent-friendly:

- **`/agent-ready`** (or `--full`) — full run: audit → scaffold
  (`llms.txt`/`llms-full.txt`/`AGENTS.txt`) → offer a clean-agent evaluation.
  `--full` is the default when no switch is given.
- **`/agent-ready --scaffold`** — audit + scaffold, stop before the evaluation.
- **`/agent-ready --audit-only`** — just the prioritized gap report.

(When installed as a plugin the command is namespaced: `/agent-friendly-guide:agent-ready`.)

The skill never fabricates API facts — it fills the templates only with what it
can verify from the repo and leaves clearly-marked `{TODO}` placeholders for the
rest.

### Install

**As a plugin (recommended — bundles the templates + eval harness):**
```
/plugin marketplace add mixcode/agent-friendly-guide
/plugin install agent-friendly-guide@agent-friendly-guide
```
Local development install: `claude --plugin-dir /path/to/agent-friendly-guide`.

**Standalone (not recommended):** copying just the skill directory
(`cp -r skills/agent-ready ~/.claude/skills/`) makes the `/agent-ready` command
available, but the skill reads its checklist, templates, language guides, and
evaluation harness from the plugin via `${CLAUDE_PLUGIN_ROOT}` — a bare copy
won't find them. Install as the plugin above instead.

## The short version

1. Ship a machine-readable manual **in the repo** — `llms.txt` plus a full manual
   (`llms-full.txt`, or an existing doc that already serves as one), discoverable
   from the README and the in-language doc comment.
2. Keep **one source of truth**; let docs and code never drift.
3. **Name the traps** with the *why* and a rule — don't hide them.
4. Give **copy-pasteable recipes** for the common paths.
5. **Design the API** so misuse is hard (remove manual bookkeeping; fail loud
   with precise errors). The deepest lever.
6. Add an **`AGENTS.txt`** for agents that modify the code (invariants + checklist).
7. **Measure** with a clean-agent evaluation on the *published* artifact; fix the
   friction; repeat.
