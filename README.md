# agent-friendly-guide

[![llms.txt](https://img.shields.io/badge/llms.txt-compliant-blue)](llms.txt)

> [!NOTE]
> **AI agents**: start with [llms.txt](llms.txt); the full method/manual is
> [GUIDELINE.md](GUIDELINE.md).

A practical, reusable method for making codebases **agent-friendly** — repos that
LLM coding agents can use, extend, and not break, without a human in the loop.

It is distilled from real-world conversions of production libraries and CLI
tools, validated by having fresh agents build non-trivial things against the
results, and tested across ten language ecosystems.

## Contents

- **[GUIDELINE.md](GUIDELINE.md)** — the method: a contract-legibility diagnostic,
  the work in leverage order, an audit checklist, anti-patterns, and
  language/ecosystem notes. Start here.
- **[templates/](templates/)** — fill-in skeletons for `llms.txt`,
  `llms-full.txt`, `llms-full-cli.txt` (the CLI/tool-facing manual),
  `llms-monorepo.txt` (a monorepo root's package map), and `AGENTS.txt`.
- **[evaluation/](evaluation/clean-agent-eval.md)** — the clean-agent evaluation
  harness: a reusable prompt and how to run it.
- **[languages/](languages/)** — per-ecosystem specifics (signals, doc surface,
  distribution model, traps) for Go, Python, JavaScript, C, Rust, JVM, Shell, Swift, PHP, Ruby.
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
- **`/agent-ready --monorepo`** — **monorepo survey**: point it at a workspace root and
  it enumerates every member package, triages each, and writes a root *monorepo
  map* (`llms.txt`) showing where each package's manual lives and how agent-ready
  it is. Deep per-package work still runs one package at a time
  (`/agent-ready packages/foo`).

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

### Other agent harnesses

The **method** (`GUIDELINE.md`) is vendor-neutral — it reasons about "an agent," not a
specific product, and has been validated on more than one model family (Claude and
Gemini both apply it correctly). The **skill** is packaged as a Claude Code plugin, but
it runs on any harness that can (1) load a skill and (2) let that skill read the plugin's
support files (`GUIDELINE.md`, `templates/`, `languages/`, `evaluation/`). Harnesses that
import *only* the skill component need those files placed alongside the skill.

From **v0.3.10** the skill self-recovers: if a harness imported only the skill component
and you have web access, it fetches the missing support files directly from the
published release, **version-pinned** to the skill (so the guide always matches). The
clone+symlink below is then only needed for **offline / no-web** harnesses.

Worked example — **Antigravity (`agy`)**, running on a Gemini model:

```
# 1. import the skill from the official repo
agy plugin install https://github.com/mixcode/agent-friendly-guide
# 2. agy keeps only the skill component, so make the support files reachable:
git clone https://github.com/mixcode/agent-friendly-guide ~/agent-friendly-guide
for f in GUIDELINE.md templates languages evaluation; do
  ln -sfn ~/agent-friendly-guide/$f ~/.gemini/config/plugins/agent-friendly-guide/$f
done
# 3. run it
agy -p "/agent-ready --scaffold" --model "Gemini 3.1 Pro (High)" --dangerously-skip-permissions
```

(Paths are `agy`-specific; adapt the plugin dir to your harness. `git pull` the clone to
track new releases.)

## The short version

1. **The law.** A manual's value = what the task needs **−** what's already *legible*
   to the agent (from the **code**, the **API names**, and its **training prior**).
   Spend effort on the **residue** — the project-specific knowledge that's in none of
   those — and not where the agent already sees the contract.
2. **Diagnose before you invest.** Is the contract legible from what the agent actually
   reads? *Legible* (an ordinary source-shipped library) → invest in the source and kill
   drift, **don't** write a manual. *Not legible* (a CLI, service, or indirection-heavy
   code) → write a manual **aimed at the residue**, shipped so it travels per the
   ecosystem's distribution model.
3. **Keep the read surface true — and on the path the agent actually reads.** Drift is
   worse than absence (a wrong doc fails silently); prioritize the *unverifiable*
   conventions an agent can't re-derive from adjacent code. Verify which surface a clean
   agent opens and inline the decisive traps + recipe **there** — don't assume it reads a
   separate manual.
4. **Design the API so misuse is hard** — remove manual bookkeeping; fail loud with
   precise errors. The deepest lever.
5. **Name the traps** (with the *why* + a rule) and give **copy-pasteable recipes** for
   the common paths.
6. **Always: a selection surface** (a one-line "use when / not for") and an
   **`AGENTS.txt`** for agents that modify the code (invariants + checklist).
7. **Measure** with a clean-agent evaluation on the *published* artifact — **pass-rate
   first, tokens second**; fix the friction; repeat.
