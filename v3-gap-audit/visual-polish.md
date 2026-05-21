# v3+ gap audit — Visual polish

> Filed: 2026-05-21
> Trigger: user looked at `https://reeloo-v2.alpsmind.com/practice/mundart/l1` and said "a mountain of buttons". The structural feature-parity work is at ~85–90 % — APIs hook up, state machines fire, testids match — yet the impl looks visibly worse than the four source apps every reference frame compares it to. The v5 pixel-diff report in `reeloo-v2/verify/design-parity/design-parity-report.md` corroborates: `/practice` 62 %, `/practice/learn-anything` 63 %, `/book` 93 %, `/` 37 %, `/me/progress` 19 %, `/extension/youtube` 29 %. Every single reference route is in the "drift" band.

This audit is the visual counterpart to `v3-gap-audit/{learn-stations,book-app,podcast,readalong,cross-cutting}.md`. Those documents asked *"is the feature there?"*. This one asks *"does it look like the thing the user already loves on the live host?"*. They turn out to be different audits — and the second one was never run.

---

## 1. Root cause

The 12 design briefs in `requests/01..12` all describe **primitives + state machines + testid contracts + per-surface integration matrices**. That was the right altitude for the structural pass. What none of them produced is **reference-quality production frames** — a screenshot of a real, single-purpose surface at the visual density a user would actually meet.

The hub pattern Claude design returned is great as a designer-onboarding tool and useless as a production target. Three concrete examples:

- `/design/stations` (`components/design/DesignStationsHub.tsx:1-838`) renders **21 station-family cards stacked vertically**, each with its own eyebrow, sub-title, sample interaction, and section anchor. That is the entire "mountain of buttons" the user reported — except the user wasn't even on `/design/stations`, they were on `/practice/mundart/l1`, which inherits the same chrome density because the production component was wired from the explorer card layout.
- `/design/dictionary` (`components/design/DesignDictionaryHub.tsx:1-326`) renders **5 form-factor variants in a grid** (popover · rail · inline · modal · flash · page). For a designer evaluating which form-factor to use on which surface, this is correct. For a learner who tapped a word, none of those frames exist — production needs *one* dictionary card, not a grid of five.
- `/design/book` (`components/design/DesignBookHub.tsx:1-570`) is a 570-LOC scrolling document with section anchors and inline interaction demos. The actual book in `/home/claw/work/book-app/app/read/[slug]/ch/[n]/page.tsx:40-63` is **a 5-LOC route** that mounts one `<ChapterReader>` against a cinematic background and disappears all chrome past a single header strip.

The v2/v3/v4 briefs got the structural ontology right and never asked for a single production-density mockup. The output reflects exactly what was asked for, which is why the impl looks structurally complete and visually flat.

---

## 2. Visual primitives in real apps that v2's design bundle is missing

The next 8 items are visual-quality features that live in the source apps and have no design-bundle equivalent in `reeloo-v2/components/`. Each names a file:line and a one-paragraph caption describing what the user actually sees.

### 2.1 — Cinematic chapter-art background with paper grain + scrim

**Source:** `book-app/components/deck/DeckBackground.tsx:1-146`.

The book-app chapter-reader, in deck mode, paints a full-bleed background that resolves through three layers: (1) a chapter-cover image if `/books/<slug>/<chapterId>/cover.jpg` HEADs 200, (2) a chapter-title→gradient lookup with named tuples for Shire / Mordor / Rivendell / Lothlórien / Fangorn / Rohan / Gondor / Moria / Bree, (3) a default warm-parchment gradient. On top of *that* the component lays an inline SVG fractal-noise paper-grain texture at 18 % opacity (overlay blend mode) plus a top→bottom legibility scrim. The visual result is unmistakably book-like — a Hobbiton oil-painting under a layer of paper grain — and is the single feature that makes the source app feel cinematic rather than wireframe.

**v2 equivalent:** none. `reeloo-v2/components/routes/book.tsx:1-787` paints flat solarized surfaces. The pixel-diff report has `/book` at **92.6 %** diff — almost certainly because the reference frame is the dialog-overlay-on-cinematic-bg and our impl has no cinematic bg at all.

