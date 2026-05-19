# Reeloo — Unified Design Brief (v1, source-grounded)

> v0.5 was an unscoped speculative brief (`00-v0.5-seed-brief.md`, 221 KB). v1 (this document) is grounded in actual source code — every claim cites a file:line.
> Per-app evidence: `v1/products/{podcast,learn-stations,flashcard,readalong,book-app}.md`.
> Token evidence: `v1/design-system/{current-state,proposed-tokens}.md`.

---

## 1. Executive summary

Reeloo's surface today is **five Next.js apps** that share a vision (learn languages by reading + listening + speaking through your daily content) but share almost no code. After auditing every source tree:

- **podcast-app** is a thin 8-route demo viewer (no DB, no auth) for a single high-craft player.
- **book-app** is a static-rendered EPUB reader with inline translation + 6-category annotation, audio in GCS.
- **learn-stations** is a 30+30+40+50 lesson catalogue with 18 drill-station kinds, the most token-disciplined theme system, NextAuth+Google, and a working sequential-drill speaking pipeline.
- **readalong** is the 134-component / 119-route workshop where new ideas land first (deck generation from URL, persona drill tutor, scenario sim, word-rain game, MP4 export). Firestore-backed multi-user.
- **flashcard** is a *category* not an app — three independent implementations (notes/practice, learn-stations/cards, readalong/decks) that all show vocab cards but share no contract.

**The opinionated recommendation:** keep **three** apps as distinct domains; merge two pairs; promote the most-used contracts to shared services.

### Three kept apps, two merges, three services

| Decision | Why |
|---|---|
| **KEEP** `podcast.reeloo.ai` as a standalone CDN-friendly read-only viewer | The 2,442-line `KarottenPlayerV3` is a focus-pin column optimised for ear-first listening at 0.6–1.0×. Different posture from text reading. Already ships at the right shape (zero DB / auth). |
| **KEEP** `book.reeloo.ai` as a force-static EPUB reader | The reading column (`max-w-720px`, IBM Plex Serif 19px, 1.75 line-height) is text-first. Two reader modes (Page / Deck) are honest about the eyes-vs-eyes+ears tradeoff. Force-static + GCS audio + IndexedDB uploads is the right architecture. |
| **KEEP** `learn-stations` ("Pip") as the active-drill surface, **rename to** `practice.reeloo.ai` | The 18-station discriminated-union taxonomy + 4-theme token system + Azure STT pipeline are the most mature pedagogy assets in the codebase. The brand mascot "Pip" should travel into the unified marketing. |
| **MERGE** notes `/practice/flashcards` + learn-stations `/cards` into `practice.reeloo.ai/library/cards` | Both surface vocab cards; one has illustrations (notes), one has SRS + audio + decks (learn-stations); neither has all of it. Consolidate at the strongest surface. |
| **MERGE** readalong → split across the three kept apps | Deck-as-presentation goes into `practice.reeloo.ai` as a new station type. Deck-generation tooling (URL → deck) goes into the operator's admin path. Persona-drill voice merges with learn-stations sequential-drill. Word-rain + scenario-sim land in `practice.reeloo.ai/games/`. Multi-user Firestore stays for the admin/operator. |
| **PROMOTE** `dictionary.reeloo.ai/api/dictionary` to be consumed by all five surfaces | Already exists and works; book-app and learn-stations are the holdouts. |
| **PROMOTE** `/audio/<sha256>.mp3 + .timing.json` to a shared `audio.reeloo.ai` service | Already exists in two implementations (learn-stations build-time + book-app GCS); consolidate behind one URL pattern. |
| **PROMOTE** the depot LLM gateway `${DEPOT_URL}/api/llm` to be the only LLM client | Already wired from book-app + notes; readalong should drop its direct Gemini calls and route through here for quota uniformity. |

### Defence of the recommendation

The brief asked for one unified app. I am proposing three. Here's why:

1. **Posture matters.** Reading a book and listening to a podcast are different *physical* postures (eyes-only vs ears-first). The two existing surfaces already make different layout decisions — `max-w-720px` reading column vs 42%-pinned focus column — because the user is doing different things with their body. Merging them into one route tree forces a chimera that serves neither posture well.

