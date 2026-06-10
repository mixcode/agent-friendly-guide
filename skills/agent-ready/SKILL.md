---
name: agent-ready
description: Make a repository agent-friendly. Audits the current repo against the agent-readiness checklist, scaffolds an agent manual (llms.txt / llms-full.txt) and a contributor guide (AGENTS.txt), and offers a clean-agent evaluation. Use when asked to make a repo or library agent-friendly or LLM-friendly, to add llms.txt / llms-full.txt / AGENTS.txt, to write an agent manual, or to assess how easily an AI agent can use a codebase. Not for general code review or bug-hunting, writing application/runtime code, or authoring docs unrelated to agent-readiness.
argument-hint: "[--full | --audit-only | --scaffold | --monorepo]"
---

# agent-ready

Make a target repository agent-friendly, following the method in
`${CLAUDE_PLUGIN_ROOT}/GUIDELINE.md` (read it if you need the full rationale; the
operational steps are below).

**Preflight.** This skill reads its checklist, templates, language guides, and
evaluation harness from `${CLAUDE_PLUGIN_ROOT}`. If that variable is empty, or
`${CLAUDE_PLUGIN_ROOT}/GUIDELINE.md` is unreadable (e.g. the skill was bare-copied
without the plugin), **stop and tell the user to install the plugin** —
`/plugin marketplace add mixcode/agent-friendly-guide` then
`/plugin install agent-friendly-guide@agent-friendly-guide` — rather than
proceeding with missing guidance.

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
  and the **distribution model** taxonomy in GUIDELINE §2 (A ships-all / B opt-in
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

**For a library, score the agent's *default path*, not just file presence.** A
library's agent reads the in-language docs (godoc/docstrings) and examples first
and often won't open a separate root manual on its own. So score Discoverability
on whether the **decisive traps + canonical recipe are reachable inline on that
path** (godoc/`doc.go`, a runnable example), and assess whether the **existing
docs already serve the agent** — the highest-leverage gap is usually content the
conventional docs *don't* carry, not a missing `llms-full.txt`. Flag a manual that
is bloated or merely duplicates an already-good godoc as `present-but-weak`, not a ✅.

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
   §2): for an **opt-in allowlist** (B) add it to `package_data`/`MANIFEST.in` /
   npm `files` / JVM resources; for a **single copied file** (D: single-header C,
   one `.sh`) put the pointer **and the key traps inside that file**, since a
   separate `llms-full.txt` won't travel with it. For files that already exist
   (README, the doc surface), make **additive** edits — insert a callout/pointer
   and show what changed; never silently rewrite them, and **never edit inside a
   generated region** (e.g. a SHDOC-generated README, a `<!-- generated -->`
   block) — put the callout in a hand-maintained section or the generator's source.

**Match the manual to the repo type** (from Phase 1). For a **CLI or MCP server**,
the manual is tool-facing, not an API cheat sheet — start from the
`llms-full-cli.txt` template and lead with the **selection surface** ("use when /
not for"), then the **invocation contract** (commands/flags or tool name + input
schema, auth/env), then **output and error legibility** and which operations
mutate state. For a **library**, use `llms-full.txt` — the API cheat sheet +
recipes lead — but because a library's agent reads the **in-language docs and
examples first and often won't open a separate manual**, also **inline the
decisive traps and the one canonical recipe into the doc surface** (godoc/`doc.go`,
a docstring) **and add or annotate a runnable example** (a Go `Example` test) that
shows the trap-prone path. Keep `llms-full.txt` a **lean delta** over the existing
docs — front-load the traps/recipe and link out to good existing docs; don't
restate an already-complete godoc (a bloated, redundant manual just goes unread,
and wastes the agent's tokens if it's ever forced to read it).

Then wire **discoverability** (the cheapest, highest-return fixes):
- Add a short callout near the top of the README pointing to `llms-full.txt`
  (in a hand-edited region, not a generated one; if the README lives in
  `.github/`, edit it there).
- Add a one-line pointer in the ecosystem's in-language doc surface to the raw
  manual URL (see `languages/<lang>.md` for which surface).
- For a **library**, the *inline* traps/recipe in the doc surface + example
  (above) are the **primary** discoverability surface; the raw-URL pointer is a
  fallback an agent may not follow once the godoc answers it. Spend the effort
  inline, not just on the pointer.
- **Never** wire a rule/hook (CLAUDE.md, a pre-task instruction) that forces "read
  `llms-full.txt` first" — make it findable and let the agent pull; forcing a read
  is at best neutral and counterproductive for well-documented libraries.

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

---

## Notes

- Full method, rationale, and anti-patterns: `${CLAUDE_PLUGIN_ROOT}/GUIDELINE.md`.
  Per-language specifics: `${CLAUDE_PLUGIN_ROOT}/languages/<lang>.md`.
- This skill must run as part of the `agent-friendly-guide` plugin: it reads the
  checklist, templates, and evaluation harness from `${CLAUDE_PLUGIN_ROOT}`
  (`GUIDELINE.md`, `templates/`, `evaluation/`). A standalone copy of just this
  skill directory will not find them.