### 2.2 — Per-type annotation underlines (solid + dotted) with color-coded popovers

**Source:** `book-app/components/AnnotationPopover.tsx:1-99` + `book-app/app/globals.css:7-13`.

Six annotation types, each with its own underline color: `vocab` (blue solid), `grammar` (green solid), `idiom` (violet solid), `place` (orange solid), `archaism` (yellow **dotted**), `proper_noun` (blue solid). The underline is 2 px with a 4 px underline-offset. Tapping a span opens a paper-card popover; the top of the popover repeats the type-color as an 11-px uppercase eyebrow ("VOCAB", "ARCHAISM") followed by the gloss in sans. The Solarized palette gives the whole reader an academic-edition feel.

**v2 equivalent:** chip-style boxes (cf. `DesignBookHub.tsx`). The dotted-underline archaism marker is missing entirely; everything reads as the same form-factor regardless of annotation type.

### 2.3 — Character SVG illustration system (9,138 vector glyphs)

**Source:** `learn-stations/public/dict-svg/` (9,138 SVGs) + `learn-stations/public/pip-poses/` (10 pose SVGs) + `learn-stations/lib/vocab-cards.ts:186-200` (`SVG_OVERRIDES` for deictic pronouns).

Every flashcard front in `learn-stations/app/cards/CardsDeck.tsx:546-552` paints an SVG on a `#F6E8C8` cream tile. The illustrations are minimal, monochrome, semantic — a stick figure pointing at themselves for `わたし`, a person near the speaker for `ここ`, a person far away for `あそこ`. They are not generic emoji; they are word-specific, single-color, vector-clean. The `pip-poses/` set adds 10 mascot poses (`happy`, `sad`, `angry`, `surprised`, `thinking`, `tired`, `point-self`, `point-near`, `point-far`, `neutral`) used when the per-word SVG is missing.

**v2 equivalent:** none. The v2 design bundle has `<Pip>` (12 moods, geometric) but no per-word illustration pool. Cards in `/practice/cards` render text-only or fall back to emoji.

### 2.4 — Cream-tile illustration container (visual anchor inside a dark card)

**Source:** `learn-stations/app/cards/CardsDeck.tsx:535-553` — front face has a top-half cream tile (`#F6E8C8`, 12 px radius) that contains the illustration. The rest of the card sits on the MNL palette dark surface (`#163134`). The cream-on-dark contrast is what makes the illustration feel "lit". Without it the SVG would just float on dark felt and read as a sticker.

**v2 equivalent:** the practice surfaces use the same dark surface but without the cream illustration container; illustrations (when present) sit directly on the surface and lose pop.

### 2.5 — Four-tier grade panel with colour-coded borders + Chinese+English labels

**Source:** `learn-stations/app/cards/CardsDeck.tsx:790-855`.

The SM-2 grade panel is a 4-column grid (`gridTemplateColumns: "repeat(4, 1fr)"`). Each tile shows a Traditional-Chinese verb (再來 / 困難 / 通過 / 簡單) above the English grade (Again / Hard / Good / Easy) in 9 px monospace at 72 % opacity. Each tile has a 1 px coloured border — red `#c8362f` for Again, yellow `#D9A559` for Hard, green `#7FAE93` for Good, teal `#7faeb3` for Easy. Below the grid, after a grade is chosen, a feedback chip slides in with the next-due timestamp ("再來一次 · in 1 min").

