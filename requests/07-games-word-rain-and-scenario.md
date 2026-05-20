# Design request: Games — Word-rain (Pixi.js) + Scenario simulator

**Status:** open
**Filed:** 2026-05-20
**Author:** orchestrator (audit pass)
**Context refs:**
- `v1/products/readalong.md` §5 Flow D (scenario, ~6 lines), §5 Flow E (word-rain, ~5 lines)
- `v3-gap-audit/readalong.md` §1.A (features 27–36)
- Source code: `/home/claw/work/readalong/nextjs/src/components/game/word-rain/` (5 files, ~1,760 LOC), `src/components/game/scenario/` (11 files, ~3,300 LOC), `src/components/scenario-sim/` (2 files)

## What we asked for in v2 (and what we got)

The v2 brief mentioned "games" exactly twice — in the IA route tree (`/games/`, `v2/06` §3.1) and in §10 anti-patterns. No deliverable, no mockup, no component. Claude design produced nothing.

The impl has no game surface.

This is a meaningful product loss because **readalong's two game modes (word-rain + scenario sim) are explicitly called out as readalong's third uniqueness pillar** in `v1/products/readalong.md` §10 ("Closer to a game than a learning tool"). They're the variable-reward layer that drives daily engagement.

## What's actually in the source app

### Word-rain (`/game/word-rain`)

- `<WordRainGame>` (895 LOC) — main game container
- `<PixiRainCanvas>` (519 LOC) — Pixi.js WebGL canvas with falling-word entities
- `<MatchedWordFlash>` (123 LOC) — glow-pulse animation when a word is matched
- `<GameSummary>` (144 LOC) — end-of-game results
- `<GameSettings>` (78 LOC) — in-game settings panel

**Game features (`WordRainGame.tsx`):**
- Words fall from the top; user types OR speaks (STT integration) to match
- Combo system (`playComboSound`), level-up (`playLevelUpSound`), heartbeat (`playHeartbeat`), sparkle (`playSparkle`), click (`playClickSound`) — 5 distinct audio events
- Settings: music on/off, beginner-mode toggle, dict-card flash toggle, background-image toggle, settings panel
- Pause: Space/P key (`:97-117`)
- Background image: AI-generated or pre-built (`bgImageUrl` state)
- `vocabMastery` per-game tracker
- `engineLog` — visible activity stream (developer + power-user)
- `dictFlashItems` — when a word matches, a dictionary card briefly flashes in
- `onComplete` callback — can unlock quests if score > threshold

**Pixi.js canvas:**
- WebGL-backed
- Resizable (`rainSize` state, default 800×500)
- Falling-word entities with gravity + spawn rate + per-word difficulty
- Imperative handle via `PixiRainCanvasHandle`

### Scenario simulator (`/scenario-sim`, `/game/scenario`, `/game/scenario/[id]`)

11 components in `game/scenario/`:
- `<ScenarioGame>` (373 LOC) — main game container
- `<ScenarioSimulator>` (in `scenario-sim/`) — alt entry point
- `<ScenarioMap>` (283 LOC) — overworld map / scene picker
- `<DialogOverlay>` (539 LOC) — branching dialogue UI
- `<NPCSprite>` (319 LOC) — character render with animation
- `<EpisodeSyllabus>` (791 LOC) — pre-episode vocab + grammar summary
- `<EpisodeWizard>` (269 LOC) — scenario authoring flow
- `<EchoReplay>` (189 LOC) — replay-your-recording in-scene
- `<GuideAvatar>` (233 LOC) — the player's avatar character
- `<InventoryBar>` (51 LOC) — collected items / vocab
- `<WinScreen>` (75 LOC) — end-of-scenario celebration
- `<CompanionContext>` (145 LOC) — companion-character state

**Scenario engine** (`ScenarioSimulator.tsx`):
- Branching dialogue with `<ScenarioNode>` shape (text, choices, requirements, effects)
- `checkRequirements()` (`:42`) and `applyEffects()` (`:70`) — RPG-like state mutation
- Persona-coloured dialogue choices (the user's response options are tinted by which NPC/persona they're framed for)
- STT integration — user speaks a response; matched against expected choice
- `autoAdvanceMs` per node (`:165`)
- Interactable icons resolved per node (`:170`)

