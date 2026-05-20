# Design request: Practice — the 21 station types

**Status:** open
**Filed:** 2026-05-20
**Author:** orchestrator (audit pass)
**Context refs:**
- `v1/products/learn-stations.md` §3, §4 (lists "18 station kinds" as a flat enum, describes none)
- `v3-gap-audit/learn-stations.md` §1 (features 1–21 = the 21 station-component table), §3 (the deep-dive gap)
- Source code: `/home/claw/work/learn-stations/components/stations/` (21 .tsx files, 7,115 LOC total)
- Memory: `project_pattern_stack_drill.md`, `reference_readalong_drill_patterns.md`

## What we asked for in v2 (and what we got)

The v2 brief named `<StationRunner>` as a single component (`v2/06` §7.1) and assumed v1's 18-station list was a sufficient design contract. Claude design delivered `routes/practice.jsx` (444 LOC) with **one** card-shaped UI hardcoded for three lessons (`minna`, `mundart`, `deutsch-b1`). No station-type specialization. The impl `reeloo-v2/components/routes/practice.tsx` ports that one UI faithfully and ships zero of the 21 distinct station interactions.

This is the largest single design gap.

## What's actually in the source app

21 distinct station components live at `/home/claw/work/learn-stations/components/stations/`. Each is its own micro-app with its own affordance grammar:

| Station | LOC | What the user sees + does |
|---|---|---|
| `intro` | 80 | Static intro card with vocab + audio + "Continue" CTA |
| `vocab-intro` | 105 | New word surface — reading, gloss, POS, play before drill |
| `mc` | 153 | 4-option multiple-choice |
| `image-mc` | 95 | SVG illustration + 4 word options |
| `listen-mc` | 311 | Hear audio (Azure TTS), pick from 3 word options with slow-replay |
| `cloze` | 375 | Fill-in-the-blank: sentence with a hole, pick from word chips |
| `complete` | 58 | Lesson-end success card (score + streak + "Next") |
| `grammar` | 118 | Rule + POS-colored example sentence (`JpPosColored`) |
| `vocab-draw` | 108 | Freehand-trace warmup before `trace` |
| `reshuffle` | 251 | Drag-and-drop word tiles to reconstruct a sentence |
| `substitution-grid` | 165 | Substitution grid — pick the replacement that fits the slot |
| `substitution-pattern` | 676 | Pattern drill — same template, cycling slot; matched value goes green + slides up; next slides from bottom |
| `word-match` | 189 | Match words to translations (two columns, draw line / tap pair) |
| `pattern-stack` | **1,258** | STT-primary syllabus drill; 3-phase substitution/transformation/extension |
| `tier-text` | 231 | Tier-conditional text variant rendering |
| `trace` | 283 | Kanji/character trace — finger-trace on canvas |
| `echo` | 397 | Listen → repeat; STT scores against expected transcript; audio-level waveform |
| `sequential-drill` | **1,668** | Locked-design speaking drill (see request 04 — separate file) |
| `sequential-drill-station` | 51 | Wrapper |
| `particle-burst` | 255 | JP particle drill — radial menu (を/に/で/が/は…), particle-tinted feedback |

Each station has its own affordances (chips? draggable tiles? canvas? mic? radial menu?), its own feedback states (correct/wrong/warning), its own scoring rule, its own telemetry shape.

The user-visible chrome is shared across stations: `<AutoAdvancePill>` countdown (2s default, cancellable), `<BottomCTA>` morphing button driven by `lib/stage.ts` (states: `no-grade → grading → graded → next`), `<Mascot>` Pip in 6 moods.

## What we need from Claude design

This is a **grouped request**. Rather than 21 separate mockups, we want them grouped by affordance family so the design system can compose.

### Group A — Static reveal stations (intro, vocab-intro, grammar, complete, tier-text)

5 stations that present information without scoring. Need: shared `<RevealCard>` layout with optional audio chip, optional illustration, optional POS-colored example, single "Continue" CTA. 5 frame variants showing the differences.

### Group B — Multiple choice family (mc, image-mc, listen-mc)

3 variants of "pick 1 of N". Need: shared `<MCCard>` with prompt area (text / image / audio), 2/3/4-option grid (responsive), per-option click feedback (correct: green ring + chime; wrong: warm pulse + transcript shown — NEVER red flash), explanation reveal on grade. Listen-mc needs a **slow-replay button** (the 0.6× variant) prominently placed.

