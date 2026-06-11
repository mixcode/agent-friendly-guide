# Making a Codebase Agent-Friendly

A practical method for turning a repository into one that LLM coding agents can
use, extend, and not break — derived from real conversions of production
libraries and CLI tools, and the friction fresh agents hit while building
non-trivial things against them. A companion **`agent-ready` skill** automates the
audit (§5) and scaffolds the artifacts; references to it throughout are optional tooling,
not prerequisites.

> The premise: a growing share of your docs are now read by machines, not just
> people. An agent-friendly repo is not "docs with more keywords" — it is a repo
> whose **entry points, invariants, traps, and APIs** are arranged so an agent
> reaches a correct result on the first try, with no human in the loop.

> **Validity — read this before trusting the numbers.** The empirical findings below
> (cited `(E1)…(E6)`, detailed in Appendix A) were measured in **2026-06** against
> then-current frontier coding agents (Claude Opus-class, medium reasoning effort). Agent
> behavior shifts as models evolve, so a finding true today can change. Treat the dated
> measurements as a snapshot, and **re-run the clean-agent evaluation (§3f) to confirm
> they still hold for your agent** before relying on them.

---

## 1. The one law

Everything below follows from a single result:

> **A manual's value = what the task needs − what's already legible to the agent.**

> **Key terms** (defined here, used throughout):
> - **Legible** — what the agent infers for *free* from three inputs: the **code** it
>   reads (signatures, control flow, in-language docs), the **API names**
>   (`arrayRepeat`, `spineBlank`, `timeout_ms` advertise their own meaning), and its
>   **training prior** (general knowledge it already has — manga reads right-to-left;
>   HTTP 404 = not found).
> - **Residue** — what the task needs that those three *don't* supply. The manual's only
>   irreducible content.
> - **The manual** — the `llms.txt` index + `llms-full.txt` pair: a machine-facing
>   reference for the *human-prompted* "read the manual" case (not the thing a
>   self-directed agent opens on its own — see §3a).
> - **(E1)…(E6)** — citations to the measured evidence in **Appendix A**; the body
>   states the finding, the appendix has the experiment.

So the residue is **project-specific, non-obvious knowledge that lives in none of those
three inputs** — house conventions, units/encodings, business rules,
locale/binding/format quirks, the *why* behind a deliberate wart. **It is the only thing
a manual supplies that nothing else can** (E1). Every decision in this guide is about
spending effort where the residue is, and not spending it where the agent already sees
the contract.

You are writing for two readers:

- **Consumer agents** — writing code that *uses* your library/service. They need how to
  call it, the common patterns, and the specific places they'll otherwise guess wrong.
- **Contributor agents** — *modifying* your codebase. They need the invariants to
  preserve, where parallel implementations live, and a checklist for "I changed this
  correctly."

---

## 2. Diagnose before you invest

Two quick reads of your repo decide everything downstream. Do them first.

### 2a. Is the contract legible? — *this routes where your effort goes*

Ask one question: **is the contract the agent needs legible from what it will actually
read at the point of use?** That — not the repo's type label — determines where
agent-readiness pays off.

- **Contract legible → invest in the source and kill drift; *don't* write a manual.**
  For an ordinary source-shipped **library**, the agent reads the signatures and
  godoc/docstrings and reconstructs the contract cheaply. A separate manual is a
  near-empty delta — neutral when ignored, a tax when forced (E2). The leverage is
  making the read surface *true and complete*: accurate godoc/docstrings, zero drift
  (§3a), a clear selection surface, inlined traps where the API has a sharp edge.
- **Contract *not* legible → invest in a manual / cheat-sheet aimed at the residue.**
  When the consumed artifact hides the contract, a dedicated manual is the surface that
  helps. This covers: a **compiled binary / CLI** (the agent invokes it and reads
  `--help`, not `main.go`; operational semantics — exit codes, side effects, unattended
  flags — live in no signature); a **service / HTTP API** or **MCP server** (no source
  reaches the consumer); and **source-shipped but indirection-heavy** code where
  signatures don't tell the story (facades, `__call`/reflection, code generation, a
  tag/DSL grammar).

