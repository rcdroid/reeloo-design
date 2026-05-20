# v3 gap audit — Learn-stations

> Source: `/home/claw/work/learn-stations/` (49,739 LOC, the biggest source app)
> v1 spec: `/home/claw/work/reeloo-design/v1/products/learn-stations.md` (261 lines)
> Impl: `/home/claw/work/reeloo-v2/components/routes/practice.tsx` (545 lines)
> Headline: **v1 spec lists 18 station types but doesn't describe any of them.** The impl ships ONE generic "speaking drill" card and gestures at 12 hardcoded lesson titles. This is the largest spec-and-impl gap of the four apps.

The v1 spec is structurally right (routes, data model, four themes, sequential-drill state machine) but **never opens the 21 station tsx files**. Design got a list of names — not a station-by-station interaction inventory. The impl reflects this perfectly: one card UI, 18 missing UIs.

---

## 1. Feature inventory — STATION-BY-STATION

The crown jewel. `components/stations/` contains **21 distinct station components** (the v1 spec says 18; counting `intro`, `vocab-intro`, `tier-text`, `particle-burst`, `sequential-drill-station`, the actual surface is 21). Each is a distinct interaction with its own UX rhythm.

| # | Station | LOC | What the user does | Key file + lines |
|---|---|---|---|---|
| 1 | `intro` | 80 | Static intro card with vocab + audio + "Continue" CTA — first card of every lesson | `stations/intro.tsx:1-80` |
| 2 | `vocab-intro` | 105 | Surface a new word with reading, gloss, POS, and a play button before any drill | `stations/vocab-intro.tsx` |
| 3 | `mc` (multiple-choice) | 153 | Read a prompt, pick 1 of 4 options; correct/wrong feedback | `stations/mc.tsx` |
| 4 | `image-mc` | 95 | Look at an SVG illustration, pick the matching word from 4 options | `stations/image-mc.tsx` |
| 5 | `listen-mc` | 311 | Hear audio (Azure TTS), pick the matching word; 3-option layout, slow-replay button | `stations/listen-mc.tsx` |
| 6 | `cloze` | 375 | Fill-in-the-blank: sentence with a hole, pick from word chips | `stations/cloze.tsx` |
| 7 | `complete` | 58 | Lesson-end success card — score, streak bump, "Next lesson" CTA | `stations/complete.tsx` |
| 8 | `grammar` | 118 | Display a grammar rule + POS-colored example sentence (`JpPosColored`) | `stations/grammar.tsx` |
| 9 | `vocab-draw` | 108 | Vocab + freehand-trace warmup before a `trace` station | `stations/vocab-draw.tsx` |
| 10 | `reshuffle` | 251 | Drag-and-drop word tiles to reconstruct a sentence | `stations/reshuffle.tsx` |
| 11 | `substitution-grid` | 165 | Substitution grid — pick the replacement that fits the slot | `stations/substitution-grid.tsx` |
| 12 | `substitution-pattern` | 676 | Pattern-drill substitution: same template, cycling slot; matched value goes green + slides up; next slides from bottom | `stations/substitution-pattern.tsx` |
| 13 | `word-match` | 189 | Match words to translations (two columns, draw line / tap pair) | `stations/word-match.tsx` |
| 14 | `pattern-stack` | **1,258** | The pattern-stack drill — STT-primary syllabus drill; 3-phase substitution/transformation/extension (the **biggest** station component) | `stations/pattern-stack.tsx` |
| 15 | `tier-text` | 231 | Tier-conditional text variant rendering — same sentence, different complexity per tier | `stations/tier-text.tsx` |
| 16 | `trace` | 283 | Kanji/character trace — finger-trace the glyph stroke order on a canvas | `stations/trace.tsx` |
| 17 | `echo` | 397 | Listen → repeat (echo-method); STT scores against expected transcript; audio-level waveform | `stations/echo.tsx` |
| 18 | `sequential-drill` | **1,668** | The locked-design sequential drill — one card per slot, never overlap, state machine `card_load → tts → pause → listening_armed → scoring → card_correct\|card_retry → next_card`, smart-stop VAD, scoreTranscript scoring | `stations/sequential-drill.tsx:8-28` (locked rules) |
| 19 | `sequential-drill-station` | 51 | Thin wrapper that lets `sequential-drill` be used as a station via the `Station` dispatcher | `stations/sequential-drill-station.tsx` |
| 20 | `particle-burst` | 255 | JP particle drill — pick particle (を/に/で/が/は…) from a radial menu; particle-tinted feedback | `stations/particle-burst.tsx` |
| 21 | (variants) | — | Tier filtering reshuffles which stations appear in a lesson per `casual / modest / serious` (`app/[course]/[lesson]/page.tsx:51`) | — |

