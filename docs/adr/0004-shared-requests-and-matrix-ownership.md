# ADR 0004 — Shared search requests, owner on the matrix, visibility by join

- **Status:** Accepted, 2026-11-06
- **Scope:** data model (DESIGN §8), `Business` authorization predicate

## Context

A dozen unrelated users will each keep a few matrices active. Two requirements pull on the model:
a user must see only their own matrices and results; and overlapping criteria — Resort A 2/15–2/28
for one user, 2/21–3/5 for another — must be searched **once**, with both users getting the result.
A `SearchItem` today is `(matrix run, resort, check-in, check-out, adults, children, infants, cabin)`
and owns its `RoomResult`s.

## Decision

1. **`SearchRequest`** is the deduped unit of work per run, unique on
   `(Run, Resort, CheckIn, CheckOut, Adults, Children, Infants)`; results hang off it.
   `CheckOutDate` is in the key (different nights = different price); **`Cabin` is not** (it
   affects flights, not rooms) and stays on the item. Priority = max over the linking matrices.
2. **`SearchItem`** becomes a link `(MatrixRun, SearchRequest, Cabin)`, one per matrix wanting the
   request. Status, tries, last error and captured time live on the request: a failed scrape fails
   for every matrix that wanted it.
3. **Results are shared, not copied.** Copies would double-count every aggregate (price-history
   averages, dashboard counts, vocab `LastSeenUtc`) and force "distinct by request" into every
   query; shared rows make aggregates right by construction.
4. **Owner on `SearchMatrix`** (`OwnerId`, a `Guid` — the Entra object id — FK to `AppUser`), not
   on results. Visibility is the join
   `RoomResult → SearchRequest → SearchItem → MatrixRun → SearchMatrix.OwnerId = @me`,
   one predicate applied in the BLL to matrices, items, results, digests and price history.
   Nothing a user sees reveals another user.
5. **Flights** keep their per-run key and gain the same lease/status columns; an item links to one
   flight set per departure airport via `SearchItemFlightSet` (fixes #220 structurally).
6. **Dedupe is per run only.** Prices change between runs; that is the point of running twice a
   week.
7. **`AppUser`** (`Oid` PK, display name, e-mail, last seen), upserted at sign-in: names for lists
   without Graph calls, FK integrity for owners, a home for per-user settings. Seeded with
   `Guid.Empty` "Unassigned"; `bf-maint assign-owner` replaces it.
8. **Concurrency:** a portable `Version` column (`long`, concurrency token, incremented by the
   context) on `SearchMatrix`, `SearchRequest`, `FlightSet`, `Resort`, `Airport`. SQL Server
   `rowversion` is not available on the SQLite test provider, so the portable form keeps one code
   path. The run lock is a *uniqueness* rule and uses the filtered unique index, not a token.
9. **Fresh start:** v1 migrations are squashed into one `InitialCreate`; no data is carried.

## Alternatives considered

- *Own copy of results per user.* Simpler to reason about row-by-row; rejected for the aggregate
  double-counting and the duplicated storage.
- *An explicit `(OwnerId, ResultId)` visibility table.* Not needed while visibility derives from
  matrix ownership; it is the additive change for a future "share this result with Carol".
- *Resolve flight sets at read time by matching the six-column key.* No link rows, but a six-column
  join predicate in every digest/results query; rejected for the silent-wrong-answer risk.
- *`rowversion` + SQLite triggers.* Tests would run a different mechanism than production.

## Consequences

- One migration-free principle for authorization: filter the matrix, everything else follows.
- The explode step gets a dedupe pass; `complete` for a matrix waits on requests it shares.
- The digest can show cheapest airfare per departure airport (#220/#247) because the links exist.
- A user whose request is shared cannot tell; a shared failure is reported in each digest without
  attribution.
