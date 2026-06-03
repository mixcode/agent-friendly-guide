# Making a Codebase Agent-Friendly

A practical method for turning a repository into one that LLM coding agents can
use, extend, and not break — derived from a real conversion of a Go library
(`github.com/mixcode/binarystruct`) and the friction a fresh agent hit while
building a non-trivial tool against it.

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

- **Doc-level** (pillars 1–4, 6): cheap, applies to any repo, large immediate
  return. Start here.
- **Design-level** (pillar 5): changing the API/code itself so it is hard to
  misuse. Higher effort, highest ceiling. This is where "agent-friendly" stops
  being about documentation.

The feedback loop (pillar 7) is what tells you when you're done.

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

The seven pillars serve all three; **find** and **select** both live in pillar 1.

---

## 2. The method — seven pillars

### Pillar 1 — Find & select: an entry point an agent can reach and choose

This pillar covers the first two of find/select/use. **Find** is discoverability —
the machine-readable entry point below. **Select** is the *selection surface* — how
the agent decides this is the right thing for the job.

Ship a dedicated agent manual **inside the repo**, not only on a docs site.

- `llms.txt` — a short index page (the emerging convention; see https://llms.txt.org)
  pointing at the deeper docs.
- `llms-full.txt` — the full manual: a syntax/API cheat sheet, the crucial rules
  and constraints, the documented traps (pillar 3), copy-pasteable recipes
  (pillar 4), debugging guidance, and error diagnosis.

Why *in the repo*: when your package is fetched (vendored into the module cache,
cloned, installed), the agent finds the manual with a single `ls` — no network
round-trip to a docs site, no guessing. In our case study the evaluating agent
called this out explicitly as the single biggest reason it never had to read
library source to make things work.

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

### Pillar 2 — One source of truth, zero drift

Designate one **ground truth** for behavior and make every other doc agree with
it. Drift is worse than missing docs: an agent that trusts a confident-but-wrong
statement fails in ways that are hard to debug.

Ground truth is usually **the code and its in-language docs (godoc/docstrings)** —
for a simple library that *is* the authority, and the agent manual derives from
it. Reach for a dedicated spec document (we used `SPECIFICATION.md`) only when
behavior is complex enough that it isn't self-evident from the code (binarystruct
earned one: a tag grammar plus three parallel execution paths). Don't add a
parallel spec to a simple project — it just becomes one more thing to drift. When
in doubt, name the authority in the manual ("if the docs and code disagree, the
code wins") rather than creating a new document.

- When you change behavior, update ground truth in the same change.
- Periodically diff docs against code. (In our conversion this caught a
  `SPECIFICATION.md` claim that contradicted what the code generator actually
  did — exactly the kind of thing an extending agent would have trusted.)
- If you maintain translated docs, keep them at parity or mark them as possibly
  stale; a half-updated translation is a drift source.

### Pillar 3 — Name the traps; don't hide them

Every API has sharp edges. The agent-friendly move is to **document the sharp
edge with the *why* and a rule**, not to pretend it isn't there (and not always
to "fix" it — a breaking change can be worse than a documented wart).