This held across repos and languages in testing: agent-readiness artifacts cut token
cost substantially for CLIs/tools (contract not legible) but were
neutral-to-counterproductive for well-documented libraries (contract already legible) —
agents are source-first and won't open a manual they don't need (E2).

### 2b. What type is it? — *this shifts what to emphasize*

| Type | How an agent consumes it | Emphasize |
| :--- | :--- | :--- |
| **Library / API** | writes code against it (import, call) | API ergonomics, recipes, traps |
| **CLI tool** | invokes commands (flags, stdin/stdout, exit codes) | usage/`--help`, machine-readable output, side-effect clarity |
| **Service / HTTP API** | sends requests (endpoints, auth, schemas) | endpoint/auth/schema docs, actionable error bodies |
| **MCP server** | calls tools (tool name, input schema, when-to-use) | per-tool `description`s, schemas, read-vs-mutating |
| **Plugin / skill** | the harness loads it (descriptions, slash commands) | the `description` / when-to-use surface |
| **Docs / methodology** | reads it | structure, a clear entry point, internal consistency |

Libraries are *imported*; tools are *found, chosen, and operated* — a distinct target.
Most agent work today is tool use, so the method covers both.

### 2c. What must the agent do? — *the cross-cutting check*

Whatever the type, the agent must do three things in order — make each easy:

- **Find** — learn the thing exists and is relevant (discoverability).
- **Select** — decide to use *this* over the alternatives, and know its scope and limits.
- **Use** — invoke it correctly the first time (traps, recipes, legible errors).

The work in §3 covers all three. Keep the trio in mind as a coverage check, not as the
structure of the work.

---

## 3. The work — highest leverage first

Two levels of effort with different payoff curves:

- **Doc-level** (§3a, §3c–f): cheap, applies to any repo, large immediate return. Start
  here.
- **Design-level** (§3b): changing the API/code itself so it's hard to misuse. Higher
  effort, highest ceiling — where "agent-friendly" stops being about documentation.

The ordering below is by **leverage** — what most moved an agent's success in
evaluations. Keeping the read surface *true* and the API *unmissable* comes first;
shipping a separate manual comes last, because agents read the code's own surfaces and
can't be reliably steered to a bolt-on doc.

### 3a. Keep the read surface true — and on the path the agent actually reads

This is the deepest doc-level lever, and it has two inseparable halves: the surface must
be **true**, *and* it must be the surface the agent **actually reads**. A wrong surface
fails silently; a true surface the agent never opens is wasted.

**Half 1 — one source of truth, zero drift.** Designate one **ground truth** for
behavior and make every other doc agree with it. The asymmetry is the whole point:

- A **missing** fact costs the agent *tokens* — it reads the source and recovers.
- A **wrong** fact costs *correctness* — the agent trusts it and fails, often silently,
  never re-checking the code.

So drift is worse than absence, and for a source-shipped library **eliminating drift
between code, comments, and docs is the highest-value work — above writing any manual.**

But not all drift bites equally; the danger scales with how **unverifiable** the wrong
fact is:

- **Verifiable drift mostly self-corrects.** When the wrong doc contradicts something
  the agent can re-derive from adjacent code — a signature, a macro/`#ifdef`, a flag, a
  compiler error, a checkable output — the agent trusts the code and routes around the
  doc (E3).
- **Unverifiable-convention drift is catastrophic.** When the wrong fact is a
  *convention* the agent **can't** check against adjacent code, with no output it can
  independently confirm (e.g. "amount is 0–100 percent" when the code means a 0–1
  fraction), the agent has nothing to catch it on and fails completely (E4).

**So when you diff docs against code, prioritize the unverifiable**: conventions, units,
defaults, ordering, semantics — the things not re-derivable from a nearby signature or a
checkable result. This is the same residue a manual exists to carry (§3d); wherever it
lives, it must be true.

