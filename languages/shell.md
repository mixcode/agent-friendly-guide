# Shell / Bash — agent-readiness notes

Language-specific specifics for the `agent-ready` method; read alongside the
agnostic `GUIDELINE.md`. Shell has **no manifest**, so detection is shape-based.

## Detect
- No package file — use `.sh`/`.bash` extensions, shebangs (`#!/usr/bin/env bash`),
  a `bin/` dir of scripts, and the **sourced-functions-vs-`main`** shape: a library
  is meant to be `source`d for its functions; a CLI has a `main`/dispatch.

## Doc surface (where the manual pointer goes)
- Library: SHDOC `# @description` comments (often generate the README).
- CLI: `--help` + man pages.

## Distribution model — does the manual travel?
- **Model D.** A **vendored repo** (git submodule/clone) carries `llms-full.txt`
  along; a **copied single `.sh`** does **not** — for single-file helpers put the
  pointer and key traps **inside the script**. Document results-on-stdout +
  status-via-exit-codes (SHDOC `@exitcode`).

## Traps & idioms
- **Filename / source-path drift:** the README's `source bash-utility.sh` may not
  match the real `bash_utility.sh`, and relative `source` paths break when the lib
  is vendored elsewhere — verify the documented source command against the files.
- **Generated README:** if the README is generated (e.g. by `generate_readme.sh`
  from SHDOC), do **not** insert a callout into the generated region — edit the
  generator's source or a hand-maintained section.
- **API shape — results on stdout, status via exit codes.** A sourced-function lib
  doesn't "return" values or raise typed errors; read the checklist's "errors name
  the location" item as **meaningful exit codes + stderr messages** (document the
  SHDOC `@exitcode` for each function), not return-value/field naming.
