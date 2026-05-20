# Design request: Pip — full mood matrix + per-app firing rules + AvatarCompanion port

**Status:** open
**Filed:** 2026-05-20
**Author:** orchestrator (audit pass)
**Context refs:**
- `v2/02` §5 (the Pip primitive spec — 6 moods named, no firing rules)
- `v3-gap-audit/cross-cutting.md` §7 (full theme analysis)
- `v3-gap-audit/readalong.md` §3 (AvatarCompanion is 473 LOC — much richer than v2's Pip)
- Source code: `/home/claw/work/readalong/nextjs/src/components/drill/AvatarCompanion.tsx` (473 LOC), `/home/claw/work/learn-stations/components/Mascot.tsx`, `/home/claw/work/learn-stations/data/avatars.ts`

## What we asked for in v2 (and what we got)

The v2 brief named Pip with 6 moods (`idle, encouraging, concerned, proud, thinking, apologetic`) and proactive/passive/in-shell modes (`v2/02` §5). Claude design produced `/home/claw/work/reeloo-v2/design-source/project/pip.jsx` (239 LOC) with the 6 moods as states. The impl wires `PipBay` and `PipInfoButton` with mood-mapping from station state.

**What's missing:** per-app firing rules (when does each mood show?), additional moods needed by source apps (`listening` for mic-armed, `celebrating` for lesson-end, `concerned-with-whiteboard` distinct from plain `concerned`), the relationship to readalong's 473-LOC `<AvatarCompanion>`, and the per-mood animation contract.

## What's actually in the source apps

### learn-stations Mascot

`/home/claw/work/learn-stations/components/Mascot.tsx`. Pip with body, leaf, expression. Moods are mostly **facial expressions**, not poses. Used in station player; lifts a whiteboard on 3rd-miss (proactive mode).

### readalong AvatarCompanion (473 LOC — the production-grade source)

`/home/claw/work/readalong/nextjs/src/components/drill/AvatarCompanion.tsx`. Used in DrillShell. Richer state model than learn-stations' Mascot:
- Pre-tts state ("about to speak")
- During-tts state ("speaking" — mouth/body animation)
- Listening state ("mic armed")
- Scoring state ("waiting for the system")
- Verdict states (pass/retry/timeout)
- Inter-attempt states (idle-but-engaged)
- Cross-card states (between cards, the companion gestures toward the next slot)
- Persona-flavored states (different personas have different idle behaviors)

It's a full character animation system, not a 6-mood sprite sheet.

### v2 brief Pip (the proposed unified primitive)

6 moods: `idle / encouraging / concerned / proud / thinking / apologetic`. 3 modes: in-shell (header), proactive (drill-time), passive (info-button). Anti-pattern: "Don't make Pip chat" (no speech bubbles, whiteboard only).

## What we need from Claude design

### 1. Expanded mood matrix (target: 12 moods)

The 6 v2 moods are insufficient. The source apps need:

| Mood | Used by | Description |
|---|---|---|
| `idle` | all | resting; neutral expression |
| `encouraging` | practice (after retry), book (long dwell) | "you got this"; warm |
| `concerned` | practice (3rd miss), book (>30s on para) | "hmm, let me help"; light worry |
| `proud` | practice (correct), card (easy), podcast (save) | celebrative; arms up |
| `thinking` | practice (during TTS), book (loading), podcast (preteach) | hand-on-chin |
| `apologetic` | practice (system error), `<FlagIssue>` resolved | "sorry about that"; bow |
| **`listening`** (NEW) | practice (mic armed), word-rain (spoken), scenario (echo) | leans in, ears perked |
| **`celebrating`** (NEW) | practice (lesson-end), card (deck-end) | confetti pose; arms wide |
| **`speaking`** (NEW) | scenario (NPC turn), drill (tts playing) | mouth animation |
| **`pointing`** (NEW) | passive info-button | small Pip with arrow gesture |
| **`whiteboard`** (NEW) | practice (concerned + hint) | concerned + holds a whiteboard with text |
| **`waving`** (NEW) | onboarding (welcome), session-start | greeting pose |

### 2. Per-mood animation spec

Each mood needs:
- Entry animation (how Pip arrives at the mood) — 240–360ms typical, `--motion-avatar-bounce` token applies
- Idle loop (does the mood loop? subtle blink? subtle sway?)
- Exit animation (how Pip leaves the mood)
- 6 frames minimum (per state) at 64×64, 128×128, 256×256

The "Don't randomize Pip's mood" anti-pattern still holds — moods are state-driven, never time-randomized. But within a mood, subtle idle motion (blink every 3–5s) is fine.

### 3. Per-app firing matrix (the gap)

| App | Event | Mood | Duration |
|---|---|---|---|
| **practice station** | mount | `idle` | persist |
| | TTS playing | `speaking` | until TTS end |
| | pre-mic-pause (300ms) | `thinking` | 300ms |
| | mic armed | `listening` | until VAD stop |
| | scoring | `thinking` | until result |
| | correct | `proud` | 1.5s → `idle` |
| | wrong (1st) | `encouraging` | 800ms → `listening` |
| | wrong (3rd) | `whiteboard` | until next attempt |
| | lesson-end (all correct) | `celebrating` | 3s → `idle` |
| **practice cards** | card mounted | `idle` | persist |
| | flip | `thinking` | brief |
| | grade `again` | `encouraging` | 600ms |
| | grade `easy` | `proud` | 800ms |
| | deck-end | `celebrating` | 3s |
| **book PageReader** | scroll active | `idle` | passive |
| | dwell > 30s on one para | `concerned` + tap to show whiteboard hint | until action |
| | save 5+ words in 10min | `proud` | 800ms |
| **book DeckReader** | paragraph play | `speaking` (small, lower-right) | during audio |
| | paragraph pause > 10s | `idle` | persist |
| **podcast player** | pre-teach modal | `thinking` | until dismiss |
| | save word | `proud` | 600ms |
| | (otherwise) | absent | — |
| **scenario sim** | (separate — NPC characters, NOT Pip) | — | — |
| | echo-replay attempt | `listening` | during attempt |
| **word-rain** | spoken mode listening | `listening` | during recording |
| | combo ≥10 | `proud` | 1s overlay |
| | (otherwise) | absent | — |
| **onboarding** | first land | `waving` | until first interaction |
| | "Welcome" step | `idle` | persist |

### 4. AvatarCompanion → Pip migration

Recommendation: **port `<AvatarCompanion>` semantically into `<Pip>`.** The 12-mood expanded matrix above + the per-app firing rules cover most of AvatarCompanion's behaviour. Decisions to make:

- **Persona-flavored states**: AvatarCompanion has variants per persona. Does Pip get persona overlays (e.g. when a specific persona is active, Pip wears that persona's hat)? Recommend NO — Pip is one identity, personas are separate characters.
- **Inter-attempt gestures**: AvatarCompanion gestures toward the next slot. Pip can do this via the `pointing` mood with a directional arrow.
- **Cross-card states**: between cards, AvatarCompanion has small idle behaviors. Pip can borrow.

### 5. NPC vs Pip — keep them separate

Scenario sim has NPCs (different characters in the world). They are NOT Pip. Design must:
- Confirm NPC sprites are a separate primitive (`<NPCSprite>`, 319 LOC)
- Provide consistent direction: same sprite size, same animation framework, but distinct visual identity
- Mockup at least 2 NPC characters as examples

### 6. Sprite-sheet deliverables

- Pip in 12 moods × 2 lighting variants (vanilla light / cobalt-dark) at 64×64, 128×128, 256×256
- Whiteboard variant (concerned + whiteboard card overlay) — separate spec for the whiteboard card (background, max text length, typography)
- Pointing variant — define the arrow direction (8 directions or 4?)

### 7. Whiteboard contract

When Pip is in `whiteboard` mood:
- Pip stands next to a small whiteboard card
- Card background: `--color-avatar-whiteboard-bg` token
- Card text: `--color-avatar-whiteboard-fg` token (high contrast for legibility)
- Max ~30 characters per line, 3 lines
- Comes from the station's `hintText` field (per v2 brief §4.3 J3)
- Animations: card slides up from behind Pip with 360ms ease-out

### 8. `<PipInfoButton>` (passive mode)

Already in impl. Spec needs:
- Trigger affordance: ?-emoji or small Pip icon (32×32) anchored next to questions
- On-tap: opens a popover with explanation text
- The popover position: above-right of the button (avoid overlapping question text)
- The popover has Pip in `pointing` mood inside

### 9. PipBay (in-shell mode)

Already in impl. Spec needs:
- Position: top-right of viewport, persistent across pages within an app
- Size: 64×64 default
- Mood reflects the currently-active app context (e.g. on `/me/progress`, mood is `proud` if user is on streak)
- Tap → opens the user menu (per v2 brief §8.1)

### 10. Proactive mood transitions

When Pip transitions from one mood to another:
- 240ms cross-fade by default
- 320ms when the new mood is celebratory (entry should feel earned)
- 0ms when state is rapidly cycling (don't animate transitions faster than the state changes)

### 11. Anti-patterns to reinforce in design

(All from v2 brief §12 + new additions)
- No speech bubbles. Whiteboard text only.
- No random mood changes. State-driven only.
- Don't put Pip on top of `<FlagIssue>`.
- Pip never shows during audio-only listening (podcast player without interaction).
- NEW: Pip never shows in word-rain except for combo overlays. The game has its own celebration UI.
- NEW: NPC characters in scenarios are NOT Pip. Keep separate identities.

## Out of scope

- The illustration art for Pip — content/illustration team. Design specs the contract.
- The NPC character library — separate content effort.
- The persona portraits — content team.

## Success criteria

- 12-mood sprite sheet delivered (vs v2's 6).
- Per-app firing matrix is the canonical contract (no `TBD`).
- Whiteboard contract documented with token + max-text rules.
- `<PipInfoButton>` and `<PipBay>` positions + sizes spec'd.
- AvatarCompanion ↔ Pip migration plan documented.
- NPC vs Pip distinction is explicit.
- All transitions have defined durations + curves.
- WCAG: animations respect `prefers-reduced-motion` (fall back to instant state change).