2. **The active-drill surface is the gravity well.** Vocab introduced in the podcast or the book lands in `practice.reeloo.ai` for retention. That's the operator's actual workflow; we should reinforce it instead of fighting it. Hence the rename.

3. **Readalong is the right place to put the operator's R&D, not the user-facing app.** 134 components and 119 API routes are too many for a public surface; let it stay as the studio (`/admin/...` + `/test/...` + `/test/drill-engine/...` already document that intent). Carve out the three user-facing wins (deck-presentation, word-rain, scenario-sim) and ship them through the user-facing apps.

4. **Shared services beat shared apps.** A typography/token shared package + a dictionary contract + an audio service is enough. We don't need a monorepo with one Next.js app — we need three Next.js apps with one design system and three backend contracts.

---

## 2. Information architecture (unified route tree)

### Public surfaces

```
reeloo.ai/                                # marketing landing (TBD; out of scope here)
├── podcast.reeloo.ai/                    # 4 routes
│   ├── /                                 # curated catalog (language tabs + cache-hit tiles)
│   ├── /podcast/<slug>                   # focus-pin player (V3 default; ?layout=v1 opt-in)
│   ├── /wizard                           # on-demand dialogue generation (depot proxy)
│   └── /library                          # NEW — bookmarks + recent
│
├── book.reeloo.ai/                       # ~5 routes
│   ├── /                                 # DropZone + curated BookCards
│   ├── /read/<slug>                      # TOC
│   ├── /read/<slug>/ch/<n>?mode=&target= # Page or Deck reader
│   ├── /search?q=&slug=                  # full-text flexsearch
│   └── /library                          # NEW — uploaded books (IndexedDB-backed)
│
└── practice.reeloo.ai/                   # the active-drill app (formerly learn-stations)
    ├── /                                 # course picker (Pip mascot)
    ├── /<course>                         # lesson list (minna / mundart / deutsch / url)
    ├── /<course>/l<N>?s=<station>        # station player — 18 station kinds (the heart)
    ├── /library/                         # NEW — unified card / deck surface
    │   ├── /cards                        # Anki-style deck (merge of learn-stations/cards)
    │   ├── /cards/random                 # illustrated discovery loop (merge of notes/practice/flashcards)
    │   └── /decks/<id>                   # deck-as-presentation (from readalong/deck/[id])
    ├── /games/                           # NEW (carved from readalong)
    │   ├── /word-rain                    # Pixi.js falling-words
    │   └── /scenario                     # branching-dialogue sim
    ├── /daily-review                     # SRS review session (from readalong)
    ├── /settings, /auth/signin, /profile # auth + prefs
    └── /api/                             # STT, TTS, cards-state, telemetry, dictionary proxy (thin)
```

### Operator surfaces (internal / auth-gated)

```
studio.reeloo.ai/  (or admin.alpsmind.com for internal)
├── /deck-generator                       # URL → deck (from readalong url-deck-queue)
├── /admin/*                              # the rest of readalong's 8 admin pages
└── /test/*                               # drill-engine + unit-generate tests
```

### Shared services (under `*.reeloo.ai` or internal domain)

```
dictionary.reeloo.ai/api/dictionary?word=&lang=    # EXISTS — used by readalong + notes
audio.reeloo.ai/<hash>.mp3 + <hash>.timing.json    # NEW — promote learn-stations + book-app GCS patterns
depot.reeloo.ai/api/{llm,tts,wizard/generate,podcast-audio/<hash>.mp3}   # EXISTS — X-App-Token
```

---

## 3. Personas & journeys

The single user today is the **immigrant-Mandarin operator** (Switzerland, learning German + Japanese — memory `user_profile.md`). Four cross-app journeys are the load-bearing ones.

### J1. Morning podcast on a phone, weak hands

User opens **podcast.reeloo.ai** on the way to work. Selects a German tile. The focus-pin column auto-plays at 1.0×. They hit a word they don't know, tap it — `seekToWord` rewinds 0.5s, the annotation rail slides in showing the zh-TW gloss. They tap "save" — the word lands in `practice.reeloo.ai/library/cards/random` queue.

**Unified routes:**
- entry: `podcast.reeloo.ai/podcast/<slug>`
- save action: POST `practice.reeloo.ai/api/library/save` `{word, lang, context: {podcastSlug, segmentN}}`
- new card visible at: `practice.reeloo.ai/library/cards/random?lang=de`

