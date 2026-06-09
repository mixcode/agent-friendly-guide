# Ruby — agent-readiness notes

Language-specific specifics for the `agent-ready` method; read alongside the
agnostic `GUIDELINE.md`.

## Detect
- Manifest: `*.gemspec` (`Gem::Specification.new`) + usually a `Gemfile`. A lone
  `Gemfile.lock` indicates an app, not a publishable gem.
- CLI = a `bin/` (or `exe/`) executable + `spec.executables` in the gemspec, plus a
  CLI lib: **Thor** (`class X < Thor`, `desc`/`method_option`) or stdlib
  **`OptionParser`/`optparse`**. (Thor is Ruby's cobra/clap; optparse is stdlib and
  easy to miss — grep `OptionParser.new` and `< Thor`.)
- Library = `lib/<name>.rb` + gemspec, no executable. A gem can be both.

## Doc surface (where the manual pointer goes)
- Library: a top-of-file **YARD/RDoc doc-comment** on the entry module/class in
  `lib/<name>.rb`, rendered on **rubydoc.info** (auto-generates from YARD for every
  published gem; YARD renders both RDoc and Markdown).
- CLI: `--help` — for **Thor** the per-command `desc '...'` strings *are* the help
  text; for optparse the `OptionParser` banner. Add the pointer there and/or atop
  the `bin/`/`exe/` executable.

## Distribution model — does the manual travel?
- **Model B (opt-in allowlist) — but read `spec.files` per repo.** The gemspec's
  `spec.files` is the allowlist of what gets packaged; the idiom decides behavior:
  - `s.files = \`git ls-files\`.split` → ships **all tracked files**, so a committed
    root `llms-full.txt` travels for free (behaves like Model A).
  - `spec.files = Dir.glob("{bin,lib}/**/*") + %w(README.md …)` → a **literal
    allowlist**; a root `llms-full.txt` is **silently dropped** unless added to the
    glob/`%w(...)`. **This is the common trap — verify it.**
  Fix: ensure `spec.files` matches the manual, then confirm with `gem build *.gemspec`
  + `gem contents`/unpack.
- **Model D** when consumed unpackaged (clone, vendor, `bundle` from a git ref, or a
  copied `bin/` script): the checkout is the artifact — manual in-repo + `--help`;
  for a single copied script, put the pointer inside it.
- **Model C** when referenced by `gem '<name>', git: '<url>'` — manual travels with
  the resolved checkout; document the git URL.

## Traps & idioms
- **`spec.files` allowlist (above)** — the #1 Ruby distribution trap.
- **`require` path ≠ gem name** — `gem install foo-bar` may be `require 'foo/bar'`;
  document the real `require` line.
- **Mixin vs singleton dual API** — modules often expose both `Module.method` and an
  `include`-able instance API; document both call forms.
- **Install command vs manifest** — verify README `gem install <x>` matches
  `spec.name`; check `required_ruby_version` and runtime deps for drift.
- **Generated-docs target** — the consumer reads rubydoc.info (YARD), not the repo;
  a pointer buried in a non-doc comment won't render there.
