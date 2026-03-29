<!--
  Sync Impact Report
  ==================
  Version change: [initial template] → 1.0.0
  Modified principles: N/A (initial ratification)
  Added sections:
    - 10 Core Principles (I–X)
    - Technical Constraints
    - Development Workflow & Quality Gates
    - Governance
  Removed sections: None
  Templates requiring updates:
    - .specify/templates/plan-template.md ✅ aligned (Constitution Check gate
      already present; will reference these 10 principles)
    - .specify/templates/spec-template.md ✅ aligned (no constitution-specific
      content to change)
    - .specify/templates/tasks-template.md ✅ aligned (task phases map to
      principles naturally)
  Follow-up TODOs: None
-->

# SmartFoundation Constitution

## Core Principles

### I. Housing Baseline

The Housing module's `WaitingListByResident` flow is the canonical
implementation pattern for this repository. Every new page or feature that
resembles a Housing-style flow MUST follow this style first, unless the task
explicitly requires a different approach.

- Controllers MUST follow the partial-class pattern used by
  `HousingController.WaitingListByResident.cs`
- The reference execution path is: controller → `HousingController.Base`
  → `MastersServies` → gateway SP → downstream feature SP
- Server-built UI models (`FormConfig`, `SmartTableDsModel`,
  `SmartPageViewModel`) are the standard pattern, not client-side rendering
- Views MUST stay thin and render through `SmartRenderer`

**Rationale**: This pattern is the most reliable and well-understood style
in the codebase. Deviating without explicit direction introduces
inconsistency and risk.

### II. Gateway SP Architecture

`ProcedureMapper` maps entry/gateway procedures exposed to the application
layer only. Downstream feature-specific stored procedures MUST NOT be
individually mapped.

- Application code reaches gateway procedures (`Masters_DataLoad`,
  `Masters_CRUD`) which route by `@pageName_` and `@ActionType`
- Downstream procedures (e.g. `[Housing].[WaitingListByResidentDL]`)
  hold business logic, validations, and audit logging
- This two-tier routing separation MUST be preserved when adding new
  Housing-style pages

**Rationale**: Flattening the gateway layer into `ProcedureMapper` would
bypass the shared permission checks, notification triggers, and error
handling that `Masters_DataLoad` and `Masters_CRUD` provide.

### III. DataSet Pattern

`DataSet` and `DataTable` are established team patterns in active Housing
code. They MUST NOT be auto-refactored away.

- Returned table 0 is always permissions
- Later tables are feature data, DDL sources, lookup lists
- Controllers commonly map `DataTable.Columns` to `TableColumn`,
  `DataRow` objects to `Dictionary<string, object?>`, and add `p01..p50`
  aliases for modal forms
- Multiple `rowsList_*` and `dynamicColumns_*` collections per returned
  table are the expected pattern

**Rationale**: This is a deliberate architectural choice, not a legacy
artifact. Refactoring it breaks every Housing-style page simultaneously.

### IV. Generic CRUD Contract

`CrudController` is not incidental plumbing — many pages depend on it.
Its contract MUST be preserved unless the task explicitly replaces the
entire mechanism.

- Form fields bind as `p01` through `p50`
- `CrudController` converts them to `parameter_01` through `parameter_50`
- Hidden context fields (`pageName_`, `ActionType`, `idaraID`,
  `entrydata`, `hostname`, `redirectAction`, `redirectController`) are
  part of the expected contract
- CRUD actions post to `/crud/insert`, `/crud/update`, or `/crud/delete`
- Feedback uses Toastr-style `TempData` buckets: `Success`, `Warning`,
  `Error`, `Info`

**Rationale**: Removing or redesigning `CrudController` without a full
replacement plan breaks every page that posts through it.

### V. Controller Pattern

Housing-style controllers MUST follow the shared controller pattern
established in `HousingController.Base`.

- Call `InitPageContext(out redirectResult)` early in the action
- Read shared session-backed fields: `usersId`, `IdaraId`, `HostName`
- Set `ControllerName` and `PageName`
- Build positional stored procedure argument arrays
- Call a `MastersServies` DataSet method
- Split the result using `SplitDataSet(...)`
- Treat the first table as permissions; use later tables as feature data
  (`dt1`, `dt2`, `dt3`, `dt4`, ...)
- Build server-side UI config objects instead of pushing logic into
  Razor views

**Rationale**: This pattern ensures consistent session handling,
permission gating, and data assembly across all Housing pages.

### VI. Layer Separation

The application follows a strict dependency rule:
Presentation → Application → DataEngine → Database.

- `SmartFoundation.Mvc` controllers MUST inject Application Layer
  services only — never call `ISmartComponentService` directly
- `MastersServies` is the active gateway service for existing Housing
  flows and MUST be preserved in those flows
- `BaseService` is the correct pattern for new narrow JSON-style services
- New focused services MUST inherit from `BaseService`, use
  `ProcedureMapper` for entry procedure resolution, and register in
  `ServiceCollectionExtensions.cs`
- `SmartFoundation.DataEngine` contains data access only — no business
  logic

**Rationale**: Violating the dependency rule couples layers incorrectly
and makes the codebase harder to test and maintain.

### VII. Preserve Naming

Database-side parameter names and form field names are part of a live
runtime contract. They MUST NOT be renamed casually.