**File evidence:** segment + word interaction at `podcast-app/components/KarottenPlayerV3.tsx:828–887`; rail at `:807–813`.

### J2. Evening reading on a tablet, eyes-only

User opens **book.reeloo.ai/read/atomic-habits.../ch/1** in Page mode. Each paragraph has zh-TW translation below; an underlined idiom expands to a popover gloss on tap. After 20 minutes they hit a long word and switch to Deck mode — the Azure TTS reads the paragraph while words highlight in karaoke. They save three glossary entries.

**Unified routes:**
- entry: `book.reeloo.ai/read/<slug>/ch/<n>`
- mode toggle: `?mode=deck` (URL-driven, no JS state)
- save action: same `practice.reeloo.ai/api/library/save` endpoint with `{context: {bookSlug, chapterId, paragraphId}}`
- next morning: those words appear in the SRS deck at `practice.reeloo.ai/library/cards?lang=en&recent=24h`

**File evidence:** Page mode at `book-app/components/PageReader.tsx:82–122`; Deck mode at `book-app/components/DeckReader.tsx`; reader prefs at `book-app/lib/reader-prefs.ts:13–25`.

### J3. Active speaking drill before lunch

User opens **practice.reeloo.ai/minna/l8?s=1**. Hits the sequential-drill station. The state machine drives `tts → 300ms pause → mic on → VAD smart-stop → STT → score → next` (`learn-stations/components/stations/sequential-drill.tsx:53–57`). They get 8/10 right, the chord SFX fires on each, the lesson auto-advances to the next station (cloze, image-mc, etc.). Telemetry lands in `data/learning-events.jsonl`.

**Unified routes:**
- entry: `practice.reeloo.ai/<course>/l<N>?s=N`
- mid-flow events: append-only to `practice.reeloo.ai/api/telemetry/learning`
- exit: `?s=last` → `complete` station → success animation

**File evidence:** station discriminated union at `learn-stations/data/stations.ts:67–412`; drill state machine at `learn-stations/components/stations/sequential-drill.tsx:79–98`.

### J4. Weekly SRS review

User opens **practice.reeloo.ai/library/cards?lang=de**. SM-2 SRS picks 30 due cards from the union of (saved from podcast J1) + (saved from book J2) + (vocab introduced in practice J3). Each card flips on tap; "got it" / "missed" updates `easeFactor` + `nextReviewAt` (`learn-stations/types/drill.ts:48–51`). Audio plays on flip; illustrations from the Flux LoRA appear for words that have them (`notes/app/api/dictionary-illustration/route.ts`).

**Unified routes:**
- entry: `practice.reeloo.ai/library/cards?lang=de`
- state read: `GET practice.reeloo.ai/api/cards/state`
- state write: `PUT practice.reeloo.ai/api/cards/state` (atomic write, `learn-stations/app/api/cards/state/route.ts:1–25`)
- audio: `audio.reeloo.ai/<sha256>.mp3`
- illustration: `dictionary.reeloo.ai/illustration?word=&lang=` (NEW endpoint promotion)

---

## 4. Visual system

> Full token evidence: `v1/design-system/proposed-tokens.md`. Summary here.

### Color (Phase B sources cited)

**Light theme** (default):
- bg `#fdf6e3` (Solarized base3 — `learn-stations:58`)
- container `#ffffff` · elevated `#fffcf0`
- text `#073642` · muted `#586e75` · subtle `#657b83`
- primary `#cb4b16` (Solarized orange — universal across 3/5 apps)
- border `#8a8670`

**Cobalt-dark** (audio-cinematic, podcast default):
- bg `#0f1419 → #1a2332` linear-gradient (podcast `app/page.tsx:188`)
- text `#e8eef5` · muted `#a8b8c8`
- primary `#cb4b16` (unchanged)
- border `#7a8e93`

**High-contrast** (a11y):
- bg `#000000` · primary `#ffaa00` · border `#6b7280`
- All ratios verified ≥ 4.5:1 (learn-stations `:263–277`)

