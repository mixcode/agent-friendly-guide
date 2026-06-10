# Making a Codebase Agent-Friendly

A practical method for turning a repository into one that LLM coding agents can
use, extend, and not break — derived from real conversions of production
libraries and CLI tools, and the friction fresh agents hit while building
non-trivial things against them.

> The premise: a growing share of your docs are now read by machines, not just
> people. An agent-friendly repo is not "docs with more keywords" — it is a repo
> whose **entry points, invariants, traps, and APIs** are arranged so an agent
> reaches a correct result on the first try, with no human in the loop.

---

## 1. Who and what this is for

Two audiences you are writing for, and they need different things:

- **Consumer agents** — an agent writing code that *uses* your library/service.
  They need: how to call it, the common patterns, and the specific places they
  will otherwise guess wrong.
- **Contributor agents** — an agent *modifying* your codebase. They need: the
  invariants they must preserve, where the parallel implementations live, and
  the checklist for "I added a feature correctly."

And two levels of effort, with very different payoff curves:

- **Doc-level** (principles 1–4, 6): cheap, applies to any repo, large immediate
  return. Start here.
- **Design-level** (principle 5): changing the API/code itself so it is hard to
  misuse. Higher effort, highest ceiling. This is where "agent-friendly" stops
  being about documentation.

The feedback loop (principle 7) is what tells you when you're done.

### Repo types — what "consume" means

How an agent consumes the repo differs by type, and that shifts what matters:

| Type | How an agent consumes it | Emphasize |
| :--- | :--- | :--- |
| **Library / API** | writes code against it (import, call) | API ergonomics, recipes, traps |
| **CLI tool** | invokes commands (flags, stdin/stdout, exit codes) | usage/`--help`, machine-readable output, side-effect clarity |
| **Service / HTTP API** | sends requests (endpoints, auth, schemas) | endpoint/auth/schema docs, actionable error bodies |
| **MCP server** | calls tools (tool name, input schema, when-to-use) | per-tool `description`s, schemas, read-vs-mutating |
| **Plugin / skill** | the harness loads it (descriptions, slash commands) | the `description` / when-to-use surface |
| **Docs / methodology** | reads it | structure, a clear entry point, internal consistency |

Libraries are *imported*; tools are *found, chosen, and operated* — a distinct
target. Most agent work today is tool use, so the method covers both.

### The find → select → use lens

Cutting across every type, an agent must do three things — make each easy:

- **Find** — learn the thing exists and is relevant (discoverability).
- **Select** — decide to use *this* over alternatives, and know its scope and
  limits. The *selection surface* — a one-line "use when … / not for …", trigger
  phrases, an MCP tool `description`, a CLI's purpose line — is the most
  underweighted dimension, and it is exactly what tool use hinges on.
- **Use** — invoke it correctly the first time (traps, recipes, error legibility).

The seven principles serve all three; **find** and **select** both live in principle 1.

---

## 2. The method — seven principles

### Principle 1 — Find & select: an entry point an agent can reach and choose

This principle covers the first two of find/select/use. **Find** is discoverability —
the machine-readable entry point below. **Select** is the *selection surface* — how
the agent decides this is the right thing for the job.

Ship a dedicated agent manual **inside the repo**, not only on a docs site.

