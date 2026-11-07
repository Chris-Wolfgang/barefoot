# ADR 0009 — Naming: `Wolfgang.Barefoot` is the code name; the product name is a display string

- **Status:** Accepted, 2026-11-06
- **Scope:** namespaces, assemblies, binaries, repository, database, secrets id, Entra app, digest sender

## Context

"Sandals" and "Beaches" are registered marks; using one as the product or code name is the classic
nominative-use grey zone (the earlier tool was named for the mark throughout, which was the biggest
item in its public-readiness review). A new repository and a from-scratch codebase make this the
cheapest moment to choose a name that is ours; renaming after the API, the clients and the Entra
registration exist costs a day and a round of broken references.

The product's display name is not settled ("Barefoot Vacation" is the placeholder in the mockup).
The code name must not depend on it.

## Decision

1. **Code name `Barefoot`; root namespace `Wolfgang.Barefoot`**, following the fleet's `Wolfgang.*`
   convention. Projects: `Wolfgang.Barefoot.Core`, `.Contracts`, `.Data`, `.Business`, `.Api`,
   `.ApiClient`, `.Obe`, `.Scraper`, `.Admin`, `.Maintenance`, `.Presentation`, `.Web`.
2. **The site adapter is `.Obe`** ("online booking engine" — generic). One driver with brand
   arguments is the likely shape for Beaches; `.Sandals` / `.Beaches` adapters only if two become
   necessary. The word *Sandals* therefore appears in code only as data (the brand code) and in
   nominative text; the redacted `docs/API-NOTES.md` keeps it as the description of the site.
3. **Binaries `bf`, `bf-scraper`, `bf-maint`.** `bf` is the command-line face of the product (as
   `gh` and `az` are for theirs): the API client with the scriptable verbs, admin-leaning at first
   (`send-digests`, `digests status`), later the user verbs too (`bf search list`). The other two
   read as the specialised workers they are — `bf-scraper` needs a browser and a home IP, `bf-maint`
   needs DDL rights. `bf-admin` was rejected as describing who we expected to run it rather than
   what it is; `bf-cli` as saying nothing. One shared exit-code table across the three.
4. **Product display name is a string** — UI header, sign-in page, browser title, digest sender
   display name, Entra app registration name — changeable at any time without touching code names.
   Wherever it appears it is followed by the nominative tagline *"for Sandals & Beaches resorts"*,
   never their logo, type or colours, and the sign-in page and every page footer carry the
   not-affiliated statement; the first-sign-in disclaimer (ADR 0008) says it in full.
5. **Database `Barefoot`, user-secrets id `Barefoot`, repository `barefoot`**; container image
   names follow the repository.

## Not chosen

- *Name the product for the brand it searches.* Trademark exposure and a rename later.
- *A name tied to the display name (e.g. `Wolfgang.BarefootVacation`).* The display name is not
  settled; the code name must survive its change.

## Consequences

- The archive branch of the earlier repository keeps its old names forever; nothing here refers to
  them except as *context* lines on re-filed issues.
- The trademark concern shrinks to documentation items (the API-NOTES redaction, the disclaimer)
  — the product and the repository no longer *are* the mark.