**Annotation palette** (shared across podcast, book, readalong):
- `--anno-vocab #268bd2` (blue, solid)
- `--anno-grammar #859900` (green, solid)
- `--anno-idiom #6c71c4` (violet, solid)
- `--anno-cultural #cb4b16` (orange, solid; alias `--anno-place`)
- `--anno-archaism #b58900` (yellow, **dotted**; book-app only)
- `--anno-proper-noun #268bd2` (alias of vocab)

**State colors** (from `learn-stations` graded-state tokens — only app with AA-verified ratios):
- correct olive `#eaf3c2 / #2d3f00`, wrong maroon `#fbdcd9 / #6e0a08`, warning brown `#fdf3cf / #5c4400`

**TTS karaoke highlight** (sourced):
- bg `#daa520` (goldenrod) · fg `#073642` · ring `#cb4b16`

### Typography

- **Sans:** IBM Plex Sans (loaded via `next/font/google`, weights 400/500/600/700). Source: podcast-app `layout.tsx:8`, book-app `layout.tsx:4`.
- **Serif:** IBM Plex Serif (weights 400/500/600/700). Source: same.
- **Mono:** IBM Plex Mono (weights 400/500/600). Source: podcast-app `layout.tsx:21`.
- **CJK:** Noto Sans TC (weights 400/500/700; `preload: false`). Source: podcast `layout.tsx:27`, learn-stations `:26`.
- **JP serif:** Noto Serif JP (furigana over kanji). Source: learn-stations `:19`. Conditionally loaded.

**Size scale** (sourced):
- 11 / 13 / 15 / 19 / 22 / 30 / 34 / 38 (px) — see `proposed-tokens.md` for source citations.
- 19px is the canonical reading-column body (book-app PageReader). 34px is the focus-pin active line (podcast). 38px is the deck-mode reader.

**Line-height + tracking:**
- `--reading-line-height 1.75` (book-app `globals.css:16`)
- UI uppercase tracking `0.06em – 0.10em` (podcast `CatalogClient.tsx:303`)
- Heading tight `-0.01em` (learn-stations `globals.css:104`)

### Spacing + radius

- Page padding `32 / 48px`; reading-column `max-w-720px`; catalog `max-w-1000px`.
- Cards `border-radius 12–16px`; pills `999px`; buttons `8px`.
- Touch-target minimum `44–48px`.

### Shadow

- `--shadow-card 0 1px 2px rgba(0,0,0,0.04)` (notes flashcards `:183`)
- `--shadow-elevated` — INVENTED layered shadow for deck-card raise
- `--shadow-focus 0 0 0 2px var(--color-border-strong)` — INVENTED; replaces ad-hoc `outline` rules

### Motion

- `--dur-fast 120ms`, `--dur-med 240ms`, `--dur-slow 460ms`
- `--ease-out cubic-bezier(0.22, 1, 0.36, 1)`
- All sourced from `podcast-app/KarottenPlayerV3.tsx:1482–1484`. Book-app's `deck-in` keyframe (`globals.css:47–56`) already uses 460ms.

### Dark-mode strategy

`document.documentElement.dataset.theme` with three values: `vanilla` (default light), `cobalt-dark`, `high-contrast`. Pre-hydration `<script>` reads `localStorage.getItem('reeloo.theme')` (pattern from `learn-stations/app/layout.tsx:84–89`).

Podcast surfaces default to `cobalt-dark`; book + practice default to `vanilla`. User preference overrides.

---

## 5. Component inventory (target — 18 components)

