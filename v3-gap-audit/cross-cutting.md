# v3 gap audit — Cross-cutting

> Themes that span 2+ apps but the v2 brief sketched at a level too thin to wire. Each theme below names the shared affordance, lists the per-app UI hooks needed, and points to the source-of-truth implementation that should win the merge.

---

## 1. Dictionary popups + word-tap surfaces

**Shared affordance.** Every text surface (podcast transcript, book paragraph, drill prompt, scenario dialogue, deck slide, flashcard back) should let the user tap a word and see a dictionary card.

**Real-world implementations:**
- podcast-app: annotation rail toggled per-word, scoped to baked-in `annotations[]` (NOT live dictionary lookup) — `KarottenPlayerV3.tsx:807`, `:179`
- book-app: `<AnnotationPopover>` hover/tap popover on annotated spans (NOT live lookup) — `AnnotationPopover.tsx`
- learn-stations: per-word audio chip only; no dictionary popup
- readalong: 5 dictionary-UI variants (`<DictionaryPanel>`, `<DictionaryLookup>`, `<DictionaryCard>`, `<AnnotationPanel>`, `<AnnotatedText>`) — `src/components/`
- notes-app `/practice/flashcards`: `cards-dict-link` testid + dictionary card render

**What v2 brief said.** §3.3 lists `dictionary.reeloo.ai/api/dictionary?word=&lang=` as a service. §7.1 has `<DictionaryCard>` and `<AnnotatedSpan>` as canonical components. **What it didn't specify**: where in each app the dictionary card appears, what triggers it, how it dismisses, what fallback when the word is OOV.

**Per-app UI hooks needed (the v3 ask):**

| Surface | Hook | Source-of-truth implementation |
|---|---|---|
| podcast player word-span | tap → dictionary card slides in to rail | readalong `<DictionaryCard>` |
| podcast annotation rail entry | tap "explain" → dictionary card | (new) |
| book PageReader inline word | tap → popover above word | book-app `<AnnotationPopover>` adapted to live lookup |
| book DeckReader active word | tap → rail dictionary card | new — needs design |
| learn-stations vocab-intro | "look this up" chip → dictionary card | new — needs design |
| learn-stations cards back | dict-link icon → full dictionary page | learn-stations `cards-dict-link` |
| readalong slide word | already wired | already wired |
| readalong drill slot | tap word → dictionary card overlay | already wired |
| scenario dialogue line | tap word → dictionary card | already wired |
| word-rain dict-flash | match → dictionary card flashes | already wired (`WordRainGame.tsx:73` `dictFlashItems`) |

Design needs to specify: (a) the popover vs slide-in vs full-card form factor per surface, (b) the loading state during fetch (cold-start can be 200ms+), (c) the OOV fallback (Gemini gloss? "We don't know this word yet" + flag-issue chip?), (d) the "save to SRS" CTA inside every dictionary card, (e) where the audio "hear it" button sits.

---

## 2. Audio playback chrome (player-bar + speed control)

**Shared affordance.** Five surfaces ship a player bar with play/pause + scrub + speed:
- podcast-app `KarottenPlayerV3` — `.kt3-bar` block
- book-app `<DeckPlayerBar>` (276 LOC)
- learn-stations station player — TTS chip, no scrub
- readalong `<SlideViewer>` — slide-paced audio
- learn-stations `/cards` — auto-TTS on flip
- notes-app `/learn/_renderers/PodcastPlayerRenderer` (1,124 LOC)

**Speed ladders are inconsistent across apps:**
| App | Ladder | Source |
|---|---|---|
| podcast-app | 0.6 / 0.75 / 1.0 | `KarottenPlayerV3.tsx:92` |
| book-app | continuous (1.0 default, range unspecified) | `DeckReader.tsx:234` |
| reeloo-v2 podcast | 1.0 / 0.85 / 0.7 (different!) | impl |
| reeloo-v2 practice | 0.7 / 1.0 / 1.25 (different again!) | impl |

**The v2 brief did not specify a canonical ladder.** Design must pick one — or specify per-surface (e.g. podcast = 0.6/0.75/1.0 for slow-listening, practice = 0.7/1.0/1.25 for drill stretch).

**Keyboard shortcuts.** Podcast wires R = replay sentence, Space = play/pause, arrows = seek/speed. Book wires Space + arrows. Learn-stations + readalong: spotty. **The v2 brief did not spec a shared keyboard contract.** Design needs to (a) pick the shared shortcuts, (b) decide which apps inherit them, (c) provide a discoverable affordance (`?` opens the shortcut panel).

**MediaSession API.** Only podcast-app wires it (`:356-365`). Book + readalong don't, despite being the more obvious lock-screen candidates. Design needs a per-app policy.