Each station has its OWN feedback state (correct/wrong/warning), its OWN affordance grammar (chips? draggable tiles? mic? touch canvas?), and its OWN telemetry shape. **None of this is in any design artefact.** v2's mockup is the `STATION_DATA` block in `design-source/project/routes/practice.jsx:205` — three hardcoded prompts, one shared UI.

### Surface: course picker (`/`)

| # | Feature | Source | Tier |
|---|---|---|---|
| 22 | 3-column grid of course cards (URL / Minna / Mundart / Deutsch) with accent color, brand chip, hero title (zh + en), lesson count, progress | `app/page.tsx`, `data/courses/index.ts:384-423` | composed |
| 23 | Pip mascot intro animation on first land (`demo/intro/page.tsx`) | `app/demo/intro/page.tsx` | composed |
| 24 | Session probe via `auth()` — fails open (anonymous OK) | `app/page.tsx:7` | atomic |
| 25 | Mascot avatar variants in `data/avatars.ts` (referenced from layout) | — | atomic |

### Surface: lesson list (`/<course>`)

| # | Feature | Source | Tier |
|---|---|---|---|
| 26 | Lesson rows with locked / unlocked state per `LessonStub.unlocked` | `app/[course]/page.tsx` | atomic |
| 27 | Per-course lesson counts: minna=40, mundart=50, deutsch=30, url=open-ended | `data/courses/index.ts:226, 306, 358, 382` | atomic |

### Surface: station player (`/<course>/l<N>?s=<station>`) — the central screen

| # | Feature | Source | Tier |
|---|---|---|---|
| 28 | URL-driven `?s=N` (1-indexed) station deep-link; back-syncs via `router.replace` | `app/[course]/[lesson]/page.tsx:58, 75` | composed |
| 29 | Auto-advance with 2s default countdown (cancellable) — `<AutoAdvancePill>` | `app/[course]/[lesson]/page.tsx:21`, `components/AutoAdvancePill.tsx` | composed |
| 30 | `<StationView>` dispatcher reads `data.type` and renders the right station component | `components/Station.tsx:80` | atomic |
| 31 | Tier filter — stations missing `tiers` default to all; tier change clamps station index, not resets to 0 | `app/[course]/[lesson]/page.tsx:50, 84` | composed |
| 32 | Stage state machine: `no-grade → grading → graded → next` orchestrated via `lib/stage.ts` — drives single `<BottomCTA>` morphing button | `components/BottomCTA.tsx`, `lib/stage.ts` | composed |
| 33 | `<Mascot>` (Pip) in idle/thinking/proud/concerned/encouraging/apologetic moods, lifted by station logic | `components/Mascot.tsx`, `data/avatars.ts` | composed |
| 34 | `<EtymologyBubble>` popover for word-origin info | `components/EtymologyBubble.tsx` | composed |
| 35 | `<DrillDensityBadge>` UI tag indicating how dense this lesson's drilling is | `components/DrillDensityBadge.tsx` | atomic |
| 36 | `<TierPicker>` — casual/modest/serious selector | `components/TierPicker.tsx` | atomic |
| 37 | `<RubyText>` — furigana renderer for JP | `components/RubyText.tsx` | atomic |
| 38 | `<JaText>` + `<RomajiOnly>` helpers (toggleable romaji visibility) | `components/JaText.tsx` | atomic |
| 39 | `<JpPosColored>` — POS-tinted Japanese sentence rendering (place/subject/object/verb/particle each get a distinct hue) | `components/JpPosColored.tsx`, tokens `globals.css:45-49` | composed |
| 40 | `<PlayTTSButton>` and `<PlaySlowTTSButton>` — two-speed TTS chips | `components/PlayTTSButton.tsx`, `PlaySlowTTSButton.tsx` | atomic |
| 41 | `<TTSDebugPanel>` + `<STTDebugPanel>` — dev-mode diagnostic panels (gated) | `components/TTSDebugPanel.tsx`, `STTDebugPanel.tsx` | composed |
| 42 | `<SettingsButton>` — opens settings modal | `components/SettingsButton.tsx` | atomic |
| 43 | `<Scene>` — illustration frame for image-mc and intro stations | `components/Scene.tsx` | atomic |
| 44 | TTS pipeline: first tries pre-built `/audio/<sha256>.mp3` + `.timing.json`, falls back to `POST /api/tts` (depot proxy) | `lib/tts.ts:165, 300` | composed |

