# v2 UI mockup (work in progress, 2026-11-06)

`ui-mockup.html` — a single-page HTML mockup of the v2 web UI (DESIGN §12, ADR 0008), iterated in the browser
during the design session. Open it in a browser; the dark bar at the top is mockup chrome: Light/Dark,
Editor/Admin, Laptop/Phone, App/Sign-in page, Re-show notices.

**Locked (approved by Chris, 2026-11-06):** first-sign-in disclaimer → notices pop-up; Editor Dashboard (status strip, Open issues card, Recent results, movers, latest digest); Admin System dashboard. UI noun is **Search** (code: SearchMatrix); individual scrapes are **lookups**.

**Locked 2026-11-06 (editor):** **Search editor** — left pane *What to search*: Name (unique per owner, inline "you already have" hint), **Brand** (Sandals · adults only / Beaches · families; Children 2–15 and Infants appear only for Beaches), **Check-in window** as *Recurring every year* (month + day pickers, resolves-to hint) or *One-time dates* (date pickers styled like every other input; deactivates itself when the last check-in passes), Adults, **Resorts** grouped by island with the gateway code, then the **flight section** — *Include flights* toggle, Cabin, Departure airports (type-ahead on focus + chips) — which disappears entirely when the toggle is off. Right pane: *Digest filters* (room level / view / type / must-have vocabularies, a value not seen recently shows "not seen in N runs" *under* its name; then **Max total price**, **Max drive to resort** (minutes, from the resort's distance to its gateway, so it applies with or without flights), **Max flight time** incl. layovers + **Max stops** (flights only)), then **Run order** as its own section (Low / Below normal / Normal / Above normal / High ↔ 1 / 25 / 50 / 75 / 99; equal run orders run one after the other), then **Recipients**. Sticky fan-out footer: resorts × check-in dates → room lookups · airports → flight lookups (hidden with flights off) · minutes per run · needs-attention line; Cancel + Save live only in the footer, Deactivate in the header.

**Locked 2026-11-06 (results):** **Results** — one search, one run. Pickers stacked down the left: Owner (Admin only), **Search name**, **Run** (capture date + outcome, "latest" marked; runs are never shown as objects), **Find**; then a scope toggle *Matching my digest filters (n)* / *Everything found (n)* and Sort (default **cheapest total**). Summary strip: matches of found, cheapest room, cheapest incl. flights (where), delta vs the previous run, offline-resort warning. Grid, one row per room option: resort + check-in + nights; room name with level pill and view; **Room price** = regular struck, sale, promo wrapped beneath in green; **Flights** = one button per departure airport showing cheapest–dearest fare and option count; **Total** = room + cheapest flight, with "▼ $486 vs Sat" / "new this run" / "— unchanged" under it. Clicking an airport opens that airport alone, full row width: the cheapest round trip with legs, then *"N more from PHL"* which loads the next **ten** cheapest options per click ("25 more · showing 11 of 36") until "All 36 shown". Offline resorts appear as a row ("No rooms looked up this run · resort offline"). Phone: cards (price/promo hidden), one airport panel at a time.

**Locked 2026-11-06 (later):** **Searches list** — name (unique per owner) + generated subtitle; window on two lines with a recurring icon (tooltip "Repeats every year") or an amber *passed* tag; resorts by short name with airports (codes) on their own line, the column never breaks a pill; last run as date + outcome pill (*Complete / In progress / Incomplete / Failed*); cheapest; vs last; status; icon actions ✎ ⏸ ▶ 🗑 with tooltips. No priority column (it lives in the editor as *Run order*).

Screens so far: **First sign-in disclaimer** (one-time, per user per disclaimer version; must tick to continue; Decline signs out; precedes the notices modal), **Dashboard** (Editor: status strip, notices, Open issues, Recent results, biggest movers, latest digest;
Admin: System dashboard with budget & health tiles), **Searches list**, **Search editor** (two panes +
fan-out footer), the **notices modal** (per-item acknowledge), the **sign-in page**.
Not yet drawn: My digests, Price history, Profile, Runs, Deliveries, Reference data, Users, System
detail, the startup states, the e-mail.

Open questions: whether Admin system notices share the modal; whether the fan-out footer shows
"already looked up by other searches" savings. Product name is a placeholder. (Settled while locking
the editor: resorts stay grouped by island; filters stay a pane, not a tab; "Later" is not offered —
acknowledging only stops the pop-up, the issue stays on the Dashboard.)

## Decisions taken while iterating

- Search **name is unique per owner** (case-insensitive, trimmed): unique index on `(OwnerId, NormalizedName)`, inline validation in the editor, 409 from the API on a race. Other users may reuse the name.
- **Open issues** stay on the Dashboard until their `ResolvesWhen` condition clears; acknowledging only stops the pop-up (per user). The digest's needs-attention line comes from the same list.
- First sign-in **disclaimer** is accepted once per user per disclaimer version (`AppUser.DisclaimerVersionAccepted`).
- **Brand** is a field on the search (Sandals / Beaches). Sandals is adults-only, so Children and Infants are shown and stored only for Beaches; the Beaches option is present now so the shape is right when those resorts become searchable.
- **Window modes**: recurring = month + day (year resolved at run time, wrap-around allowed); one-time = first/last check-in dates, the search deactivates itself and raises an Open issue when the last date passes.
- **Run order** is stored as 1–99 but edited as five words (Low 1 · Below normal 25 · Normal 50 · Above normal 75 · High 99); searches with the same value run in creation order.
- **Flights are optional per search**: with the toggle off no flight lookups are made, and Cabin, airports, Max flight time and Max stops are neither shown nor stored.
- **Max drive to resort** is a digest filter derived from `Resort` ↔ gateway distance (reference data), so it works for no-flight searches too. **Max flight time** includes layovers and is per direction.
- Vocabulary values that have stopped appearing are greyed with "not seen in N runs" under the name — never deleted (Admin prune is a later feature).
- **Results compare** "same resort + room + check-in" across runs for the vs-previous delta; rows absent last run read "new this run".
- **Flight options are paged per flight set**: the grid carries only range + count per airport, the expanded panel the cheapest with legs, and a paged endpoint (10 per page, price ascending) feeds "N more". The digest keeps carrying only the cheapest per airport.
- Resort pickers show the full name; the short name (`Resort.ShortName`) is for the Searches list and digests.