**Per-app UI hooks needed:** shared `<MediaPlayerBar>` component with prop variants: `compact` (drill chip), `standard` (single audio playback), `karaoke` (with word-highlight cursor), `pinned-bottom` (book deck mode). All four variants need mockups in vanilla + cobalt-dark + high-contrast.

---

## 3. Annotation underline taxonomy

**Shared affordance.** Both podcast-app and book-app underline annotated spans with kind-specific colors. The hex values overlap but the taxonomies differ.

| Kind | podcast | book | readalong dictionary | proposed unified |
|---|---|---|---|---|
| vocab | `#268bd2` solid | `#268bd2` solid | (none — uses POS) | `#268bd2` solid |
| grammar | `#859900` solid | `#859900` solid | — | `#859900` solid |
| idiom | `#6c71c4` solid | `#6c71c4` solid | — | `#6c71c4` solid |
| cultural | `#cb4b16` solid | — (no kind) | — | merge into a new combined `cultural / place` |
| place | — | `#cb4b16` solid | — | (same hue; merge or split?) |
| proper_noun | — | `#268bd2` solid | — | reuse vocab blue? or new hue? |
| archaism | — | `#b58900` **dotted** | — | **dotted** is unique to archaism; preserve |
| slang (legacy) | normalized → cultural | — | — | drop |

**The v2 brief did not unify these.** Design needs to: (a) pick a final taxonomy (6 kinds? 7?), (b) document the dotted/dashed/solid styling for each, (c) decide whether `proper_noun` and `vocab` share blue (currently they do in book-app) or get different hues for distinction.

**Per-app UI hooks:** every word span (podcast + book + readalong) renders the underline via `data-kind` selector, all four themes specify the kind-color mapping consistently.

---

## 4. Bookmark / save-to-SRS

**Shared affordance.** "I saw this word and want to review it later" → save to a shared SRS queue.

**Current state:**
- podcast-app: annotation rail "save" button calls `useShell().saveWord(word)` — local state only, no SRS write
- book-app: rail "Add to SRS" button — no wiring
- learn-stations `/cards`: SM-2 + `data/cards-state/<userId>.json` — file-backed
- readalong: per-deck `masteryTracker.ts` → Firestore `decks/<id>/mastery/*`
- notes-app: no SRS

**The v2 brief named `/library/cards` as the unified destination** but didn't specify the save protocol. Design needs:
- A shared `<SaveToSRS>` button visual (probably a "+" icon with subtle bounce when saved)
- A toast confirmation ("Saved to today's review")
- A queue indicator in the header (`/me/notifications` bell or a dedicated SRS dot)
- The mobile gesture (swipe right? long-press?)

---

## 5. Signal-collection / personalization UI hooks (v2 themes B & C)

**The v2 brief proposes per-language CEFR scalars + 18-type event taxonomy + level-mismatch dialog.** For this to actually work, each app must emit signals from user interactions. Today: only learn-stations emits (`POST /api/telemetry/learning`). Podcast + book emit NOTHING. Readalong has its own `firestore-analytics.ts` writing to Firestore.

**Per-app UI hooks needed before signals can flow:**

| Surface | Signal | UI hook |
|---|---|---|
| podcast catalog | `tile.click` | already exists; needs SDK wire |
| podcast player | `word.tap`, `annotation.open`, `word.save`, `seg.seek` | every word click needs an event handler with metadata (kind, segment_idx, time_ms) |
| podcast player | `paragraph.dwell_ms` | new — needs scroll observer in focus-pin |
| podcast wizard | `wizard.submit`, `wizard.success` | already exists; needs SDK wire |
| book chapter list | `chapter.view`, `chapter.level_chip_dismiss` | new — needs LevelChip click handler |
| book page reader | `paragraph.dwell_ms`, `annotation.open`, `word.save`, `translation.toggle`, `target.change`, `mode.toggle` | scroll observer + click handlers on every annotated span |
| book deck reader | `paragraph.play`, `paragraph.replay`, `font.size.change` | needs hooks on DeckPlayerBar |
| practice course picker | `course.start`, `course.silence` (from mismatch dialog) | already partially wired |
| practice station | `station.start`, `station.attempt`, `station.pass`, `station.fail`, `station.skip`, `mascot.hint.shown`, `mascot.hint.dismissed` | every station type needs its own attempt-emit |
| /cards | `card.shown`, `card.rated` (with sm-2 grade), `card.flip`, `card.audio_play` | already partly wired in learn-stations; needs port |
| readalong drill | already heavily instrumented | needs port to shared sink |
| word-rain | `game.start`, `game.match`, `game.miss`, `game.end` (score, accuracy) | already wired |
| scenario | `scenario.choice`, `scenario.branch`, `scenario.complete` | already wired |
| me/diary | `diary.summary_view`, `diary.deep_link` | new |
| flag-issue | `flag.submit` | already wired (learn-stations only) |