- `llms.txt` — a short index page (the emerging convention; see https://llms.txt.org)
  pointing at the deeper docs.
- `llms-full.txt` — the full manual: a syntax/API cheat sheet, the crucial rules
  and constraints, the documented traps (principle 3), copy-pasteable recipes
  (principle 4), debugging guidance, and error diagnosis.

Why *in the repo*: when your package is fetched (vendored into the module cache,
cloned, installed), the agent finds the manual with a single `ls` — no network
round-trip to a docs site, no guessing. In testing, the evaluating agent
called this out explicitly as the single biggest reason it never had to read
library source to make things work. **Whether the manual actually reaches the
consumer depends on the ecosystem's distribution model** — for some it's free,
for others you must opt it into the package, and for some the artifact *is* the
repo/binary; see the distribution taxonomy in §2's "languages and ecosystems."

Then make it **discoverable** from the places an agent already lands:
- A callout at the top of the README (`> [!NOTE] AI Agents: read llms-full.txt`).
- A one-line pointer at the top of the in-language doc comment (Go `doc.go`,
  module docstring, etc.) to the raw URL of `llms-full.txt`.
- Optionally a badge.

Keep the human-facing docs human-facing: put the dense agent material in
`llms-full.txt` and *link* to it, rather than bloating the README/godoc.

**Selection surface (so the agent picks you for the right job).** Discoverability
gets the agent *to* you; the selection surface gets it to use you *correctly*.
State, where the agent looks first:
- **what it's for in one line, and what it's *not* for** — scope and limits;
- the **trigger phrases / when-to-use** matching how an agent would describe the
  task. For a skill or MCP tool this is the `description`; for a CLI, the purpose
  line in `--help`; for a library, the README's opening sentence.

A precise "use when / not for" prevents both non-use (the agent never picks you)
and misuse (it picks you for the wrong job). This matters most for **tools**, where
the agent chooses among many at invocation time. Worked example: the `agent-ready`
skill's `description` — verb-first, with the trigger phrases an agent would think
in ("make a repo agent-friendly", "add llms.txt").

### Principle 2 — One source of truth, zero drift

Designate one **ground truth** for behavior and make every other doc agree with
it. Drift is worse than missing docs: an agent that trusts a confident-but-wrong
statement fails in ways that are hard to debug.

Ground truth is usually **the code and its in-language docs (godoc/docstrings)** —
for a simple library that *is* the authority, and the agent manual derives from
it. Reach for a dedicated spec document (e.g. a `SPECIFICATION.md`) only when
behavior is complex enough that it isn't self-evident from the code — for
example, a library with a custom tag/DSL grammar plus several parallel execution
paths that must stay in lockstep earns one. Don't add a parallel spec to a simple
project — it just becomes one more thing to drift. When in doubt, name the
authority in the manual ("if the docs and code disagree, the code wins") rather
than creating a new document.

- When you change behavior, update ground truth in the same change.
- Periodically diff docs against code. (In practice this catches a spec claim
  that contradicts what the code actually does — exactly the kind of thing an
  extending agent would trust and get wrong.)
- If you maintain translated docs, keep them at parity or mark them as possibly
  stale; a half-updated translation is a drift source.

### Principle 3 — Name the traps; don't hide them

Every API has sharp edges. The agent-friendly move is to **document the sharp
edge with the *why* and a rule**, not to pretend it isn't there (and not always
to "fix" it — a breaking change can be worse than a documented wart).

