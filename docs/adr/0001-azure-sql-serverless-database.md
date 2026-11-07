# ADR 0001 — The database is Azure SQL Database (free offer, serverless) and the app tolerates it sleeping

- **Status:** Accepted, 2026-11-06 (amended 2026-11-06: readiness split, migrate-tool rules)
- **Scope:** `Data`, the API host (ADR 0002), the maintenance tool, every UI host

## Context

One database is shared by a scraper on a home machine and an API in Azure that serves a dozen
users who look at the site a few times a week. The workload is tiny and bursty: two runs a week of
roughly ten minutes each, plus occasional interactive use. The budget is zero.

The database is the **Azure SQL Database free offer**: General Purpose, serverless, auto-pause
enabled. Numbers from Microsoft Learn on 2026-11-06 (the free-offer FAQ, the serverless auto-pause
and billing pages):

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

With a 15-minute auto-pause delay, a **semiweekly** run costs roughly `10 min awake + 15 min idle
tail ≈ 1 000 vCore-seconds`, under 10 000 a month, leaving well over 25 hours of interactive use
inside the allowance. *Distinct wake-ups* are the resource to protect: every wake-up costs at least
a 15-minute idle tail.

SQLite is the test provider. The Data layer contains no raw SQL and no SQL Server-only column types
outside the migration, so nothing in this decision widens the provider gap.

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
| `WarmUpGraceSeconds` | 2 | how long a UI waits before showing "Warming up" |

The SQLite path never sees these options. Every Data service inherits the execution strategy from
the context; no explicit transactions are wrapped by hand.

### 2. Warm-up is an observable state the UI can show, not a silent retry

`DatabaseReadiness` (singleton in `Data`, surfaced by the API's readiness endpoint) owns the first
contact with the database and re-probes after any connection loss:

```
Unknown ──probe──► Waking ──connect ok──► Ready
                     │
                     ├──quota error (§3)──► QuotaExhausted   (terminal until ResetsAtUtc)
                     ├──schema behind (§4)► MissingMigration (until bf-maint migrate runs)
                     └──budget spent──────► Unavailable      (re-probed on a timer; LastError set)
```

It exposes `State`, `Elapsed`, `Attempts`, `LastError`, `ResetsAtUtc`, a `Changed` event and
`WaitUntilReadyAsync(CancellationToken)`. The budget is the same one as §1 (connect timeout ×
retries), computed from `TimeProvider` so tests drive it with a fake clock. Every terminal failure
logs the raw SQL error number and message at Warning, so the first real quota pause tells us
whether 42119 is right.

**The API always starts.** Readiness gates the *data* endpoints (they answer `503` with a typed
error code, ADR 0012) and `GET /readiness` is always answerable. The web host and `bf` call
`WaitUntilReadyAsync` through the API before anything else; if `Ready` has not arrived within
`WarmUpGraceSeconds` the UI shows **"Warming up. Please wait…"** with the elapsed time, dismisses on
`Ready`, and on `Unavailable` shows the error with Retry. A headless host logs the transitions and
the elapsed time. The scraper applies the same retry posture to the API.

### 3. The quota-exhausted screen

When a connect failure matches `FreeQuotaErrorNumbers` or `FreeQuotaMessagePattern`, the state is
`QuotaExhausted` and `ResetsAtUtc` is **00:00 UTC on the first day of the next calendar month**,
computed from `TimeProvider`. The hosts show:

> Monthly free database allowance used up. The database is paused until **1 December 2026 00:00 UTC**
> (7:00 PM 30 November, local time). Retry

A headless host prints the same sentence and exits with a distinct code so a scheduled task's
history reads "quota", not "failed". A later enhancement (once managed identity has *Reader* on the
database) reads the Azure Monitor *Free amount remaining* metric to show a gauge before the wall is
hit; it is out of scope here.

### 4. `bf-maint migrate` owns the schema; nothing else migrates

- `bf-maint migrate` (ADR 0007) applies migrations with the DDL identity, from the operator's
  machine or the release pipeline, after warm-up so a migration never starts against a sleeping
  database. The API's managed identity is `db_datareader` + `db_datawriter` only (ADR 0005).
- The tool is **declarative**: you name a target (`--to <id|name|prefix|0>`, default latest) and it
  states the direction it took; `--status` reports current vs expected; `--script [--from] [--to]
  --output` renders SQL offline. Exit codes: `0` ok, `1` a migration failed, `2` usage or
  configuration, `3` the downgrade would lose data and `--confirm-data-loss` was not given, `4`
  refused. A downgrade is screened for data loss (dropped tables/columns, narrowing column changes,
  raw SQL) and the script carries a `-- Downgrade` header naming each step.
- **The database must exist**; the tool never creates one (a typo must not create a stray
  database). **A schema newer than the build is never touched** (exit `4`), and that is the one
  condition on which the API refuses to start — it cannot know what the newer columns mean.
- **A schema behind the build is a readiness state**, not a crash: the API reports
  `MissingMigration` with the pending names ("Missing migration: `<names>`. Run `bf-maint
  migrate`.") and the UI renders it full-page. `GET /system/schema` answers `{current, expected,
  upToDate}` from the same code path the tool uses.
- Every migration has a working `Down`; CI applies up → down → up against SQL Server. Backup restore
  is the primary rollback; `Down` is for the pipeline and for development.

### 5. Azure-side settings that are part of this decision

- Auto-pause delay **15 minutes**; free-limit exhaustion behaviour **auto-pause until next month**
  (never surprise billing — once switched to paid for a month it cannot be switched back).
- `Authentication=Active Directory Default` on the connection string: no SQL password anywhere.
- Firewall: the scraper host's public IP is allow-listed; home IPs change, so the scheduled task may
  run `az sql server firewall-rule update` first, or the operator re-adds the rule when a run fails.
- Nothing may hold a connection open between runs (SSMS Object Explorer, Azure Data Studio or a
  forgotten `sqlcmd` keeps the database awake and burns the allowance). There is **no manual
  pause** for a serverless database; the 15-minute inactivity delay is the only trigger, so session
  hygiene is the whole lever. The Admin dashboard shows the live `status` and
  `free_amount_remaining` so an `Online` at an odd hour is visible (ADR 0008).

## Not chosen

- *SQL Server on the home machine.* Ties the API to a residential IP and a machine that sleeps.
- *Azure SQL provisioned / Basic tier.* ~$5 a month for a workload that fits the free offer.
- *PostgreSQL flexible server free trial / SQLite on a volume.* Time-limited or single-writer.
- *Refuse to start when the database is unreachable* (a common pattern). Under auto-pause the API
  would crash-loop on every wake-up; readiness as data is the right shape.

## Consequences

- Every host has a mandatory first step: readiness through the API; only `bf-maint` touches the
  database directly, and only after warm-up.
- **Latency changes the cost of chatty code.** Each round-trip is tens of milliseconds instead of
  sub-millisecond; the run lifecycle (ADR 0003) is shaped for ~2 calls per lookup, and the DAL
  writes a lookup's results in one `SaveChanges`.
- **Tests:** the readiness state machine is unit-tested with a fake connector and a fake clock
  (grace period, retry budget, quota detection, reset-date arithmetic across a year boundary); the
  SQLite composition path is unchanged; the migrate tool runs in-process with injected writers.
- **Known uncertainty:** the quota error number. It is configuration with a logged fallback, so a
  wrong guess costs one config change, not a release.
