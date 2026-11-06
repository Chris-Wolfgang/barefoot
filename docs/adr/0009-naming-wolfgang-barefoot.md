# ADR 0009 — Naming: `Wolfgang.Barefoot` is the code name; the product name is a display string

- **Status:** Accepted, 2026-11-06
- **Scope:** namespaces, assemblies, binaries, repository, database, secrets id, Entra app, digest sender

## Context

v1 is named `SandalsSearch` throughout — namespaces, binaries (`sandals-cli`), repository, README
title. "Sandals" and "Beaches" are registered marks; using one as the product name is the classic
nominative-use grey zone and the biggest item in the public-readiness review (#261). v2 is built on
`develop` by importing v1 files one layer at a time, so every file is touched once anyway: a rename
during the import is free, while a rename after the API, the clients and the Entra registration
exist costs a day and a round of broken references.

The product's display name is not settled ("Barefoot Vacation" is the placeholder in the mockup).
The code name must not depend on it.

## Decision

1. **Code name `Barefoot`; root namespace `Wolfgang.Barefoot`**, following the fleet's `Wolfgang.*`
   convention. Projects: `Wolfgang.Barefoot.Core`, `.Contracts`, `.Data`, `.Business`, `.Api`,
   `.ApiClient`, `.Obe`, `.Scraper`, `.Admin`, `.Maintenance`, `.Presentation`, `.Web`.
2. **The site adapter stays `.Obe`** ("online booking engine" — generic). One driver with brand
   arguments is the likely shape for Beaches; `.Sandals` / `.Beaches` adapters only if two become
   necessary. The word *Sandals* therefore appears in code only as data (the brand code) and in
   nominative text; `docs/API-NOTES.md` keeps it as the description of the site.
3. **Binaries `bf`, `bf-scraper`, `bf-maint`.** `bf` is the command-line face of the product (as `gh` and `az`
   are for theirs): the API client with the scriptable verbs, today admin-leaning (`send-digests`, `digests
   status`), later the user verbs too (`bf matrix list`). The other two read as the specialised workers they
   are — `bf-scraper` needs a browser and a home IP, `bf-maint` needs DDL rights. `bf-admin` was rejected as
   describing who we expected to run it rather than what it is; `bf-cli` as saying nothing. One shared
   exit-code table across the three.
4. **Product display name is a string** — UI header, sign-in page, browser title, digest sender
   display name, Entra app registration name — changeable at any time without touching code names.
   Wherever it appears it is followed by the nominative tagline *"for Sandals & Beaches resorts"*,
   never their logo, type or colours, and the sign-in page and every page footer carry the
   not-affiliated statement.
5. **Database `Barefoot`, user-secrets id `Barefoot`.**
6. **Repository rename `sandals-search` → `barefoot`: any time before the first container image is
   pushed** (image names follow the repository). GitHub redirects the old URL for clones, `gh`,
   issues, PRs and links; the rename touches scripts with the hard-coded name, README badges,
   `docfx_project` `_baseUrl`, the sweep ledger and local worktree names (~30 minutes).

## Consequences

- The import PRs (DESIGN §13) rename namespaces as they copy; no separate rename PR.
- Issues and session notes refer to `SandalsSearch` types until the import reaches them; the
  archive branch `archive/v1-wpf` keeps the old names forever.
- #261 shrinks to the documentation items (API-NOTES redaction, disclaimer) — the product and the
  repository no longer *are* the mark.
