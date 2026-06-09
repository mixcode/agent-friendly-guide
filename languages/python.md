# Python — agent-readiness notes

Language-specific specifics for the `agent-ready` method; read alongside the
agnostic `GUIDELINE.md`.

## Detect
- `pyproject.toml` or `setup.py` — but **verify the role**: a `pyproject.toml`
  may be *tool-config only* (just `[tool.black]`/`[tool.ruff]`); confirm a
  `[project]` or `[build-system]` table before treating it as a package manifest.
- CLI = a `console_scripts`/`[project.scripts]` entry point, or a `__main__.py` /
  `if __name__ == "__main__":` with argparse/click/typer. **Beware decoys:** an
  `argparse` block under `examples/` or a demo `__main__` is not the CLI.
- Library = an importable package (`src/<pkg>/` or `<pkg>/__init__.py`).

## Doc surface (where the manual pointer goes)
- Library: the top-level package `__init__.py` module docstring.
- CLI: argparse/click `--help` / the parser `description`.

## Distribution model — does the manual travel?
- **Model B (opt-in allowlist)** when published to PyPI as a wheel: a root
  `llms-full.txt` is **not** shipped unless you add it — `MANIFEST.in`
  (`include llms-full.txt`) and/or `package_data` / `[tool.setuptools.package-data]`.
- **Model D** when consumed as an unpackaged script/clone (no wheel): the repo is
  the artifact — keep the manual in the repo + `--help`.

## Traps & idioms
- **Install command must match reality.** READMEs often say `pip install <dep>`
  while `requirements.txt`/lock pins a fork or a different version — verify the
  advertised install/deps against the manifest.
- Optional/extra dependencies (`[project.optional-dependencies]`) change which
  imports are available — note feature-gated behavior.
