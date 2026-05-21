# Design request: Visual polish pass — production-density mockups for 6 user-facing routes

**Status:** open
**Filed:** 2026-05-21
**Author:** orchestrator (visual-polish audit pass)
**Trigger:** user looked at `https://reeloo-v2.alpsmind.com/practice/mundart/l1` and said "a mountain of buttons".

**Context refs:**
- `v3-gap-audit/visual-polish.md` — the audit that produced this ask (cinematic backgrounds missing, theme floating, illustration system missing, chrome over-controlled, explorer-density bleeding into production routes)
- `reeloo-v2/verify/design-parity/design-parity-report.md` — every reference route in the drift band: `/book` 93 %, `/practice/learn-anything` 63 %, `/practice` 62 %, `/` 37 %, `/extension/youtube` 29 %, `/me/progress` 19 %
- The 12 prior requests (`requests/01..12`) — all primitives + state machines + integration matrices, none of which produced a production-density frame

---

## 1. What we asked for in v3/v4 vs what we got

The first 12 design requests asked Claude design for primitives, state machines, testid contracts, and per-surface integration matrices. The output is structurally excellent — `<Pip>` (12 moods), `<DictionaryCard>` (5 form-factors), `<MediaPlayerBar>` (3 variants), `<Flashcard>` (4 layout templates), 21 station families with full testid coverage. Each of those primitives has a hub page at `/design/<sub>` that shows every variant side by side.

What we never asked for is a **production-density mockup**: a screenshot of one route, with one purpose, at the density a user would actually meet. The result of that gap:

- **`/design/dictionary`** (`reeloo-v2/components/design/DesignDictionaryHub.tsx:34-47`) lays out 5 form-factors (popover · rail · inline · modal · flash · page) plus an 11-surface integration table. Excellent designer reference. But the production user who taps a word in a podcast transcript sees *one* dictionary form-factor, not five — and Claude design never produced a frame showing which one, at what size, with what scrim around it, in what theme.
- **`/design/stations`** (`reeloo-v2/components/design/DesignStationsHub.tsx:1-838`) renders 21 station-family cards in a vertical stack. Excellent designer reference. The user reads it as the mountain. The actual lesson route at `/practice/mundart/l1` then *inherits* the explorer chrome density because the production component was wired from the explorer card layout — `AutoAdvancePill` lives above the card as a separate SVG-arc chip (`DesignStationsHub.tsx:23-56`) when the real app at `learn-stations/app/[course]/[lesson]/page.tsx:188-218` makes the CTA itself the progress indicator with a `bg-white/20` sweep.
- **`/design/book`** (`reeloo-v2/components/design/DesignBookHub.tsx:1-570`) is a 570-LOC scrolling document with anchored sections. The actual book at `book-app/app/read/[slug]/ch/[n]/page.tsx:40-63` is a **5-LOC route** that mounts one chapter against a cinematic background. Nothing on screen except the text, a 5-control header (TOC link · mode toggle · language toggle · annotation toggle · chapter nav), and silence.

The structural pass landed. The visual pass is now overdue.

---

## 2. What's actually in the source apps

These are 8–12 visual-quality features that live in the source codebases (`/home/claw/work/{learn-stations,book-app,readalong,notes}/`) and have no equivalent in `reeloo-v2/components/`. They are not new primitives — they are the visual recipes that, applied to the existing primitives, are what makes the source apps feel polished.

**B.1 — Cinematic chapter art with paper grain + scrim.** `book-app/components/deck/DeckBackground.tsx:1-146`. Full-bleed background resolves through a 3-step chain: chapter cover image at `/books/<slug>/<chapterId>/cover.jpg` → chapter-title→gradient lookup (Shire / Mordor / Rivendell / Lothlórien / Fangorn / Rohan / Gondor / Moria / Bree) → warm-parchment default. Layers an inline-SVG fractal-noise paper-grain at 18 % opacity (overlay blend) plus a top→bottom legibility scrim.

