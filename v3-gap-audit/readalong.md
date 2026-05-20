# v3 gap audit — Readalong + Flashcards (and the notes-app surfaces that share the family)

> Source paths:
> - Canonical: `/home/claw/work/readalong/nextjs/` (~48,000 LOC, 48 page routes, 108 API directories)
> - Three flashcard implementations: `/home/claw/work/notes/app/practice/flashcards/page.tsx` (308 LOC), `/home/claw/work/learn-stations/app/cards/` (covered in learn-stations.md), and the readalong DeckBrowser family
> - Related notes-app surfaces: `/home/claw/work/notes/app/deck/[id]/`, `/home/claw/work/notes/app/practice/[id]/page.tsx` (2,811 LOC!), `/home/claw/work/notes/app/practice/pronunciation/page.tsx` (640 LOC), `/home/claw/work/notes/app/practice/series/[language]/[level]/`, `/home/claw/work/notes/app/learn/_renderers/` (5 renderers, 2,541 LOC total), `/home/claw/work/notes/app/learn/_minna-core/` (Exercises.tsx = 1,576 LOC, Whiteboard.tsx = 646 LOC), `/home/claw/work/notes/app/learn/mundart-100/` (scenes.tsx = 780 LOC)
> - MP4 export pipeline: `/home/claw/work/notes/app/api/tutorial/[id]/mp4/`, `social-video/`, `videos/tag_ingest/`
> - v1 spec: `/home/claw/work/reeloo-design/v1/products/readalong.md` (271 lines) + `flashcard.md` (183 lines)
> - Impl: nothing in `reeloo-v2/components/routes/` is dedicated to the readalong family. There's no `readalong.tsx`, no `word-rain.tsx`, no `scenario.tsx`, no `drill-shell.tsx`. The product is **entirely absent** from the v2 impl.

> **Headline: this is the largest gap of any app.** The v1 spec sketches the surface area honestly (134 components, 119 routes, 7 themes, persona drill, word-rain, scenario-sim, MP4 export pipeline), but Claude design never produced mockups for any of it. The impl doesn't even attempt the surfaces. Worse: the actual workhorse code now lives partly in `notes/app/` (which the v2 brief calls "depot internal" but actually hosts production-equivalent surfaces like `/practice/[id]` 2,811 LOC and `/learn/_minna-core/Exercises.tsx` 1,576 LOC).

---

## 1. Feature inventory (real apps)

### 1.A Readalong canonical surfaces (`/home/claw/work/readalong/nextjs/`)

#### Surface: library + browse + deck

| # | Feature | Source | Tier |
|---|---|---|---|
| 1 | `<LibraryViewPage>` — Firestore-backed deck grid with profiling-journey instrumentation; first-time users redirect to `/intro` | `app/page.tsx`, `src/components/LibraryViewPage.tsx:31-203` | multi-step flow |
| 2 | `/browse` — public deck browser | `app/browse/page.tsx` | composed |
| 3 | `/discover` — recommendations surface | `app/discover/page.tsx` | composed |
| 4 | `/library`, `/library-new`, `/owned`, `/favorites`, `/history` — 5 distinct library views | (5 page.tsx files) | composed |
| 5 | `/deck/<id>` — full deck player (`<ClientApp>` mounts `<SlideViewer>` 1,282 LOC) | `app/deck/[id]/page.tsx:17`, `SlideViewer.tsx` | multi-step flow |
| 6 | `/deck/<id>/detail` — deck details + settings | `app/deck/[id]/detail/page.tsx` | composed |
| 7 | `<SlideContent>` — the deck renderer (1,838 LOC) — composes per-slide layouts | `src/components/SlideContent.tsx` | composed |
| 8 | **4 deck-player layouts**: `CardLayout`, `StackLayout`, `SplitLayout`, `BridgeLayout` — discriminated by `page.layout` enum | `src/components/deck-player/layouts/*.tsx` | composed |
| 9 | `<WordHighlightedText>` (340 lines) — word-level karaoke driven by Azure TTS word boundaries | `src/components/WordHighlightedText.tsx` | composed |

#### Surface: drill (`/drill`)