## What we need from Claude design

### Word-rain

1. **Game-screen mockup** — Pixi.js canvas centered, falling words visible at multiple depths, input field at bottom (typed mode), STT mic chip alongside (spoken mode). Beginner-mode treatment (slower fall, fewer words). Background-image option (blurred AI art OR solid color).

2. **HUD chrome** — score, level, lives/streak, combo counter, pause button (Space/P shortcut hint). Mobile: minimal HUD; settings collapse to a single gear.

3. **`<MatchedWordFlash>`** — glow-pulse animation spec. Color, radius, duration, fade curve. The "match" moment must feel rewarding without being noisy.

4. **`<GameSummary>` end-screen** — final score, accuracy %, top 5 hardest words, "Play again" CTA, "See these words in your SRS" link.

5. **`<GameSettings>` panel** — music toggle, beginner-mode toggle, dict-card flash toggle, background-image toggle, language picker. Slide-in from right.

6. **Dict-card flash** — when a word matches, a small dictionary card flashes in-frame. Spec: position, size, dismiss timing, contents (word + reading + meaning).

7. **Audio choreography** — 5 SFX events (combo / level-up / heartbeat / sparkle / click). Design doesn't author audio but does spec the trigger moments + timing.

8. **Pause state** — Space/P pauses; resume countdown 3-2-1-go OR instant. Design picks.

9. **STT mode** — when speaking, mic icon glows; audio-level waveform; transcript appears briefly above the canvas.

10. **Embedded mode** — `onComplete` callback suggests this game can embed elsewhere (e.g. as a station inside a lesson). Mockup the embedded variant (smaller, no settings).

### Scenario simulator

11. **`<ScenarioMap>` overworld** — node-graph or branching-map view of available scenarios. User picks a scene; checkmark on completed ones.

12. **`<DialogOverlay>`** — NPC sprite on left, dialogue text bottom-center, 3–4 choice chips bottom. Persona-coloured choice tints. Tap-to-advance and auto-advance variants.

13. **`<NPCSprite>` sheet** — multiple emotions per NPC (idle, talking, surprised, happy). Sprite sheet contract or per-state PNGs.

14. **`<GuideAvatar>`** — the player's character. Walks across the map; appears in dialogue scenes (subtle).

15. **`<EpisodeSyllabus>` pre-episode** — vocab list + grammar points + estimated minutes + "Begin" CTA. Pre-warm the user before the dialogue.

16. **`<InventoryBar>`** — collected items / vocab tokens. Bottom-bar style with item icons.

17. **`<EchoReplay>` in-scene** — when the user is asked to speak, they hear the NPC's expected phrase; they speak; STT scores; on pass, scene advances; on fail, they hear the NPC's expected phrase again with a "Try again" affordance.

18. **`<WinScreen>` end-of-scenario** — XP / vocab / new-words-mastered + "Continue to next chapter" CTA.

19. **`<EpisodeWizard>`** — IF this is operator-facing (likely), reduce scope; spec only the "what fields exist" outline. IF user-facing (unlikely), full mockup.

20. **`<CompanionContext>`** — companion-character state (a sidekick that progresses with the user). Design picks: surface as a persistent UI element (top-right avatar)? Episodic only?

## Out of scope

- The Pixi.js library choice + WebGL implementation — backend.
- The scenario engine state machine (requirements / effects) — game-logic team.
- Scenario content authoring (the actual stories) — content team.
- The `gameAudio.ts` audio assets — audio team; design only specs triggers.

## Success criteria

- Word-rain has a designed game-screen with all HUD elements + the canvas treatment.
- The 5 SFX events have defined trigger moments in the design annotations.
- Scenario sim has a designed overworld + dialogue + win-screen flow (3 frames minimum).
- `<DialogOverlay>` choice-chip persona-color contract is documented.
- All games specify their integration with the SRS (matched vocab → SRS queue).
- All games specify their integration with the signal sink (game events emit per cross-cutting §5).
- Mobile variants for both games (word-rain particularly hard on small screens — design must pick a layout that works at 360px).