**B.2 — Six-type annotation underline (solid + dotted).** `book-app/app/globals.css:7-13` + `book-app/components/AnnotationPopover.tsx:8-99`. Solarized palette: vocab/blue, grammar/green, idiom/violet, place/orange, archaism/yellow-dotted, proper-noun/blue. 2 px line, 4 px offset. Popover shows the type-color as an 11 px uppercase eyebrow above the gloss.

**B.3 — Word-specific SVG illustration pool.** `learn-stations/public/dict-svg/` ships 9,138 word-specific SVGs (`de-Abendessen.svg`, `de-Aare.svg`, `ja-わたし.svg`, etc.) plus `learn-stations/public/pip-poses/` with 10 mascot poses (`happy.svg`, `point-self.svg`, …). Each glyph is monochrome, semantic, vector-clean — a stick figure pointing at themselves for `わたし`, not a generic clipart person.

**B.4 — Cream illustration tile inside a dark card.** `learn-stations/app/cards/CardsDeck.tsx:535-553`. The card-front top half is a `#F6E8C8` cream tile, 12 px radius. The illustration sits on the cream; the surface word + audio chip sits below on the MNL dark surface (`#163134`). Cream-on-dark is what makes the SVG read as illuminated, not stickered.

**B.5 — Four-tier grade panel with colour-coded borders.** `learn-stations/app/cards/CardsDeck.tsx:790-855`. 4-column grid: 再來 / 困難 / 通過 / 簡單 with English subtitles in 9 px monospace at 72 % opacity. Borders are red `#c8362f` / yellow `#D9A559` / green `#7FAE93` / teal `#7faeb3`. Post-grade feedback chip with next-due timestamp ("再來一次 · in 1 min").

**B.6 — Section-banded progress (5 colours, not N dots).** `learn-stations/app/[course]/[lesson]/page.tsx:362-401`. The stepper is segmented into 理解 / 辨識 / 組句 / 口說 / 完成 — one coloured bar per section, fill ratio = `idxInSection / totalInSection`. Current section bright, complete sections 85 % opacity, label `{section.label}` `{idx}/{total}` to the right.

**B.7 — Auto-advance fill CTA.** `learn-stations/app/[course]/[lesson]/page.tsx:188-218`. On correct answer the green CTA stays in place; a `bg-white/20` overlay sweeps left-to-right over 2 s. Tap-during-sweep cancels; tap-after-sweep advances. No separate "auto-advancing in 2.0 s" pill.

**B.8 — Horizontal-scroll chip rows above the card.** `learn-stations/app/cards/CardsDeck.tsx:270-311`. Two rows: lesson chips ("全部 · 第1課 · 第2課 …"), then JLPT/CEFR level chips ("全等級 · N5 · N4 · A1 · A2"). 22 px tall, IBM Plex Mono 11 px. `overflow-x: auto` with `WebkitOverflowScrolling: touch`, negative margin to bleed past the page gutter.

**B.9 — Reading rhythm tokens.** `book-app/app/globals.css:16-17`. `--reading-line-height: 1.75`, `--reading-max-width: 720px`. The chapter column sits in a generous gutter (`max-w-5xl mx-auto p-8`); the translation paragraph is half a size down in `font-sans` with reduced opacity, immediately below the source paragraph.

**B.10 — Auto-TTS toggle chip with status dot.** `learn-stations/app/cards/CardsDeck.tsx:312-339`. Small (26 px tall) pill, 6 px coloured dot + label `自動發音 ON|OFF`, accent-coloured ring + tinted background when on, transparent when off. Settles in the top-right of the chip rows; never dominates the layout.

**B.11 — Dictionary affordance as a discreet footer link.** `learn-stations/app/cards/CardsDeck.tsx:752-773`. The back-of-card "look it up" link is a 12 px monospace chip with `↗ dictionary.alpsmind.com` — bottom-left of the card, low contrast, never compete-with-content. The form-factor explorer in `/design/dictionary` shows 5 form-factors; production picks **one per surface** and the back-of-card surface picks the footer-chip.

