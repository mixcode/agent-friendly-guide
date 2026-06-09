# Swift — agent-readiness notes

Language-specific specifics for the `agent-ready` method; read alongside the
agnostic `GUIDELINE.md`.

## Detect
- `Package.swift`. Library = a `.library` product/target; CLI = an `.executable`
  (SwiftCLI / swift-argument-parser).

## Doc surface (where the manual pointer goes)
- Library: DocC `///` comments, rendered on the Swift Package Index.
- CLI: ArgumentParser/SwiftCLI `--help`.

## Distribution model — does the manual travel?
- **Model A / C:** SwiftPM has no central package registry by default — a consumer
  references a **git URL + tag** (Model C), and SwiftPM resolves the **full git
  checkout**, so a repo-root `llms-full.txt` travels with it (Model A behavior).
  Document the git-URL + tag form as the "install" story (there's no one-word name).

## Traps & idioms
- **`@dynamicCallable` / dynamic member APIs have no enumerable surface** — the
  callable interface isn't visible as normal methods, and a dynamic call may
  return a wrapper type (e.g. `FancyResult`) rather than the obvious `String`.
  Document the real call shape and return type explicitly.
