---
name: agent-ready
description: Make a repository agent-friendly. Audits the current repo against the agent-readiness checklist, scaffolds an agent manual (llms.txt / llms-full.txt) and a contributor guide (AGENTS.txt), and offers a clean-agent evaluation. Use when asked to make a repo or library agent-friendly or LLM-friendly, to add llms.txt / llms-full.txt / AGENTS.txt, to write an agent manual, or to assess how easily an AI agent can use a codebase. Not for general code review or bug-hunting, writing application/runtime code, or authoring docs unrelated to agent-readiness.
argument-hint: "[--full | --audit-only | --scaffold | --monorepo]"
---

# agent-ready

Make a target repository agent-friendly, following the method in
`${CLAUDE_PLUGIN_ROOT}/GUIDELINE.md` (read it if you need the full rationale; the
operational steps are below).

**Plugin root (harness-agnostic).** This skill reads its checklist, templates,
language guides, and evaluation harness from the **plugin root** — the directory that
contains `GUIDELINE.md`, `templates/`, `languages/`, and `evaluation/`. On Claude Code
that root is `${CLAUDE_PLUGIN_ROOT}`; on any other agent harness it is the installed
plugin's directory (typically the parent of this skill's `skills/agent-ready/`).
**Resolve it first**, then read from it. Throughout this file, `${CLAUDE_PLUGIN_ROOT}`
is shorthand for that root — substitute your harness's path if the variable is unset.