**B.12 — Minimal lesson chrome.** `learn-stations/app/[course]/[lesson]/page.tsx:302-430`. The entire chrome is: a 1-row header (clickable URL strip + romaji toggle + settings cog), a 1-row progress strip (segmented bands + section label), the station main, and a 2-button footer (← + morphing CTA). No tabs, no breadcrumbs, no sidebar. Everything else is the station's job.

---

## 3. What we need from Claude design

Six production-density route mockups + one illustration system + a theme assignment table + a chrome-reduction spec. **No new primitives.** This pass composes what already exists.

### 3.1 — Six production-density route mockups

Each mockup is a single full-bleed frame at 1280 × 800 desktop **and** 390 × 844 mobile. The frame must show what a real user lands on after navigating to that URL — not the explorer view, not a contact-sheet of variants.

| Route | Purpose | Density target | Reference frame |
|---|---|---|---|
| `/practice/cards` | One flashcard centered with chip rows above + grade panel below | 1 card, 2 chip rows, 4-tier grade panel — that's it. No sample grid, no variant strip. | `learn-stations/app/cards/CardsDeck.tsx` rendered with one card visible |
| `/practice/<course>/l<n>` | One station, full-bleed | 5-LOC header + section-banded progress + station main + 2-button footer + morphing CTA. No `AutoAdvancePill` (the CTA *is* the pill). No flag-issue button in chrome (it lives in the cog menu). | `learn-stations/app/[course]/[lesson]/page.tsx:302-430` |
| `/book/<slug>/ch/<n>` | Cinematic chapter reader | Full-bleed `<DeckBackground>` + 720 px reading column + 6-type annotation underlines + per-paragraph translation gutter + 3-control audio bar pinned bottom. | `book-app/app/read/[slug]/ch/[n]/page.tsx` + `book-app/components/DeckBackground.tsx` |
| `/podcast/<slug>` | Focus-pin player, dark cobalt | Focus-pin transcript column + 3-control audio bar (play · speed · scrub) + word-tap dictionary as **rail** form-factor only + sticky footer. No prev/next/replay/gear chrome. | `notes/app/learn/_renderers/PodcastPlayerRenderer` adapted |
| `/me/progress` | Calm progress dashboard | Streak chip + section-banded weekly bar + recent-cards strip + diary preview card. Max 5 surfaces on screen, generous gutters. No grade history table. | new — current `/me/progress` is 19 % off because it has too many buttons |
| `/extension/<host>` | Panel-only overlay | The 380 px panel in floating + docked states **without** the surrounding `<BrowserChrome>` mock-browser frame. The mock-browser stays as an explorer prop at `/design/extension` for designers; production frames the panel against a real screenshot or a transparent backdrop. | `reeloo-v2/components/extension/ExtensionPanel.tsx:1-437` |