Ground truth is usually **the code and its in-language docs (godoc/docstrings)** — for a
simple library that *is* the authority. Reach for a dedicated spec (`SPECIFICATION.md`)
only when behavior isn't self-evident from the code — e.g. a custom tag/DSL grammar with
parallel execution paths that must stay in lockstep. Don't add a spec to a simple
project; it's just one more thing to drift. When in doubt, name the authority in the
manual ("if docs and code disagree, the code wins") rather than creating a document.

Keep it true over time:
- When you change behavior, update ground truth in the *same* change.
- Periodically diff docs against code (catches the spec claim that contradicts the code
  — exactly what an extending agent trusts and gets wrong).
- Keep translated docs at parity or mark them stale; a half-updated translation is drift.

**Half 2 — put it on the *verified* read path.** A true surface only helps if it's the
one the agent lands on, and the intuitive answer is usually wrong:

- **Agents find docs by `ls` + *filename* (task-relevance), not by grepping content —
  and you cannot bait them to a separate doc by placement or naming.** In a small repo
  they read essentially everything; in a doc-rich repo they `ls`, open the files whose
  *names* look task-relevant (the examples, a task-named reference doc, the source), get
  enough, and **stop**. A bolt-on manual — `llms-full.txt`, `AGENTS.md`, even a file
  named `read_first_for_coding_agent.md` — is **skipped** once the task-relevant reads
  satisfy the agent (E5). No filename, convention, or imperative draws it in.
- **The read surface is repo-specific — don't assume `doc.go`/godoc/README.** In testing,
  a library's agent went straight to the tag-reference doc and the `Example` tests and
  **never opened `doc.go`**, so a pointer placed there was never seen (E6).

So: **identify, empirically, the surfaces a clean agent reads for representative tasks**
(watch what it opens in the eval, §3f), and inline the decisive traps + canonical recipe
*there*, true and drift-free:

- **Library:** commonly **runnable `Example` tests + a task-named reference doc + the
  source** — verify which, don't assume.
- **CLI / service / MCP:** the README / `--help` / root docs / tool `description`.

A README callout and an in-doc pointer to a fuller manual are cheap and worth adding —
but treat them as a **fallback**, not the mechanism. Agents often don't follow them.

### 3b. Shrink the error surface by design — the deep lever

Docs reduce *mistakes about* the API; design reduces *the opportunity for* mistakes.
The biggest wins live here, and it's the part teams skip. Three moves:

- **Eliminate manual bookkeeping.** When two things must always agree — a length field
  and the data it sizes, a count and its list, a checksum and its payload — derive one
  from the other automatically. The most error-prone step (a hand-maintained value that
  must match another field) simply disappears.
- **Fail loud, with precise errors.** Make a failure say *where* and *what* — the
  location/field and expected-vs-actual — not a generic "invalid input." A precise error
  hands the agent the answer; a vague one sends it guessing.
- **Make misuse hard or impossible.** Validate constraints early (at construction/parse
  time) with a clear message; and if an operation has more than one mode or
  implementation, keep them behaviorally identical so there are no per-mode surprises.

Heuristic: every place your docs say "remember to…" is a candidate for a design change
that removes the need to remember.

### 3c. Name the traps; recipe the common path

Put both on the surface §3a identified as the read path (inline for a legible library;
in the manual for a not-legible CLI/service).

**Name the traps; don't hide them.** Every API has sharp edges. Document the edge with
the *why* and a rule — don't pretend it isn't there, and don't always "fix" it (a
breaking change can be worse than a documented wart). Examples:

