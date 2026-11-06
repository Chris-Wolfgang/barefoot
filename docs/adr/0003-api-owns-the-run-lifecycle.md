# ADR 0003 — The API owns the run lifecycle; the scraper pulls work

- **Status:** Accepted, 2026-11-06
- **Scope:** `Business` (run lifecycle), `Api` (run resources), `Scraper`

## Context

`RunOrchestrator` does everything in one process today: lock, create run, explode each matrix,
execute items through `SearchExecutor`, persist through `RoomResultSink`, download flights,
aggregate, e-mail, derive status, resume. It has 13 save points. With the database behind an API
(ADR 0002) each of those would be a round-trip to Azure in the middle of a transaction the scraper
thinks it owns, the run lock would stay a file on the home machine, and the lifecycle logic would
live in the one process that is supposed to be "just a scraper".

## Decision

Split by *what the decision is about*:

- **Data decisions → API/Business:** run creation; the lock (a filtered unique index on
  `Runs(Status) WHERE Status = InProgress` — the database guarantees one live run); explode and
  dedupe (ADR 0004); leasing requests and flight sets to a worker with an expiry; persisting
  results; vocabulary discovery; stage advancement; aggregation and digest storage; status
  derivation; resume semantics; recording an abort.
- **Site decisions → Scraper:** Playwright, politeness delay and jitter, per-item retries, the
  circuit-breaker decision, time-zone arithmetic on scraped times.

The scraper is a pull loop over `claim` / `result` for requests, then for flight sets, then
`complete` per matrix and `finish` (resource list in DESIGN §5). Leases are ~5 minutes; a crashed
worker's items return to Pending on expiry; a `result` carries the `attempt` token it was leased
with, so a late duplicate is ignored. Claims hand out one request at a time (exact priority order,
cheap abort). Completion (aggregate + store digest) is an explicit call from the scraper, never a
side effect of a `result`.

## Alternatives considered

- *Scraper drives, API is a remote DAL.* Minimal code change; rejected because it is chatty at
  every save point, keeps the lock machine-local and puts the lifecycle in the scraper.
- *Automatic completion when the last result lands.* Rejected: it would put aggregation (and in v1
  e-mail) inside a `result` request.

## Consequences

- `RunOrchestrator` splits into `RunLifecycle` (Business) and `ScrapeLoop` (Scraper); its
  checkpoint tests move server-side.
- `FileRunLock` / `IRunLock` go; a second scraper host needs no new design.
- Checkpoint atomicity (#116) is one server transaction per `result`.
- A run knows its complete work list at creation (requests and flight sets are rows), so progress
  is countable and the UI can show a run in flight.
- ~2 HTTP calls per search item; ~200 items a night is under a minute of overhead.
- The scraper needs the same retry posture against a cold API that ADR 0001 gives the API against a
  cold database.
