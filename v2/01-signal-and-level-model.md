# v2/01 — Signal collection and the user-level model

> Resolves themes **B** (signal collection → user-level model) and **C** (level-mismatch UX pattern) as one connected spec.
>
> Status: net-new. No v1 file describes a unified level model; v1 products mention isolated state (`v1/products/learn-stations.md` §4 "Telemetry" — `data/learning-events.jsonl`; `v1/products/readalong.md` §4 "Storage" — Firestore mastery). v2 unifies them.

---

## 1. The signal taxonomy — what every app emits

Every meaningful user interaction emits a signal to a single append-only event sink at `signals.reeloo.ai/api/events`. Each event is one JSON object on one line (JSONL), following the schema already used by `learn-stations` (`v1/products/learn-stations.md` §4 "Telemetry"):

```ts
{
  ts: "2026-05-19T07:31:04Z",
  userId: "u_8f3a…",
  sessionId: "s_…",
  app: "podcast" | "book" | "practice" | "studio",
  surface: string,        // route, e.g. "/podcast/karotten"
  eventType: SignalType,  // see table below
  lang: "de" | "ja" | "en" | "zh-tw",
  payload: Record<string, unknown>
}
```

### 1.1 The 18 signal types (cited to v1 products where the interaction already exists)

| `eventType` | Emitter app(s) | What it tells the level model | v1 evidence |
|---|---|---|---|
| `dict.lookup` | podcast, book, practice | recent vocab familiarity | `v1/products/podcast.md` §5 Flow B `seekToWord`; `v1/products/book-app.md` §5 Flow D "Runtime annotation" |
| `dict.save` | podcast, book, practice | the user actively claimed unknown | new (v2/06 §3.2 cross-app save action) |
| `card.shown` | practice | exposure event for SRS gap analysis | `v1/products/learn-stations.md` §5 Flow C |
| `card.rated` | practice | `easeFactor` / `missed` outcome | `v1/products/learn-stations.md` §4 "Cards (SRS deck)" |
| `card.flip_latency_ms` | practice | recognition speed proxy | new |
| `drill.station_start` | practice | station begin | `v1/products/learn-stations.md` §4 "Telemetry" |
| `drill.attempt` | practice | per-attempt score + STT confidence | `v1/products/learn-stations.md` §5 Flow B |
| `drill.station_pass` | practice | pass with accuracy | same |
| `drill.station_skip` | practice | user bailed | new |
| `substitution.score` | practice | grammar template mastery | `v1/products/learn-stations.md` §7 `substitution-pattern` station |
| `quiz.struggle` | practice | multiple wrong on one item | derived from `drill.attempt` |
| `read.paragraph_dwell_ms` | book, podcast | comprehension speed proxy | `v1/products/book-app.md` §5 Flow A |
| `read.translation_toggled` | book, podcast | comprehension fallback | `v1/products/book-app.md` §5 Flow F |
| `audio.replay_word` | podcast, book | listening confidence proxy | `v1/products/podcast.md` §5 Flow B |
| `audio.speed_change` | podcast, book | listening confidence proxy | `v1/products/podcast.md` (player speed slider) |
| `deck.complete` | practice | full deck pass | `v1/products/readalong.md` §5 Flow A |
| `game.session` | practice | engagement proxy (no level signal) | `v1/products/readalong.md` §5 Flow E |
| `flag.issue` | all | content quality signal, not user level | `v1/products/learn-stations.md` §5 Flow F |

### 1.2 Where each app calls the sink

| App | Existing emit point | v2 change |
|---|---|---|
| `podcast.reeloo.ai` | none today | NEW: client SDK wraps `KarottenPlayerV3` word + speed + replay events |
| `book.reeloo.ai` | none today | NEW: client SDK wraps `PageReader` / `DeckReader` events |
| `practice.reeloo.ai` | `POST /api/telemetry/learning` → `data/learning-events.jsonl` (`v1/products/learn-stations.md` §4) | MIGRATE: same shape, new URL `https://signals.reeloo.ai/api/events`. Local JSONL kept as buffer in dev. |
| `studio.reeloo.ai` | none | NEW: emits only `flag.issue` (operator surface, no level model) |

The signal SDK is a single shared package (`@reeloo/signals-client`) so each app imports `recordEvent(type, payload)` — that's the only API any product surface needs to know.

---

## 2. The user-level model

### 2.1 Shape

A per-user, per-language object that the model materializer rebuilds every N events (or on a 5-minute timer, whichever is sooner):

