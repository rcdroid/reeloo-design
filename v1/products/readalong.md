# Product Spec — Readalong (`readalong.reeloo.ai`)

> Source path: `/home/claw/work/readalong/nextjs/` (the inner Next.js app inside the larger `readalong/` workspace; the outer Vite app at `/home/claw/work/readalong/src` is older and inactive)
> Live: <https://readalong.reeloo.ai> returned 302 at 2026-05-19 22:32 UTC (likely auth-gated). Local dev at `http://localhost:3000/` returned 200 and rendered the LibraryView.
> Dev port: 3000 (default `next dev`)
> Screenshots: `readalong-home.png`, `readalong-browse.png`, `readalong-drill.png`, `readalong-game-word-rain.png`

---

## 1. Identity

- **Name:** "reeloo" per `<title>` (`http://localhost:3000/` HTML head). The internal workspace name in `package.json` is `readalong-app` (`package.json:2`).
- **Tagline (synthesised):** *AI-generated language decks with word-level audio karaoke, persona-driven speaking drills, scenario simulations, and gamified word-rain — Firestore-backed, multi-user, the workhorse of the language-learning surface.*
- **The one user it serves today:** the operator's testing ground for new pedagogy ideas before they harden into learn-stations. Also the only one of the five apps with multi-user accounts via Firebase Auth.
- **The user goal it advances:** *creative-production* learning — generate a deck from a URL or YouTube video, drill it with a persona, replay echoes, render an MP4 to share. The "studio" arm of the family.

## 2. Tech stack

| Concern | Choice | Source |
|---|---|---|
| Framework | Next.js ^14.2.28 (App Router) | `package.json:62` |
| React | ^18.2.0 | `package.json:65` |
| Auth | `next-auth` ^4.24.14 (cookie sessions) + `firebase` ^12.7.0 (legacy) | `package.json:63,46` |
| DB | Firestore (admin SDK 13.6.0); Postgres for export-job store (`pg` ^8.18.0); `@vercel/kv` for rate-limit | `package.json:40,67,38` |
| TTS | Azure Neural (`microsoft-cognitiveservices-speech-sdk` ^1.47.0) + Google generative AI for content (Gemini) | `package.json:60,42,43` |
| STT | Google Cloud Speech ^7.2.1 | `package.json:36` |
| Animation | `framer-motion` ^12.23.26 | `package.json:48` |
| UI primitives | `@radix-ui/react-{dialog,dropdown-menu,label,select,slot,switch,tabs}`, `lucide-react`, `class-variance-authority`, `tailwind-merge` | `package.json:28–35` |
| Game runtime | `pixi.js` ^8.17.1 (for word-rain WebGL) | `package.json:66` |
| Subtitles | `@fontsource/noto-sans-tc` ^5.2.9 | `package.json:34` |
| Theme | `next-themes` ^0.4.6 | `package.json:64` |
| Video export | `@google-cloud/run` ^3.1.0 — kicks Cloud Run Jobs | `package.json:35` |
| Misc | `pako` 2.1 (gzip), `kuromoji`, `@mozilla/readability` | (`package.json`) |

**How it differs from siblings:** the largest by 5–10× — 48,383 lines across 134 top-level `.tsx` components, 119 API routes. Multi-user. Firestore-backed deck store. Has its own admin panel (`/admin/*`), drill engine, scenario simulator, word-rain game, and Cloud Run video export pipeline. **Tailwind v3** (not v4 like the other four) — old config (`tailwind.config.js`) drives a 7-theme HSL-token system.

## 3. Route map

48 page routes; an abbreviated list of the public ones:

