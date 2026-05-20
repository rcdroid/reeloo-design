# Design request: Test-id migration plan + theme reconciliation + signal SDK hooks

**Status:** open
**Filed:** 2026-05-20
**Author:** orchestrator (audit pass)
**Context refs:**
- `v2/05` (test-id contract — defines the format, not the migration plan)
- `v2/06` §6.2 (theme propagation, locked to 3 themes)
- `v3-gap-audit/cross-cutting.md` §5 (signal-hook inventory), §8 (existing testid precedent), §9 (theme reconciliation)
- Source code: ~200 existing data-testid attributes across the 4 source apps

## What we asked for in v2 (and what we got)

The v2 brief proposed `<surface>:<component>:<element>[:<modifier>]` colon-format testids (§10) and locked themes to `vanilla / cobalt-dark / high-contrast` (§6.2). Claude design produced testids in the proposed format inside the impl mockups — but did not produce:
- A migration plan for the ~200 existing testids in source apps (each in different flat-kebab-case formats)
- A reconciliation map for the 9 readalong themes + 4 learn-stations themes vs the locked 3
- A signal-hook inventory documenting which event each interactive element emits

## What's actually in the source apps

### Test-id conventions (5 different)

| App | Convention | Example |
|---|---|---|
| podcast | `kt-*` flat | `kt-play-pause`, `kt-tweaks-toggle` |
| book | flat-kebab | `deck-card`, `drop-zone`, `search-result` |
| learn-stations | station-prefixed flat | `cloze-feedback`, `echo-prompt`, `cards-flip` |
| readalong | flat-kebab | `drill-card`, `persona-card`, `card-frame` |
| reeloo-v2 impl | NEW: `surface:component:element` | `podcast-catalog:media-card:karotten` |

### Themes (4 + 9 = 13 → locked to 3)

| Source | Themes |
|---|---|
| podcast | hard-coded cobalt for catalog; player toggles cream/night |
| book | `prefers-color-scheme` only (no explicit toggle) |
| learn-stations | `vanilla / pip-duo / solarized-dark / high-contrast` (data-theme attr) |
| readalong | 9 themes: default-light, ocean-light, forest-light, lavender-light, default-dark, midnight-dark, emerald-dark, amethyst-dark, rose-dark |
| v2 locked | `vanilla / cobalt-dark / high-contrast` |

### Signal emissions (today)

| App | Emits to | What |
|---|---|---|
| podcast | nothing | (no SDK) |
| book | nothing | (no SDK) |
| learn-stations | `POST /api/telemetry/learning` → JSONL | `learning.station_attempt`, `learning.station_pass`, etc. |
| readalong | `firestore-analytics.ts` → Firestore | `deck.shown`, `card.rated`, `drill.attempt`, `mastery.update`, etc. |

The v2 brief proposes `signals.reeloo.ai/api/events` as the single sink. Migration plan needed.

## What we need from Claude design

### Part A — Test-id migration plan

1. **Generator helper spec** — `getTestId(surface, component, element, modifier?)` returns the colon-formatted string. Used inside every component. Should be:
   - Type-safe (TypeScript template literal types)
   - Lint-enforced (ESLint rule: every `data-testid` value goes through the helper)
   - Documented in component-gallery alongside the component

2. **Migration map for ~200 existing testids** — a table mapping every existing flat-kebab name to the new colon format. Examples:
   - `kt-play-pause` → `podcast-player:media-player-bar:play-button`
   - `deck-card` → `book-deck-reader:deck-card:root`
   - `cloze-feedback` → `practice-station:cloze:feedback`
   - `drill-card` → `readalong-drill:drill-card:root`

   Design's deliverable: a CSV (or JSON map) with `{old, new, surface, component, element}` columns covering all existing testids. The impl team uses this to mechanically migrate.

3. **Per-component test-id checklist** — for each of the 24+ components in the v2 brief, document the canonical set of testids. Example for `<DictionaryCard>`:
   - `<surface>:dictionary-card:root`
   - `<surface>:dictionary-card:play-button`
   - `<surface>:dictionary-card:save-button`
   - `<surface>:dictionary-card:close-button`
   - `<surface>:dictionary-card:sense:def-0` (parameterized)
   - `<surface>:dictionary-card:sense:example-0:audio` (nested)

4. **Anonymous-element ban enforcement** — per v2 brief §10, "anonymous interactive elements are designed-out." Design contributes a CI lint rule that rejects PRs with `<button>` / `<a>` / clickable `<div>` without `data-testid`. The rule must allow well-known exceptions (e.g. `<button type="submit" inside <form>` may inherit form testid).