Examples:
- **Inconsistent argument order** across functions — prefer documenting the trap
  with a unifying rule over a breaking reorder (e.g. "stream first, then order,
  then value; the value-first variant has no stream"). One clear paragraph can
  stop an agent from writing the wrong call.
- **Concurrency**: which entry points are safe to call concurrently, and the one
  object you must not share across goroutines.
- **Surprising-but-deliberate semantics** (e.g. an encode that intentionally
  does not write back): state it *and* the reasoning, so an agent doesn't "fix"
  it or build a wrong mental model.

Rule of thumb: if a competent engineer could plausibly write the wrong call from
the signature alone, that is a trap — document it.

Some traps are **ecosystem-idiomatic** and worth checking for by language — e.g.
single-header `#define IMPLEMENTATION`-in-one-TU rules and inverted return codes
(C), feature-gated APIs that vanish under `--no-default-features` (Rust), static
`from()` factories instead of constructors (JVM), results returned on stdout with
status via exit codes (Shell), an install command in the README that doesn't
match the real filename/manifest (any). See `languages/<lang>.md`.

### Principle 4 — Recipes for the common path

For the few patterns that cover most real usage, give a **copy-pasteable** recipe
and put it where the agent looks (README + the agent manual + the tag/API
reference). Pick the one or two patterns that cover most real usage — the typical
request, the common record shape, the standard call sequence — and show each as a
known-correct, copy-pasteable snippet. A recipe converts "the agent assembles the
pattern from primitives and maybe gets it wrong" into "the agent copies a
known-correct shape."

### Principle 5 — Design the API to shrink the error surface (the deep lever)

Docs reduce *mistakes about* the API; design reduces *the opportunity for*
mistakes. This is where the biggest wins live, and it's the part teams skip.

Three concrete moves:
- **Eliminate manual bookkeeping.** When two things must always agree — a length
  field and the data it sizes, a count and its list, a checksum and its payload —
  derive one from the other automatically instead of making the caller keep them
  in sync. The most error-prone step (a hand-maintained value that has to match
  another field) simply disappears.
- **Fail loud, with precise errors.** Make a failure say *where* and *what* — the
  location/field and expected-vs-actual — not a generic "invalid input." A precise
  error hands the agent (or human) the answer; a vague one sends it guessing.
- **Make misuse hard or impossible.** Validate constraints early (at
  construction/parse time) with a clear message; and if the same operation has
  more than one implementation or mode, keep them behaviorally identical so there
  are no per-mode surprises to memorize.

Heuristic: every place your docs say "remember to…" is a candidate for a design
change that removes the need to remember.

### Principle 6 — A contributor guide for agents (`AGENTS.txt`)

If you want agents to *extend* the code correctly, write down the invariants they
must uphold. A common critical one: when the same behavior lives in more than one
place (e.g. a reference implementation plus an optimized or generated one), a
change must land in all of them identically — pair this with a step-by-step
extension checklist and a "test every path/mode" rule. Without it, an agent
confidently updates one place and silently breaks parity.

On the name: `AGENTS.md` is the recognized convention (agent harnesses auto-read
it), but it's often a file the *maintainer* writes and owns. To avoid clobbering
that, the scaffolded contributor guide is `AGENTS.txt` — a plain-text sibling,
with a self-describing header saying what it is. If the repo already has an
`AGENTS.md` (or `CLAUDE.md`), **align with / point to it rather than duplicating**.

### Principle 7 — Measure it with a clean-agent evaluation

You cannot judge agent-friendliness from the inside; you know too much. Run the
loop:

1. Spin up a **fresh agent** with no prior context.
2. Have it build something **real and non-trivial** using only your *published*
   artifacts (the released version, not your working tree).
3. Require a **candid friction report**: where it got stuck, guessed wrong, had
   to read source, or hit a confusing error.
4. Fix the friction. Re-run.

This is the loop in action: a clean-agent build can score highly *and* surface a
concrete capability or doc gap that becomes the next improvement. Treat the
friction log as your backlog.

### A note on languages and ecosystems

The principles are language-agnostic, but the **cost** of each varies by ecosystem,
and it pays to know which ones your toolchain hands you for free. Two things vary
the most: **where the manual must live so it reaches the consumer** (the
distribution model, below) and **which in-language surface the pointer goes in**
(the doc surface — named per ecosystem in `languages/<lang>.md`). Validated across Go, Python, JS/TS, C,
Rust, JVM, Shell, Swift, PHP, and Ruby.

#### The distribution model decides where the manual lives ("does the manual travel?")

This is the single most ecosystem-divergent question. There are four models —
identify which one the repo is in, then apply its fix:

- **Model A — central registry, ships *all* files by default.** Go (module
  cache), Rust (crates.io), Swift (SwiftPM full checkout), PHP/Composer (the git
  archive of the tag). `llms-full.txt` travels **for free** from the repo root.
  *Caveats — opt-out filters:* Rust silently opts files **out** via `Cargo.toml`
  `include = [...]`; PHP via `.gitattributes` `export-ignore` (an opt-*out* list —
  the inverse of an allowlist). Check for either before assuming the manual ships.
- **Model B — central registry, opt-in allowlist.** Python wheel, npm, JVM JAR.
  A root file is **not** shipped unless you add it: Python `package_data` /
  `MANIFEST.in`, npm `package.json` `files`, JVM `src/main/resources` (and note
  Maven also publishes a separate `-javadoc` jar agents read via javadoc.io).
- **Model C — no registry; consumed by git URL.** SwiftPM, Go modules, Zig, Mint.
  There is no one-word `install <name>` — the consumer references a **git URL +
  tag**; the manual travels with the resolved checkout (document the URL form).
- **Model D — no package; the repo / a built binary / a single file IS the
  artifact.** C (vendored header or a compiled binary), unpackaged scripts
  (Python/Shell), Bun-compiled or other prebuilt-binary CLIs, JVM application
  distributions (launcher + jars; deb/rpm/Docker/Homebrew). Registry-bundling is
  **N/A** — ship the manual **in the repo, the in-file doc comment, man pages,
  and/or release assets**. *Sub-split:* vendoring the **whole repo/dir** (git
  submodule, module cache) carries the manual along; **copying a single file** (a
  single-header C lib, one `.sh`) does **not** — so for those, put the pointer and
  the key traps **inside the file itself**.

**Classifying an unlisted ecosystem (A–D).** (1) Is there a package registry the
consumer installs from *by name*? If **no** → it's **C** (consumed by a git URL +
tag) or **D** (no package — the repo, a built binary, or a single file is the
artifact). (2) If **yes**, does publishing ship the whole repo by default, or only
an allowlist? Check the packaging config: ships-all → **A**; opt-in allowlist (you
must name the files to include) → **B**. Then apply that model's "where the manual
lives" fix.