```
src/app/
├── page.tsx                          # GET / → LibraryViewPage (deck grid)
├── browse/page.tsx                   # GET /browse — public deck browser
├── discover/page.tsx                 # GET /discover — recommendations
├── intro/page.tsx                    # GET /intro — first-run onboarding
├── library/page.tsx                  # GET /library — current user's decks
├── library-new/page.tsx              # GET /library-new — newer iteration
├── owned/page.tsx                    # GET /owned — purchased decks
├── favorites/page.tsx                # GET /favorites
├── history/page.tsx                  # GET /history — recently-opened
├── deck/[id]/page.tsx                # GET /deck/<id> — full deck player (ClientApp)
├── drill/page.tsx                    # GET /drill → DrillShell
├── drill/manager/page.tsx            # GET /drill/manager
├── daily-review/page.tsx             # GET /daily-review — SRS review session
├── playlist/page.tsx + [id]/page.tsx # GET /playlist[/[id]] — sequenced decks
├── map/[slug]/page.tsx               # GET /map/<slug> — visual learning map
├── game/word-rain/page.tsx           # GET /game/word-rain — Pixi.js WebGL game
├── game/scenario/page.tsx            # GET /game/scenario — branching scenario sim
├── scenario-sim/page.tsx             # GET /scenario-sim — simulator entry
├── statistics/page.tsx               # GET /statistics
├── dashboard/page.tsx                # GET /dashboard
├── profile/page.tsx                  # GET /profile
├── dictionary/page.tsx               # GET /dictionary
├── settings/(via modal)              # SettingsModal.tsx 616 lines
├── support/, about/, privacy/, tos/, account-deletion/ — boilerplate
├── admin/                            # GET /admin, /admin/{dictionary,studio,analytics,cache,export-jobs,users,image-pool,performance,logs}
├── demo/word-highlighting/, demo/maltese-avatar/ — design playgrounds
├── test/drill-engine/, test/unit-generate/, test/drill-mastery/ — internal tests
└── api/    # 119 routes (see §8)
```

Per-route highlights:

- `GET /` (`app/page.tsx:1–37`) — wraps `<LibraryViewPage>` (`components/LibraryViewPage.tsx:31–203`) in `<ThemeProvider> + <LanguagePreferencesProvider> + <VoiceProvider> + <I18nProvider>`. Auth-aware (`getCurrentUser`, `:34`); first-time users redirect to `/intro` (`:64`).
- `GET /deck/<id>` (`app/deck/[id]/page.tsx:17`) — wraps `<ClientApp>` which parses the URL and loads the deck.
- `GET /drill` (`app/drill/page.tsx:5`) — one-liner: `return <DrillShell />`.
- `GET /api/dictionary?word=&lang=` (`app/api/dictionary/route.ts:1–35`) — local Firestore-or-disk dictionary lookup with locale-specific normalization (German umlauts → ae/oe/ue/ss; Japanese unicode block preservation).

## 4. Data model

The richest of the five. Sources:

### Deck types (`src/types/deckFile.ts` — not fully opened, but referenced)

`DeckFile` defines the entire portable deck: metadata, slides, per-slide `page.layout` enum (`stack | card | split | bridge` from `components/deck-player/layouts/types.ts`), audio refs, annotations.

### Drill types (`src/types/drillContentTypes.ts`, `drillMasteryTypes.ts`)

Source of `learn-stations/types/drill.ts` (`learn-stations` says so explicitly in its file header). Five extra types beyond what learn-stations imports: `LearnerDrillState`, `DrillUnit`, `SkillVector`, `PersonaCard`, scenario-sim related types.

### Scenario types (`src/types/scenarioTypes.ts`, `src/types/scenarioSimulator.ts`)

Branching dialogue with persona-coloured choices; consumed by `src/lib/scenarioEngine.ts` and `src/components/game/scenario/`.

### Game types

- `wordRainTypes.ts` — falling-word Pixi.js game
- `gameVocabTypes.ts` — score / streak / vocab-tracker state
- `episodeTypes.ts` — episodic content for the game

### Other type modules

`syllabusTypes`, `quizTypes`, `quizGenerationTypes`, `learningMapTypes`, `presentationTypes`, `personaDrillTypes`, `drillSequenceTypes`, `drillUnitTypes`, `analyticsTypes`, `languageTerminology`, `deckGenerationJob` — ~22 type files in `src/types/`.

### Storage

- **Firestore** is primary. `src/lib/firebase-admin.ts` is the admin entry. Decks, user state, mastery, deck generation jobs all live in Firestore. Project id `readalong-fb463` (`src/config/firebase.ts:13`).
- **Postgres** (`pg`) — export-job state (`src/lib/postgres-export-store.ts`).
- **Vercel KV** — rate limit + caching (`@vercel/kv` 3.0.0).
- **localStorage** — settings, deck preferences, theme, recent.
- **GCS** — exported MP4s (bucket `readalong-exports`, `.env.local:GCS_BUCKET`).

## 5. Key user flows

### Flow A — Open library, click a deck, present