| # | Feature | Source | Tier |
|---|---|---|---|
| 10 | `<DrillShell>` (4,257 LOC) — the big drill UI with 6 steps: welcome → setup → level → method → drill → summary | `src/components/drill/DrillShell.tsx:73, 129` | multi-step flow |
| 11 | **Mini-test placement** — 3-question estimator that infers starting level | `DrillShell.tsx:606-616` ("estimateLevel") | composed |
| 12 | Tutor i18n localization — UI labels switch to source-language | `DrillShell.tsx:491-616` (all the `levelBadge / continueLevel / personaSectionLabel` strings) | composed |
| 13 | `<PersonaCard>` — pick an AI tutor persona (name, origin, hints) that shapes prompts + voice + verdict phrasing | `src/components/drill/PersonaCard.tsx`, `personaDrillBuilder.ts` | composed |
| 14 | `<DrillCardDisplay>` (331 LOC) — per-card render with substitution / transformation / extension slot rendering | `DrillCardDisplay.tsx` | composed |
| 15 | `<DrillDeckBrowser>` (242 LOC) — pick which deck to drill | `DrillDeckBrowser.tsx` | composed |
| 16 | `<DrillSettingsPanel>` (224 LOC) — drill prefs | `DrillSettingsPanel.tsx` | composed |
| 17 | `<AvatarCompanion>` (473 LOC) — proactive AI tutor avatar (the production-grade version of v2's Pip primitive) | `drill/AvatarCompanion.tsx` | composed |
| 18 | `<ActivityLog>` (175 LOC) — per-session activity stream | `ActivityLog.tsx` | composed |
| 19 | `<MeasuredSlot>` (189 LOC) — the slot is the unit of substitution; measures + renders one slot | `MeasuredSlot.tsx` | atomic |
| 20 | `<PressureRing>` (167 LOC) — countdown / reaction-time pressure visual | `PressureRing.tsx` | atomic |
| 21 | **Reaction-time bands**: `reflex / fast / normal / slow / timeout` from `lib/reactionTime.ts` | `lib/reactionTime.ts`, `drillMasteryTypes.ts` | atomic |
| 22 | **Mastery sync** — `POST /api/drill/mastery-sync` writes per-slot mastery to Firestore | `lib/mastery-tracker.ts`, `api/drill/mastery-sync` | atomic |
| 23 | Drill API surface — 18 routes under `/api/drill/`: catalog / config / content-search / curriculum / generate / generate-from-text / generate-lesson / history / interleave-generate / level-estimate / library / mastery-sync / pattern-generate / personas / rl-signal / test-session-log / unit-generate / units | `/api/drill/*` | composed |

#### Surface: daily-review + playlist

| # | Feature | Source | Tier |
|---|---|---|---|
| 24 | `/daily-review` — SRS-driven session that surfaces saved vocab across all apps | `app/daily-review/page.tsx` | composed |
| 25 | `/playlist` + `/playlist/[id]` + `/playlist/[id]/play` + `/playlist/[id]/detail` — sequenced decks playback | (4 page.tsx files) | composed |
| 26 | `/map/<slug>` — visual learning map; SkillVector + LearnerDrillState rendered as a path/radar | `app/map/[slug]/page.tsx`, `learningMapTypes.ts` | composed |

#### Surface: games

| # | Feature | Source | Tier |
|---|---|---|---|
| 27 | **Word-rain game** — Pixi.js WebGL canvas; words fall, user types or speaks to match | `app/game/word-rain/page.tsx`, `components/game/word-rain/WordRainGame.tsx` (895 LOC), `PixiRainCanvas.tsx` (519 LOC) | composed |
| 28 | Game features: beginner-mode toggle, music on/off, dict-card flash on match, background-image toggle, settings panel, pause (Space/P key), STT integration | `WordRainGame.tsx:65-117` | composed |
| 29 | `<MatchedWordFlash>` (123 LOC) glow-pulse animation on match | `MatchedWordFlash.tsx` | atomic |
| 30 | `<GameSummary>` (144 LOC) end-of-game score + streak + accuracy | `GameSummary.tsx` | composed |
| 31 | Audio: `gameAudio.ts` — combo SFX, level-up, heartbeat, sparkle, click | `gameAudio.ts`, called `WordRainGame.tsx:13` | atomic |
| 32 | `gameVocabTracker.ts` — per-game vocab mastery state separate from main SRS | `lib/gameVocabTracker.ts` | atomic |
| 33 | **Scenario simulator** — `/scenario-sim` + `/game/scenario` + `/game/scenario/[id]` — branching dialogue with persona-coloured choices, NPC sprites, an inventory bar | `components/scenario-sim/ScenarioSimulator.tsx` (373 LOC + `ScenarioGame.tsx` 373 LOC + `ScenarioMap.tsx` 283 LOC) | multi-step flow |
| 34 | Scenario sub-components: `<DialogOverlay>` (539 LOC), `<NPCSprite>` (319 LOC), `<EpisodeSyllabus>` (791 LOC), `<EpisodeWizard>` (269 LOC), `<EchoReplay>` (189 LOC), `<GuideAvatar>` (233 LOC), `<InventoryBar>`, `<WinScreen>`, `<CompanionContext>` | 11 components in `game/scenario/` | composed |
| 35 | Scenario engine: requirements check, effects application, runtime state cloning, hub-node resolution | `ScenarioSimulator.tsx:42-127` | atomic |
| 36 | Scenario generation API: `POST /api/game/generate-episode`, `POST /api/game/generate-scenario` | `/api/game/*` | atomic |

#### Surface: video export pipeline

| # | Feature | Source | Tier |
|---|---|---|---|
| 37 | `<VideoExportTab>` (1,576 LOC) — the full export UI: aspect ratio, theme override (dark mode, romaji, word highlight, segment highlight, annotations), destination (video / youtube), preview, progress | `src/components/VideoExportTab.tsx:427-460` | multi-step flow |
| 38 | Export stages with human-readable labels: `local-export-started`, `rendering-frames`, `rendering-complete`, `uploading-youtube`, `local-export-failed` | `VideoExportTab.tsx:62-75` | atomic |
| 39 | YouTube OAuth grant flow + upload + privacy-status selection | `VideoExportTab.tsx:146-160` | multi-step flow |
| 40 | Export job state in Postgres (`postgres-export-store.ts`); Cloud Run Jobs for render (`cloud-run-{job,deck-job}.ts`); output to `gs://readalong-exports/` | `lib/postgres-export-store.ts`, `lib/cloud-run-{job,deck-job}.ts` | composed |
| 41 | Export API: `/api/export/{deck,download,log,play,status,v2}` (6 routes) | `/api/export/*` | atomic |
| 42 | SessionStorage cache of export state per deck — survives reload | `VideoExportTab.tsx:86-110` | atomic |
| 43 | Stuck-detection timeout for uploading-youtube stage | `VideoExportTab.tsx:50` | atomic |

#### Surface: dictionary + speaking-test + statistics

| # | Feature | Source | Tier |
|---|---|---|---|
| 44 | `/dictionary` + `/dictionary/[word]` — dedicated dictionary surface | `app/dictionary/page.tsx`, `[word]/page.tsx` | composed |
| 45 | `<DictionaryPanel>`, `<DictionaryLookup>`, `<DictionaryCard>`, `<AnnotationPanel>`, `<AnnotatedText>` — 5 variants of dictionary UI | `src/components/*.tsx` | composed |
| 46 | `<SpeakingTestCard>` (643 LOC) — speaking-skill assessment card | `SpeakingTestCard.tsx` | composed |
| 47 | `/statistics`, `/dashboard`, `/profile` — user data surfaces | (3 page.tsx files) | composed |

#### Surface: settings + auth + admin

| # | Feature | Source | Tier |
|---|---|---|---|
| 48 | `<SettingsModal>` (616 LOC) — modal-based global settings | `src/components/SettingsModal.tsx` | composed |
| 49 | `/admin/{dictionary,studio,analytics,cache,export-jobs,users,image-pool,performance,logs}` — 9 admin pages | (9 page.tsx files) | composed |
| 50 | `<UserManagementPage>` (380 LOC) | `UserManagementPage.tsx` | composed |
| 51 | Auth: Firebase Auth (legacy) + NextAuth (cookies) dual-path | `src/utils/firebaseAuth.ts`, `lib/nextauth-options.ts` | atomic |
| 52 | 9 themes total: 4 light (default / ocean-light / forest-light / lavender-light) + 5 dark (default / midnight / emerald / amethyst / rose) | `src/app/globals.css`, HSL token system | composed |

### 1.B notes-app readalong-family surfaces (production-equivalent)

These are NOT mentioned in v1's readalong.md but they are *the same product family* and ship live:

| # | Feature | Source | Tier |
|---|---|---|---|
| 53 | `/practice/[id]` — practice runner (2,811 LOC, single file) | `notes/app/practice/[id]/page.tsx` | multi-step flow |
| 54 | `/practice/pronunciation` — pronunciation practice (640 LOC) | `notes/app/practice/pronunciation/page.tsx` | composed |
| 55 | `/practice/flashcards` — random LoRA-illustrated card (308 LOC) | `notes/app/practice/flashcards/page.tsx` | composed |
| 56 | `/practice/series/[language]/[level]` — series-based progression | `notes/app/practice/series/[language]/[level]/page.tsx` (230 LOC) | composed |
| 57 | `/deck/[id]` — 118-LOC `DeckClient` + page wrapper | `notes/app/deck/[id]/` | composed |
| 58 | `/learn/_renderers/` — 5 lesson renderers, totaling 2,541 LOC: `<AnnotatedReaderRenderer>` (332), `<BilingualStoryRenderer>` (373), `<DuolingoRenderer>` (191), `<PodcastClozeRenderer>` (521), `<PodcastPlayerRenderer>` (1,124) | `notes/app/learn/_renderers/` | composed |
| 59 | `/learn/_minna-core/` — Minna no Nihongo engine: `<Exercises>` (1,576), `<Whiteboard>` (646), `<TypewriterCard>` (259), `<EnginePicker>` (171), `<LearnApp>` (468), `<Home>` (155), `<Shared>` (159) | `notes/app/learn/_minna-core/` | composed |
| 60 | `/learn/minna/components/` — second-generation Minna engine: `<Exercises>` (904), `<Whiteboard>` (710), `<Home>` (302), `<Shared>` (204) | `notes/app/learn/minna/components/` | composed |
| 61 | `/learn/mundart-100/` — 100-phrase Mundart course with scene-based progression: `scenes.tsx` (780), `home-client.tsx` (193), `InstallPrompt.tsx` (137), `[lesson]/page.tsx` (236) | `notes/app/learn/mundart-100/` | composed |
| 62 | `/learn/record` — recording practice (493 LOC) | `notes/app/learn/record/page.tsx` | composed |

### 1.C Notes-app MP4 + tutorial pipeline (production)

| # | Feature | Source | Tier |
|---|---|---|---|
| 63 | `POST /api/tutorial/[id]/mp4` — generate MP4 from a tutorial | `notes/app/api/tutorial/[id]/mp4/route.ts` | composed |
| 64 | `/api/tutorial/[id]/animate` — animation rendering | `notes/app/api/tutorial/[id]/animate/` | composed |
| 65 | `/api/social-video` — social-share-formatted video | `notes/app/api/social-video/route.ts` | composed |
| 66 | `/api/videos/tag_ingest` — video-tagging ingest | `notes/app/api/videos/tag_ingest/` | atomic |
| 67 | `/api/tutorial-sessions`, `/api/tutorials/{series,generate,[id]}`, `/api/tutorial/{from-url, generate, podcast, podcast-v2, wizard}` — full tutorial-generation API surface (~10 routes) | `notes/app/api/tutorial*` | composed |

### 1.D Flashcard surfaces — THREE implementations

| # | Feature | Source | Tier |
|---|---|---|---|
| 68 | **A. notes/practice/flashcards (308 LOC)** — random LoRA-illustrated card discovery loop; calls `GET /api/dictionary-illustration/random?lang=`; entry enrichment from dictionary service at `localhost:18112` | `notes/app/practice/flashcards/page.tsx:115`, `notes/app/api/dictionary-illustration/random/route.ts:20-21` | composed |
| 69 | **B. learn-stations/cards** — Anki-style deck with SRS (covered in learn-stations.md #54-63) | `learn-stations/app/cards/` | — |
| 70 | **C. readalong DeckBrowser family** — `<DeckBrowser>`, `<DeckList>`, `<DeckHomePage>`, `<DeckDetailPage>`, `<DeckEditorModal>`, `<DeckLoadingView>`, `<DeckPromptsModal>`, `<DeckSettingsTab>`, `<DeckGenerationProgress>`, `<DeckCreationWizard>` — 10 components | `readalong/nextjs/src/components/*.tsx` | composed |
| 71 | LoRA illustration pipeline: Flux.1 dev `flashcard-stroke-v1`, offline `scripts/illustrations/generate_flashcards.py`, served by `app/api/dictionary-illustration/route.ts`, stored in libsql + `~/cache/illustrations/dictionary/<lang>/<word>.png` | `notes/app/api/dictionary-illustration/route.ts:1-18, 51` | multi-step flow |
| 72 | SRS contract: SM-2 (interval, easeFactor, recentResults); two implementations (`learn-stations/lib/srs.ts`, readalong's `adaptiveEngine.ts`) | — | atomic |

---

## 2. What v1 spec captured (readalong.md + flashcard.md)

readalong.md (271 lines) is the longest of the five and covers a lot:
- §3 (route map): 48 page routes named
- §4 (data model): all 22 type modules referenced
- §5 (flows): 6 flows including drill, deck generation from URL, scenario sim, word-rain, MP4 export
- §6 (themes): 9 themes with HSL triplets
- §7 (components): the 9 biggest components by LOC
- §8 (integrations): Firestore, Azure TTS, Google STT, Gemini, Cloud Run, GCS, Vercel KV, Postgres
- §10 (uniqueness pitch): the deck-from-URL, persona-drill, word-rain triumvirate

flashcard.md (183 lines) honestly admits the 3-implementation mess and proposes consolidation.

The spec is **honest about depth** — but it does so by listing top-level features and trusting the reader to drill in. The v2 brief and Claude design did not drill in.

---

## 3. What v1 spec MISSED

- **`<DrillShell>` is 4,257 LOC** — v1 §7 lists it as one of the top components but doesn't show the 6-step flow (welcome → setup → level → method → drill → summary) or the per-step UI. Design has zero mockups for the placement test, the persona picker, the method-selection screen, the summary card.
- **Mini-test placement (3-question level estimator)** — produces `estimatedLevelLabel`, drives initial drill difficulty. Critical onboarding moment; missing from design.
- **`<PersonaCard>` UI** — persona-driven drill is one of readalong's three uniqueness pillars. v1 §10 mentions it; design produced nothing.
- **Per-persona voice + verdict tone** — `drillVoiceClient.ts` and `drillTutor.ts` produce different audio + different verdict copy per persona. Design needs to specify the affordance: does the user see the persona name + flag while they drill? A small portrait? A different verdict bubble color?
- **`<MeasuredSlot>` as the unit of substitution** — slots, not cards, are the drill atom. v1 doesn't articulate this; design built around "cards."
- **`<PressureRing>` — countdown/reaction-time visual** — what does the ring look like? Where does it sit relative to the card? How does it animate? Token: `--motion-pressure-ring`?
- **Reaction-time bands** (`reflex / fast / normal / slow / timeout`) — these are user-visible verdict states. Each needs a color, a copy phrase, a feedback rhythm.
- **`<AvatarCompanion>` (473 LOC)** — this is readalong's production-grade tutor avatar, much richer than v2's "Pip primitive" sketch. v2 should adopt this as the canonical source, not invent.
- **Word-rain game UX** — beginner mode, music toggle, dict-card flash, background-image toggle, STT input mode, pause behaviour, combo SFX sequence (`playComboSound`, `playLevelUpSound`, `playHeartbeat`, `playSparkle`, `playClickSound`). v1 §10 calls word-rain "pure entertainment / variable-reward" but design has no game-screen mockup.
- **Pixi.js canvas integration** — the WebGL canvas is a real physical layer. Design needs to know where the canvas sits, how it resizes (`rainSize` state at `WordRainGame.tsx:82`), how the input field sits relative to the canvas.
- **`<MatchedWordFlash>` glow-pulse** — defined animation, presumably reusable; could feed v2 tokens.
- **Scenario simulator surface** — 11 components, 791-LOC EpisodeSyllabus, 539-LOC DialogOverlay. v1 §10 calls it "branching dialogue with persona-coloured choices" but produces no screen mock. Design needs: NPC sprite layout, dialogue overlay positioning, inventory bar position, scenario-map navigation.
- **`<NPCSprite>` (319 LOC)** — character animation. Sprite sheet contract?
- **`<EpisodeWizard>` (269 LOC)** — scenario authoring flow. Is this user-facing or operator?
- **`<EchoReplay>` (189 LOC)** — replay-your-recording in the scenario flow. Distinct from drill's echo.
- **MP4 export full UX** — `<VideoExportTab>` is 1,576 LOC. v1 §10 mentions "Cloud Run job runner" but no UI mockup for the panel itself.
- **YouTube OAuth grant flow** — what does the user see between "I want to publish" and "publishing"? Consent screen + permission scope + return-to-app sequence. v2 brief §4.2 J2 mentions it but no design.
- **Export stage-by-stage progress UI** — the 6 stages each need a copy phrase and a visual state. Currently in source code as text strings.
- **Export privacy controls** — public / unlisted / private dropdown. Designer hasn't seen it.
- **`<SettingsModal>` (616 LOC)** — the full settings surface. v2 specs `/me/settings` but the readalong version has more controls (TTS voice, drill defaults, export defaults). Reconcile.
- **`<SpeakingTestCard>` (643 LOC)** — assessment surface. What does it look like? When does it fire?
- **`/map/<slug>` learning-map visualization** — v1 mentions it; design produced no mockup. v2 brief proposes `/me/map` but the readalong version is `slug`-scoped (per deck or per language?), different IA.
- **Deck-generation-from-URL UX** — `<CreateFromUrlDialog>` + SSE progress + `<DeckGenerationProgress>`. v1 §5 Flow C names the components; design produced no mockup.
- **Statistics + Dashboard + Profile** — three separate surfaces in readalong; v2 collapses to `/me/progress` but the existing surfaces have different data shapes worth preserving.
- **Dictionary `[word]` deep-link** — `/dictionary/<word>` direct URL; v2 brief promotes `dictionary.reeloo.ai` but doesn't spec the deep-link page UI.

### 3.B notes-app readalong-family — entirely missing from v1

v1 doesn't acknowledge any of these surfaces:
- `/practice/[id]` (2,811 LOC) — the most-LOC single component in the codebase.
- The 5 learn-renderers (`AnnotatedReader`, `BilingualStory`, `Duolingo`, `PodcastCloze`, `PodcastPlayer`).
- The two Minna engines (`_minna-core` + `minna/components`) — TWO independent implementations of the same course inside one app.
- `mundart-100` scenes (780 LOC) — a separate course tree.
- `/learn/record` recording surface.

If the v3 deliverable is "feature parity with what ships," these are essentials. If it's "feature parity with the readalong canonical app," they're out of scope. **Decision needed before design.**

### 3.C Flashcard spec gaps

`flashcard.md` is honest about the 3-implementation problem but doesn't specify:
- The LoRA-illustration generation pipeline UX. Who triggers it? What happens when the cache misses?
- The "Show meaning" reveal animation on the discovery loop.
- The deck-as-curriculum layout differences between the 4 readalong layouts (Card/Stack/Split/Bridge).
- The dict-link affordance — `cards-dict-link` testid exists but the visual treatment isn't specced.
- The four-state grade panel (`again/hard/good/easy`) — v2 brief §9 says it must exist but the visual is missing.

---

## 4. What the impl skipped

**There is no readalong route in `reeloo-v2/components/routes/`.** Zero. No drill, no word-rain, no scenario, no MP4 export panel, no deck library, no daily review, no /map, no flashcards.

The closest thing is `practice.tsx` (Mundart/Minna/Deutsch-B1 course picker) which is structurally a learn-stations port, NOT a readalong port. The "speaking drill" in `practice.tsx` is a generic flashcard card — not the `<DrillShell>` 4,257-LOC multi-step flow.

`me.tsx` (938 LOC) handles personal surfaces (`/me/progress`, `/me/map`, etc.) but `/me/map` does not visualize the SkillVector + LearnerDrillState the way readalong's `/map/<slug>` does.

**Score: 0 of 50+ readalong-specific features ported.**

---

## 5. What's actually pretty good

- The decision to PORT `<AvatarCompanion>` semantically into the v2 `<Pip>` primitive is sound; the 473-LOC source is a tested production component, much further along than a fresh design would be.
- `practice.tsx` adopts the 6-mood mood system that aligns with readalong's `AvatarCompanion` states (`idle / proud / encouraging / concerned / thinking / apologetic`), so when v3 wires readalong-the-product in, the moods are already a shared contract.
- v2's `<LevelMismatchDialog>` is the right pattern, even if readalong itself didn't have it — readalong had the *data* (`level-estimate` API) but not the *UX*.
- The `<surface>:<component>:<element>` test-id convention is internally consistent and extensible; readalong's existing flat-kebab testids can be migrated mechanically (`drill-card` → `drill:card-display:card`).
- The notes-app `/learn/_renderers/` are the actual product-equivalent of the v2 brief's "3 learning postures" — each renderer is a posture (read, listen, drill) realised. If v3 acknowledges them, they become design's source-of-truth instead of being designed-around.
