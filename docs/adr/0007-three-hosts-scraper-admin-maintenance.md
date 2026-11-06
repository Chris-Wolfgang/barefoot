# ADR 0007 — Three small hosts: scraper, admin, maintenance

- **Status:** Accepted, 2026-11-06
- **Scope:** `SandalsSearch.Scraper`, `SandalsSearch.Admin`, `SandalsSearch.Maintenance`;
  retirement of `SandalsSearch.Cli` and `SandalsSearch.App`

## Context

`sandals-cli` does three unrelated kinds of work: scraping (`refresh`, `run`, `search one`),
schema (`migrate`) and user administration (`matrix …`, `email`, `vocab list`). It was also the
end-to-end test harness. With an API (ADR 0002) each kind has different dependencies and a
different place it can run: scraping needs Chromium and a residential IP; schema work needs a
direct DDL connection; "API, do X" steps need only HTTPS and a token.

## Decision

| Host | Binary | Depends on | Identity | Phase-1 verbs |
|---|---|---|---|---|
| `Scraper` | `bf-scraper` | `Obe`, `ApiClient`, `Core` | `Scraper` service principal | `refresh-airports`, `refresh-resorts`, `run [--resume]`, `search one` (diagnostic), `version` |
| `Admin` | `bf` | `ApiClient`, `Core` | service principal (`Admin` role) or interactive login | `send-digests [--run]`, `digests status`, `matrix list`, `version`; later `prune-vocab` |
| `Maintenance` | `bf-maint` | `Data`, `Core` | DDL identity | `migrate`, `verify-schema`, `grant-roles`, `assign-owner` |

- Each host has one reason to exist and one dependency set; the project graph forbids the others
  (`Scraper` ⟂ `Data`; `Admin` ⟂ `Obe`, `Data`; `Maintenance` ⟂ `ApiClient`, `Obe`).
- `bf` is the only host whose requirements are met **anywhere the API is reachable**:
  the NUC job step today, a Container Apps cron job or a pipeline step later, with the same binary.
  That is why every "API, do X" step lives there and not in the scraper.
- One executable per host, many verbs, so each step is testable alone; the semiweekly job is a step
  list (`refresh-airports → refresh-resorts → run → send-digests`) gated on a shared exit-code
  table: stop on 7/8 (API or database unavailable), continue past 2 (partial).
- End-to-end testing moves to API contract tests (`WebApplicationFactory` + SQLite +
  `ApiClient`); the admin CLI exists for operations, not for testing.

## Alternatives considered

- *One "agent"/"worker" binary with every verb.* Honest name, but it ships Chromium beside a 20-line
  HTTP call and attracts every future "API, do X" verb by gravity.
- *No admin CLI; UI only.* Loses scriptable steps (`send-digests`, `matrix list`) that a scheduled
  task and a cron job need.

## Consequences

- `sandals-cli` disappears as a binary name (binaries are `bf-*`, ADR 0009); the README, the scheduled task and DESIGN §5 change.
- `SandalsSearch.App`'s composition root splits: database + business registration into `Api`,
  scraping registration into `Scraper`; the not-wired stubs go with `Cli`.
- Exit codes live in one shared table used by all three binaries.