**Preflight.** If you cannot locate the plugin root, or `GUIDELINE.md` is unreadable
there (e.g. the skill was bare-copied, or your harness imported only the skill
component without the rest of the plugin), **stop and tell the user to make the full
plugin available** rather than proceeding with missing guidance — on Claude Code:
`/plugin marketplace add mixcode/agent-friendly-guide` then
`/plugin install agent-friendly-guide@agent-friendly-guide`; on other harnesses, ensure
the repo's `GUIDELINE.md` / `templates/` / `languages/` / `evaluation/` are reachable
beside the skill (see the README's "Other agent harnesses" note).

## Target repository

The invocation arguments are appended to this skill (look for an `ARGUMENTS:`
line). Decide which repo to work on, in this order:

1. If the arguments name a path, use that path.
2. Otherwise use the current working directory — but **first confirm** it is the
   target the user means: state the absolute path you're about to audit and what it
   appears to be, and proceed unless that looks wrong.

**Then resolve the *scope* — the package boundary, which is not always the repo
root.** In a monorepo/megarepo each publishable package lives in its own
subdirectory, and *that subdirectory* — not the repo root — is the unit you audit
and scaffold. Detect this before Phase 1:

- **Is the target inside a workspace?** Look upward and around for a workspace
  manifest: `pnpm-workspace.yaml`, npm/yarn `workspaces` in `package.json`, Cargo
  `[workspace]` in `Cargo.toml`, `go.work`, `nx.json`, `turbo.json`, `lerna.json`,
  Maven `<modules>` in `pom.xml`, Gradle `include`s in `settings.gradle(.kts)`.
  **A workspace manifest is sufficient but not necessary** — a repo can hold
  multiple independently-publishable packages with no top-level workspace file at
  all. The decisive signal is **more than one package manifest in the tree**: most
  tellingly **multiple `go.mod` files** (nested Go modules need no `go.work`), but
  also several `Cargo.toml`/`package.json`/`pyproject.toml` each defining a real
  package. If you see that, treat it as a workspace even when no workspace manifest
  ties them together.
- **Resolve the target to one of three:**
  - **standalone** — no workspace manifest; the repo root *is* the package root.
    Proceed as normal (everything below collapses to "repo root").
  - **workspace-member** — the target is (or is inside) one member package. **This
    member's directory is the package root.** Scaffold *its* manual into *its*
    subdir; evaluate "does the manual travel?" against *its* own publish config
    (its `package.json` `files`, its `pyproject.toml`, its `.gitattributes`), not
    the root's. The monorepo root is *often* not separately published, so a manual
    there may reach no consumer — but **don't assume it**: in some ecosystems the
    root manifest is itself a published package (notably **Go**, where the root
    `go.mod` is the primary module). When the root is published too, it needs its
    own manual, not only a map of the others.
  - **workspace-root with no member named** — the target is the monorepo root and
    the user didn't pick a package. **Don't audit the root as if it were a
    package.** If `--monorepo` is set, run the **monorepo survey** (below). Otherwise
    list the member packages you found and ask whether to survey them all
    (`--monorepo`) or target one.
- **Read upward for context, write within the boundary.** You may read the
  monorepo root (shared build config, tooling, top-level README) to *understand*
  the package, but scope every file you create or edit to the package root.

When this section and the phases below say "repo root," read it as **package
root** — they're the same for a standalone repo and the member's subdir for a
workspace member.

## Modes

The arguments may include a mode switch. Pick the stopping point:

- **`--audit-only`** → run Phase 1 only, then stop.
- **`--scaffold`** → run Phases 1–2, then stop (do not offer the evaluation).
- **`--full`** → full run: Phases 1–3. **This is the default** — also used when no
  switch is given.
- **`--monorepo`** → **monorepo survey** (breadth, not depth): when the target is a
  workspace root, enumerate every member package, triage each, and write a
  root-level **monorepo map**. See "Monorepo survey mode" below. `--monorepo` overrides
  the depth switches — it does **not** scaffold each member (run a single-package
  pass per member for that). Only meaningful at a workspace root; on a standalone
  repo it falls back to a normal `--full` run.
- **`--drift-audit`** → **drift audit & fix only**: run Phase 1's drift check
  (below) over the native doc surface, then **correct every contradiction against
  the code** (Phase 2's "fix drift first"), and stop. Do **not** scaffold a manual.
  This is the highest-value pass for a well-documented, source-shipped library —
  where the read surface, not a manual, is the lever (see GUIDELINE §2a, "is the
  contract legible?").

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

## Monorepo survey mode (`--monorepo`)

Run this **instead of** Phases 1–3 when `--monorepo` is set and the target is a
workspace root. It maps the whole monorepo and triages each package; it does
**not** scaffold them (deep work stays one-package-at-a-time). The guardrails
above still apply.

1. **Enumerate the members.** Prefer the workspace manifest when there is one
   (don't guess from the directory tree alone); when there isn't, fall back to
   **finding every package manifest in the tree** — a repo can be multi-package
   with no workspace file.
   - **pnpm** — `packages:` globs in `pnpm-workspace.yaml`.
   - **npm / yarn** — the `workspaces` array/globs in the root `package.json`.
   - **Cargo** — `members` (and `exclude`) under `[workspace]` in `Cargo.toml`.
   - **Go** — `use` directives in `go.work` **if present; otherwise every `go.mod`
     in the tree** (`find . -name go.mod`, excluding `vendor/`). Nested modules
     need no `go.work`, so go.work-only enumeration misses them. Treat each
     distinct `go.mod` as a member — **including the root `go.mod` itself**, which
     in Go (unlike most ecosystems) is normally a published module in its own right.
   - **lerna / nx / turbo** — `packages` in `lerna.json`; for nx/turbo the members
     are usually the `workspaces` packages — resolve those.
   - **Maven** — `<modules>` in the parent `pom.xml` (recurse for nested modules).
   - **Gradle** — `include`/`includeBuild` in `settings.gradle(.kts)`.
   - **No manifest convention matches** — fall back to "every directory with its
     own package manifest is a member."
   Expand the globs to concrete directories and keep only those that are actually
   publishable packages (have their own manifest).
2. **Triage each member** — a *lightweight* scan, not the full audit (that's the
   single-package pass). For each, record: its **type** (Phase 1 signals), whether
   it has its **own `llms.txt`** and a **full manual**, whether its **README points
   to the manual**, and whether the **manual would travel** per *that member's own*
   publish config (its `files`/`package_data`/`.gitattributes`/`spec.files`). Roll
   that into a coarse status — **✅ ready / ⚠️ partial / ❌ not started** — plus the
   one-line top gap.
3. **Write the monorepo map** to the **monorepo root as `llms.txt`** (the name an
   agent looks for at a repo root), from
   `${CLAUDE_PLUGIN_ROOT}/templates/llms-monorepo.txt`. Fill the package table
   (name, path, type, manual link, status) and the "Working in this monorepo"
   notes (workspace manager, how to build one package, shared-config locations,
   the build-from-root trap). If a root `llms.txt` already exists, show a diff and
   confirm before overwriting. Verify the build/filter commands you write are real
   (read the workspace manifest / scripts) — don't invent them.
   **If the root is itself a published package** (e.g. a Go repo whose root
   `go.mod` is the primary module), don't reduce its `llms.txt` to a bare map: that
   root also needs a real manual. List it as a member in the table like any other,
   recommend a single-package run on it (step 5), and let the sub-module map live
   *alongside* the root's own index rather than replacing it.
4. **Cap and be honest about coverage.** For a large megarepo, if there are more
   than ~30 members, confirm scope first or triage a named subset — and **never
   silently truncate**: list any package you did not survey.
5. **Recommend the next moves.** End with the highest-leverage packages to make
   agent-ready first (e.g. the most-depended-on or most-published), each as a
   ready-to-run single-package command — `/agent-ready {path/to/member}`.

---

## Phase 1 — Audit

Detect the basics, then score the repo against the checklist and produce a
**prioritized gap report** (group findings as: missing / present-but-weak / good).

Detect first:
- **Repo type** — how an agent consumes it. State which it is and the signals you
  used:
  - **CLI tool** — a `bin`/`cmd/` dir, a `main` that parses flags/subcommands
    (cobra/clap/argparse/picocli/SwiftCLI/ArgumentParser), or a `package.json`
    `bin` / Python `console_scripts` / Cargo `[[bin]]` / a shell `bin/` of
    scripts.
  - **MCP server** — a `.mcp.json`, MCP SDK imports
    (`@modelcontextprotocol/sdk`, the `mcp`/`fastmcp` Python pkg), or code
    registering tools.
  - **Library / API** — exported API, no `main` / no CLI entry point (Cargo
    `src/lib.rs`, Package.swift `.library`, a JVM jar with no app entry, an
    importable package, a sourced-functions shell lib).
  - **Plugin / skill** — a `SKILL.md` or `.claude-plugin/plugin.json`.
  - **Docs / methodology** — mostly Markdown, no package to install.
  A repo can be more than one (e.g. a library that also ships a CLI) — note both.
  Judge by **real artifacts** — a manifest/config file actually present, or an
  import/dependency in *source* — not by a marker string appearing in prose. A
  doc or skill that merely *describes* a signal is not that type: exclude
  `*.md`, `SKILL.md`, and `templates/` from marker greps (e.g. a file that
  mentions `modelcontextprotocol` to *explain* MCP detection is not an MCP
  server). When unsure, confirm the signal's source before classifying.
  **Know each ecosystem's manifest** — `go.mod`, `Cargo.toml`, `package.json`,
  `pyproject.toml`/`setup.py`, **`pom.xml`/`build.gradle(.kts)`**,
  **`Package.swift`**, `Makefile`/`CMakeLists.txt` — and **verify its role, don't
  just key on presence**: a `pyproject.toml` may be tool-config only (check for a
  `[project]`/`[build-system]` table), a `package.json` may target Bun/Deno, a
  `bin` field may be `null`. **Shell has no manifest** — use `.sh`/`.bash` files,
  shebangs, a `bin/` dir, and sourced-functions-vs-`main` shape.
- Language/ecosystem. **Read `${CLAUDE_PLUGIN_ROOT}/languages/<lang>.md`** if it
  exists (`go`, `python`, `javascript`, `c`, `rust`, `jvm`, `shell`, `swift`,
  `php`, `ruby`) — it
  gives that ecosystem's detection signals, doc surface, distribution model, and
  traps. If there's no file for the language, fall back to the agnostic guidance
  and the **distribution model** taxonomy in GUIDELINE §4a (A ships-all / B opt-in
  allowlist / C git-URL / D repo-binary-file-is-the-artifact) — this tells you
  where the manual must live to reach the consumer.
- Entry-point docs: README (also check non-root locations — `.github/README.md`,
  `docs/`), the ecosystem's in-language doc surface (named in
  `languages/<lang>.md` — e.g. Go `doc.go`, Rust rustdoc, JVM `package-info.java`/
  Javadoc, Swift DocC, Python `__init__.py` docstring, a C header top-comment,
  shell SHDOC), reference docs, any ground-truth/spec doc.
- Existing agent artifacts: `llms.txt`, `llms-full.txt`, `AGENTS.txt`/`AGENTS.md`,
  and other agent-facing docs — **`CLAUDE.md`, `.cursorrules`,
  `.github/copilot-instructions.md`**. If a contributor-agent doc already exists,
  **align with / point to it rather than authoring a duplicate `AGENTS.txt`** that
  will drift from it.

Then score the repo against the **agent-readiness checklist**, which lives in one
place: `${CLAUDE_PLUGIN_ROOT}/GUIDELINE.md` §5 ("The audit checklist"). **Read it
now** and score each item ✅ good / ⚠️ weak / ❌ missing with a one-line note. Its
groups are *Discoverability*, *Correctness & trust*, *Usability*,
*Contributor-readiness*, and *Evidence* — use them as section headers in your
report.

**Apply the type overlay.** §5 ends with **Type overlays** (Library / API, CLI tool, MCP server).
If the repo is one of those, score its overlay items too — for tools the
selection surface (a clear "use when / not for"), the invocation contract, output
and error legibility, and read-vs-mutating clarity are usually the highest-value
gaps. For any type, mark inapplicable core items **N/A** with a one-line reason
(e.g. a docs/methodology repo has no "errors name the location" item) rather than
forcing a ❌.

**For a library, score the agent's *observed read surface*, not file presence —
and don't assume it's `doc.go`.** A library's agent won't open a separate root
manual on its own, and the surface it *does* read is repo-specific: in testing
agents went to the example tests and a task-named reference doc and **skipped
`doc.go` entirely**. So **identify, empirically, which surfaces a clean agent reads
for a representative task** — the fastest way is the Phase-3 clean-agent eval (or a
quick throwaway `claude -p` on a real task) and watch which files it opens; commonly
the **example tests + a task-named reference doc + source**. Then score
Discoverability on whether the **decisive traps + canonical recipe sit inline on
those surfaces**, and whether the **existing docs already serve the agent** (the
highest-leverage gap is content the read surfaces *don't* carry, not a missing
`llms-full.txt`). Flag a manual that is bloated or merely duplicates a surface the
agent already reads as `present-but-weak`, not a ✅. **Do not rely on a separate
manual — or any filename — to attract the agent; it won't.**

**Drift audit — the read surface vs the code (highest-value for legible-contract repos).**
For every behavioral claim in the surface the agent actually reads — the
in-language docs (godoc/`doc.go`, docstrings, header comments), the README's usage
prose, and type hints — **verify it against what the code actually does**: units,
defaults, argument order, keyword names, valid value sets, return type/semantics,
and error conditions. **Trace into helpers/delegates where the real behavior lives**
— do not assume a public docstring matches the implementation it calls. Any claim
that contradicts the code is a **drift finding** and ranks at the **top** of the
fix list: per GUIDELINE §3a this is the asymmetric risk — a *wrong* doc makes an
agent fail silently (it trusts the doc and never re-checks the code), whereas a
*missing* doc only costs it some reading. Report each as
`claim → actual code behavior → file:line`.

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
`${CLAUDE_PLUGIN_ROOT}/templates/` (`llms.txt`, `llms-full.txt`, `AGENTS.txt`;
plus `llms-full-cli.txt` for a CLI/tool — see below). For each:

1. Read the template.
2. Fill every section you can from verified repo facts; leave `{TODO: ...}`
   placeholders for anything you can't confirm (especially the traps and
   recipes — those need real API knowledge).
3. Write the file at the **package root** (the repo root for a standalone repo;
   the member's subdir for a workspace member — see "Target repository"; confirm
   first if it already exists), then
   **make it reach the consumer per the distribution model** (Phase 1 / GUIDELINE
   §4a): for an **opt-in allowlist** (B) add it to `package_data`/`MANIFEST.in` /
   npm `files` / JVM resources; for a **single copied file** (D: single-header C,
   one `.sh`) put the pointer **and the key traps inside that file**, since a
   separate `llms-full.txt` won't travel with it. For files that already exist
   (README, the doc surface), make **additive** edits — insert a callout/pointer
   and show what changed; never silently rewrite them, and **never edit inside a
   generated region** (e.g. a SHDOC-generated README, a `<!-- generated -->`
   block) — put the callout in a hand-maintained section or the generator's source.

**Fix drift first.** Before (and often instead of) scaffolding any manual, correct
every **drift finding** from Phase 1: make a minimal edit to the offending
docstring/comment/README line so it states **what the code actually does** — the
code is the source of truth. For a source-shipped **library** this is the
single highest-value action, because the agent reads that surface and a separate
manual won't be read (and could not rescue a wrong surface anyway). Show each
correction as a diff against the original. If `--drift-audit`, stop after this.

**Match the manual to the repo type** (from Phase 1). For a **CLI or MCP server**,
the manual is tool-facing, not an API cheat sheet — start from the
`llms-full-cli.txt` template and lead with the **selection surface** ("use when /
not for"), then the **invocation contract** (commands/flags or tool name + input
schema, auth/env), then **output and error legibility** and which operations
mutate state. For a **library**, use `llms-full.txt` — the API cheat sheet +
recipes lead — but because a library's agent **reads the task-relevant surfaces and
won't open a separate manual on its own**, the real work is to **inline the decisive
traps and the one canonical recipe onto the surface(s) the agent was *observed* to
read in Phase 1** — commonly an **annotated runnable example** (a Go `Example` test)
and a **task-named reference doc**, and the in-language doc comment *only if* the
agent actually reads it (don't default to `doc.go`). Make those surfaces **true and
drift-free** (fix-drift step above). Keep `llms-full.txt` a **lean delta** for the
human-prompted case, and **aim that delta at the domain residue** — the
project-specific knowledge that's in neither the code, the API names, nor the agent's
training prior: house conventions, units/encodings, business rules, locale/binding/
format quirks, the *why* behind a deliberate wart. Don't restate signatures the agent
reads or general facts it already knows (a bloated, redundant manual goes unread, and
wastes tokens if forced).

Then wire **discoverability** — but treat pointers as a *fallback*, not the
mechanism (testing shows agents often don't follow them; the lever is content on the
read surface):
- Add a short callout near the top of the README pointing to `llms-full.txt`
  (hand-edited region, not generated; if the README lives in `.github/`, edit there).
- Add a one-line pointer in the in-language doc comment to the raw manual URL (see
  `languages/<lang>.md`) — cheap insurance, but **do not rely on it**.
- The **primary** move is the inlined traps/recipe on the agent's observed read
  surface (above). Do **not** try to attract the agent with a clever or imperative
  manual filename — it doesn't work; spend the effort on the read surface.
- **Never** wire a rule/hook (CLAUDE.md, a pre-task instruction) that forces "read
  `llms-full.txt` first" — at best neutral, counterproductive for well-documented repos.

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
cross-check). **Spawn it** with your sub-agent tool (e.g. the Task/Agent tool) as
a fresh general-purpose agent, instructed to work in an isolated temp dir with no
other context; if you have no way to spawn one, hand the filled-in prompt to the
user to run in a clean session. Then triage its friction log into a backlog
mapped to the checklist. **If the sub-agent can't
build/compile/run** (read-only or sandboxed environment), use that harness's
**discovery-only mode** — it reads the published artifact and reports
first-contact friction without building, still yielding the discoverability and
selection signal.

If the user declines, point them to `evaluation/clean-agent-eval.md` so they can
run it later, and finish with the Phase-1 gap report as the backlog.

**Quantitative A/B (when a change is contested or two designs compete).** Beyond the
qualitative friction report, you can put a number on it: run a fresh agent N times
(N≈4) per condition on a task with a *checkable* answer and compare **pass-rate**
first, novel-token cost second. Conditions: **A** = artifacts stripped, **B1** =
artifacts present + told to read the manual, **B2** = artifacts present, self-discover
(B2≈A means the agent never found the manual — a read-path failure, GUIDELINE §3a). For
the token figure, read the reasoning model's *intake (input + cache-creation) + output*
from the run's structured usage (e.g. `--output-format json`, per-model block); exclude
cache reads and any background helper model. **Decisive caveat:** at *equal success* on
small, well-documented repos the token delta saturates to ≈neutral — the agent solves
from source regardless — so a flat token result does **not** mean the artifact is
worthless. The artifact's value shows up in the **failure regime**: inject the exact
condition it targets (an *unverifiable* doc/code drift, or *missing* domain residue) and
measure pass-rate there. Lead with correctness; use tokens only to break ties between
designs that both pass.

---

## Notes

- Full method, rationale, and anti-patterns: `${CLAUDE_PLUGIN_ROOT}/GUIDELINE.md`.
  Per-language specifics: `${CLAUDE_PLUGIN_ROOT}/languages/<lang>.md`.
- This skill must run as part of the `agent-friendly-guide` plugin: it reads the
  checklist, templates, and evaluation harness from `${CLAUDE_PLUGIN_ROOT}`
  (`GUIDELINE.md`, `templates/`, `evaluation/`). A standalone copy of just this
  skill directory will not find them.