- **Inconsistent argument order** across functions — prefer a unifying rule over a
  breaking reorder (e.g. "stream first, then order, then value; the value-first variant
  has no stream"). One clear paragraph stops the wrong call.
- **Concurrency** — which entry points are safe to call concurrently, and the one object
  you must not share across goroutines.
- **Surprising-but-deliberate semantics** (e.g. an encode that intentionally does not
  write back) — state it *and* the reasoning, so an agent doesn't "fix" it.

Rule of thumb: if a competent engineer could plausibly write the wrong call from the
signature alone, that's a trap — document it. Some traps are **ecosystem-idiomatic** and
worth checking for by language — single-header `#define IMPLEMENTATION`-in-one-TU rules
and inverted return codes (C), APIs that vanish under `--no-default-features` (Rust),
static `from()` factories instead of constructors (JVM), results on stdout with status
via exit codes (Shell), a README install command that doesn't match the real
filename/manifest (any). See `languages/<lang>.md`.

**Recipe the common path.** For the one or two patterns that cover most real usage — the
typical request, the common record shape, the standard call sequence — give a
known-correct, **copy-pasteable** snippet. A recipe converts "the agent assembles the
pattern from primitives and maybe gets it wrong" into "the agent copies a known-correct
shape."

### 3d. Write a manual *only when the contract isn't legible* — and aim it at the residue

If §2a put you in the not-legible bucket (CLI, service, MCP, indirection-heavy code),
ship a dedicated reference (`llms.txt` + `llms-full.txt`) inside the repo. Three rules:

- **Aim every word at the residue.** Don't restate docs that are already good; a manual
  duplicating an excellent godoc is dead weight (E2). The reference's irreducible content
  is the residue from §1 — the project-specific domain/cultural knowledge in neither the
  code, the API names, nor the agent's prior. Lead with that and the decisive
  traps/recipe; link out for everything the agent can already read or already knows.
- **Write it as a lean *delta*, not a restatement.** Front-load the traps and recipe;
  link out for the rest. (When such a convention was unguessable and counter-prior,
  agents failed it 0/4 without the manual and 4/4 with it — E1.)
- **Invite, never force.** Make the reference findable and let the agent pull. Do **not**
  wire a rule/hook that commands "read `llms-full.txt` first" — at best neutral, and
  counterproductive for well-documented repos (E2).

Whether the manual actually *reaches* the consumer depends on the ecosystem's
distribution model — see §4.

### 3e. Always: a selection surface and a contributor guide

These pay off *regardless* of legibility — reading the API tells an agent *how* to call
the thing, not *whether* it should, nor how to safely *modify* it.

**Selection surface — so the agent picks you for the right job.** Before *use* comes
*select*. Where the agent looks first, state:

- **what it's for in one line, and what it's *not* for** — scope and limits;
- **trigger phrases / when-to-use**, matching how an agent would describe the task.

For a skill or MCP tool this is the `description`; for a CLI, the purpose line in
`--help`; for a library, the README's opening sentence. A precise "use when / not for"
prevents both non-use (never picked) and misuse (picked for the wrong job) — and it
matters most for **tools**, where the agent chooses among many at invocation time.
(Worked example: the `agent-ready` skill's `description` — verb-first, with the trigger
phrases an agent would think in.)

**Contributor guide (`AGENTS.txt`) — so agents extend the code correctly.** Write down
the invariants they must uphold. The critical one: when the same behavior lives in more
than one place (a reference implementation plus an optimized or generated one), a change
must land in *all* of them identically — pair this with a step-by-step extension
checklist and a "test every path/mode" rule. Without it, an agent confidently updates
one place and silently breaks parity.

> On the name: `AGENTS.md` is the recognized convention (harnesses auto-read it), but
> it's often a file the *maintainer* writes and owns. To avoid clobbering it, the
> scaffolded guide is `AGENTS.txt` — a plain-text sibling with a self-describing header.
> If the repo already has `AGENTS.md` (or `CLAUDE.md`), **align with / point to it
> rather than duplicating.**

### 3f. Prove it with a clean-agent evaluation

You cannot judge agent-friendliness from the inside; you know too much. Run the loop:

1. Spin up a **fresh agent** with no prior context.
2. Have it build something **real and non-trivial** using only your *published*
   artifacts (the released version, not your working tree).
3. Require a **candid friction report**: where it got stuck, guessed wrong, had to read
   source, or hit a confusing error.
4. Fix the friction. Re-run.

The eval is an **instrument, not just a grade**: watch *which files the agent opens* —
that's how you discover its read path (§3a) — and note any claim it trusted that the code
contradicts (drift). Treat the friction log as your backlog.

**Put a number on it (when a change is contested or two designs compete).** Run a
controlled A/B instead of asking an agent how it felt. Two axes, in priority order:

1. **Correctness — the primary signal.** Define a task with a *checkable* answer; run a
   fresh agent N times per condition (N≈4 sees a regime; more for a close call). Compare
   **pass-rate**, not vibes.
2. **Cost — the secondary signal.** Count the **novel tokens** the reasoning model
   processes — *intake* (input + cache-creation) **plus** output — and **exclude** cache
   *reads* (re-reading the same context is nearly free) and any smaller background/helper
   model (harnesses often run a cheaper model for side tasks like summarization; count
   only the model doing the reasoning). Read it from the run's structured usage (e.g. a
   JSON run-summary with a per-model token block), not a guess. Compare medians; with
   small N a rank test (Mann–Whitney U) tells you whether a gap is real or noise.