### Surface: sequential-drill — the locked-design crown jewel

| # | Feature | Source | Tier |
|---|---|---|---|
| 45 | Locked state machine: `card_load → tts → pause → listening_armed → scoring → card_correct\|card_retry → next_card` | `stations/sequential-drill.tsx:8-28` | multi-step flow |
| 46 | Rule: ≤1 sound + ≤1 visual change + ≤1 action at any moment | (same block) | — |
| 47 | Rule: TTS and STT **never overlap** | (same) | — |
| 48 | Rule: Wrong = silence + transcript shown. **Never red flash, never wrong-buzz haptic.** | (same) | — |
| 49 | **Smart-stop VAD**: 15s hard ceiling / 8s if no voice / 4s silence after voice / 1.2s after match ≥0.7 / immediate stop on ≥0.99 | `stations/sequential-drill.tsx:43-52` | composed |
| 50 | Scoring: 60% slot match + 40% full-frame match; pass at 0.7 | `:55-57`, `lib/score-transcript.ts` | composed |
| 51 | 300ms `pre_mic_pause` before mic-on | `:` pipeline | atomic |
| 52 | Chord SFX on pass (audio reward) | pipeline | atomic |
| 53 | `recordEvent()` telemetry call per attempt (transcript, stt_confidence, accuracy, reaction_ms) | `lib/learning-telemetry.ts:69` | atomic |

### Surface: /cards Anki deck (FLASHCARD surface #1)

| # | Feature | Source | Tier |
|---|---|---|---|
| 54 | `?lang=ja|de|gsw` deck selection; `?lesson=<key>` optional scope | `app/cards/page.tsx:25` | atomic |
| 55 | SVG illustration on front, surface word + auto-TTS on mount | `app/cards/CardsDeck.tsx` | atomic |
| 56 | Flip → back: meaning + romaji (toggle via `useShowRomaji`) + example sentence + pipLine | `CardsDeck.tsx` | composed |
| 57 | SRS state via `useDeckState.ts` — local-first, then `PUT /api/cards/state` if signed in | `useDeckState.ts`, `app/api/cards/state/route.ts` | composed |
| 58 | `lib/srs.ts` SM-2 algorithm + tests | `lib/srs.ts`, `srs.test.ts` | atomic |
| 59 | File-backed SRS at `data/cards-state/<userId>.json` with atomic tmp+rename writes | `app/api/cards/state/route.ts:32` | atomic |
| 60 | User ID from `X-Forwarded-User` (oauth2-proxy) or `?userId=anon-XXXX` localStorage fallback | `route.ts:39-42` | atomic |
| 61 | Front audio: pre-built or live TTS | `lib/tts.ts:165, 300` | atomic |
| 62 | `cards-autotts-toggle`, `cards-dict-link`, `cards-example-play` testids — full card affordance set | `data-testid` scan | atomic |
| 63 | `cards-grade-feedback` + `cards-grade-panel` — flip-with-grade UX | — | atomic |

### Surface: STT pipelines

| # | Feature | Source | Tier |
|---|---|---|---|
| 64 | `POST /api/stt-azure` — Azure Speech-to-Text REST: webm/opus → ogg/opus ffmpeg transcode, then Azure REST | `app/api/stt-azure/route.ts:40-63, 106` | multi-step flow |
| 65 | `POST /api/stt-gcp` — Google Cloud Speech v1 alt path; SA at `~/.config/gcloud/reeloo-media-sa.json`; billing-pending fallback | `app/api/stt-gcp/route.ts:35, 89, 142` | multi-step flow |
| 66 | Phonetic-distance scorer (`lib/phonetic-distance.ts` + `score-transcript.ts`) — tolerates near-matches | `lib/score-transcript.ts` | atomic |

### Surface: theming + design system

