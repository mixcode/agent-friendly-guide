# Rust — agent-readiness notes

Language-specific specifics for the `agent-ready` method; read alongside the
agnostic `GUIDELINE.md`.

## Detect
- `Cargo.toml`. CLI = `[[bin]]` / `src/main.rs` (usually clap); library =
  `src/lib.rs`. A crate can expose both.

## Doc surface (where the manual pointer goes)
- Library: rustdoc — crate-level `//!` and item `///`, rendered on docs.rs.
- CLI: clap `///` doc-comments double as `--help`.

## Distribution model — does the manual travel?
- **Model A (ships-all-by-default):** crates.io publishes the crate contents, so a
  repo-root `llms-full.txt` travels **for free** — **unless** `Cargo.toml` sets
  `include = [...]` (or `exclude`), which silently opts files out. **Check for it**
  and add the manual to `include` if present.

## Traps & idioms
- **Feature-gated public API:** the available API changes with Cargo features
  (`--no-default-features`, optional features) — document which items are behind
  which feature; the default surface ≠ the full surface.
- The derive-CLI doc comments are a single unified surface (they are both the
  rustdoc and the `--help`).