Conditions worth separating: **A** = artifacts stripped; **B1** = artifacts present *and*
the agent told to read the manual first; **B2** = artifacts present, agent self-discovers
(the realistic case — and a direct read-path check: if **B2 ≈ A**, the agent never found
the manual).

**The caveat that makes or breaks this:** on small, well-documented targets the token
axis *saturates* — at **equal success** the delta collapses to ≈neutral, because a
capable agent solves from the source either way and the manual changes only how it
narrates. Read naively that says "artifacts don't matter." It doesn't — it says *this
task couldn't see the difference.* The token matrix only discriminates when cost
co-varies with a real knowledge gap. So the decisive experiment is the **failure
regime**: construct the exact condition your artifact targets — an *unverifiable*
doc/code drift (§3a) or a *missing* piece of residue (§3d) — and measure **pass-rate**.
That's where a true, well-placed artifact moves the number from near-zero to
near-perfect, and where tokens-at-equal-success is blind. Lead with correctness; use
tokens only to break ties between designs that both pass.

---

## 4. Ecosystems, distribution, and monorepos

The principles are language-agnostic, but the **cost** of each varies by ecosystem, and
it pays to know which ones your toolchain hands you for free. Validated across Go,
Python, JS/TS, C, Rust, JVM, Shell, Swift, PHP, and Ruby. Two things vary most: **which
in-language surface the pointer goes in** (the doc surface, named per ecosystem in
`languages/<lang>.md`) and **where the manual must live to reach the consumer** (the
distribution model, below).

For a **library**, the in-language doc surface and **runnable examples** are *among* the
surfaces the agent reads — but which one it opens for a given task is repo-specific and
must be **verified, not assumed** (in testing, agents preferred a task-named reference
doc and example tests and skipped `doc.go` — E6). The per-language files name the
*candidate* surfaces; inline the decisive traps/recipe on the one(s) a clean agent is
**observed** to read.

### 4a. The distribution model — "does the manual travel?"

The single most ecosystem-divergent question. Identify the model, then apply its fix:

- **Model A — central registry, ships *all* files by default.** Go (module cache), Rust
  (crates.io), Swift (SwiftPM full checkout), PHP/Composer (git archive of the tag).
  `llms-full.txt` travels **for free** from the repo root. *Caveats — opt-out filters:*
  Rust silently opts files **out** via `Cargo.toml` `include = [...]`; PHP via
  `.gitattributes` `export-ignore` (an opt-*out* list — the inverse of an allowlist).
  Check for either before assuming the manual ships.
- **Model B — central registry, opt-in allowlist.** Python wheel, npm, JVM JAR. A root
  file is **not** shipped unless you add it: Python `package_data` / `MANIFEST.in`, npm
  `package.json` `files`, JVM `src/main/resources` (and note Maven also publishes a
  separate `-javadoc` jar agents read via javadoc.io).
- **Model C — no registry; consumed by git URL.** SwiftPM, Go modules, Zig, Mint. There
  is no one-word `install <name>` — the consumer references a **git URL + tag**; the
  manual travels with the resolved checkout (document the URL form).
- **Model D — no package; the repo / a built binary / a single file IS the artifact.** C
  (vendored header or compiled binary), unpackaged scripts (Python/Shell), Bun-compiled
  or other prebuilt-binary CLIs, JVM application distributions (launcher + jars;
  deb/rpm/Docker/Homebrew). Registry-bundling is **N/A** — ship the manual **in the repo,
  the in-file doc comment, man pages, and/or release assets**. *Sub-split:* vendoring the
  **whole repo/dir** (git submodule, module cache) carries the manual along; **copying a
  single file** (a single-header C lib, one `.sh`) does **not** — so for those, put the
  pointer and the key traps **inside the file itself**.