| Component | Purpose | Apps that need it | Existing implementations that merge |
|---|---|---|---|
| `<ReelooLogo>` | Wordmark + dot | all 3 | new |
| `<MediaCard>` | Card with cover/gradient + emoji + title + meta + CEFR chip | podcast, book, practice library | `podcast-app/CatalogClient.tsx:130`, `book-app/BookCard.tsx`, `readalong/DeckList.tsx` |
| `<MediaLibrary>` | Lang tabs + grid + recent row + filter pills | podcast, book, practice library | `podcast-app/CatalogClient.tsx`, `readalong/LibraryViewPage.tsx`, `notes/app/podcast/_components/CatalogClient.tsx` |
| `<RecentRow>` | localStorage-backed recently-opened strip | podcast, book, practice | `podcast-app/RecentRow.tsx` (already canonical) |
| `<ProgressModal>` | Animated waiting spinner with status text | podcast, book (upload), practice | `podcast-app/ProgressModal.tsx` (canonical) |
| `<MediaPlayerBar>` | play/pause + ±15s + scrub + speed (🐢🚶🐇) + CC + replay | podcast, book deck mode | merge `KarottenPlayerV3` `.kt3-bar` block + book-app `DeckPlayerBar.tsx` |
| `<FocusPinColumn>` | Vertical 42%-pinned segment column with active-line typography | podcast | `KarottenPlayerV3.tsx` (extract from the 2,442 lines) |
| `<ReadingColumn>` | `max-w-720px`, Plex Serif 19px, line-height 1.75, inline annotations | book Page mode | `book-app/PageReader.tsx` (canonical) |
| `<DeckCard>` | Card-mode raised reader with audio bar + annotation rail | book Deck mode, practice library/decks | `book-app/DeckReader.tsx` + readalong `CardLayout.tsx` |
| `<AnnotatedSpan>` | Underlined word/phrase with popover gloss | podcast, book, practice | `book-app/AnnotationPopover.tsx`, `readalong/AnnotationPanel.tsx`, podcast inline |
| `<AnnotationRail>` | Right-side panel with numbered notes per segment/paragraph | podcast, book | `KarottenPlayerV3` rail + `book-app/DeckAnnotationRail.tsx` |
| `<WordKaraoke>` | Word-boundary-driven highlight on TTS playback | podcast (active line), book Deck mode, practice cards | `KarottenPlayerV3` word-span, `book-app/lib/word-timing.ts` + `DeckPlayerBar.tsx`, `readalong/WordHighlightedText.tsx` |
| `<LanguageTabs>` | Horizontal pills with flag + label + count, with active styling | podcast, book, practice | `podcast-app/CatalogClient.tsx:101–117` (canonical) |
| `<StationRunner>` | The 18-station discriminated-union dispatcher with auto-advance + stage CTA | practice | `learn-stations/components/Station.tsx:80` (canonical) |
| `<SpeakingDrill>` | TTS → pause → mic → VAD → STT → score loop | practice | `learn-stations/components/stations/sequential-drill.tsx` (canonical) |
| `<Flashcard>` | Front (illustration + surface) / back (gloss + audio + example) flip | practice library | merge `learn-stations/cards/CardsDeck.tsx` + `notes/app/practice/flashcards/page.tsx` |
| `<DictionaryCard>` | Word + readings + POS + level + senses + examples + audio button | book popover, practice card-back, podcast rail | `readalong/DictionaryCard.tsx` (canonical) |
| `<ThemeProvider>` | data-theme attribute + pre-hydration script + localStorage sync | all 3 | adopt `learn-stations/app/layout.tsx:84–89` pattern |

Plus shared primitives: `<Button>`, `<Pill>`, `<Toggle>`, `<TextInput>`, `<Select>`, `<Slider>`, `<FlagIssue>` (already canonical in learn-stations).

---

## 6. Migration map

> Format: `existing-route → new-route` (or `kept` if unchanged).

### Podcast (kept)
- `podcast-app/`               → `podcast.reeloo.ai/`                              kept
- `podcast-app/wizard`         → `podcast.reeloo.ai/wizard`                        kept
- `podcast-app/podcast/<slug>` → `podcast.reeloo.ai/podcast/<slug>`                kept
- `podcast-app/api/wizard`     → `podcast.reeloo.ai/api/wizard`                    kept (proxy)
- `podcast-app/api/tts`        → `audio.reeloo.ai/tts`                             promoted
- `podcast-app/api/audio/<h>`  → `audio.reeloo.ai/<h>.mp3`                         promoted

### Book (kept, with library section added)
- `book-app/`                              → `book.reeloo.ai/`                  kept
- `book-app/search`                        → `book.reeloo.ai/search`            kept
- `book-app/read/<slug>`                   → `book.reeloo.ai/read/<slug>`       kept
- `book-app/read/<slug>/ch/<n>`            → `book.reeloo.ai/read/<slug>/ch/<n>` kept
- `book-app/api/import-epub`               → `book.reeloo.ai/api/import-epub`   kept
- `book-app/api/annotate`                  → `depot.reeloo.ai/api/llm` (proxied) consolidate