#### Per-ecosystem specifics

The signals, doc surface, distribution model, and the language-specific **traps**
for each ecosystem live in `languages/<lang>.md` — read the one for the repo's
language alongside this guide. Available: `go`, `python`, `javascript`, `c`,
`rust`, `jvm`, `shell`, `swift`, `php`, `ruby`. If none matches, apply the agnostic guidance
above and pick the closest distribution model (A–D); when in doubt, let the
clean-agent evaluation surface the ecosystem's quirks.

#### Other cross-cutting notes

- **Wire a formatter + linter into CI** (principle 2's drift defense) in any
  ecosystem that doesn't hand you one (`gofmt`/`go vet` are Go-only freebies).
- **The README isn't always at the repo root** (e.g. `.github/README.md`,
  `docs/`) — check there before concluding it's missing.
- **If the in-language doc surface doesn't exist yet, *create* it.** Sometimes the
  pointer has no home — no `doc.go`, no `package-info.java`, zero rustdoc/DocC
  comments. Don't skip the pointer: add the doc comment / module docstring so the
  surface exists, then point it at the manual.
- **Monorepos & sub-packages: the package root is not the repo root.** In a
  monorepo/megarepo each publishable package lives in its own subdirectory, and
  *that subdirectory is the unit* — everything above that says "repo root" means
  the **package root**. This changes three things: (1) **placement** — the manual
  lives in the package's subdir, where its consumers land, not at the monorepo
  root (which is usually **never published**); (2) **distribution** — "does the
  manual travel?" is decided by the *package's own* publish config (its
  `package.json` `files`, its `pyproject.toml`/`MANIFEST.in`, its `.gitattributes`
  `export-ignore`), not the root's; (3) **discoverability** — the pointer goes in
  the *package's* doc surface and README, since a consumer reaches the package's
  registry page, not the repo's top README. Detect a workspace via its manifest
  (`pnpm-workspace.yaml`, npm/yarn `workspaces`, Cargo `[workspace]`, `go.work`,
  `nx.json`/`turbo.json`/`lerna.json`, Maven `<modules>`, Gradle `settings`
  includes). Read upward for shared context, but scope writes to the package
  boundary. Watch the **build-from-root trap**: a member often can't build/test in
  isolation (hoisted deps, shared tooling, path deps) — a clean-agent eval must
  evaluate it *as a workspace member*, not treat "won't build alone" as a failure.
  Give the monorepo root its own discoverability surface: a **root `llms.txt`
  that is a package map** — which packages exist, where each one's manual lives,
  and how agent-ready each is — so a contributor agent landing on the repo can
  navigate to the right package. (The `agent-ready` skill's `--monorepo` mode surveys a
  workspace and writes this map; deep per-package work still runs one package at a
  time.)
- **The clean-agent evaluation (principle 7) is the equalizer.** It surfaces
  ecosystem-specific gaps — "the manual wasn't in the installed package," "there
  was no formatter so a generated change looked malformed" — regardless of
  language. When in doubt about a language's quirks, let the eval find them.