**Classifying an unlisted ecosystem.** (1) Is there a package registry the consumer
installs from *by name*? If **no** → **C** (git URL + tag) or **D** (the repo, a binary,
or a single file is the artifact). (2) If **yes**, does publishing ship the whole repo by
default (→ **A**) or only an allowlist you must name files into (→ **B**)? Then apply that
model's fix.

### 4b. Monorepos & sub-packages — the package root is not the repo root

In a monorepo each publishable package lives in its own subdirectory, and *that
subdirectory is the unit* — everything above that says "repo root" means the **package
root**. This changes three things:

1. **Placement** — the manual lives in the package's subdir, where consumers land, not at
   the monorepo root (which is *often* not separately published — **though don't assume
   it**: in some ecosystems, notably **Go**, the root `go.mod` is itself the primary
   published module and needs its own manual).
2. **Distribution** — "does the manual travel?" is decided by the *package's own* publish
   config (its `package.json` `files`, `pyproject.toml`/`MANIFEST.in`, `.gitattributes`
   `export-ignore`), not the root's.
3. **Discoverability** — the pointer goes in the *package's* doc surface and README,
   since a consumer reaches the package's registry page, not the repo's top README.

Detect a workspace via its manifest (`pnpm-workspace.yaml`, npm/yarn `workspaces`, Cargo
`[workspace]`, `go.work`, `nx.json`/`turbo.json`/`lerna.json`, Maven `<modules>`, Gradle
`settings` includes) — **or, with no workspace file at all, by finding multiple package
manifests in the tree** (several `go.mod`/`Cargo.toml`/`package.json`; nested Go modules
need no `go.work`). Read upward for shared context, but scope writes to the package
boundary. Watch the **build-from-root trap**: a member often can't build/test in
isolation (hoisted deps, shared tooling, path deps) — a clean-agent eval must evaluate it
*as a workspace member*, not treat "won't build alone" as a failure. Give the root its
own discoverability surface: a **root `llms.txt` that is a package map** (which packages
exist, where each one's manual lives, how agent-ready each is) so a contributor agent can
navigate to the right package. (The `agent-ready` skill's `--monorepo` mode surveys a
workspace and writes this map; deep per-package work still runs one package at a time.)

### 4c. Cross-cutting ecosystem notes

- **Wire a formatter + linter into CI** (the drift defense from §3a) in any ecosystem
  that doesn't hand you one (`gofmt`/`go vet` are Go-only freebies).
- **The README isn't always at the repo root** (e.g. `.github/README.md`, `docs/`) —
  check before concluding it's missing.
- **If the in-language doc surface doesn't exist yet, *create* it.** No `doc.go`, no
  `package-info.java`, zero rustdoc/DocC comments? Add the doc comment / module docstring
  so the surface exists, then point it at the manual.
- **Per-ecosystem specifics** — signals, doc surface, distribution model, and
  language-specific **traps** — live in `languages/<lang>.md`. Available: `go`, `python`,
  `javascript`, `c`, `rust`, `jvm`, `shell`, `swift`, `php`, `ruby`. If none matches,
  apply the agnostic guidance and pick the closest distribution model; when in doubt, let
  the clean-agent eval surface the quirks — **it is the equalizer**, finding gaps like
  "the manual wasn't in the installed package" or "no formatter, so a generated change
  looked malformed" regardless of language.

---

## 5. The audit checklist

A quick pass over any repo. The `agent-ready` skill scores against this exact checklist
automatically (see the README to install it).

> This checklist is the **single source** the `agent-ready` skill scores against —
> `SKILL.md` references it rather than keeping its own copy. Edit it here only.

**Discoverability**
- [ ] `llms.txt` index present at the **package root** (the repo root, or the member's
      subdir in a monorepo — see §4b).
- [ ] `llms-full.txt` (or equivalent) present and complete.
- [ ] README links the agent manual near the top (check non-root locations too:
      `.github/`, `docs/`).
- [ ] In-language doc comment points to the raw manual URL (use the ecosystem's doc
      surface — see `languages/<lang>.md`).
- [ ] **The decisive content is on the surface a clean agent is *observed* to read for
      representative tasks** (verify — don't assume `doc.go`/godoc). For a **library**
      that is commonly the **example tests + a task-named reference doc + the source**;
      inline the key traps + canonical recipe there, true and drift-free — not only inside
      (or linked from) a separate `llms-full.txt` the agent won't open on its own.
- [ ] **The manual is a lean *delta* over existing docs**, not a restatement of
      already-good docs: it front-loads the traps/recipe and links out for the rest (a
      manual that duplicates an excellent godoc goes unread, and costs tokens for nothing
      if forced).
- [ ] **The manual reaches the consumer per the repo's distribution model** (§4a): for a
      registry **opt-in allowlist** (B) it's added (`package_data`/`MANIFEST.in`, npm
      `files`, JVM resources/`-javadoc` jar); for **ships-all** (A) it's at the package
      root (and not excluded by Rust `include` or `.gitattributes export-ignore`); when
      the **artifact is the repo/binary/single file** (D) it's shipped in-repo + in the
      in-file doc comment + man/release assets, and registry-bundling is marked **N/A** —
      for a single copied file (single-header C, one `.sh`) the pointer and key traps live
      *inside the file*.

**Correctness & trust**
- [ ] The source of truth is clear (a spec doc, or the code/godoc) and the docs match it.
- [ ] README/quick-start **claims and install/source commands match the manifest, code,
      and real filenames** (no advertised-capability or install-path drift).
- [ ] Translated/secondary docs are at parity or marked.
- [ ] Common traps (argument order, concurrency, surprising-but-deliberate behavior,
      modes/flags) are documented with the *why* — prioritizing **unverifiable**
      conventions (units, defaults, ordering, semantics) over facts the agent can check
      against adjacent code.

**Usability**
- [ ] Copy-pasteable recipe for each common real-world pattern.
- [ ] Errors name the location/field and expected-vs-actual.
- [ ] Constraints validated early with clear messages.

**Contributor-readiness**
- [ ] `AGENTS.txt` lists invariants + an extension checklist.
- [ ] Parallel implementations (if any) are called out as "keep in sync."

**Evidence**
- [ ] A clean-agent evaluation has been run on the *published* artifact, and its friction
      log triaged.
- [ ] For a contested change, a controlled A/B was run — **pass-rate** as the primary
      signal (and, where it discriminates, **novel-token** cost from structured run
      usage), with the decisive comparison in the **failure regime** the artifact targets,
      not just an equal-success token count.

### Type overlays

Apply on top of the core checklist for the repo's type; mark inapplicable core items
**N/A** with a reason rather than ❌.

**Library / API** (consumed by *writing code against it*)
- [ ] The decisive traps and canonical recipe are on the surface the agent **actually
      reads** for the task — **verified empirically** (commonly the **example tests + a
      task-named reference doc + source**), not assumed to be `doc.go`/godoc, and reachable
      without opening `llms-full.txt`.
- [ ] `llms-full.txt` is the *exhaustive* reference (for the "read the manual" case) and a
      **lean delta** over the in-language docs — not a duplicate of an already-good godoc.
- [ ] **Select:** the README / doc-surface opening line states what it's for and *not* for,
      so an agent imports it for the right job.

**CLI tool** (consumed by *invoking commands*)
- [ ] `--help` / usage is complete: subcommands, flags, defaults.
- [ ] **Select:** a one-line "use when / not for" — the purpose is clear from `--help` or
      the README's first line.
- [ ] Output is machine-parseable (a `--json`/quiet mode, or stable format) and exit codes
      are meaningful.
- [ ] Side effects are clear: read-only vs mutating commands are distinguishable (so a
      harness can gate them); a dry-run exists for destructive ones.
- [ ] Auth / env / config prerequisites are documented for headless use.

**MCP server** (consumed by *calling tools*)
- [ ] **Select:** each tool's `description` says when to use it — and when not — in task
      terms.
- [ ] Input schemas are complete, with per-field descriptions and at least one example.
- [ ] Tool outputs are structured and self-explanatory; errors say what the agent should
      do next.
- [ ] Read-only vs mutating tools are distinguishable (annotations or naming) for gating.
- [ ] Server setup/auth (env vars, config) is documented for headless/non-interactive runs.

---

## 6. Anti-patterns

- **Keyword-stuffing the README** instead of giving a correct entry point and recipes.
  Agents need structure, not SEO.
- **Docs on a website but not in the repo.** The agent often has the code, not the browser.
- **Hiding warts.** An undocumented sharp edge is a guaranteed wrong call.
- **Fixing a wart with a breaking change** when a documented rule would do.
- **Stopping at docs.** If your docs are full of "remember to…", the API is the problem.
- **Self-assessment only.** Without a clean-agent run you are grading your own homework.
- **A manual that duplicates already-good in-language docs.** Agents read
  godoc/docstrings and examples first and won't open a redundant root manual — inline the
  decisive delta where they land, keep `llms-full.txt` a lean reference.
- **Forcing "read the manual first."** A rule/hook that mandates it on every task taxes
  the agent when it isn't needed (measurably so for well-documented libraries). Make the
  manual discoverable; let the agent pull.
- **Trying to *attract* an agent to a separate manual by placement or naming.** A bolt-on
  doc — any name, even `read_first_for_coding_agent.md` — gets skipped when the
  task-relevant reads already satisfy the agent. Put the content *on* the surface they
  read; don't hope a name pulls them off it.
- **Assuming a fixed read surface (`doc.go`/godoc) without checking.** The right surface
  is repo-specific; content on a surface the agent doesn't open is invisible. Verify with
  the eval which surfaces actually get read.

---

## Appendix A — The evidence

**Measured 2026-06** against frontier coding agents (Claude Opus-class, medium reasoning
effort), N≈4 per condition. These are a dated snapshot — agent behavior drifts as models
evolve, so re-run the relevant eval (§3f) before relying on a figure that's aged.

The claims above are grounded in a multi-study evaluation program (controlled A/B with
fresh agents, pass-rate as primary signal). Consolidated so the method reads as
imperatives; consult here when you want the number behind a claim.

- **E1 — The residue is the manual's irreducible value.** With a counter-prior,
  unguessable convention (the code localizes one way; general knowledge says the
  opposite; nothing in the API names hints at it), fresh agents failed the task **0/4**
  without the manual and **4/4** with it. When the same convention leaked through
  well-named knobs, the manual added little — confirming value = needed − legible.
- **E2 — Manuals help only where the contract isn't legible.** Across repos and
  languages, at *equal success*: a redundant ~8k-token library manual *raised* an agent's
  novel-token cost ~20% when forced and was ignored when not; a focused ~1k-token CLI
  manual *cut* cost ~35%. Self-discover (B2) ≈ no-artifact (A) for well-documented
  libraries — agents are source-first.
- **E3 — Verifiable drift self-corrects.** An injected drift on a *checkable* fact
  (signature / macro / option) left the task passing **4/4** across Go, JS, C, and PHP;
  agents verified against the code and ignored the wrong comment.
- **E4 — Unverifiable-convention drift is catastrophic.** An injected drift on an
  *unverifiable* convention (units, with no checkable output) failed **0/4** across four
  languages; a drift audit that rewrites the convention to match the code restored it to
  **4/4**.
- **E5 — You can't bait the read path by filename.** A bolt-on manual was skipped even
  when named `read_first_for_coding_agent.md`; agents read task-relevant surfaces by
  `ls` + filename and stop once satisfied.
- **E6 — The read surface is repo-specific.** A library's agent went to the tag-reference
  doc and `Example` tests and **never opened `doc.go`** — so decisive content must go on
  the surface the agent is *observed* to read, not the assumed one.

---

*This guideline is the source of truth for the companion `agent-ready` skill, which
executes the audit and scaffolds the artifacts above.*