Examples from the case study:
- **Inconsistent argument order** across functions — we did *not* reorder
  (breaking change); we documented the trap with a unifying rule ("stream first,
  then order, then value; the value-first variant has no stream"). The eval
  agent confirmed this single paragraph stopped it from writing the wrong call.
- **Concurrency**: which entry points are safe to call concurrently, and the one
  object you must not share across goroutines.
- **Surprising-but-deliberate semantics** (e.g. an encode that intentionally
  does not write back): state it *and* the reasoning, so an agent doesn't "fix"
  it or build a wrong mental model.

Rule of thumb: if a competent engineer could plausibly write the wrong call from
the signature alone, that is a trap — document it.

### Pillar 4 — Recipes for the common path

For the few patterns that cover most real usage, give a **copy-pasteable** recipe
and put it where the agent looks (README + the agent manual + the tag/API
reference). Our highest-value recipe was the single most common real layout in
the domain (a variable-length record). A recipe converts "the agent assembles
the pattern from primitives and maybe gets it wrong" into "the agent copies a
known-correct shape."

### Pillar 5 — Design the API to shrink the error surface (the deep lever)

Docs reduce *mistakes about* the API; design reduces *the opportunity for*
mistakes. This is where the biggest wins live, and it's the part teams skip.

Three concrete moves, all from the case study:
- **Eliminate manual bookkeeping.** We added a tag that auto-computes
  length/count fields at encode time, so the agent never hand-maintains a
  `len(x)` that must match a field — the single most error-prone part of the
  format simply disappeared.
- **Fail loud, with precise errors.** A mismatch now returns an error naming the
  byte offset, the field, and got-vs-want. An agent (or human) debugging a bad
  input is handed the answer instead of a generic failure.
- **Make misuse hard or impossible.** Validate constraints at construction/parse
  time with a clear message; keep behavior identical across all execution modes
  so there are no per-mode surprises to memorize.

Heuristic: every place your docs say "remember to…" is a candidate for a design
change that removes the need to remember.

### Pillar 6 — A contributor guide for agents (`AGENTS.txt`)

If you want agents to *extend* the code correctly, write down the invariants they
must uphold. In our case the critical one was: a feature must be implemented
identically across three parallel code paths (reflection, unsafe, codegen), with
a step-by-step extension checklist and a "write tests in all modes" rule.
Without this, an agent confidently implements one path and silently breaks
parity. (Note: `AGENTS.md`/`AGENTS.txt` is increasingly a recognized convention —
align with whatever your tooling reads.)

### Pillar 7 — Measure it with a clean-agent evaluation

You cannot judge agent-friendliness from the inside; you know too much. Run the
loop:

1. Spin up a **fresh agent** with no prior context.
2. Have it build something **real and non-trivial** using only your *published*
   artifacts (the released version, not your working tree).
3. Require a **candid friction report**: where it got stuck, guessed wrong, had
   to read source, or hit a confusing error.
4. Fix the friction. Re-run.

This is what closed the loop for us: the clean-agent build scored highly *and*
surfaced a concrete gap (a missing feature for fixed/magic values) that became
the next improvement. Treat the friction log as your backlog.

### A note on languages and ecosystems

The pillars are language-agnostic, but the **cost** of each varies by ecosystem,
and it pays to know which ones your toolchain hands you for free.

- **Go is unusually well-suited** to this work: `gofmt` gives consistent style,
  `go vet` catches a class of misuse, `doc.go` / `go doc` provide an in-language
  manual surface, module vendoring means `llms-full.txt` ships *with* the package
  into the consumer's module cache, and pkg.go.dev renders docs automatically.
  Several pillars are nearly free.
- **Other ecosystems need explicit equivalents.** Wire a formatter + linter into
  CI (pillar 2's drift defense); pick the in-language doc surface for the pointer
  to `llms-full.txt` (a module/package docstring, or a packaged README); and —
  easy to miss — make sure the manual is **included in the published artifact**
  so it travels with installs (npm `files`, Python package data / `MANIFEST.in`,
  a crate's `include`, etc.), not just present in the git repo.
- **The clean-agent evaluation (pillar 7) is the equalizer.** It surfaces
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
- [ ] `llms.txt` index present at repo root.
- [ ] `llms-full.txt` (or equivalent) present and complete.
- [ ] README links the agent manual near the top.
- [ ] In-language doc comment points to the raw manual URL.
- [ ] The manual is included in the *published* artifact, not just the repo.

**Correctness & trust**
- [ ] The source of truth is clear (a spec doc, or the code/godoc) and the docs match it.
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

## 5. Case study: binarystruct (worked example)

A condensed before/after lives in [`case-study/binarystruct.md`](case-study/binarystruct.md):
what each pillar looked like in practice, the clean-agent evaluation that scored
5/5, and the gap it surfaced that drove the next feature.

---

*This guideline is the source of truth for the companion `agent-ready` skill,
which executes the audit and scaffolds the artifacts above.*
