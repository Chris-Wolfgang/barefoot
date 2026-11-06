# SandalsSearch — Design (v2)

v1 (2026-07-03) described one Windows machine running a WPF app and a CLI over a local SQL Server.
**v2 (2026-11-06)** moves the application to a hosted shape: one Web API in Azure owns every
database access, a scraper on a home machine feeds it, and a web UI serves up to a dozen unrelated
users. The reasoning for each decision is in `docs/adr/`; this document is the spec that results.

Sections that still describe the shipped code are marked **Current**; sections describing the
target are marked **Target**. Nothing marked Target is built yet.

---

## 1. What it is

Sandals.com's search works well when you know the resort, departure airport and dates — and is
tedious when any of those are flexible. SandalsSearch lets a user define a **search matrix**
(resorts × departure airports × a flexible check-in window × nights), expands it into every
individual search, executes them against the Sandals online booking engine (driven in a real
browser — §3), stores **full-fidelity** results (every room option, every flight option), keeps
history so prices can be compared **over time**, runs unattended on a schedule, and e-mails a
**digest** per matrix — including filtered views the Sandals site can't express.

v2 adds: **several users**, each seeing only their own matrices and results; **one search per
distinct request per run**, however many users asked for it; and a hosted UI reachable from
anywhere.

## 2. Architecture — Target

Everything reaches the database through one Web API. The API hosts the business logic and the
data access; every other process is a client of it. One exception: the maintenance tool, which
owns the schema and connects directly with a DDL identity (ADR 0002, ADR 0007).

```
 home machine (NUC)                        Azure                                   anywhere
 ┌─────────────────────┐   HTTPS + token   ┌──────────────────────────┐            ┌────────────┐
 │ bf-scraper          │ ────────────────► │ SandalsSearch.Api        │ ◄────────  │ Web UI     │
 │  Playwright/Chromium│                   │  Business (BLL)          │  OIDC user │ (Blazor)   │
 │  residential IP     │                   │  Data (DAL, EF Core)     │            └────────────┘
 └─────────────────────┘                   │  Entra JWT validation    │            ┌────────────┐
 ┌─────────────────────┐   HTTPS + token   │  audit: who did what     │ ◄────────  │ bf         │
 │ bf-maint            │ ─┐                └────────────┬─────────────┘            │            │
 │  migrate, repair    │  │ direct, DDL identity        │ managed identity         └────────────┘
 └─────────────────────┘  └───────────────► Azure SQL Database (free offer, serverless, auto-pause)
```

Namespaces are `Wolfgang.Barefoot.*` (ADR 0009 — `Barefoot` is the code name; the product display name is a
string, "Barefoot Vacation" as the placeholder); the v1 `SandalsSearch.*` names below map one-to-one.

| Project | Layer | Contents |
|---|---|---|
| `SandalsSearch.Core` | Domain + contracts | entities, the service interfaces every client consumes, pure services (window resolver, expander, validators, digest renderer), `DatabaseMessages` |
| `SandalsSearch.Contracts` | DTOs | request/response shapes shared by the API and `ApiClient` (most Core DTOs move here) |
| `SandalsSearch.Data` | **DAL** | `DbContext`, configurations, migrations, pure persistence queries/commands behind Core interfaces |
| `SandalsSearch.Business` | **BLL** | run lifecycle, matrix editing + authorization rules, catalog snapshot diffing, digest building + delivery, vocab rules |
| `SandalsSearch.Api` | host | ASP.NET Core minimal API over Business + Data; Entra JWT validation; `HttpContextAuditUserProvider`; database warm-up (ADR 0001) |
| `SandalsSearch.ApiClient` | client | HTTP implementations of the same Core interfaces, used by every client |
| `SandalsSearch.Obe` | adapter | Playwright booking, flight and catalog clients, selectors, parsers (unchanged) |
| `SandalsSearch.Scraper` (`bf-scraper`) | host | `Obe` + `ApiClient`; the only process with a browser and a home IP |
| `SandalsSearch.Admin` (`bf`) | host | `ApiClient` only; scriptable "API, do X" steps; runs anywhere the API is reachable |
| `SandalsSearch.Maintenance` (`bf-maint`) | host | `Data` only; `migrate`, `grant-roles`, `assign-owner`, repair; the one direct-DB client |
| `SandalsSearch.Presentation` | UI logic | slimmed to framework-free screen logic (`MatrixDraft`, pickers, chart scaling, paging); the v1 view models retire with WPF (§12) |
| `SandalsSearch.Web` | host | Blazor Web App (Fluent UI Blazor) over `Presentation` + `ApiClient`; user sign-in |