For each route deliver: the desktop frame, the mobile frame, and a 50-word visual treatment note (theme, dominant colour, typography, what's intentionally absent).

### 3.2 — Character illustration system (10–20 starter glyphs)

A starter set of 10–20 vector glyphs in the `learn-stations/public/dict-svg` style:

- **Style:** single-colour stroke (default `currentColor` so theme tokens drive the tint), 24 × 24 viewBox, semantic — a stick figure or pictogram that *names* the word, not a generic decoration.
- **Coverage:** 10 deictics (`私 / あなた / ここ / そこ / あそこ / これ / それ / あれ / 上 / 下` or the German equivalents `ich / du / hier / dort / oben / unten / links / rechts / vorne / hinten`) + 10 high-frequency nouns (`Brot / Wasser / Buch / Haus / Auto / Zug / Hund / Katze / Sonne / Mond`). 20 total is enough to seed; the LoRA pipeline in `scripts/illustrations/generate_flashcards.py` handles the long tail.
- **Naming:** `/public/illustrations/<lang>-<surface>.svg` — `/illustrations/de-Brot.svg`, `/illustrations/ja-私.svg`. Filenames mirror the existing `dict-svg` convention so the lookup helper in `lib/vocab-cards.ts:svgUrlFor()` ports cleanly.
- **Plus** 10 mascot poses (`happy.svg`, `sad.svg`, `thinking.svg`, `surprised.svg`, `tired.svg`, `neutral.svg`, `point-self.svg`, `point-near.svg`, `point-far.svg`, `cheering.svg`) that compose with the existing `<Pip>` mood matrix. These are the "Pip-as-character" referenced in the brief — vector poses, not 12-mood face geometry.

Deliver as SVGs ready to commit under `reeloo-v2/public/illustrations/`.

### 3.3 — Theme assignment table (one theme per app)

The v2 token system at `reeloo-v2/app/styles/tokens.css:20-150` defines 4 themes. Currently every surface inherits whatever the body class is, so the apps look identical. Spec:

| App | Theme | Dominant accent | Why |
|---|---|---|---|
| `/podcast/*` | cobalt-dark (new variant of `solarized-dark` with stronger blue) | `--accent-blue` `#268bd2` | Listening surface; user closes their eyes; high-contrast text needs to recede |
| `/book/*` | sepia-paper (extend existing `sepia` to layer the paper-grain noise globally) | `--accent` brown `#8b5a2b` | Reading surface; cinematic chapter art demands warm palette |
| `/practice/*` | cream + warm orange accent (the MNL palette from `learn-stations/CardsDeck.tsx:30-42` — `#0a1517` bg, `#F0E9D8` ink, `#D97757` accent) | `#D97757` warm orange | High-energy drill surface; CTA needs to pop |
| `/extension/<host>` | cobalt panel on cream backdrop | `--accent-blue` | Panel must read as a tool *over* host content, not as a website |
| `/me/*` | solarized-light (the calm default) | `--accent-cyan` | Reflective surface; should feel cool, not energetic |
| `/design/*` | solarized-light (designer reference) | mixed | Explorer mode; theme is whatever the designer is auditing |

Deliver the cobalt-dark and cream+warm variants as additions to `tokens.css` (new `[data-theme]` blocks). The reeloo-v2 app shell already supports `data-theme` so wiring per-route is a single attribute on the segment layout.

### 3.4 — Annotation typography system

Adopt the book-app palette + form (`book-app/app/globals.css:7-13`, `book-app/components/AnnotationPopover.tsx:1-99`) across **every text surface** that supports annotation — book chapter, podcast transcript, learn-stations grammar station, readalong slide:

- `vocab` → `--anno-vocab` blue solid 2 px underline, 4 px offset
- `grammar` → `--anno-grammar` green solid
- `idiom` → `--anno-idiom` violet solid
- `place` → `--anno-place` orange solid
- `archaism` → `--anno-archaism` yellow **dotted**
- `proper_noun` → `--anno-proper-noun` blue solid

Tap-popover: type-color as 11 px uppercase eyebrow above the gloss in sans. **Not** the chip-style boxes currently in `DesignBookHub.tsx`. Replace those mockups.

### 3.5 — Audio chrome reduction

Replace `<MediaPlayerBar variant="podcast">` (7 controls) and `<MediaPlayerBar variant="book-deck">` (5 controls) with a **single 3-control bar** for all listening surfaces:

- **Play / pause** (left, 44 px primary button)
- **Speed chip** (single button showing current speed, tap to cycle 0.75× → 1.0× → 1.25×; long-press to open full ladder; never render the ladder inline)
- **Scrub** (track with current/total time)

Deprecate: prev / next (the focus-pin column or paragraph anchors are the navigator), replay (R-key + visible kbd hint), gear (settings live in `/me/settings`). The `card` variant stays as today (2 controls — play + speed). The `book-deck` variant collapses to the same 3-control bar.

Deliver a new `<MediaPlayerBar>` mockup showing the single bar in podcast theme + book theme; mark the deprecated controls as struck-through on the existing hub page.

### 3.6 — Negative space + max-content guidance

For each of the 6 routes above, the mockup notes must specify:

- The dominant column max-width (e.g. book: 720 px chapter column; podcast: 560 px focus-pin column; practice: 360 px card centered).
- The minimum gutter (e.g. 24 px mobile / 64 px desktop).
- What should NOT be on screen at production density (e.g. practice route excludes flag-issue button, settings link, breadcrumb, station-counter chip; they live in the cog menu).

This becomes the chrome-budget contract: anything not on the mockup should not ship to production. Designers can keep adding to `/design/<sub>` hubs; that's where additional affordances are evaluated before promotion.

---

## 4. Out of scope

- **No new primitives.** `<Pip>`, `<DictionaryCard>`, `<MediaPlayerBar>`, `<Flashcard>`, station families — all stay. This pass polishes the composition of existing primitives, not the primitive set.
- **No new states or interaction flows.** State machines from requests #1–12 stay verbatim.
- **No new routes.** The 6 routes above already exist in `reeloo-v2/app/`.
- **No removal of `/design/<sub>` hubs.** Those stay as explorer / designer-onboarding pages. They simply stop being the source for production composition.
- **No backend changes.** Theme variants, illustration assets, and chrome reduction are all CSS + JSX work; no API contracts move.

---

## 5. Success criteria

- Re-run `reeloo-v2/verify/design-parity/` against the new mockups. Target: every route falls below **15 % pixel diff** (current band 19 %–93 %).
- The user's vibe test: *"would I screenshot this and put it in a presentation?"* Answer at `/practice/mundart/l1` flips from no to yes.
- Theme assignment is legible at half-a-second glance — a user landing on a route can tell which app they're in by colour palette alone, no logo needed.
- Chrome budget audit: every production route renders ≤ 8 distinct interactive elements on first paint. (The cog menu absorbs the rest.)
- Annotation underlines are everywhere they need to be, dotted-archaism included; chip-box annotations are gone.
- Audio chrome is the 3-control bar on every listening surface; the 7-control podcast variant and 5-control book-deck variant are deprecated.
- Illustration pool has ≥ 20 vector glyphs committed at `reeloo-v2/public/illustrations/`; flashcard front falls back to the LoRA cache only when the static SVG is missing.

---

## 6. Open questions

1. **Character illustration tone.** Childish stick-figure (closer to the existing `pip-poses/` set) or sophisticated minimal (closer to airport-pictogram + Noun Project)? The dict-svg pool is somewhere between — closer to airport pictogram than childish. Pick one and commit; mixed-tone is what makes the current Pip mood matrix feel slightly off when next to a flashcard SVG.
2. **`<Pip>` mood matrix vs character illustration system.** Are they the same character (Pip-as-character with poses *and* mood faces) or two distinct visual systems sharing the same SVG style? Brief #11 treats Pip as a 12-mood face primitive; the source apps treat the same SVG style as a full-body pictogram. Reconcile.
3. **Primary "default" theme.** When a route doesn't declare a theme, what does the body inherit — solarized-light (calm), cream (warm), or paper (book)? Currently solarized-light by default; the audit suggests cream as a warmer default for a learning product.
4. **`/design/<sub>` hub consolidation.** The 6 hubs (`audio`, `book`, `dictionary`, `pip`, `sequential-drill`, `stations`) total ~3,100 LOC. Keep all 6 as explorer-mode (status quo), consolidate into a single `/design` token-reference page, or both (token reference at top, explorer hubs as drill-downs)? The user's "mountain of buttons" reaction suggests the hub-aesthetic itself is the problem; consolidating into a single reference page would close one bleed path between explorer and production.
5. **Annotation underline density.** All annotated spans on first paint, or only on hover/long-press? The source `book-app` shows them all the time. The chip variants in `DesignBookHub` show them on tap. Which becomes canonical?
