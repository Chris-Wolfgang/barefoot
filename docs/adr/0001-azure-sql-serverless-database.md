# ADR 0001 — The database is Azure SQL Database (free offer, serverless) and the app tolerates it sleeping

- **Status:** Accepted, 2026-11-06
- **Scope:** `SandalsSearch.Data`, the API host (ADR 0002), the maintenance tool, every UI host
- **Supersedes:** the "SQL Server Express on the same machine" assumption in DESIGN §2/§8

## Context

The application is moving from one machine (WPF + CLI + SQL Express) to two hosts that share one database: a
scraper host on a home machine and, later, a web host in Azure. The database moves first, on its own, so the
existing WPF and CLI hosts prove the remote-database behaviour before any new UI is written.

The database is the **Azure SQL Database free offer**: General Purpose, serverless, auto-pause enabled.
Numbers from Microsoft Learn on 2026-11-06 (the free-offer FAQ, the serverless auto-pause and billing pages):

| Fact | Value |
|---|---|
| Free allowance per database per month | 100 000 vCore-seconds, 32 GB data, 32 GB backup |
| Free databases per subscription | up to 10 |
| Auto-pause delay | minimum 15 min, default 60 min |
| Billing while awake | `max(min vCores, used vCores, min memory ÷ 3, used memory ÷ 3)` per second; at the 0.5-vCore minimum the memory floor makes an idle awake hour ≈ 2 400 vCore-seconds |
| Billing while paused | zero compute; storage only |
| Resume | the first connection returns SQL error **40613** and starts the resume; "generally less than one minute"; Microsoft recommends a connect timeout of at least 60 s and a retry |
| Quota exhausted (behaviour chosen: *auto-pause until next month*) | the database is inaccessible until the first day of the next calendar month; reported by the community as SQL error **42119** with a "monthly free allowance" message — **not documented**, so the app treats the number as configuration |
| SLA | none |

With a 15-minute auto-pause delay a nightly scrape costs roughly `10 min awake + 15 min idle tail ≈ 36 000
vCore-seconds a month`, which leaves about 25 hours of interactive use per month inside the free allowance.
*Distinct wake-ups per day* are the resource to protect: every wake-up costs at least a 15-minute idle tail.

SQLite stays the test provider (`configureDbContext` in `ServiceRegistration`). The Data layer contains no raw SQL
and no SQL Server-only column types outside `Migrations/`, so nothing in this decision widens the provider gap.

## Decision

### 1. Resilience lives in the provider configuration, driven by a `Database` options section

`UseSqlServer(connectionString, sql => sql.EnableRetryOnFailure(MaxRetryCount, MaxRetryDelay, [4060]).CommandTimeout(CommandTimeoutSeconds))`,
with `Connect Timeout` written into the connection string by code. Defaults:

| Option | Default | Why |
|---|---|---|
| `ConnectTimeoutSeconds` | 90 | resume can take 60 s; Microsoft says ≥ 60 |
| `CommandTimeoutSeconds` | 60 | the free tier is slow on first touch after resume |
| `MaxRetryCount` | 8 | with a 20 s cap ≈ 2 min of transient-error retries |
| `MaxRetryDelaySeconds` | 20 | |
| `AdditionalTransientErrorNumbers` | `[4060]` | "cannot open database" appears on some resume paths; EF already knows 40613/40197/40501/10928/10929 and timeouts |
| `FreeQuotaErrorNumbers` | `[42119]` | community-reported, unverified; see §3 |
| `FreeQuotaMessagePattern` | `monthly free allowance\|free limit` | second signal for §3 |
| `WarmUpGraceSeconds` | 2 | how long a host waits before showing "Warming up" |

The SQLite path never sees these options. No Data service changes: all 11 of them inherit the execution strategy
from the context, and there are no explicit transactions to wrap.

### 2. Warm-up is an observable state the UI can show, not a silent retry

`DatabaseReadiness` (singleton in `SandalsSearch.Data`) owns the first contact with the database:

```
Unknown ──WaitUntilReadyAsync──► Waking ──connect ok──► Ready
                                   │
                                   ├──quota error (§3)──► QuotaExhausted   (terminal; ResetsAtUtc set)
                                   └──budget spent──────► Unavailable      (terminal; LastError set)
```