1. `GET /` → `<LibraryViewPage>` (`:31`). Profiling journey starts (`:43–48`).
2. `initializeAuth` + `onAuthChange` (`:73–79`).
3. `getAllDeckMetadata()` reads localStorage (`:81–82`).
4. For decks missing cover/image, fetch `/api/decks/<id>` (`:97`) with `getAuthHeader()` (`:90`).
5. User clicks a deck → router pushes to `/deck/<id>?...`.
6. `<ClientApp>` (`app/deck/[id]/page.tsx:41`) mounts `<SlideViewer>` (1,282 lines, `src/components/SlideViewer.tsx`).
7. `<SlideViewer>` runs `PresentationEngine` (`src/engine/PresentationEngine.ts` + V2) to compute render segments; passes to `<SlideContent>` (1,838 lines) which renders the active page via the right deck-player layout component.
8. Audio plays via `<WordHighlightedText>` (340 lines) — word boundaries from Azure TTS drive a karaoke highlight.

### Flow B — Drill mode

1. `GET /drill` → `<DrillShell>` (the big drill UI, in `components/drill/DrillShell.tsx`).
2. Loads a drill unit via `src/lib/drillUnitLoader.ts`. Each unit has substitution / transformation / extension phases (the design learn-stations later inherited).
3. Persona-card overlay: the user picks an AI persona (`PersonaCard.tsx`), and the drill voice / coaching style adapts (`src/lib/drillTutor.ts`, `drillVoiceClient.ts`).
4. Audio loop: tutor speaks prompt → user repeats → STT (`src/lib/stt-streaming.ts` for streaming or `stt-client.ts`) → score → next.
5. Mastery sync to Firestore via `mastery-tracker.ts` + `POST /api/drill/mastery-sync`.

### Flow C — Generate a deck from a URL

1. `<CreateFromUrlDialog>` (`src/components/CreateFromUrlDialog.tsx`) accepts a URL.
2. Server queues a deck-generation job via `src/lib/url-deck-queue.ts` → `url-deck-worker.ts`.
3. Worker hits `POST /api/drill/generate-from-text` and / or `/api/drill/generate-lesson` → Gemini via `@google/genai`. URL content is fetched + cleaned via `@mozilla/readability`.
4. Status SSE'd back through `src/lib/sse.ts` to `<DeckGenerationProgress>`.
5. Deck saved to Firestore; user redirected to `/deck/<newId>`.

### Flow D — Scenario simulator (branching dialogue practice)

1. `GET /scenario-sim` → `<ScenarioSimulator>` (`components/scenario-sim/ScenarioSimulator.tsx`).
2. Engine: `src/lib/scenarioEngine.ts` + `src/lib/scenarioSimulator.ts`.
3. NPC turn → user picks an option or speaks → STT → branching state transition. `<AvatarSprite>` provides character animation.
4. Telemetry: `firestore-analytics.ts` events; `gameAchievements.ts` unlocks badges.

### Flow E — Word-rain game (Pixi.js WebGL)

1. `GET /game/word-rain` → `<WordRainGame>` (`components/game/word-rain/WordRainGame.module.css` + tsx).
2. Pixi.js canvas; words fall from the top, user types or speaks to match.
3. On match: `<MatchedWordFlash>` glow-pulse + tracker updates `gameVocabTracker.ts`.
4. `<GameSummary>` at end shows score + streak.

### Flow F — Export a deck to MP4

1. `<VideoExportTab>` (1,576 lines) collects export options (`src/lib/local-video-export.ts` is the local fallback).
2. Server creates a Cloud Run Job via `@google-cloud/run` (`src/lib/cloud-run-deck-job.ts` and `cloud-run-job.ts`) — Postgres tracks job state.
3. Worker renders frames + audio → uploads MP4 to `gs://readalong-exports/`.
4. SSE → `<ExportHistoryItem>` shows download link.

## 6. Design tokens used

Tailwind v3 + 7-theme HSL token system in `src/app/globals.css`:

| Theme | data-theme | bg HSL | primary HSL | Vibe |
|---|---|---|---|---|
| Default Light | `:root` | `40 33% 96%` (warm cream) | `220 89% 55%` (blue) | default |
| Ocean Light | `:root[data-theme='ocean-light']` | `210 40% 96%` | `199 89% 48%` (cyan) | calm blue |
| Forest Light | `:root[data-theme='forest-light']` | `140 25% 95%` | `142 71% 45%` (green) | calm green |
| Lavender Light | `:root[data-theme='lavender-light']` | `270 30% 96%` | `262 83% 58%` (violet) | calm violet |
| Default Dark | `.dark` | `222 47% 11%` | `210 100% 60%` | high-contrast blue |
| Midnight Dark | `.dark[data-theme='midnight-dark']` | `230 25% 7%` | `217 91% 60%` | deep navy |
| Emerald Dark | `.dark[data-theme='emerald-dark']` | `160 30% 8%` | `160 84% 45%` | jewel green |
| Amethyst Dark | `.dark[data-theme='amethyst-dark']` | `270 30% 8%` | `270 76% 60%` | jewel violet |
| Rose Dark | `.dark[data-theme='rose-dark']` | (in `globals.css`) | — | jewel rose |

