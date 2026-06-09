# Swift — agent-readiness notes

Language-specific specifics for the `agent-ready` method; read alongside the
agnostic `GUIDELINE.md`.

## Detect
- `Package.swift`. Library = a `.library` product/target; CLI = an `.executable`
  (SwiftCLI / swift-argument-parser).

## Doc surface (where the manual pointer goes)
- Library: DocC `///` comments, rendered on the Swift Package Index. (If the
  package has *no* `///` comments yet, **create** the doc surface — add a
  doc-comment to the entry type — then point it at the manual.)
- CLI: ArgumentParser/SwiftCLI `--help`.

## Distribution model — does the manual travel?

Swift can land in two models; pick by **how this repo is actually consumed**:

- **SPM package (library or source CLI) → treat as C for *install*, A for
  *manual-travel*.** SwiftPM has no central name registry, so the consumer
  references a **git URL + tag** (Model C) — document that as the "install" story
  (there's no one-word name). SwiftPM then resolves the **full git checkout**, so a
  repo-root `llms-full.txt` travels with it for free (Model A behavior); no
  allowlist needed.
- **CLI shipped as a prebuilt binary** (release asset, Homebrew, etc.) **→ Model
  D.** The binary is the artifact — ship the manual in the repo + `--help` + man/
  release assets; don't expect a file to travel with the binary.

**Tiebreaker:** having a `Package.swift` does *not* make it C — key on consumption.
An `.executable` shipped as an installed binary → **D**; only C/A when consumers
actually `import` the package as a dependency.

## Traps & idioms
- **`@dynamicCallable` / dynamic member APIs have no enumerable surface** — the
  callable interface isn't visible as normal methods, and a dynamic call may
  return a wrapper type (e.g. `FancyResult`) rather than the obvious `String`.
  Document the real call shape and return type explicitly.