### Part B — Theme reconciliation

5. **Decision: drop the 10 "vibe" themes or keep them as Studio presets?** — Design recommends. Three options:
   - **Drop entirely** — vanilla/cobalt-dark/high-contrast wins, the user loses the forest-light/midnight-dark variety. Risk: readalong fans miss the customization.
   - **Keep as Studio presets** — vanilla/cobalt-dark/high-contrast are the "stable" themes; the 10 vibe themes live behind a "Studio themes" toggle in `/me/settings`. Recommended.
   - **Promote all 13** — design system fully supports 13 themes. Risk: maintenance + a11y testing burden.

6. **Theme propagation cookie** — `.reeloo.ai` cookie scope, name `reeloo:theme`, value the theme slug. Pre-hydration script reads cookie before paint. Document the cookie attributes (Secure, SameSite=Lax, 1-year expiry).

7. **Per-app default theme override** — v2 brief §6.2 says: podcast → cobalt-dark, book → vanilla, practice → vanilla. Once user explicitly sets a theme, that wins globally. Design specs: how does the user know they've overridden the per-app default? (Recommendation: subtle "Auto · cobalt-dark" indicator in settings.)

8. **High-contrast theme — the third locked theme** — design must ship full high-contrast token spec. Current source apps don't have a high-contrast theme except learn-stations. Reconcile.

### Part C — Signal SDK hooks per component

9. **Component-gallery signal field** — every shared component gets a `Signals emitted` field in its component-gallery page. Example for `<Flashcard>`:
   - `card.shown` — emitted on mount
   - `card.flipped` — emitted on flip
   - `card.rated` — emitted on grade (with `grade` field: again/hard/good/easy)
   - `card.audio_play` — emitted on TTS play
   - `card.dict_open` — emitted on dict-link tap

10. **Per-app signal inventory** — for each app, document which events are emitted from which surface. Use the table format from cross-cutting §5. Design owns the inventory; engineering wires the SDK.

11. **SDK ergonomics spec** — call site: `useSignal()` hook returns `emit(event, payload)`. Payload shape includes a per-app `surface` field (auto-filled), a `timestamp_ms` (auto-filled), and event-specific fields. Document the call site in 3 example components.

12. **Privacy toggle integration** — `/me/privacy` has a "Reeloo learns from you" toggle (v2 brief §2.3). When OFF, the SDK becomes a no-op. Design specs: the toggle copy, the off-state visual, the "what we don't collect when this is off" disclosure.

### Part D — Open-question resolution

13. **The notes-app question** — cross-cutting §10 asks whether the notes-app surfaces are folded in, separated, or partially absorbed. Design takes a position:
    - Recommendation: **partially absorbed**. The notes-app `/learn/_renderers/` (5 renderers, 2,541 LOC) port to practice.reeloo.ai as `<RendererPicker>` selecting between annotated-reader / bilingual-story / duolingo / podcast-cloze / podcast-player. The MP4 export pipeline is folded into `export.reeloo.ai`. The Minna engines (`_minna-core/Exercises.tsx`, `minna/components/Exercises.tsx`) are duplicates of readalong's DrillShell and are deprecated. The mundart-100 course tree is a content artifact; folds into practice's course catalog.

14. **`<DrillTestShell>` (1,269 LOC)** — operator-only or user-facing? Design picks: operator-only (move to studio.reeloo.ai).

15. **`<EpisodeWizard>` (269 LOC)** — operator or user? Design picks: operator-only.

16. **The "for your level" chip on podcast tiles** — v2 brief §5 says "for your level" appears when `inferredCEFR ≈ targetCEFR`. Design specs: what "≈" means numerically (within 0.5? within 1.0?), the chip animation when level changes.

17. **CEFR confidence band threshold** — v2 brief open question #12: dialogue suppression at 0.6 with 1-week cooldown when confidence < 0.5. Worth user-test, but design picks a number to ship.

## Out of scope

- The signals.reeloo.ai backend.
- The Firestore migration to signals sink.
- The actual ESLint rule implementation.
- The cookie infrastructure (caddy / auth).

## Success criteria

- A migration map covers all ~200 existing testids.
- The 24+ components have per-component testid checklists.
- The lint rule (anonymous-element ban) is documented.
- Theme reconciliation has a defined position (drop/keep/promote — pick one).
- High-contrast theme has full token spec.
- Every shared component declares its emitted signals.
- The per-app signal inventory is published.
- Privacy toggle UX is designed.
- The notes-app fate is decided (no `TBD`).
- All 17 deliverables above are tracked in a single migration playbook document.
