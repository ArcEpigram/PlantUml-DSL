# AGENTS.md

## Project Overview
PlantUML DSL — reusable `.puml` macros and functions library for PlantUML diagrams.
Automation via **Task** (task runner), versioning via **Commitizen** (Conventional Commits),
testing via PlantUML CLI `-checkonly`.

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

Initialize the project:
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

### DSL files (`src/*.puml`)
- Wrap every file in `@startuml` / `@enduml`
- Separate sections with `''----...----` (60 dashes)
- Each `!function` / `!procedure` must have a doc comment with usage example
- Use `camelCase` for function names, `PascalCase` for file names
- Indent with 4 spaces
- Use `!include` with relative paths only

### Tests (`tests/*.Tests.puml`)
- One test file per source module: `Utils.puml` → `Utils.Tests.puml`
- Arrange/Act/Assert pattern inside `!procedure`
- Call the test procedure directly at the bottom of the file (after `Tests RUN!` section)
- Use `!assert` for validation
- Include source with `!include ../src/<FileName>.puml`

## Versioning and Releases
- Version stored in `version.json` and `pyproject.toml`
- Bump with: `cz bump`
- Tag format: `v<version>`
- After bump: version.json auto-updates, changelog entries appended

## Common Workflows

### Adding a new DSL function
1. Add `!function` to a file in `src/` (create a new file if needed)
2. Join the new source file into the library if creating one
3. Create corresponding `tests/<Name>.Tests.puml`
4. Run `task test`
5. Commit with Conventional Commits format

### Adding a usage example
1. Create a `.puml` file in `playground/`

### Cutting a release
1. `cz bump`
2. `task build`
3. Archive appears in `out/`

## Notes & Constraints
- PlantUML CLI (`-checkonly`) requires Java — tests will fail without it
- DSL functions may depend on `version.json` (see `$get_library_version` in `Utils.puml`)
- Do not alter the structure or keys of `version.json`
- All `@startuml` / `@enduml` blocks are mandatory
- Tests use relative `!include` paths only — no absolute paths
- Source files are zipped as-is into `out/`; keep file paths relative to `src/`