### Group C — Fill-in family (cloze, complete, substitution-grid)

Sentence with a slot or grid. Need: shared `<FillCard>` with sentence rendering (slot as visual gap with subtle dotted underline), chip palette below, drag-or-tap interaction, feedback animation when chip lands in slot.

### Group D — Pattern-drill family (substitution-pattern, pattern-stack)

The big one — substitution-pattern is 676 LOC and pattern-stack is 1,258 LOC. Per memory `reference_readalong_drill_patterns.md`: "Substitution = same template + cycling slot, NOT sentence-construction. Matched value goes green + slides up; next slides in from bottom."

Need: dedicated mockup for the **matched-slides-up / next-slides-up animation** (this is the user's repeated requirement — emphasis 10x). Pattern-stack additionally needs the 3-phase progression visualization (substitution → transformation → extension) with a phase indicator.

### Group E — Spatial-input stations (reshuffle, word-match, trace, vocab-draw, particle-burst)

5 stations with non-card UI. Need:
- `reshuffle`: drag-tile mockup with target slots and source pile. Keyboard equivalent (arrow + space).
- `word-match`: two-column layout with hit-test affordance (tap A then tap B's pair, OR draw a line on touch).
- `trace`: full-screen canvas with kanji outline guide, stroke order indicators, "redo stroke" affordance.
- `vocab-draw`: trace warmup variant — smaller canvas, less strict scoring.
- `particle-burst`: radial menu (5-8 spokes), each spoke colored by particle hue, with center-target sentence; feedback bursts particles from the center on correct.

### Group F — Speaking station (echo)

Listen → repeat; STT scores; audio-level waveform during recording. See request 04 for the related sequential-drill (more locked).

### Shared chrome — all stations

- **`<AutoAdvancePill>`** — 2s countdown chip at top-right of card. Cancel on tap. Per `components/AutoAdvancePill.tsx`.
- **`<BottomCTA>`** — morphing button: "Continue" (no-grade) → "Checking…" (grading) → "Next →" (graded). 4 states; smooth morph between. Per `components/BottomCTA.tsx` + `lib/stage.ts`.
- **Pip mood-firing matrix** — per-station mood map (see request 11 for the cross-cutting avatar spec).
- **Progress strip** at top of station player (station N of M).
- **`<EtymologyBubble>`** popover for word-origin info on tap.
- **`<DrillDensityBadge>`** small chip indicating lesson density.
- **`<TierPicker>`** casual / modest / serious (3-state segmented control).
- **`<RubyText>`** furigana renderer for JP (kanji + small superscript reading).
- **`<JpPosColored>`** POS-tinted Japanese sentence (place / subject / object / verb / particle, each token gets a tinted background per `globals.css:45-49`).
- **TTS buttons** `<PlayTTSButton>` + `<PlaySlowTTSButton>` — two-speed audio chips.

### Per-station deliverables

For each of the 21 stations, design must produce:

1. A Figma frame showing the station UI in its three core states: pre-attempt, mid-attempt, post-grade (correct + post-grade-wrong as a fork).
2. Token usage spec (which surface / which text / which graded-state token).
3. The accessibility model: keyboard nav, focus order, ARIA roles.
4. The empty/loading/error state (audio fails, image fails, STT 5xxs, kanji has no stroke data).
5. The mobile variant.
6. A `<surface>:<component>:<element>` test-id sheet.

## Out of scope

- The discriminated-union zod schema (`data/stations.ts`) — already canonical.
- The 140 lesson JSON files — design doesn't change content.
- The four themes — covered in v1; just inherit.
- Sequential-drill internals — see request 04 (separate, locked-design).
- The TTS / STT backend — see request 10 (cross-cutting).
- The Anki-style `/cards` deck — see request 05.

## Success criteria

- Each of the 21 stations has at least one Figma frame.
- Affordance families are visually consistent (all MC stations look like MC; all reveal stations look like reveal).
- Substitution-pattern animation is mocked or video'd (the matched-slides-up requirement is design's responsibility now).
- Particle-burst radial menu is documented with the particle-color tokens.
- Trace canvas accessibility model is documented (touch-first, mouse fallback, keyboard "redo stroke").
- `<BottomCTA>` morph state machine is a 4-frame animation in the design.
- `<AutoAdvancePill>` countdown has a defined timing curve.
