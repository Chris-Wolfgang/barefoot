# ADR 0004 — Shared search requests, owner on the search, visibility by join

- **Status:** Accepted, 2026-11-06 (amended 2026-11-06: keys, conventions, name uniqueness)
- **Scope:** data model (DESIGN §8), `Business` authorization predicate

## Context

A dozen unrelated users will each keep a few searches active. Two requirements pull on the model:
a user must see only their own searches and results; and overlapping criteria — Resort A 2/15–2/28
for one user, 2/21–3/5 for another — must be looked up **once**, with both users getting the
result. Prices change between runs, so sharing is only ever within a run.

## Decision

1. **`SearchRequest`** is the deduped unit of work per run, unique on
   `(Run, Resort, CheckIn, CheckOut, Adults, Children, Infants)`; results hang off it.
   `CheckOutDate` is in the key (different nights = different price); **`Cabin` is not** (it
   affects flights, not rooms) and stays on the item. Run order = max over the linking searches.
2. **`SearchItem`** is a link `(MatrixRun, SearchRequest, Cabin)`, one per search wanting the
   request. Status, tries, last error and captured time live on the request: a failed scrape fails
   for every search that wanted it.
3. **Results are shared, not copied.** Copies would double-count every aggregate (price-history
   averages, dashboard counts, vocab `LastSeenUtc`) and force "distinct by request" into every
   query; shared rows make aggregates right by construction.
4. **Owner on `SearchMatrix`** (`OwnerId`, a `Guid` — the Entra object id — FK to `AppUser`), not
   on results. Visibility is the join
   `RoomResult → SearchRequest → SearchItem → MatrixRun → SearchMatrix.OwnerId = @me`,
   one predicate applied in the BLL to searches, items, results, digests and price history.
   Nothing a user sees reveals another user.
5. **Search names are unique per owner**: `NormalizedName` (trimmed, upper-invariant) with a unique
   index on `(OwnerId, NormalizedName)`; the editor validates inline and the API answers `409` on a
   race. Different owners may use the same name.
6. **Flights** keep their per-run key and gain the same lease/status columns; an item links to one
   flight set per departure airport via `SearchItemFlightSet`.
7. **Dedupe is per run only.** Prices change between runs; that is the point of running twice a
   week.
8. **`AppUser`** (`Oid` PK, display name, e-mail, last seen, disclaimer version accepted),
   upserted at sign-in: names for lists without Graph calls, FK integrity for owners, a home for
   per-user settings. Seeded with `Guid.Empty` "Unassigned"; `bf-maint assign-owner` replaces it.
9. **Keys are `Guid`s.** The user key is given (Entra `oid`); the rest follow so one convention
   covers the model and clients can mint ids for idempotent creates. Hot tables (`SearchRequest`,
   `RoomResult`, flight rows) use sequential Guids for clustered-index locality.
10. **Concurrency:** a portable `Version` column (`long`, concurrency token, incremented by the
    context) on `SearchMatrix`, `SearchRequest`, `FlightSet`, `Setting`, `Resort`, `Airport`. SQL
    Server `rowversion` is not available on the SQLite test provider, so the portable form keeps
    one code path. `Version` doubles as the row's ETag (ADR 0012). The run lock is a *uniqueness*
    rule and uses the filtered unique index, not a token.
11. **One migration.** The model ships as a single `InitialCreate`; there is no prior schema and no
    data to carry. Later migrations follow ADR 0001 §4.
12. **Conventions:** snake_case names with `pk_/ak_/fk_/ix_/ux_` prefixes; `Restrict` on every FK
    (no cascades — deletes are explicit business operations); every index cites the query it
    serves; a `ModelConventions.Verify` pass asserted empty by a test on SQLite; `DateTimeOffset`
    in UTC for every instant.

## Not chosen

- *Own copy of results per user.* Simpler to reason about row-by-row; rejected for the aggregate
  double-counting and the duplicated storage.
- *An explicit `(OwnerId, ResultId)` visibility table.* Not needed while visibility derives from
  search ownership; it is the additive change for a future "share this result with Carol".
- *Resolve flight sets at read time by matching the six-column key.* No link rows, but a six-column
  join predicate in every digest/results query; rejected for the silent-wrong-answer risk.
- *`rowversion` + SQLite triggers.* Tests would run a different mechanism than production.
- *A global monotonic row version* (one sequence for the whole database, used as ETag, delta
  watermark and cache stamp). Needed when clients sync deltas; barefoot has no offline client, and a
  per-row `Version` avoids the trigger that SQL Server then requires.
- *`bigint` identity keys.* Fewer bytes per index, but the user key is already a Guid and clients
  could not mint ids for idempotent creates.

## Consequences

- One migration-free principle for authorization: filter the search, everything else follows.
- The explode step gets a dedupe pass; `complete` for a search waits on requests it shares.
- The digest can show cheapest airfare per departure airport because the links exist.
- A user whose request is shared cannot tell; a shared failure is reported in each digest without
  attribution.
- Collection ETags and cache invalidation, if ever needed, use `max(UpdatedUtc)` + count or a
  per-table generation row rather than a global version.
