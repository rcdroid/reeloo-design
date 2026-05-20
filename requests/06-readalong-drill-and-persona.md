# Design request: Readalong DrillShell + Persona-driven drill

**Status:** open
**Filed:** 2026-05-20
**Author:** orchestrator (audit pass)
**Context refs:**
- `v1/products/readalong.md` §5 Flow B (~10 lines on drill)
- `v3-gap-audit/readalong.md` §1.A (features 10–23), §3 (the gaps)
- `v3-gap-audit/cross-cutting.md` §11 (drill state machine — 5 implementations to reconcile)
- Source code: `/home/claw/work/readalong/nextjs/src/components/drill/DrillShell.tsx` (4,257 LOC), `PersonaCard.tsx`, `MeasuredSlot.tsx`, `PressureRing.tsx`, `AvatarCompanion.tsx` (473 LOC)

## What we asked for in v2 (and what we got)

The v2 brief named `<SpeakingDrill>` as ONE shared component and (in §4.5 J5, §5 J6) referenced "persona-driven" tutor mode. Claude design produced zero artifacts for the DrillShell flow. The impl has nothing.

DrillShell is the **biggest single component in the readalong codebase** (4,257 LOC) and represents readalong's most-developed product surface — the place where persona, mastery tracking, level estimation, and substitution/transformation/extension drilling converge.

## What's actually in the source app

`DrillShell.tsx` is a 6-step multi-step flow:

```
welcome → setup → level → method → drill → summary
                                       ↘ branch (mid-drill choice)
```

Each step is a different UI:

1. **welcome** — intro card with persona pitch + sign-in prompt + "Get started" CTA. Source: `:574-587` (the welcomeBody copy in English).
2. **setup** — source language + target language picker. Auto-pre-fill from user profile if signed in. Source: `:577-583`.
3. **level** — placement step. Two paths: "Set your starting level" (CEFR picker) OR "I am not sure" → 3-question mini-test (`:606-616` "estimateLevel"). The mini-test surfaces 3 progressively-difficult sentences; user attempts each; algorithm returns an estimated CEFR with confidence.
4. **method** — pick the drill method (substitution / transformation / extension / mixed). With a tutor note explaining when each is recommended.
5. **drill** — the actual drill. `<DrillCardDisplay>` (331 LOC) renders one card; `<MeasuredSlot>` (189 LOC) is the unit; `<PressureRing>` (167 LOC) is the reaction-time visual; `<PersonaCard>` (60 LOC) shows the persona portrait + name + origin (with native-hints localization); `<AvatarCompanion>` (473 LOC) reacts to attempts.
6. **summary** — end-of-session card with score + new-vocab-learned + weak topics + "Continue tomorrow" CTA.

**Personas** are AI tutor characters that shape:
- Voice (different Azure Neural voice per persona)
- Verdict phrasing ("Nice!" vs "Yes, that's it." vs "Hmm, almost — try again")
- Prompt style (formal / casual / kid-friendly)
- Visual portrait + flag + origin city

Per `types/personaDrillTypes.ts`: each persona has `{ name, imageUrl/imageEmoji, origin: {flag, city}, nativeHints, voice }`.

**Reaction-time bands** (`lib/reactionTime.ts`): `reflex / fast / normal / slow / timeout` — each band has a defined ms range and feeds into mastery scoring (faster = stronger memory). The `<PressureRing>` countdown is the visual.

**Mastery tracker**: `<MeasuredSlot>` measures slot width AND tracks `SlotMastery` per `makeSlotKey()` (`:71`). On every attempt, mastery state updates via SM-2-variant algorithm. `POST /api/drill/mastery-sync` writes to Firestore.

## What we need from Claude design

1. **6-step flow mockup** — one frame per step (welcome / setup / level / method / drill / summary) + the `branch` mid-drill choice frame. Show the step-progress UI (1 of 6 bar at top? Stepper? Subtle eyebrow?).

