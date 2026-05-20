# Design request: Sequential-drill — the locked-design speaking station

**Status:** open
**Filed:** 2026-05-20
**Author:** orchestrator (audit pass)
**Context refs:**
- `v1/products/learn-stations.md` §5 Flow B (the only station v1 properly documented — 30 lines)
- `v3-gap-audit/learn-stations.md` §1 (features 45–53) + §3 (`<MeasuredSlot>` gap)
- Source code: `/home/claw/work/learn-stations/components/stations/sequential-drill.tsx:1-28` (the locked-design rules block), and the 1,668-LOC implementation

## What we asked for in v2 (and what we got)

The v2 brief named `<SpeakingDrill>` as one shared component (`v2/06` §7.1) and assumed v1's Flow B was a complete contract. Claude design produced ONE generic "speaking drill" card with a play button and a "I knew this" / "Let me try" pair. The locked-design rules — never red flash, never wrong-buzz haptic, ≤1 sound + ≤1 visual + ≤1 action — are documented nowhere in the design.

This is a **high-craft** surface. Six months of co-evolution between TTS, STT, the scorer, and the player produced these rules. They are NOT defaults that can be loosely re-derived. Design must spec them.

## What's actually in the source app

`/home/claw/work/learn-stations/components/stations/sequential-drill.tsx` (1,668 LOC). The first 28 lines are an unusual "locked design rules" header comment:

```
// LOCKED DESIGN — do not modify lightly. Every rule below
// has a "do NOT relax" because it was violated in an earlier iteration.
//
// - One card = one slot. Cards are sequential, never overlap.
// - State machine: card_load → tts → pause → listening_armed → scoring →
//   card_correct | card_retry → next_card
// - ≤1 sound + ≤1 visual change + ≤1 action at any moment.
// - TTS and STT never overlap.
// - Wrong = silence + transcript shown. NEVER red flash. NEVER wrong-buzz haptic.
```

Plus the VAD rules at `:43-52`:
```
// SMART-STOP VAD
//   - 15s hard ceiling
//   - 8s if no voice detected
//   - 4s silence after voice detected
//   - 1.2s after match ≥ 0.7
//   - immediate stop on ≥ 0.99
```

Scoring at `:55-57`: **60% slot match + 40% full-frame match; pass at 0.7**.

Pipeline: card mounts → `speak()` via `lib/tts.ts` (first pre-built `/audio/<hash>.mp3`, then `POST /api/tts` fallback) → 300ms `pre_mic_pause` → mic on → smart-stop VAD → `POST /api/stt-azure` → `scoreTranscript()` (`lib/score-transcript.ts`) → chord SFX on pass → telemetry → next.

Internal sub-components include `<MeasuredSlot>` (189 LOC in readalong; analogous in learn-stations) — the slot is the unit, not the card. Each slot is measured for width and used to drive the TTS playback boundary.

The user-visible UX:
- A single card with the target sentence
- TTS plays automatically, with a "🔊" indicator pulsing during playback
- Mic-armed state: a microphone icon glows + the audio-level waveform animates
- Transcript appears below the sentence during scoring (live transcription if STT supports streaming, else final-only)
- On pass: green ring around the slot, chord SFX, 1.5s pause, auto-advance
- On fail: silence, transcript shown, retry-affordance — `<button>Try again</button>` (no red, no wrong-buzz)

## What we need from Claude design

1. **State-machine diagram** — `card_load → tts → pause → listening_armed → scoring → card_correct | card_retry → next_card`. Each state has a defined visual treatment. Design must mock all 7 states + the two terminal-card forks.

2. **The `<MeasuredSlot>` primitive** — slot-as-unit visual. A slot is a measured-width container that holds (a) the target phrase, (b) the user's transcript (when scoring), (c) a delta indicator on retry showing where the mismatch was. Mockup needed.

3. **TTS playback indicator** — pulsing icon during `tts` state. Volume? Position? Animation curve? Subtle, NOT distracting.

4. **Mic-armed state** — the most important moment of the drill. The user needs to KNOW it's listening. Standard pattern: microphone icon + audio-level waveform + subtle border-glow on the slot. Design must spec the exact treatment because the VAD rules mean the user doesn't push-to-talk — listening is automatic.

5. **Smart-stop VAD ceiling display** — the 15s hard ceiling could feel arbitrary. Design must decide: show a subtle countdown? Show nothing? The 8s "no voice" cutoff needs an explicit "didn't hear you, try again" empty state.

6. **Transcript reveal** — when STT returns, the transcript appears under the card. Streaming (word-by-word) vs final-only — design picks. If streaming, what does the cursor look like?

7. **Pass feedback** — green ring + chord SFX + 1.5s pause + auto-advance. The "chord" is a defined audio asset; design specifies the visual + timing. The ring color is `--color-state-correct-solid-bg #2d3f00` per learn-stations tokens.

8. **Fail feedback — the hardest part** — NO red, NO buzz, NO bounce, NO shake. The rule is: silence + transcript shown + retry. Design must visualize "what is the OPPOSITE of a failure animation that doesn't punish the learner." Proposed: subtle warm pulse (NOT red — warning brown `--color-state-warning-solid-bg #5c4400` at very low opacity), transcript appears below the slot, a `<button>Try again</button>` (or — better — auto-arm mic again after 600ms).

9. **Retry-affordance copy** — never "Wrong" or "Incorrect." Recommended: "Listening again…" or just silence + the mic-arm visual returning.

10. **Lesson-end celebration (`complete` station)** — score + streak bump. Pip in "celebrating" mood (NEW mood — see request 11). Confetti? Probably NO (we have a "no random delight" anti-pattern). Just clean score reveal.

11. **Locked-design contract artifact** — the locked-design rules block (`:8-28`) should be REPRODUCED in the design as an annotation panel in the Figma component, so future designers reading the spec see the contract directly.

12. **The 300ms `pre_mic_pause`** — a defined silence. Design must decide: does the audio indicator fade out, or does it just disappear? The transition matters for user attention.

13. **Pip mood transitions inside the drill** — `idle → thinking` (during TTS, audio playing) → `listening` (NEW mood — mic armed; not in v2's 6) → `proud` (on pass) → `encouraging` (on retry-after-1-miss) → `concerned` (on 3rd miss, lifts whiteboard).

14. **`<BottomCTA>` interaction with the drill** — the drill is mostly hands-free, so the bottom CTA only appears after the auto-advance window expires. Design must show the bottom CTA "Skip to next" appearing 8s into the card if the user hasn't engaged.

15. **Telemetry events surfaced in UI?** — `drill.attempt` fires every try. Should the UI show "Attempt 2 of N"? Recommend NO (pressure). But the operator-facing diagnostic should be reachable via a long-press on the slot. Design specifies.

## Out of scope

- The TTS / STT backend (Azure / GCP / depot proxy) — see request 10.
- The `lib/score-transcript.ts` phonetic-distance algorithm — backend.
- Mastery sync to Firestore / file storage — backend.
- The 18 other station types — see request 03.

## Success criteria

- The 7-state machine has a Figma mockup for each state.
- The fail state explicitly carries the "no red, no buzz" contract in the design annotations.
- The pass state is defined down to the 1.5s pause + chord SFX timing.
- The smart-stop VAD ceilings are surfaced (or explicitly hidden — design picks, but commits).
- The mic-armed state is the most prominent moment of the drill in the design.
- `<MeasuredSlot>` is documented as a primitive separate from `<StationRunner>`.
- A new "listening" mood is added to Pip's mood matrix.
- The locked-design rules are quoted verbatim in the design file as a non-negotiable contract.
