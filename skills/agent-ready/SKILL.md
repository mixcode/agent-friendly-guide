---
name: agent-ready
description: Make a repository agent-friendly. Audits the current repo against the agent-readiness checklist, scaffolds an agent manual (llms.txt / llms-full.txt) and a contributor guide (AGENTS.txt), and offers a clean-agent evaluation. Use when asked to make a repo or library agent-friendly or LLM-friendly, to add llms.txt / llms-full.txt / AGENTS.txt, to write an agent manual, or to assess how easily an AI agent can use a codebase.
argument-hint: "[--audit-only | --scaffold]"
---

# agent-ready

Make the **current repository** agent-friendly, following the method in
`${CLAUDE_PLUGIN_ROOT}/GUIDELINE.md` (read it if you need the full rationale; the
operational steps are below). Work on the repo rooted at the current working
directory.

## Modes (parse from `$ARGUMENTS`)

Inspect `$ARGUMENTS` for a switch and pick the stopping point. Default = full.

- **`--audit-only`** → run Phase 1 only, then stop.
- **`--scaffold`** → run Phases 1–2, then stop (do not offer the evaluation).
- **(no switch / anything else)** → full run: Phases 1–3.

State which mode you're running in one line before you start.

## Guardrails (read before writing anything)

- **Never fabricate API facts.** Fill scaffolded docs only with information you
  can verify from the repo's code, existing docs, README, or tests. Everything
  else becomes a clearly-marked `{TODO: ...}` placeholder. A confident-but-wrong
  manual is worse than a stub — it causes drift, the thing the method exists to
  prevent.
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
- Language/ecosystem and how the package is published & consumed (so you know
  where the manual must travel — module vendoring, npm `files`, Python package
  data, etc.).
- Entry-point docs: README, in-language doc surface (e.g. Go `doc.go`, module
  docstring), reference docs, any ground-truth/spec doc.
- Existing agent artifacts: `llms.txt`, `llms-full.txt`, `AGENTS.txt`/`AGENTS.md`.

Then score each item (✅ good / ⚠️ weak / ❌ missing) with a one-line note:

**Discoverability**
- [ ] `llms.txt` index at repo root.
- [ ] `llms-full.txt` (or equivalent) present and complete.
- [ ] README links the agent manual near the top.
- [ ] In-language doc comment points to the raw manual URL.
- [ ] The manual is included in the *published* artifact, not just the repo.

**Correctness & trust**
- [ ] A single ground-truth doc exists and matches the code.
- [ ] Translated/secondary docs are at parity or marked.
- [ ] Common traps (argument order, concurrency, surprising-but-deliberate
      behavior, modes/flags) are documented with the *why*.

**Usability**
- [ ] Copy-pasteable recipe for each common real-world pattern.
- [ ] Errors name the location/field and expected-vs-actual.
- [ ] Constraints validated early with clear messages.

**Contributor-readiness**
- [ ] `AGENTS.txt` lists invariants + an extension checklist.
- [ ] Parallel implementations (if any) are called out as "keep in sync."

End Phase 1 with the **top 3–5 highest-leverage fixes**. If `--audit-only`, stop here.

---

## Phase 2 — Scaffold

Generate or update the agent artifacts from the templates in
`${CLAUDE_PLUGIN_ROOT}/templates/` (`llms.txt`, `llms-full.txt`, `AGENTS.txt`).
For each:

1. Read the template.
2. Fill every section you can from verified repo facts; leave `{TODO: ...}`
   placeholders for anything you can't confirm (especially the traps and
   recipes — those need real API knowledge).
3. Write the file at the repo root (confirm first if it already exists).

Then wire **discoverability** (the cheapest, highest-return fixes):
- Add a short callout near the top of the README pointing to `llms-full.txt`.
- Add a one-line pointer in the in-language doc surface to the raw manual URL.

If the audit found a clear "common pattern," draft a copy-pasteable **recipe**
for it (or a `{TODO}` recipe stub if you can't verify the API).

Summarize what you created/changed and list the remaining `{TODO}` placeholders
the maintainer must fill. If `--scaffold`, stop here.

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
- This skill is most useful installed as part of the `agent-friendly-guide`
  plugin (so the templates and evaluation harness are available via
  `${CLAUDE_PLUGIN_ROOT}`). Phase 1 (audit) still works standalone using the
  checklist embedded above.