2. **Placement mini-test UI** — 3-question sequence. Each question is a `<MeasuredSlot>` attempt. After all 3, show the estimated level with confidence band ("We think you're B1 — confident" vs "We think you're B1 — not sure, try a longer set"). "Confirm" / "Try different level" affordance.

3. **Persona picker (`<PersonaCard>`)** — grid of 4–6 personas. Each card: portrait/emoji, name, origin flag + city, native-language hints ("Hi!" in target lang + source-lang hint), short tutor-style blurb. Selected state. The selected persona's portrait appears in the drill chrome.

4. **`<DrillCardDisplay>` per phase**:
   - **Substitution phase**: sentence template with ONE slot, target word replaces a wildcard. Per memory `reference_readalong_drill_patterns.md`: "Substitution = same template + cycling slot. Matched value goes green + slides up; next slides in from bottom."
   - **Transformation phase**: same meaning, different grammatical structure (e.g. active → passive).
   - **Extension phase**: complete the meaning (e.g. add a "because…" clause).
   - Each phase needs a phase-indicator chip.

5. **`<MeasuredSlot>`** — the slot is the unit, not the card. Mockup: a measured-width container with the target phrase + optional transcript reveal + per-attempt mastery state. Show 4 mastery states: `unseen / introduced / learning / familiar / mastered / reflexive` (the 6 levels from `types/drill.ts:34-51`).

6. **`<PressureRing>` countdown** — circular progress ring around the slot. When the user attempts within `reflex` band: ring fills green-pulse. `fast` = mild green. `normal` = neutral. `slow` = warm. `timeout` = retry-state (no red, per the locked-design contract).

7. **Reaction-time band feedback** — each band needs a verdict phrase (varied by persona). Design specs the phrasing slots; copywriting fills.

8. **`<AvatarCompanion>` integration** — readalong's 473-LOC AvatarCompanion is the production-grade version of v2's Pip primitive. Design must decide: (a) port AvatarCompanion's full state machine into Pip (and expand Pip's 6 moods to ~14), (b) keep AvatarCompanion for the readalong-drill surface and Pip for the others, (c) unify. Recommendation: (a) — port. Phase C request 11 carries the unified mood spec.

9. **Per-persona voice + verdict tone** — when a different persona is active, the user should sense a difference in *more than just the portrait*. Design proposes: subtle UI tint per persona, different verdict-bubble background color per persona's culture, different audio chime per persona.

10. **`<ActivityLog>`** — per-session activity stream. Each row: timestamp, slot key, attempt result (correct / retry / skip), mastery delta. Mockup the live-updating list.

11. **`<DrillSettingsPanel>`** — drill-only settings: persona, method, voice speed, pressure-ring visibility, retry budget (1/3/∞). Mock.

12. **Summary card** — score + new-vocab-learned (with mini-flashcard previews) + weak topics (with deep-link "Drill these tomorrow") + streak bump.

13. **Branch frame** — mid-drill, the user can choose to (a) skip ahead, (b) get easier cards, (c) take a break. Mockup.

14. **`<DrillTestShell>`** (1,269 LOC) — designer playground for the drill. Production-relevant? Or operator-only? Design decides.

## Out of scope

- The 18 drill API routes — backend.
- The Firestore mastery schema — backend.
- The persona content (which personas ship, their bios) — content team.
- The voice synthesis pipeline (Azure Neural per persona) — backend.

## Success criteria

- Each of the 6 steps + the branch fork has a Figma frame.
- The persona picker is a real, designed surface (not a placeholder).
- `<MeasuredSlot>` is documented as a primitive with all 6 mastery states.
- `<PressureRing>` reaction-time bands map to defined colors + verdict phrases.
- The substitution-phase "matched-slides-up / next-slides-up" animation is the design's responsibility (this is the user's repeated 10× requirement).
- AvatarCompanion → Pip mood-mapping is reconciled (see request 11).
- `<DrillCardDisplay>` is a single primitive that takes a phase prop and renders the right variant.
- Mobile variants for all 6 steps.