### Practice (renamed from learn-stations)
- `learn-stations/`                        → `practice.reeloo.ai/`              renamed
- `learn-stations/<course>`                → `practice.reeloo.ai/<course>`      kept
- `learn-stations/<course>/l<N>`           → `practice.reeloo.ai/<course>/l<N>` kept
- `learn-stations/cards`                   → `practice.reeloo.ai/library/cards` moved
- `learn-stations/settings`                → `practice.reeloo.ai/settings`      kept
- `learn-stations/test/*`                  → `studio.reeloo.ai/test/*`          moved (internal)
- `learn-stations/demo/intro`              → `practice.reeloo.ai/onboarding`    renamed
- `learn-stations/api/stt-azure`           → `stt.reeloo.ai/azure` (NEW shared)  promoted
- `learn-stations/api/stt-gcp`             → `stt.reeloo.ai/gcp`                  promoted
- `learn-stations/api/cards/state`         → `practice.reeloo.ai/api/cards/state` kept
- `learn-stations/api/auth/[...nextauth]`  → `practice.reeloo.ai/api/auth/...`   kept

### Notes flashcards (merged into practice)
- `notes/practice/flashcards`              → `practice.reeloo.ai/library/cards/random`        merged
- `notes/api/dictionary-illustration`      → `dictionary.reeloo.ai/illustration?word=&lang=`  promoted
- `notes/api/dictionary-illustration/random` → `dictionary.reeloo.ai/illustration/random`     promoted

### Readalong (split across the kept apps)
- `readalong/`                             → `practice.reeloo.ai/library/decks`               moved (deck-as-library)
- `readalong/browse`                       → `practice.reeloo.ai/library/decks/browse`        moved
- `readalong/deck/<id>`                    → `practice.reeloo.ai/library/decks/<id>`          moved
- `readalong/drill`                        → MERGE INTO `learn-stations/<course>/l<N>?s=sequential-drill` (the same station; expand input model to accept persona-driven prompts) |
- `readalong/daily-review`                 → `practice.reeloo.ai/daily-review`                kept
- `readalong/game/word-rain`               → `practice.reeloo.ai/games/word-rain`             moved
- `readalong/game/scenario`                → `practice.reeloo.ai/games/scenario`              moved
- `readalong/scenario-sim`                 → `practice.reeloo.ai/games/scenario`              merged
- `readalong/library`                      → `practice.reeloo.ai/library/decks` (alias)        moved
- `readalong/dictionary`                   → `dictionary.reeloo.ai` (promoted external surface) moved
- `readalong/admin/*`                      → `studio.reeloo.ai/admin/*`                       moved (operator-only)
- `readalong/test/*`                       → `studio.reeloo.ai/test/*`                        moved
- `readalong/api/dictionary[/*]`           → `dictionary.reeloo.ai/api/dictionary[/*]`        promoted
- `readalong/api/drill/*` (15 routes)      → split: generation→`studio.reeloo.ai`, mastery+history+units→`practice.reeloo.ai` |
- `readalong/api/synthesize, tts-cache, voices` → `audio.reeloo.ai/{synthesize,cache,voices}` promoted |
- `readalong/api/speech-to-text`           → `stt.reeloo.ai/`                                 promoted
- `readalong/api/decks/*`                  → `practice.reeloo.ai/api/library/decks/*`         moved
- `readalong/api/export/*`, jobs           → `studio.reeloo.ai/api/export`                    moved
- `readalong/api/admin/*`                  → `studio.reeloo.ai/api/admin/*`                   moved

---

## 7. In scope / out of scope for the design pass

### In scope

- **Visual tokens.** `tokens.json` for the unified palette, typography scale, spacing, radius, shadow, motion, all three themes.
- **18-component Figma library** mapping to the table in §5. Each component documented with: anatomy, states (default / hover / active / focus / disabled / loading / error), responsive breakpoints (mobile / tablet / desktop), token references, and Code Connect mapping to the React source.
- **Mockups for 12 canonical screens** (4 per kept app):
  - `podcast.reeloo.ai`: catalog grid, V3 player (focus state), wizard form, annotation rail open
  - `book.reeloo.ai`: home with DropZone, TOC, Page mode reader with annotations, Deck mode reader with karaoke
  - `practice.reeloo.ai`: course picker, station player (sequential-drill mid-listen), library/cards (front + back), daily-review summary
