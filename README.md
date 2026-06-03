# agent-friendly-guide

A practical, reusable method for making codebases **agent-friendly** — repos that
LLM coding agents can use, extend, and not break, without a human in the loop.

It is distilled from a real conversion (the `github.com/mixcode/binarystruct` Go
library) and validated by having a fresh agent build a non-trivial tool against
the result.

## Contents

- **[GUIDELINE.md](GUIDELINE.md)** — the method: seven pillars, an audit
  checklist, anti-patterns, and language/ecosystem notes. Start here.
- **[templates/](templates/)** — fill-in skeletons for `llms.txt`,
  `llms-full.txt`, and `AGENTS.txt`.
- **[evaluation/](evaluation/clean-agent-eval.md)** — the clean-agent evaluation
  harness: a reusable prompt and how to run it.
- **[case-study/binarystruct.md](case-study/binarystruct.md)** — the before/after
  worked example.
- **[skills/agent-ready/](skills/agent-ready/SKILL.md)** — an Agent Skill (this
  repo is also a Claude Code plugin) that runs the audit, scaffolds the
  artifacts, and offers the clean-agent evaluation.

## The skill: `agent-ready`

This repo is a Claude Code **plugin** that ships one skill, `agent-ready`. Run it
inside any repository to make it agent-friendly:

- **`/agent-ready`** — full run: audit → scaffold (`llms.txt`/`llms-full.txt`/
  `AGENTS.txt`) → offer a clean-agent evaluation.
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

**As a standalone skill** (audit works everywhere; scaffolding needs the plugin
assets):
```
cp -r skills/agent-ready ~/.claude/skills/
```

## The short version

1. Ship a machine-readable manual **in the repo** (`llms.txt` + `llms-full.txt`),
   discoverable from the README and the in-language doc comment.
2. Keep **one source of truth**; let docs and code never drift.
3. **Name the traps** with the *why* and a rule — don't hide them.
4. Give **copy-pasteable recipes** for the common paths.
5. **Design the API** so misuse is hard (remove manual bookkeeping; fail loud
   with precise errors). The deepest lever.
6. Add an **`AGENTS.txt`** for agents that modify the code (invariants + checklist).
7. **Measure** with a clean-agent evaluation on the *published* artifact; fix the
   friction; repeat.
