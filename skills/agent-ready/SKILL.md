---
name: agent-ready
description: Make a repository agent-friendly. Audits the current repo against the agent-readiness checklist, scaffolds an agent manual (llms.txt / llms-full.txt) and a contributor guide (AGENTS.txt), and offers a clean-agent evaluation. Use when asked to make a repo or library agent-friendly or LLM-friendly, to add llms.txt / llms-full.txt / AGENTS.txt, to write an agent manual, or to assess how easily an AI agent can use a codebase. Not for general code review or bug-hunting, writing application/runtime code, or authoring docs unrelated to agent-readiness.
argument-hint: "[--audit-only | --scaffold]"
---

# agent-ready

Make a target repository agent-friendly, following the method in
`${CLAUDE_PLUGIN_ROOT}/GUIDELINE.md` (read it if you need the full rationale; the
operational steps are below).

## Target repository

The invocation arguments are appended to this skill (look for an `ARGUMENTS:`
line). Decide which repo to work on, in this order:

1. If the arguments name a path, use that repo.
2. Otherwise use the current working directory — but **first confirm** it is the
   repo the user means: state the absolute path you're about to audit and what it
   appears to be, and proceed unless that looks wrong.

## Modes

The arguments may include a mode switch. Pick the stopping point; default to the
full run when no switch is present:

- **`--audit-only`** → run Phase 1 only, then stop.
- **`--scaffold`** → run Phases 1–2, then stop (do not offer the evaluation).
- **(no switch)** → full run: Phases 1–3.

State the target repo and the mode in one line before you start.

## Guardrails (read before writing anything)

- **Never fabricate API facts.** Fill scaffolded docs only with information you
  can verify from the repo's code, existing docs, README, or tests. Everything
  else becomes a clearly-marked `{TODO: ...}` placeholder. A confident-but-wrong
  manual is worse than a stub — it causes drift, the thing the method exists to
  prevent.
- **Stay internally consistent.** If you mark something `{TODO}` or unverified,
  do not state it as confirmed anywhere else — hedge it the same way every place
  it appears (e.g. "appears to X — unverified, see TODO"). A doc that contradicts
  itself is exactly the drift this method exists to prevent.
- **Don't overwrite existing files silently.** If `llms.txt`, `llms-full.txt`, or
  `AGENTS.txt` already exist, show a diff/summary and ask before changing them.
- **Match the repo's conventions** (style, headings, language). For non-English
  repos, mirror the existing doc language.
- **Run, don't just claim.** Use real commands to inspect the repo; quote what
  you actually found.

---

## Phase 1 — Audit

Detect the basics, then score the repo against the checklist and produce a
**prioritized gap report** (group findings as: missing / present-but-weak / good).

Detect first:
- **Repo type** — how an agent consumes it. State which it is and the signals you
  used:
  - **CLI tool** — a `bin`/`cmd/` dir, a `main` that parses flags/subcommands
    (cobra/clap/argparse), or a `package.json` `bin` / Python `console_scripts`.
  - **MCP server** — a `.mcp.json`, MCP SDK imports
    (`@modelcontextprotocol/sdk`, the `mcp` Python pkg), or code registering tools.
  - **Library / API** — exported API, no `main` / no CLI entry point.
  - **Plugin / skill** — a `SKILL.md` or `.claude-plugin/plugin.json`.
  - **Docs / methodology** — mostly Markdown, no package to install.
  A repo can be more than one (e.g. a library that also ships a CLI) — note both.
  Judge by **real artifacts** — a manifest/config file actually present, or an
  import/dependency in *source* — not by a marker string appearing in prose. A
  doc or skill that merely *describes* a signal is not that type: exclude
  `*.md`, `SKILL.md`, and `templates/` from marker greps (e.g. a file that
  mentions `modelcontextprotocol` to *explain* MCP detection is not an MCP
  server). When unsure, confirm the signal's source before classifying.
- Language/ecosystem and how the package is published & consumed (so you know
  where the manual must travel — module vendoring, npm `files`, Python package
  data, etc.).
- Entry-point docs: README, in-language doc surface (e.g. Go `doc.go`, module
  docstring), reference docs, any ground-truth/spec doc.
- Existing agent artifacts: `llms.txt`, `llms-full.txt`, `AGENTS.txt`/`AGENTS.md`.

Then score the repo against the **agent-readiness checklist**, which lives in one
place: `${CLAUDE_PLUGIN_ROOT}/GUIDELINE.md` §3 ("The audit checklist"). **Read it
now** and score each item ✅ good / ⚠️ weak / ❌ missing with a one-line note. Its
groups are *Discoverability*, *Correctness & trust*, *Usability*,
*Contributor-readiness*, and *Evidence* — use them as section headers in your
report.

