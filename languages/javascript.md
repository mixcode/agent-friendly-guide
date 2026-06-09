# JavaScript / TypeScript — agent-readiness notes

Language-specific specifics for the `agent-ready` method; read alongside the
agnostic `GUIDELINE.md`. (Covers Node, Bun, and Deno.)

## Detect
- `package.json`. CLI = a `bin` field (**verify it isn't `null`/empty**);
  library = `exports`/`main`/`module`. A repo can be both.
- The runtime may be **Bun or Deno**, not npm/Node — check the scripts/imports.

## Doc surface (where the manual pointer goes)
- Library: JSDoc/TSDoc on the entry module.
- CLI: `--help`.

## Distribution model — does the manual travel?
- **Model B (opt-in allowlist):** npm ships only what `package.json` `"files"`
  lists (plus a few defaults) — add `llms-full.txt` to `"files"` so it travels.
- **Model D:** if the CLI is compiled to a standalone binary (Bun/Deno `compile`),
  the binary is the artifact — keep the manual in the repo + `--help`.

## Traps & idioms
- **`exports` map / dual ESM-CJS:** the importable surface depends on the
  conditional `exports`; document the real entry points, not just `main`.
- **Advertised flags vs reality:** READMEs claim output modes/flags the CLI
  doesn't actually have — verify each documented flag against the parser.
