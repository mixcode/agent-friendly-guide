# PHP — agent-readiness notes

Language-specific specifics for the `agent-ready` method; read alongside the
agnostic `GUIDELINE.md`.

## Detect
- `composer.json` is the manifest — but **verify its role, and don't require it**:
  - Library = an `"autoload"` block with `psr-4` (and/or classmap), no `"bin"`. A
    `"files"` autoload entry ships global helper functions — those are part of the
    public surface, not just classes.
  - CLI = a `"bin"` key, **or** (very common, *no manifest at all*) a top-level
    script guarded by `if (PHP_SAPI === 'cli')` calling `getopt(...)`/`$argv`, often
    with a `compile.php` building a **PHAR** (`new Phar(...)`).
  - **A PHP CLI may have NO `composer.json`** (hand-rolled `autoload.php` +
    `spl_autoload_register` + git submodules for deps). Don't conclude "not a PHP
    project" just because the manifest is missing.
- Framework signals: a Laravel **Facade** (`extends Facade`, `getFacadeAccessor`)
  exposes its real surface via `@method static …` docblocks, not visible methods.

## Doc surface (where the manual pointer goes)
- Library: the **PHPDoc block on the entry class / autoloaded helper file**
  (rendered by **phpDocumentor**). For a Facade, put it next to the `@method`
  docblock — that's what an agent reads.
- CLI: the `--help` / usage text (the `getopt` handler or console `description`).

## Distribution model — does the manual travel?
- **Model A with an opt-OUT filter (the PHP shape).** Composer installs the git
  archive of the tag and ships **all tracked files by default**, so a repo-root
  `llms-full.txt` travels **for free** — *unless* the repo uses **`.gitattributes`
  `export-ignore`** to strip files from the dist tarball (the inverse of Rust's
  `include`). **Check `.gitattributes`:** if the manual is `export-ignore`d it won't
  reach consumers; if there's *no* `.gitattributes` (common), everything ships —
  the manual travels, but flag the missing filter as a hygiene issue.
- **Model D** when the artifact is a **PHAR / precompiled binary** or an unpackaged
  clone: keep the manual in-repo + `--help` + release assets; registry-bundling is
  N/A. Note `Phar::buildFromDirectory(".", '~\.php$~')` bundles only `*.php`, so a
  root `llms-full.txt` isn't inside the PHAR — put the pointer + key traps inside
  the CLI script / `--help` (the single-file copy case).

## Traps & idioms
- **`.gitattributes export-ignore` is the allowlist knob** — the #1 PHP
  distribution gotcha; verify the manual isn't export-ignored.
- **Facade / `__callStatic` magic hides the API.** A `Slugify::get(...)` class may
  define zero real methods — calls resolve through `__callStatic` and the only
  signature is a `@method static …` PHPDoc line. Document from the docblock.
- **`composer require <vendor/name>` is case-sensitive on Packagist** — verify the
  advertised install string against `composer.json` `"name"`.
- **PSR-4 namespace ↔ directory must agree**; read the `autoload` map, don't invent paths.
- **Version/ext constraints gate availability** (`"php": "^8.0"`, `"ext-mbstring"`).
- **Install command vs reality** — a PHP CLI may install via `wget` of a PHAR, not
  `composer require`; don't assume Packagist for every repo.