**Apply the type overlay.** §3 ends with **Type overlays** (CLI tool, MCP server).
If the repo is one of those, score its overlay items too — for tools the
selection surface (a clear "use when / not for"), the invocation contract, output
and error legibility, and read-vs-mutating clarity are usually the highest-value
gaps. For any type, mark inapplicable core items **N/A** with a one-line reason
(e.g. a docs/methodology repo has no "errors name the location" item) rather than
forcing a ❌.

End Phase 1 with the **top 3–5 highest-leverage fixes**. If `--audit-only`, stop here.

---

## Phase 2 — Scaffold

**First, establish the source of truth** — what the docs must agree with, so
there is one authority and the rest derive from it:
- If the repo already has a ground-truth/spec doc, defer to it and check the new
  docs for parity against it.
- If it does not, the **code and its in-language docs (godoc / docstrings) are
  the authority**. Do NOT invent a parallel spec for a simple project — that just
  adds another thing to drift. Instead add a one-line precedence note to the
  manual, e.g. *"Source of truth: the code and godoc; this file is derived — if
  they disagree, the code wins."*
- Only **recommend** creating a dedicated spec doc when behavior is complex
  enough that it isn't self-evident from the code (flag it; don't auto-create).

Then generate or update the agent artifacts from the templates in
`${CLAUDE_PLUGIN_ROOT}/templates/` (`llms.txt`, `llms-full.txt`, `AGENTS.txt`).
For each:

1. Read the template.
2. Fill every section you can from verified repo facts; leave `{TODO: ...}`
   placeholders for anything you can't confirm (especially the traps and
   recipes — those need real API knowledge).
3. Write the file at the repo root (confirm first if it already exists). For
   files that already exist (README, the doc surface), make **additive** edits —
   insert a callout/pointer and show what changed; never silently rewrite them.

**Match the manual to the repo type** (from Phase 1). For a **CLI or MCP server**,
the manual is tool-facing, not an API cheat sheet — lead with the **selection
surface** ("use when / not for"), then the **invocation contract** (commands/flags
or tool name + input schema, auth/env), then **output and error legibility** and
which operations mutate state. For a **library**, the API cheat sheet + recipes
lead. (No dedicated tool-card template yet — fill `llms-full.txt` tool-style.)

Then wire **discoverability** (the cheapest, highest-return fixes):
- Add a short callout near the top of the README pointing to `llms-full.txt`.
- Add a one-line pointer in the in-language doc surface to the raw manual URL.

If the audit found a clear "common pattern," draft a copy-pasteable **recipe**
for it (or a `{TODO}` recipe stub if you can't verify the API).

## Phase 2.5 — Cross-check (before finishing)

Re-read every file you generated or edited, against the source of truth and
against each other. Confirm and fix before reporting:
- Every factual claim traces to something you actually read (else hedge or `{TODO}`).
- Nothing asserted in one place is marked `{TODO}` / unverified in another.
- Code snippets and recipes use real identifiers and would plausibly compile.
- Internal links and file references resolve.

Then summarize what you created/changed, list the remaining `{TODO}` placeholders
the maintainer must fill, and note in one line what you cross-checked. If
`--scaffold`, stop here.

---

## Phase 3 — Offer the clean-agent evaluation

The evaluation spawns a fresh agent to build a real tool against the *published*
artifact and report friction — it is the only honest measure of agent-readiness,
but it is **expensive** (many tool calls, real build work). So **ask before
running it**.

If the user agrees, use the harness and prompt template in
`${CLAUDE_PLUGIN_ROOT}/evaluation/clean-agent-eval.md`: help fill in the
`{braces}` for this repo (language, how to fetch the published version, a
well-scoped tool to build that exercises the error-prone parts, an independent
cross-check), run it as a sub-agent in an isolated directory, then triage its
friction log into a backlog mapped to the checklist.

If the user declines, point them to `evaluation/clean-agent-eval.md` so they can
run it later, and finish with the Phase-1 gap report as the backlog.

---

## Notes

- Full method, rationale, anti-patterns, and a worked example:
  `${CLAUDE_PLUGIN_ROOT}/GUIDELINE.md` and
  `${CLAUDE_PLUGIN_ROOT}/case-study/binarystruct.md`.
- This skill must run as part of the `agent-friendly-guide` plugin: it reads the
  checklist, templates, and evaluation harness from `${CLAUDE_PLUGIN_ROOT}`
  (`GUIDELINE.md`, `templates/`, `evaluation/`). A standalone copy of just this
  skill directory will not find them.