```ts
type UserLevelModel = {
  userId: string;
  lang: "de" | "ja" | "en";
  updatedAt: ISO8601;

  // Coarse skill scalars (0..6, CEFR-aligned: A1=1, A2=2, B1=3, B2=4, C1=5, C2=6)
  skills: {
    listening: number;   // weighted from audio.replay_word, audio.speed_change, drill.attempt(listen-mc)
    reading:   number;   // weighted from read.paragraph_dwell_ms, dict.lookup density
    speaking:  number;   // weighted from drill.attempt STT confidence + substitution.score
    writing:   number;   // not measured today; reserved
    vocab:     number;   // weighted from dict.* + card.rated outcomes
    grammar:   number;   // weighted from substitution.score + quiz.struggle on grammar items
  };

  // The two windowed vocab samples the user described in theme B
  recentLookups20: Array<{word: string, lang: string, ts: ISO8601, surface: string}>;
  overallLookups100: Array<{word: string, lang: string, count: number}>;

  // Weak spots — derived from quiz.struggle clustering
  weakTopics: Array<{topic: string, miss_rate: number, last_seen: ISO8601}>;

  // Vocabulary gaps from card.rated outcomes
  vocabGaps: Array<{word: string, ease: number, dueAt: ISO8601}>;

  // Inferred course-level fit
  inferredCEFR: "A1" | "A2" | "B1" | "B2" | "C1" | "C2";
  inferredCEFRConfidence: number; // 0..1
};
```

### 2.2 Materializer

- Implemented as a depot worker reading from `signals.reeloo.ai/api/events` (or directly from the SQLite event table when co-located).
- Re-derives the level model with a simple weighted-average rule per skill, plus EMA decay so stale events lose weight.
- Writes the latest snapshot to `signals.reeloo.ai/api/level/<userId>?lang=de`.
- A "why this level" endpoint returns the top 5 events that moved each scalar in the last week — used by the `*/me/map` dashboard surface (`v2/03` §4).

### 2.3 How it influences the apps

| Consumer | Read | Effect |
|---|---|---|
| `practice.reeloo.ai/<course>` lesson picker | `GET /level/<u>?lang=de` | sort lessons by closeness to `inferredCEFR`; gray out clearly-too-easy ones |
| `practice.reeloo.ai/<course>/l<N>?s=<station>` station player | level + `weakTopics` | inject one bonus drill targeting weakTopics every 5 stations |
| `book.reeloo.ai/read/<slug>` chapter picker | level | mark chapters as "challenge", "fit", "easy" with three subtle chips |
| `podcast.reeloo.ai/` catalog | level | "for you" row ordered by `inferredCEFR` proximity |
| `practice.reeloo.ai/library/cards` SRS scheduler | `vocabGaps` | already used; now centralised |
| `practice.reeloo.ai/learn-anything/<subject>` | level + `weakTopics` | tutorial generator conditions on level (see `v2/04` §3) |
| Level-mismatch dialogue (§4 below) | level + current content target_cefr | trigger when delta ≥ 2 CEFR steps |

### 2.4 Where the model lives — service vs depot table

**Pick:** new shared service `signals.reeloo.ai`, backed by depot's SQLite. Not a separate database.

Rationale:
- v1 already routes LLM through depot (`depot.reeloo.ai/api/llm`, v1 §1) — adding two more endpoints to the same surface is the lowest-friction expansion.
- Append-only event log is a natural fit for SQLite + FTS5, which depot already uses for memory (per the global CLAUDE.md "minis directory service" note).
- The "service" framing matters for the design system because the user can see a "Reeloo learns from you" badge that explains the signals — not just an opaque internal log.

The endpoint surface is:

```
POST   signals.reeloo.ai/api/events          # append (JSONL ingest)
GET    signals.reeloo.ai/api/level/<userId>?lang=<lang>      # latest level snapshot
GET    signals.reeloo.ai/api/level/<userId>/why?lang=<lang>  # top events that moved the model
GET    signals.reeloo.ai/api/history/<userId>?since=…&app=…  # raw event stream for the diary surface
DELETE signals.reeloo.ai/api/events/<userId>?confirm=…       # full erasure (privacy)
```

---

## 3. Privacy and the "off switch"

- Every emitting app has a settings toggle (`*/me/settings/privacy`) — `Send signals to my level model` (default ON).
- When OFF, the SDK becomes a no-op locally; the level model goes stale.
- A DELETE endpoint wipes the user's row in the events table and the materialized level snapshot.

This pattern is mocked in `v2/03` §6.

---

## 4. The level-mismatch UX pattern (theme C)

### 4.1 Trigger

When the user enters any content surface (lesson, chapter, podcast tile), the surface fetches `GET signals.reeloo.ai/api/level/<u>?lang=<l>` and compares `inferredCEFR` to `content.targetCEFR`:

- **Underleveled** (content >> user): `content.targetCEFR - inferredCEFR ≥ 1` and confidence ≥ 0.6 → show "Heads up, this might be tough" dialogue.
- **Overleveled** (content << user): `inferredCEFR - content.targetCEFR ≥ 2` and confidence ≥ 0.6 → show "Your level seems higher than this" dialogue (the one the user described).
- **Fit**: no dialogue.