---

## 3. The audit checklist

A quick pass over any repo. The `agent-ready` skill scores against this exact
checklist automatically (see the README to install it).

> This checklist is the **single source** the `agent-ready` skill scores against —
> `SKILL.md` references it rather than keeping its own copy. Edit it here only.

**Discoverability**
- [ ] `llms.txt` index present at the **package root** (the repo root, or the
      member's subdir in a monorepo — see "Monorepos & sub-packages").
- [ ] `llms-full.txt` (or equivalent) present and complete.
- [ ] README links the agent manual near the top (check non-root locations too: `.github/`, `docs/`).
- [ ] In-language doc comment points to the raw manual URL (use the ecosystem's doc surface — see `languages/<lang>.md`).
- [ ] **The manual reaches the consumer per the repo's distribution model** (§2): for a registry **opt-in allowlist** (B) it's added (`package_data`/`MANIFEST.in`, npm `files`, JVM resources/`-javadoc` jar); for **ships-all** (A) it's at the package root (and not excluded by Rust `include` or `.gitattributes export-ignore`); when the **artifact is the repo/binary/single file** (D) it's shipped in-repo + in the in-file doc comment + man/release assets, and registry-bundling is marked **N/A** — for a single copied file (single-header C, one `.sh`) the pointer and key traps live *inside the file*.

**Correctness & trust**
- [ ] The source of truth is clear (a spec doc, or the code/godoc) and the docs match it.
- [ ] README/quick-start **claims and install/source commands match the manifest, code, and real filenames** (no advertised-capability or install-path drift).
- [ ] Translated/secondary docs are at parity or marked.
- [ ] Common traps (argument order, concurrency, surprising-but-deliberate behavior, modes/flags) are documented with the *why*.

**Usability**
- [ ] Copy-pasteable recipe for each common real-world pattern.
- [ ] Errors name the location/field and expected-vs-actual.
- [ ] Constraints validated early with clear messages.

**Contributor-readiness**
- [ ] `AGENTS.txt` lists invariants + an extension checklist.
- [ ] Parallel implementations (if any) are called out as "keep in sync."

**Evidence**
- [ ] A clean-agent evaluation has been run on the *published* artifact, and its
      friction log triaged.

### Type overlays

Apply on top of the core checklist for the repo's type; mark inapplicable core
items **N/A** with a reason rather than ❌.

**CLI tool** (consumed by *invoking commands*)
- [ ] `--help` / usage is complete: subcommands, flags, defaults.
- [ ] **Select:** a one-line "use when / not for" — the purpose is clear from `--help` or the README's first line.
- [ ] Output is machine-parseable (a `--json`/quiet mode, or stable format) and exit codes are meaningful.
- [ ] Side effects are clear: read-only vs mutating commands are distinguishable (so a harness can gate them); a dry-run exists for destructive ones.
- [ ] Auth / env / config prerequisites are documented for headless use.

**MCP server** (consumed by *calling tools*)
- [ ] **Select:** each tool's `description` says when to use it — and when not — in task terms.
- [ ] Input schemas are complete, with per-field descriptions and at least one example.
- [ ] Tool outputs are structured and self-explanatory; errors say what the agent should do next.
- [ ] Read-only vs mutating tools are distinguishable (annotations or naming) for gating.
- [ ] Server setup/auth (env vars, config) is documented for headless/non-interactive runs.

---

## 4. Anti-patterns

- **Keyword-stuffing the README** instead of giving a correct entry point and
  recipes. Agents need structure, not SEO.
- **Docs on a website but not in the repo.** The agent often has the code, not
  the browser.
- **Hiding warts.** An undocumented sharp edge is a guaranteed wrong call.
- **Fixing a wart with a breaking change** when a documented rule would do.
- **Stopping at docs.** If your docs are full of "remember to…", the API is the
  problem.
- **Self-assessment only.** Without a clean-agent run you are grading your own
  homework.

---

*This guideline is the source of truth for the companion `agent-ready` skill,
which executes the audit and scaffolds the artifacts above.*
