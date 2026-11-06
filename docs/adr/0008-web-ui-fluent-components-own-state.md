# ADR 0008 — Web UI: Blazor Web App on Fluent UI Blazor; components own their state; role-scoped IA

- **Status:** Accepted, 2026-11-06
- **Scope:** `SandalsSearch.Web`, the slimmed `SandalsSearch.Presentation`, the digest template

## Context

The WPF UI must be replaced by a hosted UI for up to a dozen unrelated users on laptops, with
digests read on phones. `Presentation` (2 822 lines) is WPF-shaped MVVM: ~575 lines of plumbing
(`ObservableObject`, commands, `ScreenViewModel`, the single-context `DataAccessGate`, the shell),
~900 lines of thin screen view models, ~260 lines of framework-free display logic, and the
1 033-line `MatrixEditorViewModel` of which roughly 400 lines are behaviour and the rest binding
plumbing. The accessibility (#243), theme (#244) and trademark (#261) findings all point at the
v1 XAML.

## Decision

1. **Blazor Web App, `InteractiveServer` globally**, as its own container beside the API; both
   scale to zero. Co-hosting in one process is a later optimisation if cold starts annoy.
2. **Fluent UI Blazor** as the component library: accessible by default, a neutral
   `FluentDesignTheme` with light/dark, a data grid that sorts, pages and virtualises. Charts are
   our own SVG over the existing `ChartScaler`; no chart dependency. Runner-up kept in reserve:
   Bootstrap + QuickGrid (same `EditForm` code underneath). MudBlazor rejected on look and weight;
   Radzen on vendor shape.
3. **Components own their state.** The MVVM plumbing retires with WPF. The behaviour survives as
   framework-free classes in a slim `Presentation`: `MatrixDraft` (extracted from the editor view
   model), the pickers, `ChartScaler`, `PriceHistoryRange`, a paging helper — unit-tested with no
   UI framework. Screens call the Core interfaces through `ApiClient`. Binding Blazor to the
   existing view models was rejected: setters that fire async loads cascade under two-way binding
   (#237's class of bug), `PropertyChanged` from thread-pool continuations needs per-screen
   marshalling, and the `Faulted`/MessageBox error model has no web equivalent.
4. **Information architecture by role** (DESIGN §12): an Editor's world is their matrices; an
   Admin's is the system. Runs, reference data, deliveries, users and failure details are
   Admin-only; Editors see per-matrix outcomes. Admin is also an Editor ("mine" by default).
   Viewer is kept for v1.
5. **Price history** is strict (own captures) for Editors and cross-user for Admin in v1; the v2
   priority is a shared market view in **cost per adult per night**, per resort and room level,
   weekly buckets — exact while the brand is adults-only.
6. **Mobile:** design target laptop/tablet; the matrix editor is desktop-first; the *phone
   contract* is sign-in, Dashboard, My digests and single-matrix Results. The digest itself is
   mobile-first (one card per room, 600 px single column, inline CSS, plain-text part, deep links
   to the phone-contract screens).
7. **Admin dashboard carries budget and health**: database status and free allowance remaining,
   storage, Container Apps compute vs grant (lagging), digests sent, last-run statistics, scraper
   last seen. Azure Monitor cards follow the managed-identity wiring.
8. **Readiness** (warming up / quota / missing migration) is a full-page state before the shell;
   live run progress is polled every ~30 s.

## Consequences

- `Wpf` is retired when the web reaches parity; until then it keeps the old view models untouched.
- The editor extraction is the single largest UI task; the seven thin screens are a day or two
  each; the plumbing deletion is free. Most of the 182 Presentation tests port to `MatrixDraft`
  and picker tests; the ones that test the gate, commands, `HasInputErrors` and `Faulted` go with
  the plumbing.
- Accessibility and the neutral theme come from the library rather than an audit.
- A first-run tour/wizard is deferred to a future version; the empty Dashboard carries onboarding.
