# paket-multi-project

## Probe metadata

- **Pattern**: multi-project-shared-lock
- **Target framework**: net8.0
- **Generated**: 2026-04-22
- **Purpose**: Validates that Mend correctly maps dependencies per-project when multiple .NET projects share a single paket.lock, and correctly handles the project-specific paket.references override mechanism.

## Feature exercised

This probe exercises Paket's multi-project solution support: three SDK-style .csproj projects (`App`, `Core`, `Tests`) each declare their own `paket.references` listing only the packages they consume, while a single root-level `paket.lock` contains the full resolved graph. It also includes a `Tests.csproj.paket.references` file (the project-specific override that Paket resolves with higher priority than the directory-level `paket.references`).

## Project layout

```
MultiProject.sln
paket.dependencies          <- single source of truth for all version constraints
paket.lock                  <- single shared lockfile for all three projects
src/
  App/
    App.csproj
    paket.references        <- Newtonsoft.Json, Microsoft.Extensions.Logging
  Core/
    Core.csproj
    paket.references        <- Serilog, AutoMapper
  Tests/
    Tests.csproj
    paket.references        <- NUnit, NUnit3TestAdapter  (directory-level)
    Tests.csproj.paket.references  <- NUnit, NUnit3TestAdapter  (project-specific override, takes precedence)
```

## Expected dependency tree

Each Mend project maps to one .csproj. Mend reads per-project `paket.references` to determine which root-level packages belong to each project, then resolves transitive dependencies from the shared `paket.lock`.

### App project

Direct dependencies (from `src/App/paket.references`):
- `Newtonsoft.Json` 13.0.3 — no transitive dependencies in lockfile
- `Microsoft.Extensions.Logging` 8.0.0 — transitive children:
  - `Microsoft.Extensions.DependencyInjection.Abstractions` 8.0.1
  - `Microsoft.Extensions.Logging.Abstractions` 8.0.0
    - `Microsoft.Extensions.DependencyInjection.Abstractions` 8.0.1
  - `Microsoft.Extensions.Options` 8.0.0
    - `Microsoft.Extensions.DependencyInjection.Abstractions` 8.0.1
    - `Microsoft.Extensions.Primitives` 8.0.0

### Core project

Direct dependencies (from `src/Core/paket.references`):
- `Serilog` 3.1.1 — no transitive dependencies in lockfile
- `AutoMapper` 12.0.1 — transitive children:
  - `Microsoft.Extensions.DependencyInjection.Abstractions` 8.0.1

### Tests project

Direct dependencies (from `src/Tests/Tests.csproj.paket.references`, which overrides `src/Tests/paket.references`):
- `NUnit` 3.14.0 — no transitive dependencies in lockfile
- `NUnit3TestAdapter` 4.5.0 — no transitive dependencies in lockfile

## Key detection assertions

1. Mend must produce **3 separate project entries**, one per `.csproj`.
2. `App` must show `Newtonsoft.Json` and `Microsoft.Extensions.Logging` as roots; it must NOT show `Serilog`, `AutoMapper`, `NUnit`, or `NUnit3TestAdapter`.
3. `Core` must show `Serilog` and `AutoMapper` as roots.
4. `Tests` must show `NUnit` and `NUnit3TestAdapter` as roots, sourced from the `.csproj.paket.references` override (not from the directory-level `paket.references`, which is identical in this probe but would differ in a divergence test).
5. The `dependencyFile` for every dependency must be `paket.lock` (shared lockfile, not per-project).
6. Transitive packages that appear as children of multiple roots (e.g. `Microsoft.Extensions.DependencyInjection.Abstractions`) must be correctly represented under each parent — no deduplication that drops children.