| # | Feature | Source | Tier |
|---|---|---|---|
| 67 | **Four themes** wired via `data-theme` on `<html>`: vanilla / pip-duo / solarized-dark / high-contrast | `globals.css:149-277` | composed |
| 68 | Pre-hydration theme-init script (read localStorage before paint to avoid FOUC) | `app/layout.tsx:84-89` | atomic |
| 69 | WCAG-AA test enforcement — `lib/design-tokens.test.ts` verifies all theme combos | `lib/design-tokens.test.ts` | atomic |
| 70 | POS color tokens (`--pos-place / --pos-subject / --pos-object / --pos-verb / --pos-particle`) | `globals.css:45-49` | atomic |
| 71 | TTS-highlight tokens (`--tts-highlight-bg #daa520` goldenrod / `--tts-highlight-fg`) | `globals.css` | atomic |
| 72 | Graded-state tokens (`correct` olive, `wrong` maroon, `warning` brown) | `globals.css:21-35` | atomic |

### Surface: cross-cutting

| # | Feature | Source | Tier |
|---|---|---|---|
| 73 | `<FlagIssue>` floating chip on every page, modal, POST `/api/flag-issue` → JSONL append | `components/FlagIssue.tsx`, `app/layout.tsx:106`, `app/api/flag-issue/route.ts` | composed |
| 74 | Telemetry: `POST /api/telemetry/learning` → append-only `data/learning-events.jsonl` | `app/api/telemetry/learning/route.ts`, `lib/learning-telemetry.ts:69` | atomic |
| 75 | Auth: Google OAuth via NextAuth 5.0 beta; X-Forwarded-User header propagation in production | `auth.ts:1-22`, `app/auth/signin/page.tsx` | composed |
| 76 | Settings page: theme + tier + locale + romaji toggle | `app/settings/page.tsx` | composed |

---

## 2. What v1 spec captured

- §3 route map (full, correct)
- §4 data model — the discriminated union, lesson catalogue counts, telemetry shape, deck SRS state path
- §5 Flow B (sequential-drill) — captured the locked-design rules and VAD numbers verbatim. **This is the only station with proper documentation.**
- §6 design tokens — all four themes captured with hex values
- §10 the sequential-drill craftsmanship pitch

---

## 3. What v1 spec MISSED — this is the big one

v1 names "18 station kinds" in §4 (line 108) but **does not describe a single one**. Nothing in v1 says what `cloze` looks like, what `reshuffle` lets you drag, what `particle-burst` displays, what `trace` accepts as touch input. The taxonomy is a flat enum without behavioural shape.

Concretely missing from v1:

- **Per-station interaction inventory** — what does the user see and touch in each of the 21 station tsx files? v1 has zero coverage. (See Phase C requests 03–08 for cluster grouping.)
- **Per-station accessibility model** — `reshuffle` is drag-drop (keyboard equivalent?); `trace` is touch (mouse fallback?); `particle-burst` is radial (tab order?); `cloze` chips (arrow nav?). None spec'd.
- **Per-station empty/loading/error states** — `listen-mc` shows what if audio fails to fetch? `echo` shows what if STT 5xxs? `trace` shows what if the kanji has no stroke data? None covered.
- **Per-station scoring rule** — `mc` is binary; `cloze` is binary; `echo` is phonetic-distance with a threshold; `sequential-drill` is 60/40 slot/full match; `pattern-stack` is multi-pass. v1 only mentions sequential-drill's.
- **`<BottomCTA>` morphing button states** — `lib/stage.ts` plus `components/BottomCTA.tsx` is referenced. v1 doesn't show the four-state morph. Design needs the state diagram.
- **`<AutoAdvancePill>` countdown UI** — 2s cancellable countdown. v1 §3 mentions auto-advance time; doesn't show the chip design.
- **`<EtymologyBubble>` popover** — referenced but no design specified.
- **`<DrillDensityBadge>`** — UI tag, no spec.
- **`<JpPosColored>` legend** — the POS-color token is in v1 §6, but the legend (when does the user learn what blue means?) isn't.
- **`<Mascot>` (Pip) 6-mood spritesheet** — v2 mentioned the 6 moods, but the **per-station mood mapping** (when does encouraging fire? when concerned?) isn't anywhere.
- **`<RubyText>` typography** — furigana positioning, font scale relative to base, spacing rules.
- **`<Scene>` illustration frame** — what does the frame look like? Aspect ratio? Card vs full-bleed?
- **`<TTSDebugPanel>` / `<STTDebugPanel>`** — these exist; are they user-visible or dev-only? If user-facing, what does the user see?
- **`<TierPicker>` UI** — three-button toggle or dropdown?
- **/cards full back-of-card layout** — meaning + romaji + example + audio + dict-link + grade panel. v1 gestures; impl shipped a different shape.
- **/cards grade panel** — `cards-grade-feedback` and `cards-grade-panel` testids exist; v1 §5 Flow C just says "flip-card."
- **Sequential-drill `<MeasuredSlot>`** — the 1668-line component has internal sub-components for measuring slot-width / TTS-status / mic-armed. Design needs to know the slot is the unit, not the card.
- **`particle-burst.tsx`** — radial particle picker is unique JP-pedagogy UI. Worth its own spec.
- **`pattern-stack.tsx` (1258 lines)** — STT-primary syllabus drill. Was a 7-cycle co-evolution per memory `project_pattern_stack_drill.md`. Design has no mockup.
- **`substitution-pattern.tsx` (676 lines)** — the "matched value goes green + slides up; next slides in from bottom" rule (per memory `reference_readalong_drill_patterns.md`). User has repeated this requirement 10× — must be in the design spec.
- **Theme `pip-duo` (Duolingo-green primary)** — v1 mentions it exists, but the v2 brief locked to vanilla/cobalt-dark/high-contrast. Decision needed: drop `pip-duo` or add it back?
- **Stage transitions** — `card_load` to `next_card` is the macro. The micro is: how does the card slide out? What's the transition? The 460ms / 240ms / 120ms durations are tokens but the *which-token-for-what* mapping isn't there.

