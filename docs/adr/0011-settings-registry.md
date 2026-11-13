# ADR 0011 — Settings: a typed registry stored in the database; `appsettings` holds bootstrap keys only

- **Status:** Proposed, 2026-11-06 (recommendation; decide before phase 3)
- **Scope:** `Core` (`SettingKey<T>`), `Business` (`ISettings`, registry), `Data` (`Setting` table),
  `Api` (`/settings`), `Web` (Admin → System), `appsettings.*`

## Context

Barefoot has a growing set of operational knobs — lease length, politeness delay, minimum lead
days, max active searches per user, digest retention, run-order defaults, disclaimer version,
session lifetimes, CORS allow-list, quota error numbers — that an Admin should be able to change
without a redeploy, that must be validated, audited and visible, and that every host must read the
same way. Scattering them across `appsettings.json`, environment variables and container secrets
makes "what is the effective value right now" a question nobody can answer from the UI.

`Chris-Wolfgang/wms` solved this with a typed setting registry (its `docs/SETTINGS.md`): the shape
below is that design with the multi-site cascade removed.

## Decision (recommended)

1. **A setting is a typed key**: `SettingKey<T>` declared as a `static readonly` field in `Core`
   with a lower-case dotted name (`run.lease_seconds`), a default, a one-line description, an
   optional `Validator`, and `RequiresRestart`. Names are validated at construction; there are no
   string overloads anywhere.
2. **One `SettingRegistry`** collects every key at startup and rejects duplicates; `GET /settings`
   returns the registry (key, type, default, description, current value, updated-by/at) for the
   Admin **System** page.
3. **Values are stored as invariant text** in one `Setting` table (`Key` PK, `Value`, `UpdatedUtc`,
   `Version`) through a `SettingCodec<T>` per type (bool, int, long, double, string, `TimeSpan`,
   `DateTimeOffset`, string lists). Missing row = default. The table is audited by
   `Wolfgang.AuditTrail`, so every change names the Admin who made it.
4. **Read only through `ISettings`** (`Get(SettingKey<T>)`), injected everywhere in `Business` and
   `Api`. A per-instance cache is invalidated by a cheap generation probe (`max(UpdatedUtc)` +
   count — no global row version, ADR 0004); the scraper and `bf` read settings through the API.
5. **Write through `PUT /settings/{key}`** with `If-Match` on the setting's `Version` (ADR 0012);
   the validator runs server-side; `RequiresRestart` keys show a banner on the System page until the
   API restarts.
6. **`appsettings` holds bootstrap keys only**: the connection string and database options (ADR 0001
   §1), Entra tenant/client ids, the hosting flags (`BehindProxy`), logging. An unknown key under
   the application's section produces a startup warning so a setting cannot hide there by mistake.
7. **Scope is global.** No per-site, per-zone or per-user cascade; per-user preferences (time zone,
   default occupancy) live on `AppUser`, not in the registry.
8. **Initial registry** (defaults in parentheses): `run.lease_seconds` (300), `run.min_lead_days`
   (4), `run.politeness_delay_ms` (2500) and `run.politeness_jitter_ms` (750), `run.max_active_searches_per_user`
   (5), `digest.retention_days` (365), `digest.sender_display_name` (the product name),
   `vocab.stale_after_runs` (6), `ui.disclaimer_version` (1), `auth.session_absolute_hours` (8),
   `auth.session_idle_minutes` (30), `auth.cors_origins` (empty), `database.free_quota_error_numbers`
   ([42119]), `results.flight_options_page_size` (10).

## Not chosen

- *`IOptions<T>` from `appsettings` / environment only.* Every change is a redeploy; no audit; no
  UI; the effective value is invisible.
- *A settings cascade (org → site → zone).* Right for a multi-tenant warehouse product; barefoot is
  one installation.
- *Azure App Configuration.* Another service, another identity, for a dozen keys.

## Consequences

- The Admin **System** page (DESIGN §12) is read/write rather than read-only.
- Tests inject an in-memory `ISettings`; no setting is read at type-initialisation time.
- A setting's name is part of the API surface; renaming one is a migration that copies the row.
