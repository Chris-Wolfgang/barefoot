# ADR 0002 — Every client reaches the database through one Web API; DAL and BLL are separate layers

- **Status:** Accepted, 2026-11-06
- **Scope:** solution structure; every host

## Context

Several processes in several places share one database: a scraper on a home machine, a web UI and
an admin tool used from anywhere, and an API in Azure. A dozen unrelated users will edit their own
searches and read their own results, and the audit trail (`Wolfgang.AuditTrail`) must name the
person behind each change — which only a process that authenticates people can do. Business rules
(ownership, dedupe, run lifecycle) must be applied once, not once per client.

The earlier tool compiled the data access and the business logic into every host (desktop app and
CLI); its lesson is that two processes with their own connection strings and their own copies of
the rules drift, and neither knows who the user is.

## Decision

1. **One Web API owns all database access.** `Wolfgang.Barefoot.Api` hosts the business logic
   (`Business`) and the data access (`Data`). Every other process — scraper, admin tool, web UI —
   is a client of the API and holds no connection string.
2. **The one exception is the maintenance tool.** Schema changes need DDL rights the API must not
   have; `Wolfgang.Barefoot.Maintenance` connects directly with a DDL identity for `migrate`,
   `verify-schema`, `grant-roles` and `assign-owner`. Nothing a user would do belongs there.
3. **DAL and BLL are separate projects.** `Data` holds the `DbContext`, configurations, the
   migration and persistence queries/commands; `Business` holds the run lifecycle, search editing
   and authorization rules, catalog diffing, digest building and delivery, vocabulary rules and
   settings. EF's `DbContext` is the unit of work; the DAL is the context plus focused query and
   command classes behind the Core interfaces — not a generic repository. Rules: one query class
   per read with an `AsNoTracking` projection; no `IQueryable` crosses a Core interface; no
   `Set<T>()` on the unit-of-work surface; conversion methods live where the conventions say
   (`ToDto` in `Api`, `ToEntity` in `Data`).
4. **The Core interfaces are the API surface.** `Api` exposes one resource per interface;
   `ApiClient` implements the same interfaces over HTTP (ADR 0010); `Presentation` does not know
   which implementation it has. DTOs live in `Contracts`, shared by `Api` and `ApiClient`.
5. **Dependency rules** are enforced by project references and by architecture tests (DESIGN §2,
   §14): no UI or worker project references `Data` or an EF package.

## Not chosen

- *Each host talks to the database.* Every host needs a connection string and a database identity,
  rules are duplicated, and the audit trail cannot name a person.
- *A modular monolith with vertical-slice modules.* Right for a larger product with many bounded
  contexts; for a dozen users and one domain, two layers behind one API is enough structure. The
  per-read query class and explicit registration rules are kept from that style.
- *Generic repositories over EF.* A second abstraction over the unit of work that hides the
  queries that matter for a database billed per second.

## Consequences

- Logic is in one place and audited with a real identity (ADR 0005).
- The scraper's hot path is HTTP over the internet; ADR 0003 shapes the lifecycle so this costs
  ~2 calls per lookup rather than one per save point.
- The API is a public endpoint, so authentication is not optional and precedes everything else.
- `WebApplicationFactory` integration tests over SQLite, driving the real `ApiClient`, are the
  end-to-end harness: the contract every client uses is what gets tested.
- Versioning: the API and its clients ship together but can be deployed apart; the API is
  path-versioned and its OpenAPI document is committed and frozen per version (ADR 0010); every
  client checks `GET /version` at start and fails fast on a mismatch.
