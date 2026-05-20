# Design request: Audio chrome — player bar, speed control, TTS chips, STT mic states

**Status:** open
**Filed:** 2026-05-20
**Author:** orchestrator (audit pass)
**Context refs:**
- `v3-gap-audit/cross-cutting.md` §2 (the full theme analysis with the ladder discrepancy)
- Source code: `podcast-app/components/KarottenPlayerV3.tsx` (player-bar `.kt3-bar`), `book-app/components/deck/DeckPlayerBar.tsx` (276 LOC), `learn-stations/lib/tts.ts` + `app/api/stt-azure/route.ts`, `readalong/nextjs/src/lib/stt-streaming.ts`

## What we asked for in v2 (and what we got)

The v2 brief named `<MediaPlayerBar>` as one shared component (`v2/06` §7.1). Claude design produced one `.rp-player-bar` block for podcast + one `.rb-player-bar`-equivalent for book deck (implicit), each different. The impl ships:
- Podcast speed ladder: `1.0 / 0.85 / 0.7` (impl-only; doesn't match source)
- Practice speed ladder: `0.7 / 1.0 / 1.25` (third different ladder)
- Book: speed control mentioned in spec but not in impl

The user has repeated 10× that "TTS must be high-quality" (per memory `feedback_tts_must_be_high_quality.md`) and "STT must auto-stop" (per `feedback_stt_must_auto_stop.md`). Neither of these requirements is captured in the design.

## What's actually in the source apps

### Speed ladders

| App | Ladder | Source |
|---|---|---|
| podcast (real) | `[0.6, 0.75, 1.0]` | `KarottenPlayerV3.tsx:92` |
| book deck (real) | continuous, default 1.0 | `DeckReader.tsx:234` (range unspecified) |
| reeloo-v2 podcast | `1.0 / 0.85 / 0.7` (different!) | impl |
| reeloo-v2 practice | `0.7 / 1.0 / 1.25` (different!) | impl |

### Keyboard shortcuts

- podcast: `R` replay sentence, `Space` play/pause, `← →` seek 15s, `↑ ↓` speed ladder
- book deck: `Space` play/pause, arrows seek
- learn-stations: STT auto-arms; user doesn't push-to-talk
- readalong: persona drill has its own keyboard contract

### TTS pipeline

`learn-stations/lib/tts.ts` (the canonical):
- First tries pre-built static: `fetch('/audio/<sha256>.mp3')` + sidecar `.timing.json` (`:300`)
- Fall back: `POST /api/tts` → depot `/api/tts` (Google Cloud TTS Wavenet/Studio)
- 30s timeout

`book-app/scripts/synth-audio.ts` + `synth-audio-f5.ts`: build-time generation (Coqui XTTS, F5-TTS).

The TTS pipeline produces:
- MP3 audio file
- `timing.json` sidecar with word boundaries `{word, start_ms, end_ms}[]`
- Stored content-addressed by sha256

### STT pipeline

`learn-stations/app/api/stt-azure/route.ts`: Azure Cognitive Services Speech REST
- Accepts `{audio: base64, mimeType, language?}`
- Transcodes webm/opus → ogg/opus via ffmpeg (`:40-63`)
- Calls `https://${REGION}.stt.speech.microsoft.com/...`

`learn-stations/app/api/stt-gcp/route.ts`: GCP Speech v1 alt path, billing-pending.

`readalong` has streaming STT (`stt-streaming.ts`) AND non-streaming (`stt-client.ts`).

**VAD smart-stop rules** (`sequential-drill.tsx:43-52`):
- 15s hard ceiling
- 8s if no voice
- 4s silence after voice
- 1.2s after match ≥ 0.7
- Immediate stop on ≥ 0.99

The user requirement: **STT must auto-stop** — no manual "Stop" button. Combine confidence threshold + silence VAD + max-duration.

## What we need from Claude design

### 1. `<MediaPlayerBar>` master component

One component with three layouts:

- **`variant="podcast"`** — full chrome: play/pause large + scrub + time + speed-ladder pill (3-step) + tweaks gear + rail toggle. Pin to bottom of viewport (sticky). Cobalt-dark default.
- **`variant="book-deck"`** — play/pause + scrub + speed (continuous slider) + paragraph-cycle toggle. Pin to bottom of deck-frame.
- **`variant="card"`** — minimal: play/pause + optional speed. Used in learn-stations cards + drill stations. Pin within card chrome.

### 2. Speed ladder — pick ONE

Three different ladders in the impl + a fourth (continuous) in book. **Design must pick:**

Recommendation:
- **Listening surfaces** (podcast, book-deck, scenario): `0.6 / 0.75 / 1.0` — match podcast-app source. Slow speeds make sense; faster-than-natural speech rarely helps comprehension.
- **Drill surfaces** (practice stations, /cards): `0.7 / 1.0 / 1.25` — match impl/practice. Faster speeds help reflexive recall once the user knows the word.

Document the rationale. Mockup both ladders.

### 3. Keyboard shortcuts — pick ONE contract

| Key | Action |
|---|---|
| `Space` | play/pause |
| `← / →` | seek -15s / +15s |
| `↑ / ↓` | speed up / down |
| `R` | replay current sentence (listening surfaces only) |
| `M` | mute |
| `,/.` | step backward / forward one word (NEW — for drill review) |
| `?` | open shortcut help panel |

Discoverable via `?`. `aria-keyshortcuts` on every action.

### 4. Seek overlay

Per podcast-app (`:477, 742-746`): 600ms-lived `role="status" aria-live="polite"` overlay shows the effective jump magnitude. Examples:
- "−15s"
- "0.85×"
- "▶"

Position: center of viewport, dismisses with fade.

### 5. TTS button (`<PlayTTSButton>`, `<PlaySlowTTSButton>`)

Per `learn-stations/components/PlayTTSButton.tsx` + `PlaySlowTTSButton.tsx`:
- Standard play: 1.0× speed
- Slow play: 0.6× speed (or whatever the slow-listen tier is)
- Both: small icon button, 32×32 default, with a subtle ring during playback
- Both: hit the canonical TTS pipeline (cached `/audio/<hash>.mp3` first, then live)

Loading state: spinner while fetching (for first-time word, cold cache).

### 6. STT mic states — the critical UX

Per user 10× requirement: **STT must auto-stop**. Combine:
- Confidence threshold (≥0.99 → immediate stop)
- Silence VAD (4s silence after voice → stop)
- Max-duration (15s → stop)
- No-voice fallback (8s no voice → stop with "didn't hear you" affordance)

Mic states:
1. **Idle** — mic icon, not glowing
2. **Pre-listen pause** — 300ms after TTS ends, dimmed mic
3. **Armed** — mic icon glowing + audio-level waveform animating
4. **Capturing** — same as armed + transcript appearing live (if streaming STT)
5. **Scoring** — transcript final + spinner while server scores
6. **Result** — pass (green ring fade) or retry (mic re-arms after 600ms)
7. **Timeout** — "didn't hear you" copy + manual mic chip to retry

### 7. Live transcript reveal

When STT is streaming, the transcript appears word-by-word under the prompt. Treatment:
- Same typography as the prompt (don't visually disambiguate — the user shouldn't feel "watched")
- Slightly muted color
- Last word has a subtle cursor

When STT is non-streaming: transcript appears all at once on scoring.

### 8. Audio-level waveform

A small canvas/SVG showing real-time RMS audio level. Simple bar visualizer (5–8 bars, animated). Color: neutral when listening; subtle pulse on speech detection.

### 9. "Hear it" affordance everywhere

Every word in every text surface should be tappable to hear it. The TTS button needs to be small enough not to dominate, big enough to tap on mobile (44×44 hit target minimum). Recommended: inline icon that grows on hover/focus.

### 10. MediaSession integration

Per podcast-app (`:356-365`): `seekbackward`, `seekforward`, `seekto`. Design specs the lock-screen metadata payload (title, artist, artwork) per surface.

### 11. Audio loading + error states

- **Loading**: skeleton bar, no play button until audio ready
- **Error**: "Audio unavailable" with `<FlagIssue>` chip
- **Cold-cache miss**: "Generating audio…" with a defined wait threshold (typically 2–5s for cached, 10–30s for novel)

### 12. Per-app integration table

| Surface | Player-bar variant | TTS chip location | STT? |
|---|---|---|---|
| podcast player | podcast | per-word inline | NO |
| book PageReader | (none) | per-word inline (hover popover) | NO |
| book DeckReader | book-deck | per-word inline | NO |
| learn-stations station | card | as part of the card | YES (echo, sequential-drill) |
| learn-stations cards | card | front-card auto-play | NO |
| readalong slide | book-deck variant | per-word inline | NO |
| readalong drill | card | as part of the slot | YES |
| readalong word-rain | (none) | (none) | YES (spoken mode) |
| readalong scenario | (none) | NPC dialogue auto-play | YES (echo replay) |

## Out of scope

- Coqui XTTS / F5-TTS / Azure Neural / Google Cloud TTS backend choice — see memory `feedback_tts_must_be_high_quality.md`; design doesn't pick.
- The phonetic-distance scorer (`lib/score-transcript.ts`) — backend.
- The depot TTS gateway (`/api/tts`) — backend.

## Success criteria

- One `<MediaPlayerBar>` component with three variants.
- ONE speed-ladder rule, applied consistently across each surface family.
- ONE keyboard contract, documented with `aria-keyshortcuts`.
- 7-state STT mic flow designed.
- TTS loading/error states designed.
- MediaSession metadata per surface specified.
- The audio-level waveform is a defined primitive (used by sequential-drill, word-rain spoken mode, scenario echo).
- WCAG AA in all three themes for all chrome.