**Design needs to spec the SDK ergonomics.** A naive `signal.emit(name, payload)` call is fine, but the design contract is: every interactive component must declare which events it emits. The v2 brief §10 says "the constraint matters because anonymous interactive elements are designed-out." Extend that: every named element must declare its event in a component-gallery field. Without this, signals will drift.

---

## 6. Notification center + video export pipeline

**The v2 brief proposes `notifications.reeloo.ai` and `export.reeloo.ai`** but doesn't show:

- The export panel mockup per surface (podcast / book / practice each get a "Make a video" entry point — what does the modal/sheet look like in each? Are they three different mockups or one shared `<VideoExportSheet>`?)
- The progress UI inside the sheet — readalong's 6 stages with copy phrases. Design needs to inherit these.
- The aspect-ratio + template selection — readalong's `<VideoExportTab>` exposes dark mode, romaji, word highlight, segment highlight, annotations toggles. Are these per-surface or per-template?
- The YouTube OAuth grant screen — when the user clicks "Publish to YouTube" without a linked account, what do they see? In readalong this is a redirect to Google consent. Design needs to specify the in-app transition.
- The toast vs inbox routing — v2 brief §12 says `export.done` goes to inbox; design must specify which events bypass the inbox and toast immediately.
- The notification inbox UI — `/me/notifications` is named but no mockup. Should be list-style? Grouped by app? Filterable?
- The "stuck export" UX — readalong has a `WARNING_TIMEOUT_MS` for uploading-youtube. Design needs the warning state.
- The cancel-export UX — `cancelExportJob` is wired; what's the visual?

**Per-app UI hooks:**
- podcast player "Make a video" CTA in overflow (already a stable testid)
- book chapter "Export chapter as video" CTA in header (already in impl)
- practice "Export this deck as video" CTA in summary station
- readalong deck "Export" CTA in deck-player overflow
- All five surfaces share `<VideoExportSheet>` with surface-specific defaults (podcast = 9:16 karaoke, book = 16:9 reading, practice = 9:16 vertical, readalong = configurable)

**The design pass left this entirely unsketched.** Phase C request 09 lifts the full UX.

---

## 7. Pip avatar — 6 moods are not enough, and per-app firing rules are missing

**Six moods named in v2 brief §13.1**: idle, encouraging, concerned, proud, thinking, apologetic.

**What the source apps need:**
- **learn-stations** sequential-drill: explicit "listening" mood (mic-armed, body language different from "thinking" which is during TTS); "celebrating" mood at lesson-end is distinct from "proud" per-card
- **readalong** AvatarCompanion: 14 distinct emotion states in the 473-LOC source (`drill/AvatarCompanion.tsx`)
- **scenario-sim**: NPC personalities are not the same as Pip — NPCs are characters in the world, Pip is the tutor. Two avatar systems coexisting; design needs to decide if NPCs share the Pip primitive or stay separate
- **word-rain**: no Pip — game has its own UI/sfx; design needs to decide whether Pip cheers on combos
- **podcast**: no Pip in the player today; v2 brief says it appears in the AppHeader user-menu. The actual moment Pip would matter for podcast (pre-teach modal? annotation save?) is not specified

**Mood firing rules — per-app, per-event:**
| App | Event | Mood |
|---|---|---|
| practice station | answer correct | proud (1.5s) |
| practice station | answer wrong (first try) | encouraging |
| practice station | answer wrong (3rd try) | concerned + whiteboard hint |
| practice station | mic listening | listening (NEW) |
| practice station | TTS playing | thinking |
| practice lesson-end | all correct | celebrating (NEW) |
| practice cards | card flipped | idle |
| practice cards | grade "again" | encouraging |
| practice cards | grade "easy" | proud |
| book chapter | save 5+ words in 10min | proud |
| book chapter | dwell > 30s on one paragraph | concerned + whiteboard "stuck?" |
| podcast pre-teach modal | open | thinking |
| podcast player | save word | proud |

**The v2 brief did not specify any of these.** Design needs the mood-firing matrix as a deliverable.

**Per-mood animation contract:** `--motion-avatar-bounce` is a token but the per-mood entry animation (proud = bounce up, concerned = head-tilt, encouraging = nod) is unspecified.

---

## 8. Test-id contract — actual existing precedent

**The v2 brief proposes `<surface>:<component>:<element>[:<modifier>]` colon-separated format** (§10).

