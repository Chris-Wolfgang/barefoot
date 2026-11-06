# Wolfgang.Barefoot

Flexible-date room and flight price search for Sandals(R) & Beaches(R) resorts: define a matrix of
resorts x departure airports x a check-in window x nights once, and get a digest twice a week when
prices move.

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![.NET](https://img.shields.io/badge/.NET-10.0-purple.svg)](https://dotnet.microsoft.com/)
[![GitHub](https://img.shields.io/badge/GitHub-Repository-181717?logo=github)](https://github.com/Chris-Wolfgang/barefoot)

---

## Status

**v2, under construction.** This repository is the hosted successor to
[sandals-search](https://github.com/Chris-Wolfgang/sandals-search) (v1: WPF + CLI on one machine,
archived there as `archive/v1-wpf`). The design is in [docs/DESIGN.md](docs/DESIGN.md) and the
decisions in [docs/adr/](docs/adr/); the code is imported from v1 one layer at a time on `develop`.

## Shape

- **`Wolfgang.Barefoot.Api`** (Azure Container Apps): the only process that touches the database;
  hosts the business logic and the data access; validates Entra ID tokens; audits who changed what.
- **`bf-scraper`** (a home machine): drives the booking site in a real browser and posts results
  to the API.
- **`bf`**: the command-line face of the product; scriptable steps over the API (`send-digests`,
  `matrix list`, ...).
- **`bf-maint`**: schema migrations and repair; the one direct-database client.
- **Web UI** (Blazor, Fluent UI): per-user matrices, results, price history and digests.
- **Database**: Azure SQL Database, free offer, serverless; it sleeps when idle and the app waits for it.

## License

MIT; see [LICENSE](LICENSE).

*This is an independent tool and is not affiliated with or endorsed by Sandals Resorts International.
"Sandals" and "Beaches" are trademarks of their owner, used only to identify the resorts this tool searches.*

---

This repository was generated from [Chris-Wolfgang/repo-template](https://github.com/Chris-Wolfgang/repo-template).
