# ADR 0007 — Three small hosts: scraper, admin, maintenance

- **Status:** Accepted, 2026-11-06
- **Scope:** `Wolfgang.Barefoot.Scraper`, `.Admin`, `.Maintenance`

## Context

Three unrelated kinds of unattended work exist: scraping (`refresh-*`, `run`, a diagnostic single
lookup), schema work (`migrate`), and "API, do X" steps (`send-digests`, status queries). Each has
different dependencies and a different place it can run: scraping needs Chromium and a residential
IP; schema work needs a direct DDL connection; API steps need only HTTPS and a token. One binary
with every verb would ship Chromium beside a 20-line HTTP call and attract every future verb by
gravity (the earlier tool's single CLI did exactly that).

## Decision

| Host | Binary | Depends on | Identity | Phase-1 verbs |
|---|---|---|---|---|
| `Scraper` | `bf-scraper` | `Obe`, `ApiClient`, `Core` | `Scraper` service principal | `refresh-airports`, `refresh-resorts`, `run [--resume]`, `search one` (diagnostic), `version` |
| `Admin` | `bf` | `ApiClient`, `Core` | service principal (`Admin` role) or interactive login | `send-digests [--run]`, `digests status`, `search list`, `version`; later `prune-vocab` |
| `Maintenance` | `bf-maint` | `Data`, `Core` | DDL identity | `migrate [--to] [--status] [--script]`, `verify-schema`, `grant-roles`, `assign-owner` |

- Each host has one reason to exist and one dependency set; the project graph and the architecture
  tests forbid the others (`Scraper` ⟂ `Data`; `Admin` ⟂ `Obe`, `Data`; `Maintenance` ⟂
  `ApiClient`, `Obe`).
- `bf` is the only host whose requirements are met **anywhere the API is reachable**: the home
  machine's job step now, a Container Apps cron job or a pipeline step later, with the same binary.
  That is why every "API, do X" step lives there and not in the scraper.
- One executable per host, many verbs, so each step is testable alone; the semiweekly job is a step
  list (`refresh-airports → refresh-resorts → run → send-digests`) gated on a shared exit-code
  table: stop on 7/8 (API or database unavailable), continue past 2 (partial), 3 = breaker tripped,
  9 = quota exhausted.
- End-to-end testing is API contract tests (`WebApplicationFactory` + SQLite + `ApiClient`); the
  admin CLI exists for operations, not for testing.

## Not chosen

- *One "agent"/"worker" binary with every verb.* Honest name, wrong dependency set.
- *No admin CLI; UI only.* Loses scriptable steps (`send-digests`, `search list`) that a scheduled
  task and a cron job need.
- *Scheduling inside the API (a hosted service).* The scraper must run on a residential IP, and an
  API that scales to zero cannot host a timer.

## Consequences

- Binaries are `bf-*` (ADR 0009); the README and the scheduled task use those names.
- Exit codes live in one shared table used by all three binaries.
- Failures of the scheduled job are visible: the task's history reads the exit code, and a
  `report-failure`-style step opens one issue per failure signature.