**The actual source apps use flat kebab-case:**
- podcast-app: `kt-audio`, `kt-play-pause`, `kt-tweaks-toggle`, `kt-rail-toggle`, `kt-skip-back` (the `kt-` prefix is the only namespace)
- book-app: `deck-background`, `deck-card`, `deck-rail`, `deck-player-bar`, `drop-zone`, `search-result` (no namespace — flat)
- learn-stations: `cloze-feedback`, `echo-prompt`, `cards-flip`, `exp3-bottom-cta`, `flag-issue-launcher` (station-name as the namespace, hyphenated)
- readalong: `activity-entry`, `ai-generate-button`, `drill-card`, `card-frame`, `card-romaji`, `persona-card` (flat kebab-case)
- reeloo-v2 impl: `podcast-catalog:media-card:karotten`, `practice-station:speaking-drill:tts-button` (the v2 convention — actually applied!)

**The v2 impl introduces a new convention that conflicts with all four source apps.** This is fine for new components but means migration is a lift, not a sed. Phase C request 12 specifies the migration plan.

**Recommendation:** v2 convention is the right one (namespacing prevents collision when components are reused across surfaces), but design must (a) document the migration map for the ~200 existing testids and (b) provide a `data-testid` generator helper that auto-prefixes the surface.

---

## 9. Theme propagation across apps

**Source apps' themes:**
- podcast-app: cobalt-dark (hard-coded for catalog), cream/night toggle inside the player (Tweaks panel)
- book-app: light/dark via `prefers-color-scheme` (browser-driven, no toggle)
- learn-stations: 4 themes via `data-theme` + localStorage + pre-hydration script
- readalong: 9 themes (4 light + 5 dark) via `<ThemeProvider>` + `next-themes`

**v2 brief locked to 3 themes** (vanilla / cobalt-dark / high-contrast) and proposes a `.reeloo.ai` cookie for cross-domain propagation. This means dropping:
- All readalong's "vibe" themes (forest-light / lavender-light / midnight-dark / emerald-dark / amethyst-dark / rose-dark / ocean-light) — 7 themes deleted
- learn-stations's `pip-duo` (Duolingo green) — 1 theme deleted

**Design needs to confirm the deletion** (or compromise: keep the 9 themes as an optional Studio preset). The v1 spec for readalong calls these themes "vibe" — they're a creative-production identity. Deleting them changes the product personality. Phase C request 13 names the call.

---

## 10. The notes-app/depot question — is this an app or a place?

**The notes-app `/home/claw/work/notes/` hosts production-equivalent versions of every app surface**, but the v2 brief treats it as "depot internal."

**Concrete examples:**
- `/practice/[id]` (2,811 LOC) — single-file station runner
- `/learn/_renderers/PodcastPlayerRenderer` (1,124 LOC) — readalong-equivalent
- `/learn/_minna-core/Exercises.tsx` (1,576 LOC) — drill engine
- `/learn/_minna-core/Whiteboard.tsx` (646 LOC) — drill canvas
- `/learn/minna/components/Exercises.tsx` (904 LOC) — SECOND drill engine in the same app
- `/learn/mundart-100/scenes.tsx` (780 LOC) — scene-based course
- `/api/tutorial/[id]/mp4/route.ts` — production MP4 generator
- 261 API routes total

**This is the elephant in the room.** Either (a) the v3 design pass acknowledges these as ports to fold into practice.reeloo.ai + book.reeloo.ai, (b) the v3 design pass excludes them and the notes-app becomes a separate fourth surface for "operator + experiments", or (c) the design pass merges some (the MP4 pipeline, the renderers) and excludes others (the in-house Minna engines that duplicate readalong's DrillShell).

**Design must take a position.** Phase C request 14 names this.

---

## 11. Drill state machine — three competing implementations

| Implementation | LOC | Where | Locked design? |
|---|---|---|---|
| learn-stations sequential-drill | 1,668 | `components/stations/sequential-drill.tsx` | YES (the `:8-28` block) |
| learn-stations pattern-stack | 1,258 | `components/stations/pattern-stack.tsx` | partial (per memory `project_pattern_stack_drill.md`) |
| readalong DrillShell | 4,257 | `components/drill/DrillShell.tsx` | NO (active development) |
| notes/learn _minna-core Exercises | 1,576 | `notes/app/learn/_minna-core/Exercises.tsx` | NO |
| notes/learn minna Exercises | 904 | `notes/app/learn/minna/components/Exercises.tsx` | NO (newer rewrite) |

**Five implementations of "drill the user on a sentence."** They differ on:
- State machine atomicity (sequential-drill is monastic, DrillShell is permissive)
- Persona-driven voice (only DrillShell)
- VAD smart-stop (only sequential-drill)
- Mastery tracker (DrillShell writes to Firestore; sequential-drill writes JSONL)
- Substitution / transformation / extension phases (DrillShell + pattern-stack)

**The v2 brief picked `<StationRunner>` + `<SpeakingDrill>` as the canonical components** but did not say whose engine wins. **Design needs to declare the winner** (recommendation: sequential-drill's locked-design rules + DrillShell's persona/mastery layer + pattern-stack's substitution dynamics, all stitched).

Phase C request 06 carries this.
