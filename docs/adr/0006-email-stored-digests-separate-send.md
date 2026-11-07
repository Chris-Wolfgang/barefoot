# ADR 0006 — E-mail: store the digest on completion, send in a separate step, owner-only recipients

- **Status:** Accepted, 2026-11-06
- **Scope:** `Business` (digest building and delivery), `Api` (`/digests`), `Admin` (`send-digests`)

## Context

Each search produces a digest per run, read on a phone by a user who may never open the web UI.
With several unrelated users, a free-text recipient list would make the server an open relay; a
synchronous send from inside the run would stall the scraper's call on a slow SMTP server; and mail
to a dozen strangers' Gmail/Outlook inboxes from a placeholder address lands in spam. The earlier
tool did all three; each is designed out here.

## Decision

1. **Store, then send.** `complete` builds the digest (HTML + plain text) into a `DigestDelivery`
   row (status *Pending*) and sends nothing. `POST /runs/{id}/digests/send` — the last job step,
   from `bf` — mails stored bodies and records *Sent* / *Failed*, attempts and error.
   Everyone's mail goes out together after the run is known-good; the stored body is exactly what
   was sent; moving the send to its own workflow later is a scheduling change (ADR 0007).
2. **Recipients:** the owner's sign-in address, always; Admin-added extras; verified self-service
   extras ("add my spouse": confirmation link, cap, stored consent) in phase 2. Recipients are
   Admin-managed until then.
3. **Protection rules:** one message per recipient, never CC/BCC across users; no cross-user
   content in any template; addresses and bodies never logged above Debug; failure notices
   (abort reasons, run details) to the Admin only — a user's digest says "3 of 14 lookups did not
   complete", never why. A failed delivery raises an Open issue for its recipient (ADR 0008).
4. **Sender:** interim, a **dedicated mailbox** owned by the Admin (not the Admin's personal
   mailbox — a leaked app password must expose digests, not a life), 2FA + app password, credentials
   in Container Apps secrets / Key Vault, TLS. Target: Azure Communication Services Email or a
   transactional provider on an owned domain with SPF/DKIM (DMARC `p=none` to start). One
   `ISmtpClient` implementation either way; a file-sink implementation for local runs.
5. **Views:** Admin sees every delivery and status; a user sees "my digests" rendered from the
   stored HTML, each linking to its results. `resend` is an Admin action.
6. **Markup is mobile-first** (DESIGN §9): one card per room, 600 px single column, inline CSS,
   plain-text part always, deep links to the phone-contract screens; a fixture test pins it.

## Not chosen

- *Send inside `complete`.* Couples the scraper's call to SMTP latency and makes a mail failure
  look like a run failure.
- *Free-text recipient lists per search.* An open relay with extra steps; also the thing that puts a
  stranger's address in another user's search.
- *The Admin's personal mailbox as sender.* Blast radius.

## Consequences

- `DigestDelivery` (~30 KB a row) grows ~150 MB a year at a dozen users × twice a week × a few
  searches; a retention setting (ADR 0011) can come later.
- `MatrixRunStage.Aggregated → Emailed` happens in a different step from `complete`; resume logic
  treats "aggregated, not sent" as a pending delivery, not a pending search.
- `bf send-digests` must be idempotent (it only ever sends *Pending* rows).