- **Dark + a11y theme variants** for the 4 most-used components: `<MediaPlayerBar>`, `<FocusPinColumn>`, `<ReadingColumn>`, `<Flashcard>`.
- **Iconography contract.** Decide between Lucide (already in readalong), Heroicons, custom — and document it.
- **Empty / loading / error states** for `<MediaLibrary>`, `<DeckCard>`, `<SpeakingDrill>`.
- **Onboarding flow.** First-run sequence across the three apps (Pip intro → language picker → first lesson).

### Out of scope (explicitly defer)

- Marketing site `reeloo.ai/` chrome and copy.
- App Store screenshots and metadata.
- The operator's `studio.reeloo.ai` admin UI (functional, not user-facing).
- The 7 named themes from readalong (ocean / forest / lavender / midnight / emerald / amethyst / rose) — recommend dropping all; if any survives, it's a one-page proposal not a full design pass.
- Code-level token migration scripts (each app team owns its rip-and-replace).
- The pricing / paid-tier surface that lives in `notes/app/pricing/page.tsx` — currently dormant, design later when monetisation is real.
- Email + push notification templates.
- The current `notes/app/podcast/_components/*.tsx` (the depot mirror of podcast-app) — these get deleted when podcast-app is the canonical surface.

---

## 8. Deliverables requested from Claude design

1. **`tokens.json`** in the W3C design-token JSON format, covering:
   - `color.bg.{vanilla,cobalt-dark,high-contrast}` for all 13 surface/text/border roles
   - `color.anno.{vocab,grammar,idiom,cultural,archaism,proper-noun}` with underline style metadata
   - `color.state.{correct,wrong,warning}.{bg,fg,ring,solidBg,solidFg}`
   - `color.pos.{place,subject,object,verb,particle}`
   - `font.{sans,serif,mono,cjk,jp-serif}` referencing IBM Plex / Noto Sans TC / Noto Serif JP
   - `fontSize.{xs,sm,base,md,lg,xl,2xl,3xl}` = 11/13/15/19/22/30/34/38
   - `space.{1..16}` Tailwind-base + named `--space-page-pad`, `--space-card-pad`, `--space-section-gap`
   - `radius.{sm,md,lg,xl,pill}` = 4/8/12/16/999
   - `shadow.{card,elevated,focus}`
   - `motion.duration.{fast,med,slow}` + `motion.easing.out`

2. **Figma library** with the 18 components in §5. Component variants for state + theme. Auto-layout where appropriate (`<MediaCard>` and `<MediaLibrary>` especially).

3. **12 screen mockups** in Figma. Light + dark variants for each. Annotated with token references so an engineer can implement without guessing.

4. **Code Connect map** — Figma component → React component file path:
   - `MediaCard` → `packages/ui/MediaCard.tsx` (NEW shared package)
   - `FocusPinColumn` → `packages/ui/FocusPinColumn.tsx` (extracted from `podcast-app/components/KarottenPlayerV3.tsx`)
   - …18 entries.

5. **Migration playbook** (one page per kept app) showing which existing-component files get refactored to consume which Figma component. Example: `podcast-app/components/CatalogClient.tsx:130 → use <MediaCard variant="podcast">`.

6. **Anti-patterns doc** — what NOT to do, including:
   - Don't introduce new theme names; the unified set is `vanilla / cobalt-dark / high-contrast` (+ optional `pip-duo`).
   - Don't ship hex literals in component source; use tokens.
   - Don't reintroduce HSL triplets (readalong's pattern); use named-color tokens.
   - Don't re-add the 7 readalong cosmetic themes.

---

## 9. Open questions for Claude design

1. **Cobalt-dark vs Solarized-dark for the default dark theme.** Podcast-app's cobalt (`#0f1419 → #1a2332`) is cinematic and right for audio surfaces, but on the book reader it might feel too cool. Should we ship ONE dark theme (cobalt) or TWO (cobalt for podcast, Solarized-dark for book + practice)? Sourced evidence on both in `current-state.md` §1.4.