Token style: HSL space-separated triplets (Tailwind v3 + shadcn-ui convention). Each theme defines the same 16 vars: `background`, `foreground`, `card`, `card-foreground`, `popover`, `popover-foreground`, `primary`, `primary-foreground`, `secondary`, `secondary-foreground`, `muted`, `muted-foreground`, `accent`, `accent-foreground`, `destructive`, `destructive-foreground`, plus `border`, `input`, `ring`, `chart-1`…`chart-5`, `radius`.

Theme set via `<ThemeProvider>` (`src/components/ThemeProvider.tsx`, 72 lines) which writes to `document.documentElement.classList.add('dark')` + `dataset.theme`.

Fonts: system-stack body (`globals.css:9–12`); `@fontsource/noto-sans-tc` for Chinese (`package.json:34`).

Custom animations: `glow-pulse` for correct quiz answers (`globals.css:32–56`); deck transition keyframes elsewhere; pixi.js drives game animations.

Drill-specific tokens in `src/styles/drill-theme.css` (imported at `globals.css:6`).

## 7. Components inventory

134 top-level `.tsx` files in `src/components/` — too many to list. Highlights by size:

- `SlideContent.tsx` — 1,838 lines (the renderer)
- `VideoExportTab.tsx` — 1,576 lines
- `SlideViewer.tsx` — 1,282 lines
- `TestModeOverlay.tsx` — 996 lines
- `SpeakingTestCard.tsx` — 643 lines
- `SettingsModal.tsx` — 616 lines
- `StudyModeHome.tsx` — 565 lines
- `SlideHeader.tsx` — 530 lines
- `UserManagementPage.tsx` — 380 lines

Subdirs:
- `components/drill/` — `DrillShell`, `DrillCardDisplay`, `DrillSettingsPanel`, `DrillTestShell`, `DrillDeckBrowser`, `ActivityLog`, `AvatarCompanion`, `MeasuredSlot`, `PersonaCard`, `PressureRing`, `drillTheme.ts`, plus `shared/` and `views/`
- `components/deck-player/layouts/` — `CardLayout`, `StackLayout`, `SplitLayout`, `BridgeLayout` + `types.ts` + `index.ts`
- `components/game/scenario/` — `DialogOverlay`, `EchoReplay`, `EpisodeSyllabus`, `EpisodeWizard`, `GuideAvatar`, `WinScreen`, `CompanionContext`
- `components/game/word-rain/` — `WordRainGame`, `GameSummary`, `GameSettings`, `MatchedWordFlash`
- `components/scenario-sim/` — `AvatarSprite`, `ScenarioSimulator`
- `components/shared/` — generic UI primitives
- `components/ui/` — shadcn-style Radix wrappers (Button, Dialog, Tabs etc.)

**Duplicates with siblings:**
- `DictionaryPanel`, `DictionaryLookup`, `DictionaryCard`, `AnnotationPanel`, `AnnotatedText` are all variations on what notes / book-app / podcast-app re-implement. Merge target: one `<DictionaryCard>`, one `<AnnotatedSpan>`.
- `WordHighlightedText.tsx` (340 lines) overlaps with podcast-app's word-level karaoke (`KarottenPlayerV3` word-span rendering).
- The deck-player layout templates overlap with book-app's `<DeckReader>` and podcast-app's focus-pin column.
- `useAuthManager` hook overlaps with learn-stations's NextAuth + book-app's no-auth.

## 8. External integrations

| Call | File:line | Purpose |
|---|---|---|
| `/api/dictionary?word=&lang=` (internal) | `src/components/DictionaryPanel.tsx:212`, `DictionaryLookup.tsx:265,349`, `DictionaryCard.tsx:203`, `AnnotationPanel.tsx:913`, `drill/DrillShell.tsx:1866` | Dictionary lookup; internal route at `src/app/api/dictionary/route.ts:1–35` does its own normalization |
| `/api/dictionary/search?prefix=&lang=&limit=` | `DictionaryLookup.tsx:290` | Prefix typeahead |
| Firestore | many | Decks, mastery, analytics |
| Azure Neural TTS | `src/app/api/synthesize/route.ts` | Generate audio |
| Google Cloud Speech | `src/app/api/speech-to-text/route.ts` | STT |
| Gemini (`@google/genai`) | `src/app/api/drill/{generate,generate-lesson,generate-from-text,unit-generate,interleave-generate}/route.ts` | Deck generation, drill generation |
| Cloud Run Jobs | `src/lib/cloud-run-{job,deck-job}.ts` | Video export |
| GCS | `gs://readalong-exports/` (`.env.local`) | MP4 outputs |
| Vercel KV | `src/lib/rateLimiter.ts` | Rate-limit |
| Postgres | `src/lib/postgres-export-store.ts` | Export-job state |
| Firebase Auth → NextAuth migration | `src/config/firebase.ts:1–7`, `src/lib/nextauth-options.ts` | Auth split |

