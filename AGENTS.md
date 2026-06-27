# AGENTS.md

## Project Overview
[Project description](./README.md)
[Contributing rules](./CONTRIBUTING.md)
[Pull Request Template](./.github/PULL_REQUEST_TEMPLATE.md)

## Useful links
[PlantUML Preprocessing](https://plantuml.com/preprocessing)

## Repository Structure
```
src/        —  DSL library files (*.puml): `!function`, `!procedure` macros
tests/      —  Unit tests: `*.Tests.puml`, executed by PlantUML -checkonly
doc/        —  Architecture and algorithm diagrams (*.puml)
playground/ —  Usage examples and demos
bin/        —  External tools (plantuml.jar)
out/        —  Build artifacts (zipped library)
.githooks/  —  Git hooks for conventional commits enforcement
```

## Development Environment
- **Java** (JRE 8+) — required to run PlantUML
- **Python 3.x** — used by Taskfile for build scripts
- **Task** (taskfile.dev) — task runner; `choco install go-task` / `brew install go-task`
- **Commitizen** — installed automatically via `task init`

## Initialize the project
```bash
task init    # installs commitizen, downloads plantuml.jar, registers git hooks
```

## Build, Test, and Lint Commands
```bash
task test     # Run unit tests via PlantUML -checkonly
task build    # Package src/*.puml + version.json into out/PlantUML-DSL.zip
task clean    # Remove build artifacts
```

Always run `task test` after making changes to `src/` or `tests/`.

## Coding Conventions
- Read [programming-guideline](./doc/programming-guideline.md) before read or write source code.

## Branching Strategy
- GitHub Flow + Forking Workflow

## Versioning and Releases
- Use semantic versioning (SemVer).
- Auto versioning by Commitzen util
  - on commit used git hooks that invokes Commitzen
  - Commitzen check/validate commit message
  - Commitzen increment project version number in SemVer format
- Version stored in `version.json` (and `pyproject.toml`)
- Tag format: `v<version>`

## Strong Rules
- Always read a file before changing it.
- Do not change or remove my text in MD documents. If you need it, add you text in quotes.
- After making changes and before committing, always re-read `git diff` or the modified files to verify exactly what is being committed — not just your own changes.

## Notes & Constraints
- PlantUML CLI (`-checkonly`) requires Java — tests will fail without it
- Do not alter the structure or keys of `version.json`
- Tests use relative `!include` paths only — no absolute paths
- If string concatenation is required, use the `$format()` function whenever possible (but not necessarily).
- PlantUML supports forward references.