# ADR 0002 — Every client reaches the database through one Web API; DAL and BLL are separate layers

- **Status:** Accepted, 2026-11-06
- **Scope:** solution structure; every host

## Context

v1 is one process per host (WPF, CLI) with the business logic and the data access compiled into
each. Moving the UI to the web and serving a dozen unrelated users means several processes in
several places touching the same database, and it means knowing *who* made each change — the
`Wolfgang.AuditTrail` library records a user per change, but only if the process knows one.

`Presentation` and the CLI already consume only Core interfaces (`IMatrixCatalog`, `IMatrixEditor`,
`IDashboardService`, `IResultsService`, `IRunHistoryService`, `IPriceHistoryService`,
`IReferenceDataService`, `IRoomVocabularyCatalog`, `ISettingsService`, the link editors); `Data`
implements them. The seam for an API already exists.

## Decision

1. **One Web API owns all database access.** `SandalsSearch.Api` hosts the business logic
   (`Business`) and the data access (`Data`). Every other process — scraper, admin tool, web UI —
   is a client of the API and holds no connection string.
2. **The one exception is the maintenance tool.** Schema changes need DDL rights the API must not
   have; `SandalsSearch.Maintenance` connects directly with a DDL identity for `migrate`,
   `grant-roles`, `assign-owner` and repair. Nothing a user would do belongs there.
3. **DAL and BLL are separate projects.** `Data` keeps the `DbContext`, configurations, migrations
   and pure persistence queries; `Business` takes the logic that lives in `Data` today
   (`RunOrchestrator`, `MatrixEditor` and link editors, `MatrixDigestBuilder`, the catalog syncs,
   digest delivery). Not a repository pattern: EF's `DbContext` is the unit of work; the DAL is the
   context plus focused query/command classes behind the Core interfaces.
4. **The Core interfaces are the API surface.** `Api` exposes one resource per interface;
   `ApiClient` implements the same interfaces over HTTP; `Presentation` does not know which
   implementation it has. DTOs move to `Contracts`, shared by `Api` and `ApiClient`.
5. **Dependency rules** are enforced by project references (DESIGN §2).

## Consequences

- Logic is in one place and audited with a real identity (ADR 0005).
- The scraper's fast path (process → DB) becomes HTTP over the internet; ADR 0003 shapes the
  lifecycle so this costs ~2 calls per search item rather than one per save point.
- The API is a public endpoint, so authentication is not optional and precedes everything else.
- `WebApplicationFactory` integration tests over SQLite, driving the real `ApiClient`, replace the
  CLI as the end-to-end harness: the contract every client uses is what gets tested.
- Project count grows from 12 to ~17; `Cli`, `App` and eventually `Wpf` retire.
- Versioning: the API and its clients ship separately; every client checks `GET /version` at start
  and fails fast on a mismatch.