2. **`--anno-cultural` vs `--anno-place`.** Same hex, two names. Collapse to one slot (saving a token name) or keep both for semantic clarity in book-app's annotation pipeline? Per `current-state.md` §1.2.

3. **Plex Serif at 19px reading vs 34px focus-pin.** Are these the same family setting (just different sizes), or does the focus-pin variant need tighter tracking / different optical sizing? Currently both share `var(--font-plex-serif)` with no per-variant tracking override.

4. **Touch-target floor 44 vs 48px.** podcast uses 44 (`CatalogClient.tsx:199`), notes/practice uses 48 (`page.tsx:301`). WCAG 2.5.5 says 44px minimum; Material says 48px. Pick one for the unified set.

5. **The Flux LoRA illustrations on flashcard fronts.** They're b/w line drawings (the `flashcard-stroke-v1` LoRA name implies it). Do we keep b/w-only, or design a color treatment for higher-energy cards? Current notes/practice surface treats them as static PNGs (`/practice/flashcards/page.tsx:200`).

6. **Wizard form.** Today's podcast-app `/wizard` is a Tailwind-default form (`app/wizard/page.tsx:35–73`). Should the unified wizard surface read as creative-studio (matching podcast's cobalt) or as form-utility (matching practice's solarized)?

7. **Onboarding mascot.** Pip is the learn-stations mascot. Does it travel to book and podcast, or stay as `practice.reeloo.ai`'s only? Memory `feedback_north_star.md` and `user_identity.md` suggest the operator wants ONE consistent identity; design should validate or break that.

8. **Readalong's 134-component sprawl.** A lot of that is the operator's experimentation surface. Should the Figma library treat the studio surfaces as "out of scope" (this brief's recommendation), or should we document the 30-or-so components the operator depends on so we can govern them too?

9. **The Cards/Decks distinction.** In learn-stations a "card" is one vocab-front-back unit. In readalong a "deck" is a multi-segment presentation. In the unified product, do we treat them as different objects (`/library/cards` vs `/library/decks`) or as one polymorphic thing? Vote: separate. Different shapes, different use cases.

10. **The `welcome` query param.** Podcast-app's `/podcast/<slug>?welcome=1` (`app/podcast/[slug]/page.tsx:65`) triggers a one-time greeting after fresh generation. Should the unified shell have a generic `?welcome=…` mechanism, or should each surface own its welcome state?

---

## 10. Out-of-scope items the previous agent put in scope (corrections to v0.5)

- v0.5 assumed all five apps share a single Next.js codebase. They don't — five separate package.json files, five separate deploys.
- v0.5 invented colors not present in source (e.g. specific shadcn neutrals). v1's recommended palette is sourced — every value cites a file:line.
- v0.5 named "flashcard" as if it were one app. Source-evidence shows three independent surfaces; v1's recommendation is to merge them at `practice.reeloo.ai/library/cards`, not pretend the singular exists.
- v0.5 proposed a monorepo migration. v1 proposes a **shared package** for tokens + 18 components without forcing a monorepo; each app remains independently deployable to Vercel.
- v0.5 did not address the 2,442-line duplication of `KarottenPlayerV3` between podcast-app and notes — the biggest hidden debt. v1 explicitly calls it out as a delete-on-cutover task.

---

## TL;DR (one paragraph)

The five apps share a vision (language-learning via daily content) but no code. Source audit shows **three load-bearing postures** (ear-first listen, eyes-first read, output-first drill) that are physically different — keep them as three apps (`podcast.reeloo.ai`, `book.reeloo.ai`, `practice.reeloo.ai`), promote three contracts (`dictionary.reeloo.ai`, `audio.reeloo.ai`, depot `/api/llm`) to shared services, and absorb readalong by carving out its three user-facing wins (deck-as-presentation, word-rain game, scenario sim) into `practice.reeloo.ai/library/decks` + `/games/*`, with the operator's studio (deck-generator + admin) moving to a separate auth-gated `studio.reeloo.ai`. The unified design system has **18 components**, **3 themes** (`vanilla` default light, `cobalt-dark` for podcast, `high-contrast` for a11y), **2 font families** (IBM Plex + Noto Sans TC), and **6 annotation underline colors** with three already agreed across two of the existing apps. Every token in `proposed-tokens.md` cites its source app; values that are invented are labelled.