- Preserve casing and spelling exactly for: `pageName_`, `ActionType`,
  `idaraID`, `entrydata`, `hostname`
- Preserve positional parameter names: `parameter_01` through
  `parameter_50`
- Preserve form binding names: `p01` through `p50`
- Do not add `@` prefixes to parameter names in application code
- Do not replace these names with cleaner aliases unless intentionally
  changing the whole database contract

**Rationale**: These names flow through gateway stored procedures,
downstream procedures, `CrudController`, and form bindings. A rename
in any one layer silently breaks the others.

### VIII. Database Is Reference Only

`SmartFoundation.Database` is a snapshot/reference project. It MUST NOT
be treated as the source of truth for live database behavior.

- It is not guaranteed current
- Use it to understand intent, naming, and routing — not to prove live
  behavior
- When database behavior matters, verify against active runtime code:
  `ProcedureMapper.cs`, `MastersServies.cs`,
  `SmartComponentService.cs`, and current MVC callers in
  `SmartFoundation.Mvc`

**Rationale**: Relying on SQL files that may be outdated leads to
incorrect assumptions about stored procedure signatures and behavior.

### IX. Spec-Kit Branch Isolation

Spec-kit artifacts live on the `spec-kit` branch only. They MUST NEVER
reach the `main` branch or any PR targeting upstream.

- `.specify/`, `.opencode/command/speckit*` files are confined to the
  `spec-kit` branch
- PRs to upstream (`ProgrammingSectionKFMC/SmartFoundation`) MUST branch
  from `main` and contain zero spec-kit files
- The `main` branch `.gitignore` includes guards to prevent accidental
  inclusion of spec-kit paths
- When rebasing `spec-kit` onto `main`, resolve conflicts in favor of
  keeping spec-kit files

**Rationale**: The upstream repository does not use spec-kit. Leaking
these files would pollute the upstream codebase and confuse other
contributors.

### X. Code Style

All code MUST follow the established conventions of the SmartFoundation
codebase.

- PascalCase for classes, methods, properties, and public members
- camelCase for locals, parameters, and private fields
- Prefer `Dictionary<string, object?>` for dynamic parameter bags
- Preserve async/await patterns for all I/O operations
- Keep `using` directives at the top and remove unused ones
- XML docs on public members when adding or substantially changing them
- No hard-coded stored procedure names in the Application Layer — use
  `ProcedureMapper`
- In the Presentation Layer, no hard-coded SP names and no direct
  DataEngine calls

**Rationale**: Consistent style reduces cognitive load during code
reviews and makes the codebase navigable for the entire team.

## Technical Constraints

- **Framework**: ASP.NET Core 8.0 MVC
- **Architecture**: Clean Architecture with four active projects:
  `SmartFoundation.Mvc` (presentation),
  `SmartFoundation.Application` (business logic),
  `SmartFoundation.DataEngine` (data access),
  `SmartFoundation.UI` (reusable components)
- **Database access**: Dapper via `SmartComponentService` using
  `SmartRequest`/`SmartResponse` objects
- **Testing**: xUnit with Moq in
  `SmartFoundation.Application.Tests`; no MVC test project exists today
- **Frontend**: Tailwind CSS with npm tooling in `SmartFoundation.Mvc`
- **Composition root**: `SmartFoundation.Mvc/Program.cs`
- **DI registration**: `ServiceCollectionExtensions.cs` in
  `SmartFoundation.Application`
- **Service lifetimes**: App services are scoped; `ConnectionFactory`
  is singleton; `ISmartComponentService` maps to
  `SmartComponentService`

## Development Workflow & Quality Gates

- Build the solution before committing:
  `dotnet build SmartFoundation.sln`
- Run tests when changing Application Layer code:
  `dotnet test SmartFoundation.Application.Tests`
- Full solution build includes `SmartFoundation.Database.sqlproj` which
  may require SQL project tooling locally
- Frontend changes require Tailwind rebuild:
  `npm --prefix SmartFoundation.Mvc run tw:build`
- `AGENTS.md` is the primary operating guide; if documentation
  conflicts with active code, trust active code
- The safest reference path for any Housing-style feature is:
  1. `HousingController.WaitingListByResident.cs`
  2. `HousingController.Base.cs`
  3. `CrudController.cs`
  4. `MastersServies.cs`
  5. Gateway and downstream stored procedures for that page

## Governance

This constitution is the governing document for all spec-driven
development on the SmartFoundation `spec-kit` branch.

- **Supremacy**: Constitution principles supersede individual agent
  preferences or generic best practices when they conflict
- **Amendment procedure**: Amendments require documentation of the
  change, rationale, and impact on existing code. MAJOR version for
  principle removal/redefinition. MINOR for new principles or expanded
  guidance. PATCH for clarifications.
- **Compliance review**: All generated specs, plans, and task lists
  MUST be checked against these principles before implementation
- **Complexity justification**: Any deviation from these principles
  MUST be explicitly documented in the plan's Complexity Tracking
  section with rationale and rejected simpler alternatives
- **Primary guidance file**: `AGENTS.md` is the runtime operating
  guide; this constitution encodes its rules as spec-kit governance

**Version**: 1.0.0 | **Ratified**: 2026-03-29 | **Last Amended**: 2026-03-29