**v2 equivalent:** `components/routes/practice.tsx` uses a 2-state "Not yet / Got it" pair (per request #5 audit). The 4-state colour-coded grid is exactly what request #5 specified, but Claude design never produced a mockup, so the impl shipped the older 2-state pattern.

### 2.6 — Section-banded progress stepper (5 colours, not 13 dots)

**Source:** `learn-stations/app/[course]/[lesson]/page.tsx:362-401`.

Instead of "task N of 13", the lesson stepper segments the progress bar into **5 sections by activity band** — 理解 / 辨識 / 組句 / 口說 / 完成. Each section is its own coloured bar that fills as the user moves through stations *within* that section. The current section is bright; complete sections drop to 85 % opacity; the section label sits to the right with `idxInSection/totalInSection`. The result tells the learner "you are in the speaking band" rather than "you are on dot 9".

**v2 equivalent:** the stations route has a station-by-station dot progress; no section banding.

### 2.7 — Auto-advance countdown CTA with translucent fill animation

**Source:** `learn-stations/app/[course]/[lesson]/page.tsx:188-218`.

When a user answers correctly, the green CTA stays in place but a `bg-white/20` overlay sweeps from left to right over 2 s, filling the button as the auto-advance counts down. Tap-during-countdown cancels (pauses) the auto-advance; tap-after-countdown advances. The visual is a button that is simultaneously a progress indicator — no separate "auto-advancing in 2.0 s" pill is needed.

**v2 equivalent:** the design hub has a separate `AutoAdvancePill` SVG-arc chip (`DesignStationsHub.tsx:23-56`) that lives above the card. The pill version *adds* chrome where the source app removes it.

### 2.8 — Compact lesson + level chips with horizontal scroll (sticky, not stacked)

**Source:** `learn-stations/app/cards/CardsDeck.tsx:270-311`.

The deck filter has *two* horizontal scrollable chip rows — first lesson chips ("全部 · 第1課 · 第2課 ..."), then JLPT/CEFR level chips ("全等級 · N5 · N4 · A1 · A2"). Each chip is 22 px tall, 11 px IBM Plex Mono, with a 1 px border that switches to accent when active. The two rows sit *above* the card and never push it below the fold. `overflow-x: auto` with `WebkitOverflowScrolling: touch` and a negative margin to bleed past the page gutter.

**v2 equivalent:** `/design/stations` and `/practice` stack filter chips vertically as cards. The horizontal scroll affordance is missing on every surface that needs it.

---

## 3. Additional cross-cutting visual gaps

Four more themes that aren't single primitives but are visible everywhere once you look:

### 3.1 — Audio chrome reduction

`reeloo-v2/components/primitives/MediaPlayerBar.tsx:144-205` (variant=podcast) renders **7 controls**: prev / play / next / scrub / speed-ladder / replay / gear. The book variant renders **5**: play / next / scrub / speed-ladder / cycle. The card variant renders 2 (play + speed). The real apps render **3 on every audio surface**: play, speed (single chip showing current speed; tap to cycle), scrub. There is no prev/next on podcast (the focus-pin column is the navigator), no gear (settings live in `/me/settings`), no replay (R-key is the affordance; chrome-less).

### 3.2 — Theme floating

The v2 token set (`app/styles/tokens.css:19-130`) defines 4 themes — `solarized-light` (default), `solarized-dark`, `sepia`, `high-contrast`. There is no per-app theme assignment. Every surface inherits whatever the user's body class is. The result: `/book` looks identical to `/practice` looks identical to `/podcast`. The source apps each commit to one mood — book-app dark / parchment scrim, podcast cobalt-dark, learn-stations MNL dark with `#D97757` warm orange accent. v2 reads as one homogeneous Solarized surface.

### 3.3 — Production-density vs explorer density

`/design/<sub>` is correctly an explorer hub. The mistake is that the **production routes inherited the explorer density**: `/practice` shows multiple station-family cards on one screen, `/practice/<course>/l<n>` shows a station player that is itself crowded with explorer-card chrome (auto-advance pill + grade panel + nav arrows + station counter + section header + flag-issue + Pip whiteboard). The source app at `learn-stations/app/[course]/[lesson]/page.tsx` shows **one station**, full bleed, plus a 2-row header (URL strip + segmented progress) and a 2-button footer (← + morphing CTA). That's it. Nothing else on screen.

### 3.4 — Negative space + max-content widths

Source apps use:
- `book-app/app/globals.css:17` → `--reading-max-width: 720px` for the chapter column
- `learn-stations/app/cards/CardsDeck.tsx:169` → card sized to 360 × 480 explicit pixels, *not* responsive
- `book-app` chapter route → `max-w-5xl mx-auto p-8` (line 41) so the chapter sits in a generous gutter

`reeloo-v2` tends toward `max-w-7xl` or `max-w-full` with multi-column grids. The visual density is 3–5× higher per square cm. The "mountain of buttons" feeling is partly literal button count and partly the *absence of negative space around* the buttons.

---

## 4. Which pixel-diff numbers map to which gap

Cross-referencing `reeloo-v2/verify/design-parity/design-parity-report.md` with §2:

| Route | Diff % | Most-likely root cause from §2 |
|---|---:|---|
| `/book` | 92.6 % | §2.1 cinematic chapter art missing + §3.2 theme floating |
| `/practice/learn-anything` | 62.7 % | §3.3 production-density + §2.8 chip scroll |
| `/practice` | 62.0 % | §3.3 production-density (this is the route the user audited) |
| `/` | 36.6 % | §3.4 negative space + §3.2 theme floating |
| `/extension/youtube` | 29.1 % | mock-browser frame around panel — `ExtensionSurface.tsx:51-64` renders `<BrowserChrome>` as the dominant visual, the panel is a slim 380 px column. Real Chrome extension renders panel only. |
| `/me/progress` | 18.9 % | §3.1 audio chrome (the streak chip + progress arc + grade history) + §3.4 negative space |

The diff numbers are not noise. They tell a consistent story: every reference frame is **less dense + more atmospheric** than the live impl.

---

## 5. What this means for the next design ask

The next request packet (`requests/13-visual-polish-pass.md`) must:

1. Ask for **production-density mockups**, not new primitives. The primitives are there; they need to be composed at the right density on the right route.
2. Pick **one theme per app**. The user should be able to tell which app they're in by the colour palette of the first half-second.
3. Spec a **character illustration system** — 10–20 vector glyphs in the learn-stations/dict-svg style (single-colour, semantic stick figure). Even a starter set lets the flashcard surfaces feel like the source.
4. Specify **chrome reduction** explicitly — every audio surface gets the 3-control bar; every annotation gets type-coloured underline not chip box; every progress display gets section bands not station dots; every CTA gets the auto-advance fill animation not a separate pill.
5. Ban explorer-density on production routes. The 6 `/design/<sub>` hubs survive **only** as explorer / token-reference; nothing the user ever lands on by accident should look like a hub.

The success bar is the user's vibe test: *"would I screenshot this and put it in a presentation?"*. Right now the answer at `/practice/mundart/l1` is no, and the diff report quantifies why.

---

## Appendix — sources cited

- `book-app/app/read/[slug]/ch/[n]/page.tsx:40-63` — 5-LOC route, the production-density baseline
- `book-app/components/deck/DeckBackground.tsx:1-146` — cinematic art chain
- `book-app/components/PageReader.tsx:82-110` — `--reading-max-width: 720px` + per-paragraph translation gutter
- `book-app/components/AnnotationPopover.tsx:8-99` — 6-type annotation underline + popover
- `book-app/app/globals.css:7-17` — annotation palette + reading rhythm tokens
- `learn-stations/app/[course]/[lesson]/page.tsx:302-430` — minimal lesson chrome
- `learn-stations/app/[course]/[lesson]/page.tsx:188-218` — auto-advance fill CTA
- `learn-stations/app/[course]/[lesson]/page.tsx:362-401` — section-banded progress
- `learn-stations/app/cards/CardsDeck.tsx:30-48` — MNL palette tokens
- `learn-stations/app/cards/CardsDeck.tsx:425-521` — `<FlipCard>` shell (360 × 480, cream illustration tile)
- `learn-stations/app/cards/CardsDeck.tsx:790-855` — 4-tier grade panel
- `learn-stations/public/dict-svg/` — 9,138 word-specific SVG illustrations
- `learn-stations/public/pip-poses/` — 10 mascot poses
- `reeloo-v2/components/design/DesignStationsHub.tsx:1-838` — explorer hub, NOT production
- `reeloo-v2/components/design/DesignDictionaryHub.tsx:34-47` — 11-surface integration matrix
- `reeloo-v2/components/primitives/MediaPlayerBar.tsx:51-205` — current 3-variant chrome (over-controlled)
- `reeloo-v2/components/extension/ExtensionSurface.tsx:51-64` — mock-browser chrome around panel
- `reeloo-v2/app/styles/tokens.css:20-150` — 4 themes, no per-app assignment
- `reeloo-v2/verify/design-parity/design-parity-report.md` — drift numbers across 6 routes
