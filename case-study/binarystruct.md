# Case Study: binarystruct

How the seven pillars looked in practice when converting
[`github.com/mixcode/binarystruct`](https://github.com/mixcode/binarystruct) — a
Go struct-tag-driven binary marshaller — into an agent-friendly repo, and the
clean-agent evaluation that validated it.

## Starting point

A capable, well-tested library with conventional Go docs (README + `doc.go`). A
first clean-agent build (before the conversion) succeeded but hit friction: the
agent had to infer patterns, guessed wrong on an argument-order inconsistency,
and had to hand-maintain length fields — the most error-prone part of the
target format (ZIP).

## Pillar by pillar

**1. Machine-readable entry point.** Added `llms-full.txt` (cheat sheet, rules,
traps, recipes, debugging, error diagnosis) and `llms.txt` as its index. Made it
discoverable: a `> [!NOTE] AI Agents: read llms-full.txt` callout at the top of
the README, and a one-line pointer at the top of `doc.go` to the raw URL. Because
Go vendors the file into the module cache, the consuming agent found it with one
`ls`.

**2. One source of truth.** `SPECIFICATION.md` was designated ground truth. The
conversion caught a real contradiction — the spec described a codegen fallback
for one case that the generator actually rejected — and fixed the spec to match
the code. Kept the Japanese docs (`*_ja.md`) at parity.

**3. Named the traps.**
- *Argument order*: `Marshal(v, order)` puts the value first while
  `Unmarshal/Write/Read` put the stream first. We **documented** it (with a
  unifying rule) rather than making a breaking reorder. The eval agent
  specifically credited this with stopping a wrong call.
- *Concurrency*: documented which entry points are safe concurrently and that a
  single `Marshaller` must not be shared across goroutines.
- *Emit-only semantics*: the new `valueof`/`const` tags write computed values but
  don't mutate the struct — documented as a deliberate, permanent decision with
  the reasoning, so an agent won't "fix" it.

**4. Recipes.** Added the variable-length-record recipe (a length field via
`valueof=bytelen(F)` paired with a `[len]` size expression) to the README,
`llms-full.txt`, and the tag reference — the single most common real layout.

**5. Designed the API to shrink the error surface.** This was the deepest work:
- `valueof=bytelen(F)` / `count(F)` — auto-computes length/count fields at encode
  time, so the agent never hand-maintains `len(x)`-must-equal-a-field. The most
  error-prone part of writing a ZIP simply vanished.
- `const=0x...` — pins magic numbers/signatures: emitted on encode, validated on
  decode with a precise error (`decode error at offset 0 (field Signature):
  const mismatch: got 0x50000304, want 0x504b0304`). Failure hands you the
  answer.
- Both were implemented identically across all three execution paths (reflection,
  unsafe, codegen) so there are no per-mode surprises.

**6. `AGENTS.txt` for contributors.** Captured the critical invariant: a feature
must be implemented identically across the three code paths, with an extension
checklist and a "test in all modes" rule.

**7. Clean-agent evaluation.** A fresh agent built a ZIP compressor/decompressor
against the *published* `v0.2.4`, in an isolated directory, verifying with the
system `unzip`. Result: **5/5** for agent-friendliness, full interop, and — the
valuable part — it surfaced a concrete gap ("no first-class magic-number tag"),
which directly motivated the `const` feature in the next release. The friction
log became the backlog.

## Outcome

- Friction that the *first* eval hit (argument order, manual length fields) was
  eliminated or pre-empted by docs.
- The second eval scored 5/5 and produced the next feature idea.
- The same `const=` work was then dogfooded back into the project's own ZIP
  example, replacing hand-written signature set/check code with a single
  declarative tag — and giving better error messages for free.

The loop — convert, evaluate with a clean agent, fix the friction, re-evaluate —
is the part worth copying.