**Dependency rules** (enforced by project references): `Scraper` never references `Data`;
`Admin` never references `Obe` or `Data`; `Maintenance` never references `ApiClient` or `Obe`;
`Api` never references `Obe`. The Core interfaces are the API surface: `Data`/`Business` implement
them server-side, `ApiClient` implements them over HTTP, and `Presentation` does not know which it
has.

**Retired by v2:** `SandalsSearch.Cli` (verbs split across the three hosts), `SandalsSearch.App`
(its composition root splits between `Api` and `Scraper`), `SandalsSearch.Wpf` (once `Web` reaches
parity), `FileRunLock`/`IRunLock` (the lock moves into the schema, §5), the legacy `Data/Obe` HTTP
client (#137).

### 2.1 Architecture — Current (shipped, v1)

Ports-and-adapters over a host-agnostic core: `Core`, `Data` (DbContext + the business logic that
v2 moves to `Business`), `Obe`, `App` (composition root), `Presentation`, `Cli`, `Wpf`. One
process, one local SQL Server Express. `Cli` doubles as the end-to-end test harness.

## 3. Booking-engine integration — Current

The shipped clients drive the booking UI in Chromium through Playwright and read the rendered room
and flight cards — see [API-NOTES.md](API-NOTES.md). Switched on by `Obe:UsePlaywright`;
`Obe:Headless` controls the window. Politeness: sequential with a configurable delay (~2.5 s) +
jitter. Earliest searchable check-in ≈ today + `Run:MinLeadDays`.

v2 changes nothing here except *where* the results go: the scraper posts them to the API instead of
writing them to the database.

## 4. Search matrices — Current, plus ownership (Target)

A matrix has two halves:

- **Search axes** (what gets downloaded): resorts, departure airports, nights, check-in window,
  occupancy (adults/children/infants), include-flights flag, cabin.
- **Filter criteria** (what the digest shows): room level / view / type / features, flight
  constraints (max travel time per direction, max stops), optional price cap. **Filters apply at
  aggregation time only — capture always downloads everything.**

Lifecycle: Draft → Active → Inactive (results retained). Active matrices carry a **priority**
(default 50).

**v2:** every matrix has an **owner** (`OwnerId`, the user's Entra object id). Editors see and edit
only their own matrices; Admins see all. Users are unrelated to one another: nothing a user sees
reveals that another user exists. Digests go to the owner's sign-in address; Admins may add further
recipients; self-service "add my spouse" with address verification is phase 2 (ADR 0006).

### Check-in windows — Current

- **Recurring (annual):** month/day pair; resolved at run time:
  `effectiveStart = max(resolvedStart, today + minLeadDays)`; rolls to next year when
  `today + minLead > resolvedEnd`. Year-wrapping windows are legal.
- **One-time:** explicit dates; auto-deactivates once the window passes.

## 5. Run lifecycle — Target (ADR 0003)

The API owns the lifecycle; the scraper pulls work. Decisions about *data* are the API's, decisions
about *the site* are the scraper's.

| API / Business (data decisions) | Scraper (site decisions) |
|---|---|
| create the run; at most one in progress, enforced by a filtered unique index on `Runs(Status)` | Playwright, headed/headless |
| explode every active matrix; **dedupe identical requests across matrices** (§8) | politeness delay + jitter, per-item retries |
| lease requests to a worker; expire leases from a dead worker | the circuit-breaker *decision* |
| persist results and flight sets; discover vocabulary | time-zone arithmetic on scraped times |
| advance stages; aggregate; store digests; derive run status; resume semantics | — |

```
POST /runs                      {trigger}              → run; 409 if one is in progress
POST /runs/resume                                      → the incomplete run, or 404
POST /runs/{id}/requests/claim  {worker, leaseSeconds} → next Pending request (priority order), or "none left"
POST /runs/{id}/requests/{r}/result {attempt, rooms[] | error}   → idempotent on attempt
POST /runs/{id}/flight-sets/claim                      → next Pending flight set, or "none left"
POST /runs/{id}/flight-sets/{f}/result {attempt, options[] | error}
POST /runs/{id}/matrices/{m}/complete                  → Downloaded → Aggregated; stores the digest (no send)
POST /runs/{id}/abort           {reason}               → breaker tripped; the rest waits for resume
POST /runs/{id}/finish                                 → derives Success | Partial | Aborted | Failed
POST /runs/{id}/digests/send                           → mails every stored, unsent digest (bf)
```

Scraper loop: `POST /runs` (or `/resume`) → claim → scrape → result … until none left → flight
sets the same way → `complete` each matrix whose work is done → `finish`. A tripped breaker calls
`abort` and exits 3. A crashed worker's leases expire and its requests return to Pending; a late
duplicate `result` is ignored by its `attempt` token.

**Semiweekly job** (Windows Task Scheduler on the NUC, exit-code gated — stop on 7/8, continue past 2):
```
bf-scraper refresh-airports
bf-scraper refresh-resorts
bf-scraper run            # or: bf-scraper run --resume
bf   send-digests
```

Catalog refresh posts a **snapshot** of what the site shows; the API diffs it against the catalog,
applies the add/update/deactivate rules and the safety rule against a bad scrape taking every
resort offline (#131/#219), and audits the changes as the scraper's identity.

### 5.1 Execution pipeline — Current

`sandals-cli run` explodes, executes (`SearchExecutor` → `RoomResultSink`), downloads flights,
aggregates and e-mails, per matrix in priority order, in one process; `run --resume` finishes the
most recent incomplete run; a file lock prevents overlap. Verbs listed in README.

## 6. Flights — Current, with one Target change

- Airfare is bookable only ~6 months out; beyond the horizon room-only prices are stored and airfare
  fills in once in range. `airfarePrice = combined total − room price` per option.
- **FlightSet dedupe** per run by (origin airport, gateway airport, depart date, return date,
  cabin, travellers) — not by resort; resorts on one island share a gateway.
- Full fidelity: FlightOption → FlightDirection ×2 → FlightLeg.
- **v2:** a search item links to **one flight set per departure airport** through
  `SearchItemFlightSet` (today a single FK links only the first airport — #220); flight sets are
  created at explode time as work items and deduped across every matrix in the run.

## 7. Reference data — Current

- **Resort** = permanent identity row (code, name, island, gateway airport FK).
- **ResortSpan** = effective-dated status with `1900-01-01` / `9999-12-31` sentinels; machine-
  maintained from scrape diffs.
- **Vocabularies** (RoomLevel/View/Type/Feature) are **discovered** from results, never authored.
  v2 (deferred): a BLL rule flags values not seen in *n* runs; `bf prune-vocab` or a UI
  action triggers it.
- **RoomCategory**: per-resort code + name, first/last seen; price-over-time identity =
  (resort, check-in date, room category).
- **Airports** carry their IANA time zone (`Airport.TimeZoneId`).

## 8. Data model — Target (ADR 0004)

```
AppUser (Oid GUID PK, DisplayName, Email, LastSeenUtc)          seeded: 0000…0000 "Unassigned"
SearchMatrix (+ OwnerId → AppUser, OwnerDisplayName, Version)   audited
Run (filtered UNIQUE on Status WHERE Status = InProgress)        the run lock
MatrixRun (per matrix per run; Stage Exploded → Downloaded → Aggregated → Emailed)
SearchRequest (Run, Resort, CheckIn, CheckOut, Adults, Children, Infants,    ← the deduped unit of work
               Priority, Status, Tries, LastError, CapturedUtc, LeasedBy, LeaseExpiresUtc, Attempt, Version)
               UNIQUE (Run, Resort, CheckIn, CheckOut, Adults, Children, Infants)
SearchItem (MatrixRun, SearchRequest, Cabin)   UNIQUE (MatrixRun, SearchRequest)   ← one per matrix wanting it
RoomResult (SearchRequest, …)                                     scraped once
SearchItemFlightSet (SearchItem, FlightSet)                       one per departure airport
FlightSet (+ Status, LeasedBy, LeaseExpiresUtc, Attempt, LastError, Version; CapturedUtc nullable)
DigestDelivery (MatrixRun, RecipientUserId → AppUser, RecipientAddress, Subject, HtmlBody, TextBody,
                Status, Attempts, LastError, CreatedUtc, SentUtc)
Resort, Airport (+ Version)
```

- **Dedupe is per run.** Identical `(resort, dates, occupancy)` across matrices is one
  `SearchRequest` with one set of `RoomResult`s; each matrix links through its own `SearchItem`.
  `Cabin` is not in the key — it affects flights, not rooms. Across runs nothing is deduped: prices
  change, which is the point of searching twice a week.
- **Visibility is a join**, not a column: a user sees a result because one of *their* matrices links
  to its request (`RoomResult → SearchRequest → SearchItem → MatrixRun → SearchMatrix.OwnerId`).
  One predicate, applied in the BLL, scopes matrices, items, results, digests and price history.
- **Item status is per request.** If a scrape fails, it fails for every matrix that wanted it, and
  each digest says so.
- **Concurrency:** a portable `Version` column (`long`, concurrency token, incremented by the
  context) on user-edited and claimable rows — SQL Server `rowversion` is not available on the
  SQLite test provider. Conflicts surface as 409 from the API.
- **Identity key:** `OwnerId` is the Entra object id as a `Guid` (`uniqueidentifier`), parsed once
  at the auth boundary; `Guid.Empty` is the unassigned sentinel that `bf-maint
  assign-owner` replaces.
- **Fresh start:** the v1 migrations are squashed into one `InitialCreate` for this model; no data
  is carried from v1.
- **Audit:** `Wolfgang.AuditTrail` audits Configuration and Reference entities as today; the user
  column now carries the signed-in user (UI) or the service principal (scraper/admin). Results and
  Execution tables stay `[NotAudited]`.

### 8.1 Data model — Current

Reference / Configuration / Execution (Run → MatrixRun → SearchItem, the item owning its results and
a single `FlightSetId`) / Results as shipped; `[NotAudited]` on Results and Execution. The v1 model
and its three migrations are superseded by §8.

## 9. Aggregation & e-mail — Target (ADR 0006)

- **Aggregate on `complete`, store, send later.** `complete` builds the per-matrix digest (HTML +
  plain text, unchanged renderer) into a `DigestDelivery` row, status *Pending*, and sends nothing.
  `POST /runs/{id}/digests/send` — the last job step, via `bf` — mails the stored bodies
  and records *Sent* / *Failed* + attempts + error. Everyone's mail goes out together, after the run
  is known-good, and the stored body is exactly what was sent.
- **Recipients:** the owner's sign-in address; Admin-added extras; verified self-service extras are
  phase 2. One message per recipient, never CC/BCC across users; no cross-user content in any
  template; addresses and bodies are never logged above Debug.
- **Sender:** interim — a dedicated mailbox owned by the Admin (not the Admin's personal mailbox),
  credentials in Container Apps secrets / Key Vault; later Azure Communication Services or a
  transactional provider on an owned domain with SPF/DKIM. One `ISmtpClient` implementation either
  way; a file-sink implementation for local runs.
- **Views:** Admin — every delivery and its status (`GET /digests`); user — "my digests", rendered
  from the stored HTML (`GET /digests/mine`). `resend` is an Admin action.
- **Failure notices** go to the Admin only; a user's digest says "3 of 14 searches did not
  complete", never why.
- Content: §1 summary per resort, §2 resolved criteria incl. offline-resort notices, §3 detail
  (cheapest → dearest, with amenities and cheapest flight per departure airport — #247/#220),
  plus a link to the web UI.
- **Mobile-first markup — the digest is read on a phone.** Single column, 600 px max, 16 px body,
  generous tap targets. §3 becomes **one card per room** (price large on the first line; room ·
  level · view · amenities on the second; check-in as a chip) under resort sections whose header
  carries the section's cheapest price, so the summary scans without opening anything. Table-based
  layout with inline CSS (what e-mail clients render), dark-mode-safe colours, a plain-text part
  always, and an "Open in SandalsSearch" deep link per resort and per digest that lands on the
  phone-contract Results screen with matrix and capture date pre-selected. Acceptance: renders in
  Gmail (iOS + Android), Apple Mail, Outlook mobile and Outlook desktop; a fixture test pins the
  markup (replacing the v1 exact-markup test, #247).

### 9.1 — Current

`SmtpDigestSender` sends each matrix's digest synchronously right after aggregation to the
matrix's free-text recipient list; failure notices go to the same list.

## 10. Identity & authorization — Target (ADR 0005)

- **Entra ID**, the tenant the Azure subscription already has; app registration with *assignment
  required*; the dozen users invited as **B2B guests** (Microsoft, Google or one-time-passcode
  sign-in). No sign-up page, no stored passwords.
- **App roles:** `Admin` (everything), `Editor` (create/edit/deactivate own matrices; see own
  results and own matrices' run outcomes), `Viewer` (read own), `Scraper` (service principal:
  snapshots, claims, results; no matrix access). Runs themselves are Admin-only objects.
- The API validates Entra JWTs itself (one gate for UI, scraper, admin); the web host signs users in
  with OIDC and calls the API with the user's token, so the API — and the audit trail — sees the
  person.
- **Database identities, no passwords:** API managed identity → `db_datareader` +
  `db_datawriter`; maintenance → DDL via the operator's Entra user locally or the job's managed
  identity in the pipeline (`bf-maint grant-roles` is idempotent and part of `migrate`).

## 11. Database — Target (ADR 0001)

Azure SQL Database **free offer**, serverless, auto-pause 15 min, exhaustion behaviour *pause until
next month*. The API treats a sleeping database as normal: `Connect Timeout=90`, EF retry on the
resume errors, a `DatabaseReadiness` warm-up the UI can observe ("Warming up. Please wait…" after
2 s; a quota-exhausted screen naming the reset instant), and a schema check that refuses to start
the UI on a pending migration ("Missing migration: … Run `bf-maint migrate`"). SQLite
remains the test provider; the Data layer has no raw SQL and no provider-specific column types.

There is **no manual pause** for a serverless database (portal, CLI or REST); 15 minutes of
inactivity is the only trigger and the floor. The real idle tail is ~20 minutes per wake-up (the
API's pooled connections die when Container Apps scales it to zero, then the 15 minutes run) —
about 6 000 vCore-seconds a month at twice a week, not worth engineering around. What *is* worth
a rule: **no client stays connected** (SSMS Object Explorer, Azure Data Studio, a forgotten
`sqlcmd`) — an open session holds the database awake indefinitely; the maintenance tool connects,
works and disconnects. The Admin dashboard shows the live `status` so an `Online` at an odd hour is
visible.

## 12. UI — Target (ADR 0008)

**Name.** The product name is a display string (placeholder "Barefoot Vacation"); wherever it appears it reads
"<Name> — for Sandals & Beaches resorts" in plain words, with the not-affiliated statement on the sign-in page
and in every page footer (ADR 0009). No logo, type or colour of theirs; the rail mark is a neutral glyph on the
accent, which is the Sandals light blue (`#05b2e8`, lifted to `#2cc3f0` in dark mode for contrast).

**Host and library.** A Blazor Web App (`SandalsSearch.Web`, `InteractiveServer` render mode
globally) built on **Fluent UI Blazor** — accessible by default (#243), a neutral
`FluentDesignTheme` with light/dark (retires the Sandals palette, #244/#261), a data grid that
sorts, pages and virtualises. Charts are rendered as **our own SVG** (the existing `ChartScaler`
logic); no chart dependency. Two containers (API, Web), both scale to zero.

**Components own their state.** The v1 MVVM plumbing (`ObservableObject`, commands,
`ScreenViewModel`, `DataAccessGate`, `ShellViewModel`) retires with WPF; the framework-free logic
survives in a slim UI-logic library — `MatrixDraft` (the editor's load/save mapping, window-mode
rules, picker and chip logic, fan-out wiring, extracted from `MatrixEditorViewModel`),
`ResortPickItem`, `IslandGroup`, `FilterPickItem`, `ChartScaler`, `PriceHistoryRange`, a paging
helper — unit-tested without any UI framework. Screens call the Core interfaces through
`ApiClient` directly.

**Information architecture by role.** An Editor's world is their matrices; an Admin's world is
the system. Everything an Editor sees is reachable from a matrix they own.

| | Editor / Viewer | Admin (adds) |
|---|---|---|
| Landing | **Dashboard** — my matrices at a glance (status, last searched, cheapest, ▼/▲ vs last capture), last digest, next scheduled run, needs-attention (window passed, resort offline); first sign-in shows one action, "Create your first matrix" | **System dashboard** — last run (status, duration, items, failures, seconds per item), **Budget & health** (below) |
| Work | **Matrices** → **Matrix editor** | Owner column + owner filter; "mine" by default, "all owners" toggle; can open any matrix |
| Read | **Results** — matrix + *capture date* pickers (runs are Admin objects), filters-only toggle, paged grid, cheapest flight per departure airport | any owner |
| Read | **Price history** — over *my* captures only (v1) | cross-user view over all captures |
| Read | **My digests** — every digest sent to me, rendered from the stored body, linking to its results | **Deliveries** — every delivery, status, attempts, error, resend |
| Me | **Profile** — name/e-mail (from sign-in, read-only), time zone, default occupancy; later extra recipients | — |
| Admin | — | **Runs** (history + live progress: requests and flight sets done/total, per-matrix stage, abort reasons), **Reference data** (resorts + spans, airports + time zones, vocab with the future prune action), **Users** (role, matrix count, last sign-in; inviting stays in Entra), **System** (effective configuration, read-only) |

- Admin is also an Editor (own matrices). **Viewer** = receives digests and browses results; the
  Admin creates matrices *for* them. Kept for v1 even if unused.
- Runs are invisible to Editors as objects; they see per-matrix outcomes only ("searched Tue
  06:12 — 14 of 14, digest sent"), never run ids, breaker reasons or other matrices.
- Readiness states (warming up / quota exhausted / missing migration) render full-page, before the
  shell, for every role, from the API's readiness endpoint. A run in flight is shown by polling the
  API every ~30 s while a run is `InProgress`.
- Dropped from v1: user-facing Settings (becomes Admin **System**); Run history for non-admins;
  "Refresh now" on Reference data (scraping is `bf-scraper` on the home machine, not the web).
- Future version: a first-run tour / wizard.

**Budget & health (Admin dashboard).** Table-backed cards ship with the dashboard; the Azure
Monitor cards follow once the API's managed identity has *Reader* on the database and
*Monitoring Reader* on the resource group:
database `status` (`Online | Paused`) and `free_amount_remaining` this month with days to reset;
storage used vs 32 GB; API + Web compute vs the Container Apps free grant (Cost Management, lags
8–24 h, shown "as of"); digests sent this month; last run duration / items / failures / seconds per
item; scraper last seen (last `claim`). `Online` at an odd hour means a session is holding the
database awake.

**Price history scoping.** v1: strict for Editors (own captures), cross-user for Admin — the
"no cross-user signal" rule holds for the dozen. **v2 priority:** a shared *market view* for
everyone — **cost per adult per night** (`TotalPrice ÷ Adults ÷ nights`, all derivable today, no
schema change), per resort and room level, bucketed by week — so a new user can see "March is
cheaper than February" on day one. Sandals is adults-only, so the figure is exact until the Beaches
brand (children) arrives; an "adults only" toggle is a Beaches-era item.

**Mobile.** The web app's design target is a laptop/tablet; the **matrix editor is
desktop-first**. The *phone contract* — the screens a digest links to and that must work on a
phone: sign-in, Dashboard (cards), My digests, Results for one matrix (cards instead of the grid,
cheapest flight expandable); a bottom tab bar for those plus Profile; Admin under "More".
Everything else must merely not break.

**Auth UX.** "Sign in with Microsoft / Google" (Entra does the rest); first sign-in creates the
`AppUser` row and lands on the empty Dashboard; sign-out and "who am I" in the header.

## 13. Branches and implementation phases — Target

**Repository and branch model.** v2 lives in a **new repository, `barefoot`**, cut from `repo-template`
(ADR 0009): a clean history (none of v1's history items), the current template by construction, and
the code name in the URL. `main` is the release branch; v2 is built on **`develop`** by *importing*
v1 code layer by layer from `sandals-search`'s read-only **`archive/v1-wpf`** branch rather than by
refactoring it — so `Cli`, `App`, `Wpf`, the legacy HTTP client, the v1 migrations and the v1 solution
file disappear by omission, and `git log` on the archive branch is the history of every imported file.
The v1 repository's 85 open issues get a triage pass at the import: carried here (with a *carried from
`sandals-search#N`* line), or left there as the v1 record; `sandals-search` is archived on GitHub when
the import is complete. `pr.yaml`, `protected-files.yaml`, `codeql.yaml` and `stryker.yaml` run for
PRs into `develop` from the first commit.

**Import order** (stacked PRs on `develop`, each reviewable alone). What is taken from
`archive/v1-wpf` and what is not:

| Taken | Not taken |
|---|---|
| `Core` (minus `Occupancy`, `ISubSessionProvider`, `ObeOptions.BaseUrl`) + tests | `Cli`, `App`, `Wpf` |
| `Obe` + tests, verbatim first | `Data/Obe` (legacy HTTP client), `FileRunLock`, `EnvironmentAuditUserProvider` |
| `Data`: configurations, `DbContext`, `SqliteContextFactory`, pure query services → the DAL | `Data/Migrations` (fresh `InitialCreate`), `MigrationTests`, `CliTestHost` |
| `Data`: orchestrator, editors, digest builder, syncs, sender → `Business`, then reshaped | the 14 "test the fake" tests |
| `Presentation`: framework-free logic (§12) | the view models and MVVM plumbing |
| `docs/API-NOTES.md` (redacted per #261), DESIGN v2, ADRs | — |

1. **Business extracted from Data** — pure move, no behaviour change; the Core interfaces stay.
2. **Data model v2** — `AppUser`, owner, `SearchRequest`, `SearchItem` link, `SearchItemFlightSet`,
   `DigestDelivery`, `Version`, the run-lock index; squashed `InitialCreate`; `Business` adapted
   (dedupe at explode, visibility predicate, stored digests).
3. **`Api` + `ApiClient` + `Contracts`** — the Core interfaces over HTTP; Entra JWT validation;
   `HttpContextAuditUserProvider`; ADR 0001 warm-up; `WebApplicationFactory` integration tests over
   SQLite driving the real `ApiClient` (replaces the CLI as the end-to-end harness). WPF pointed at a
   local API.
4. **`Maintenance`** — `migrate`, `grant-roles`, `assign-owner`, `verify-schema`.
5. **`Scraper`** — pull-based `run`, snapshot `refresh-*`, `search one`; **`Admin`** —
   `send-digests`, `digests status`, `matrix list`, `version`. `Cli` retired.
6. **Azure** — SQL free offer, Container App for the API (scale to zero), secrets, firewall rule;
   Akamai-from-Azure test is *not* needed (the scraper stays home).
7. **`Web`** — Blazor Server, screen by screen; `Wpf` retired at parity.

## 14. Quality bar

Unchanged from v1 and applied as we go: 95 % line coverage on every `src/` assembly and 100 % on
test assemblies; the analyzer suite with warnings-as-errors; CodeQL + DevSkim; Copilot + user
review on every PR; Stryker per test project. New in v2: API **contract tests** (the integration
suite over `WebApplicationFactory` + SQLite) are the end-to-end gate; every client talks to the API
only through `ApiClient`, so the contract is tested once.