It exposes `State`, `Elapsed`, `Attempts`, `LastError`, `ResetsAtUtc`, a `Changed` event and
`WaitUntilReadyAsync(CancellationToken)`. The budget is the same one as §1 (connect timeout × retries), computed
from `TimeProvider` so tests drive it with a fake clock. Every terminal failure logs the raw SQL error number and
message at Warning, so the first real quota pause tells us whether 42119 is right.

Host rule (identical for the API, the scraper against the API, and the UI hosts): call `WaitUntilReadyAsync` before anything else. If
`Ready` has not arrived within `WarmUpGraceSeconds`, show **"Warming up. Please wait…"** with the elapsed time;
dismiss on `Ready`; on `Unavailable` show the error with Retry / Exit. A headless host logs the state transitions and the
elapsed time instead of showing a window.

### 3. The quota-exhausted screen

When a connect failure matches `FreeQuotaErrorNumbers` or `FreeQuotaMessagePattern`, the state is
`QuotaExhausted` and `ResetsAtUtc` is **00:00 UTC on the first day of the next calendar month**, computed from
`TimeProvider`. The hosts show:

> Monthly free database allowance used up. The database is paused until **1 December 2026 00:00 UTC**
> (7:00 PM 30 November, local time). Retry · Exit

A headless host prints the same sentence and exits with a distinct code so a scheduled task's history reads
"quota", not "failed". A later enhancement (web phase, where managed identity is free) reads the Azure Monitor
*Free amount remaining* metric to show a gauge before the wall is hit; it is out of scope here.

### 4. `bf-maint migrate` owns the schema; the API and UI hosts refuse to start on a pending migration

- `bf-maint migrate` (ADR 0007) runs `MigrateAsync` after warm-up, with the DDL identity, from the
  operator's machine or the release pipeline. No other process migrates.
- The API and the UI hosts **never migrate**. After warm-up they call `GetPendingMigrationsAsync()`; if any are
  pending they stop with **"Missing migration: `<names>`. Run `bf-maint migrate`."** The API's managed
  identity is `db_datareader` + `db_datawriter` only — no DDL rights in the cloud (ADR 0005).
- Warm-up always precedes migration so a migration never starts against a sleeping database.

### 5. Azure-side settings that are part of this decision

- Auto-pause delay **15 minutes**; free-limit exhaustion behaviour **auto-pause until next month** (never
  surprise billing — once switched to paid for a month it cannot be switched back).
- `Authentication=Active Directory Default` on the connection string: no SQL password anywhere.
- Firewall: the scraper host's public IP is allow-listed; home IPs change, so the scheduled task may run
  `az sql server firewall-rule update` first, or the operator re-adds the rule when a run fails.
- Nothing may hold a connection open between runs (SSMS Object Explorer, Azure Data Studio or a forgotten `sqlcmd`
  keeps the database awake and burns the allowance). There is **no manual pause** for a serverless database; the
  15-minute inactivity delay is the only trigger, so session hygiene is the whole lever. The Admin dashboard shows
  the live `status` and `free_amount_remaining` so an `Online` at an odd hour is visible (ADR 0008).

## Consequences

- **Every host gets a mandatory first step** (warm-up, then the migration check); `#238` (synchronous
  `Migrate()` on the WPF UI thread) is fixed by this design rather than worked around.
- **Latency changes the cost of chatty code.** Each round-trip is tens of milliseconds instead of sub-millisecond.
  Nightly volume is fine; the two `SaveChangesAsync` per room result and the per-row sub-query in `ResultsService`
  (#249) stop being cosmetic.
- **`FileRunLock` stays machine-local.** Exactly one machine runs the scraper. If a second ever does, the lock
  must move to the database; noted, not built.
- **Tests:** the readiness state machine is unit-tested with a fake connector and a fake clock (grace period,
  retry budget, quota detection, reset-date arithmetic across a year boundary); the SQLite composition path is
  unchanged; the host rule is tested on the CLI (log lines, exit codes) and on the WPF/web view-models.
- **v2 note (ADR 0002):** the API is the process that holds the connection, so the warm-up runs there; the UI
  hosts observe it through the API's readiness endpoint, and the scraper applies the same retry posture to the API.
- **Known uncertainty:** the quota error number. It is configuration with a logged fallback, so a wrong guess
  costs one config change, not a release.
