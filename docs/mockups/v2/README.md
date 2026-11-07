# v2 UI mockup (work in progress, 2026-11-06)

`ui-mockup.html` — a single-page HTML mockup of the v2 web UI (DESIGN §12, ADR 0008), iterated in the browser
during the design session. Open it in a browser; the dark bar at the top is mockup chrome: Light/Dark,
Editor/Admin, Laptop/Phone, App/Sign-in page, Re-show notices.

**Locked (approved by Chris, 2026-11-06):** first-sign-in disclaimer → notices pop-up; Editor Dashboard (status strip, Open issues card, Recent results, movers, latest digest); Admin System dashboard. UI noun is **Search** (code: SearchMatrix); individual scrapes are **lookups**.

Screens so far: **First sign-in disclaimer** (one-time, per user per disclaimer version; must tick to continue; Decline signs out; precedes the notices modal), **Dashboard** (Editor: status strip, notices, my matrices, biggest movers, latest digest;
Admin: System dashboard with budget & health tiles), **Matrices list**, **Matrix editor** (two panes +
fan-out footer), the **notices modal** (per-item acknowledge), the **sign-in page**.
Not yet drawn: Results, My digests, Price history, Profile, Runs, Deliveries, Reference data, Users, System
detail, the startup states, the e-mail.

Open questions noted at the end of the session: whether "Later" should be allowed on notices or every
notice must be acknowledged; whether Admin system notices share the modal; island grouping for resort
selection; filters as a pane vs a tab; whether the fan-out footer shows "already searched by other
matrices" savings. Product name is a placeholder.