### 4.2 The overleveled dialogue (mocked copy)

```
┌──────────────────────────────────────────────────────────────────┐
│  🐦  Pip says…                                                   │
│                                                                  │
│  Your level looks higher than this lesson's target (B2 vs A2).  │
│  Two options:                                                    │
│                                                                  │
│   ▷  [Quick condensed review]                                    │
│       Skip-ahead summary of this unit's 12 cards in 90 seconds. │
│                                                                  │
│   ▷  [Free navigation]                                           │
│       Unlock the lesson grid so you can pick any station.        │
│                                                                  │
│   ▷  [Stay on the guided path]   (default; one tap to dismiss)  │
│                                                                  │
│   ▢  Don't ask again for this course                             │
└──────────────────────────────────────────────────────────────────┘
```

Visual: modal sheet, slides up from bottom on mobile; centered with backdrop blur on desktop. Pip avatar in proactive mode (`v2/02` §5.2), gesturing at the option list. Uses tokens `--color-bg-elevated`, `--shadow-elevated`, `--radius-lg`.

### 4.3 The underleveled dialogue (mocked copy)

```
┌──────────────────────────────────────────────────────────────────┐
│  🐦  Pip says…                                                   │
│                                                                  │
│  This is a stretch (target B2, you're around A2). That's okay — │
│  I'll lean on glosses + slow audio.                              │
│                                                                  │
│   ▷  [Continue with extra help]   (default)                      │
│       Auto-show translations, default playback 0.7×.             │
│                                                                  │
│   ▷  [Go easier instead]                                          │
│       Show me 3 lessons closer to my level.                      │
│                                                                  │
│   ▢  Don't ask again for this course                             │
└──────────────────────────────────────────────────────────────────┘
```

Same surface, different copy + Pip in *encouraging* facial state.

### 4.4 Condensed-view surface (option 1 of overleveled dialogue)

A new route `practice.reeloo.ai/<course>/l<N>/condensed` that renders:

- A vertical scroll of all 12 stations as **cards** instead of full drills.
- Each card shows the prompt + expected answer + a single-tap "I knew this" / "Let me try" button.
- Audio for each card available via a single `<MediaPlayerBar>` (v1 §5 component) on tap.
- Completing the condensed view emits one `drill.station_pass` per "I knew this" + zero events for "Let me try" cards (which then route into the normal station flow).

This pattern is reusable — any surface with sequential micro-units can offer condensed mode. Document it in the component inventory as `<CondensedRunner>` (`v2/05` §2).

### 4.5 Free-navigation unlock

`practice.reeloo.ai/<course>/l<N>?s=<station>` already supports query-param station selection (`v1/products/learn-stations.md` §3 Route map). The unlock simply (a) renders the full station grid as an interactive index and (b) sets `localStorage.setItem('practice.freeNav.<course>', '1')` so the user doesn't see the dialogue again on this course. The default linear flow stays the recommendation; free-nav is opt-in.

### 4.6 Sequence diagram

```
User opens /minna/l8?s=1
        │
        ▼
Lesson shell mounts ──► GET signals.reeloo.ai/api/level/<u>?lang=ja
                                                │
                       inferredCEFR = B2 ◄──────┘
                       lesson.targetCEFR = A2
                       delta = 2  →  trigger overleveled dialogue
        │
        ▼
LevelMismatchDialog renders (Pip proactive)
        │
        ├─ User taps "Quick condensed review"
        │       │
        │       ▼   route → /minna/l8/condensed
        │       │
        │       └── CondensedRunner emits drill.station_pass events
        │
        ├─ User taps "Free navigation"
        │       │
        │       └── unlocks /minna/l8 station grid, sets localStorage flag
        │
        └─ User taps "Stay on the guided path"
                │
                └── dismiss; record dialog.dismissed event so we don't repeat
```

### 4.7 Threshold tuning

The CEFR-delta thresholds (≥1 for underleveled, ≥2 for overleveled) and the 0.6 confidence floor are starting values, not laws. They live in a config file the operator can tweak from `studio.reeloo.ai/admin/level-thresholds` without a deploy.

---

## 5. Open questions for design

1. Three "your level vs this content" chips (challenge / fit / easy) on `book.reeloo.ai` — should they be visible by default or revealed under a toggle? My vote: revealed under a toggle (level-aware-mode in `*/me/settings`), to avoid scolding the user every chapter.
2. Where to show the inferred CEFR badge in the app chrome — header next to the user avatar (always visible) or only on `*/me/map`? My vote: only on `*/me/map` + the level-mismatch dialogue itself, to avoid making it feel like a score.
3. Confidence band visualization — when `inferredCEFRConfidence < 0.5`, should the dialogue suppress itself entirely or show a softer "I'm not sure yet" variant? My vote: suppress for ≥1 week of activity, then show the soft variant.