---

## 4. What the impl skipped

Against `reeloo-v2/components/routes/practice.tsx` (545 lines):

| v1-spec'd or source-app feature | In impl? |
|---|---|
| 21 station types | NO — one generic "card" UI |
| Per-station audio playback | NO — `playing` is a useState bool |
| Real TTS pipeline | NO |
| Real STT pipeline | NO |
| Sequential-drill state machine + VAD smart-stop | NO |
| `<RubyText>` furigana | NO |
| `<JpPosColored>` POS coloring | NO — example sentence is plain |
| `<AutoAdvancePill>` countdown | NO |
| `<BottomCTA>` morphing button | PARTIAL — has "Not yet" / "Got it" but no morph |
| /cards Anki deck | NO — `/practice/flashcards` route is hardcoded in shell, no implementation in routes |
| SRS state + SM-2 | NO |
| `<EtymologyBubble>` | NO |
| `<DrillDensityBadge>` | NO |
| `<TierPicker>` | NO |
| `<FlagIssue>` global chip | UNKNOWN — not in `routes/practice.tsx`; check shell |
| Theme system (4 themes) | PARTIAL — vanilla theme exists; no high-contrast, no pip-duo |
| Course picker with progress + weak topics | PARTIAL — hardcoded `COURSES` with progress/weak fields, but no signal-driven re-ordering |
| Lesson list with locked/unlocked | PARTIAL — `state: i < 2 ? "done" ...` hardcoded |
| `?s=<station>` deep-link + auto-advance | NO |
| 12 lesson titles | YES — but `LESSON_TITLES` is hardcoded (`practice.tsx:110`) |
| Level-mismatch dialog | YES — `LevelMismatchDialog` imported and wired |
| Speed ladder | PARTIAL — turtle/walk/bunny 0.7×/1.0×/1.25× exists; different from podcast's 0.6/0.75/1.0 ladder |
| Pip mascot integration | YES — `PipBay` and `PipInfoButton` are imported; mood mapping exists |
| `make-video` CTA | YES — wired to `makeVideo()` |
| Settings page | UNKNOWN — not in `routes/practice.tsx` |

**Score: 4 of 21 station UIs ported (intro-like generic card + complete-like grade panel + lesson-list-as-TOC + a course picker). 80%+ of the surface area is missing.**

---

## 5. What's actually pretty good

- `<PipBay>` + `<PipInfoButton>` is the right primitive — moods map cleanly to `pipMood`, the bay/info distinction matches v2's proactive/passive split.
- `<LevelMismatchDialog>` is wired with all three escape hatches (condensed / free-nav / silence).
- Course-picker progress bars + weak-topic display is good IA — preserves the v1 signal-aware design intent.
- Test-id naming follows the `<surface>:<component>:<element>` convention (e.g. `practice-station:speaking-drill:tts-button`) — internally consistent. (Note: source learn-stations uses flat kebab-case, so this is a NEW convention being introduced.)
- The 38% progress bar at the top of the station screen is the right shape for a multi-station lesson; just needs to wire to real station-index state.
- `data-screen-label` on the wrapper is a useful a11y/test affordance and is consistently applied.
