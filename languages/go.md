# Go — agent-readiness notes

Language-specific specifics for the `agent-ready` method; read alongside the
agnostic `GUIDELINE.md`. Go is the most-validated ecosystem and the easiest case.

## Detect
- `go.mod` present. CLI = a `cmd/` dir or a `package main` with a `main` func
  (flag/cobra/spf13). Library = exported (capitalized) API, no `main`.
- A repo can be both (a library that also ships a `cmd/` CLI) — note both.
- **Multi-module repos:** more than one `go.mod` in the tree means more than one
  independently-published module — and **Go needs no `go.work` for that** (a
  nested `go.mod` is excluded from its parent and resolved on its own). For the
  monorepo survey, enumerate every `go.mod` (`find . -name go.mod`, minus
  `vendor/`), not just `go.work` `use` entries — and the **root `go.mod` itself is
  a published module**, so it's both a member to survey *and* a package needing its
  own manual (not only a map).

## Doc surface (where the manual content lives — not just a pointer)
- Library: the *candidate* surfaces are the package doc comment / `doc.go`,
  **`Example` functions in `*_test.go`**, a **task-named reference doc**, and the
  source (godoc and examples both render on pkg.go.dev). **Don't assume `doc.go` is
  the one the agent opens** — verify it (GUIDELINE §3a, "verified read path"): in
  testing, agents went to the example tests + a task-named reference doc and **never
  opened `doc.go`**, so a pointer placed there was never seen (GUIDELINE Appendix A,
  E6). So **inline the decisive traps and the one canonical recipe on the surface(s)
  a clean agent is *observed* to read** — commonly a runnable `Example` plus the
  task-named doc — not only as a pointer, and not assumed into `doc.go`. Add the
  one-line raw-`llms-full.txt` pointer at the top of `doc.go` too, but treat it as a
  fallback, not the primary surface.
- CLI: `--help` / `flag.Usage`. Give it a one-line purpose + synopsis banner.

## Distribution model — does the manual travel?
- **Model A (ships-all-by-default).** The module cache resolves the whole repo, so
  `llms-full.txt` / `llms.txt` / `AGENTS.txt` travel **for free** from the repo
  root. No allowlist to maintain.

## Traps & idioms
- **`go install` needs the module path to match the repo URL.** If `go.mod` says
  `module foo` but the README says `go install github.com/x/foo@latest`, install
  fails. Verify they agree.
- `gofmt` + `go vet` are free in-toolchain checks (the §3a drift defense most
  ecosystems must wire up manually — see GUIDELINE §4c). A generated/edited file
  that isn't `gofmt`ed is a smell.
- For batch CLIs, exit codes should reflect per-item failure (don't exit 0 when an
  item failed) and unattended mode shouldn't block on an interactive prompt — both
  are common CLI-overlay findings.
