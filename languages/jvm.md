# JVM (Java / Kotlin) — agent-readiness notes

Language-specific specifics for the `agent-ready` method; read alongside the
agnostic `GUIDELINE.md`.

## Detect
- `pom.xml` (Maven) or `build.gradle(.kts)` (Gradle). Library = a jar with no app
  entry point; CLI/app = an application plugin / a `main` (commons-cli, picocli).

## Doc surface (where the manual pointer goes)
- Library: `package-info.java` + class Javadoc, rendered on javadoc.io.
- CLI: commons-cli/picocli `--help`.

## Distribution model — does the manual travel?
- **Model B (opt-in):** a published JAR ships only packaged resources — put the
  manual in `src/main/resources` and/or rely on the separately-published
  **`-javadoc` jar** (agents read it via javadoc.io).
- **Model D:** a CLI application distribution (launcher + jars, or deb/rpm/Docker/
  Homebrew) is the artifact — ship the manual in the repo + release assets.

## Traps & idioms
- **Static factory idiom:** many JVM types have no public constructor and are
  created via static `from()`/`of()`/`create()` methods — document the real
  construction path, not a `new` call.
- **README location:** JVM repos often keep the README at **`.github/README.md`**,
  not the repo root — check there before concluding it's missing.
