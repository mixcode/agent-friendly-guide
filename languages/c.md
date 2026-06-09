# C — agent-readiness notes

Language-specific specifics for the `agent-ready` method; read alongside the
agnostic `GUIDELINE.md`. C is the sharpest test of "does the manual travel?"

## Detect
- `Makefile` / `CMakeLists.txt`. Library = headers + `.a`/`.so` (or a
  **single-header** lib, STB-style). CLI = a `main` + `getopt`/manual argv parsing.

## Doc surface (where the manual pointer goes)
- Library: the header file's top comment (and per-function comments).
- CLI: `usage()` output **and man pages** for installed tools.

## Distribution model — does the manual travel?
- **Model D (no registry — the header/binary IS the artifact).** Registry
  bundling is N/A. Ship the manual in the repo, the header top-comment, man pages,
  and/or release assets.
- **Single-header sub-case (critical):** a single-header lib is consumed by
  **copying one file** into another project — a separate `llms-full.txt` does
  **not** travel with it. So put the pointer **and the key traps inside the
  header itself**.
- **Man pages are a real *travel* mechanism for installed CLIs**, not just a doc
  surface: ship a `.1` and have `make install` place it in `share/man` so `man
  <tool>` reaches the user. (Many C Makefiles install only the binary — check.)

## Traps & idioms
- **Single-header implementation macro:** the header is inert until the consumer
  defines e.g. `#define WFC_IMPLEMENTATION` in exactly one translation unit —
  document this prominently; it's the #1 misuse.
- **Spec as source of truth:** the authority may be an external standard (e.g. a
  protocol spec PDF), not the code — name it.