API surface: **119 routes** under `src/app/api/`. Major namespaces:
- `/api/dictionary`, `/api/dictionary/search`
- `/api/decks/*` (CRUD + image-pool)
- `/api/drill/*` (15 routes: catalog, generate, units, mastery-sync, level-estimate, personas, rl-signal, …)
- `/api/synthesize`, `/api/tts-cache`, `/api/voices`, `/api/speech-to-text`
- `/api/jobs/[id]`, `/api/export/{log,v2}`
- `/api/playlists/*`, `/api/maps`, `/api/feedback`, `/api/recordings/upload`
- `/api/admin/*` (8 admin routes)
- `/api/analytics`, `/api/cache/{stats,entries}`, `/api/progress/summary`, `/api/feedback`

## 9. Screenshots

- `v1/screens/readalong-home.png` — `localhost:3000/` (LibraryViewPage — Loading… overlay caught mid-hydration; the real view is a deck grid)
- `v1/screens/readalong-browse.png` — `localhost:3000/browse`
- `v1/screens/readalong-drill.png` — `localhost:3000/drill` (DrillShell)
- `v1/screens/readalong-game-word-rain.png` — `localhost:3000/game/word-rain`

## 10. What this app does that others can't reasonably absorb

Three behaviours, all unique:

1. **Deck generation from arbitrary URLs / YouTube videos.** The `url-deck-queue` + `url-deck-worker` + `url-plugins` pipeline (`src/lib/url-plugins/plugins/`) is an active ETL: take any web content, segment it, LLM-annotate it, store the deck. No other app does this.

2. **Persona-driven drill voice tutor.** The `<PersonaCard>` + `drillVoiceClient` + `drillTutor` stack lets the user pick a teaching persona that shapes the prompts, the voice, the verdict phrasing. The drill is conversational, not transactional.

3. **Word-rain Pixi.js game + scenario simulator.** Live WebGL game + branching dialogue. Pure entertainment / variable-reward layer on top of vocab. Closer to a game than a learning tool.

The 119-route API surface plus 134-file component tree means this is also the most operationally-mature surface in the family — multi-user, admin panel, Cloud Run job runner, telemetry. Anything reused in unification would likely pull from here.

## 11. What this app does that overlaps with others

- **Dictionary lookup** is duplicated everywhere (notes flashcards, book-app annotations, podcast-app rail). Readalong already has the most-used `/api/dictionary` — promote to `dictionary.reeloo.ai` (already exists per memory `reference_shared_dictionary_api.md`). Merge target: one canonical contract, drop the in-app re-implementations.
- **Annotation underline colors** overlap with podcast-app and book-app — three different Solarized derivatives. Merge target: shared palette.
- **TTS / STT** pipelines: Azure Neural + GCP Speech are independently re-implemented in learn-stations and readalong. Merge target: `audio.reeloo.ai` for TTS, `stt.reeloo.ai` for STT.
- **Drill engine.** `src/lib/drillStateMachine.ts` + `drillTutor.ts` + `drillUnitLoader.ts` + `adaptiveEngine.ts` are the canonical version; learn-stations has a trimmed copy (`types/drill.ts:1–8` says so). Merge target: unify the schema, share the engine, ship learn-stations's typed lesson catalogues as one feed.
- **Deck/card UI.** `<DeckBrowser>`, `<DeckList>`, `<DeckDetailPage>` overlap with podcast-app's `<CatalogClient>` and book-app's `<BookCard>`. Merge target: shared `<MediaLibrary>` component with a polymorphic `MediaItem` type.
- **Auth.** Three auth strategies across the five apps (Firebase + NextAuth here; NextAuth Google in learn-stations; none in book-app/podcast-app). Merge target: NextAuth Google as the single contract; oauth2-proxy in front for non-Next surfaces.
