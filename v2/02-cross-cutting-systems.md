# v2/02 — Cross-cutting systems

> Five new shared subsystems that span all three apps. Each has an owner-app (where the primary route lives) and integration surfaces in the others.
>
> Resolves themes **I** (video export + YouTube + notification center), **E** (issue flagging UI), **F** (analytics funnel), **J** (avatar primitive).

---

## 1. Notification center (theme I, part 1)

### 1.1 The problem v1 didn't solve

v1 mentions long-running jobs once: the readalong Cloud Run MP4 export (`v1/products/readalong.md` §5 Flow F). It says nothing about how the user knows when their export finished, what other jobs exist, or how to retry one. With v2 adding generative tutorials (theme K), more exports (theme I across podcast / book / flashcard), and the signal materializer, there are now many background jobs the user needs visibility into.

### 1.2 Notification types (initial set)

| Type | Source app | Lifecycle | Default channel |
|---|---|---|---|
| `export.started` | studio (any export) | open until `export.done` | toast pill |
| `export.progress` | studio | replaces previous progress | inline progress bar in toast |
| `export.done` | studio | sticky until tap-to-dismiss | toast pill + inbox row |
| `export.failed` | studio | sticky + retry CTA | toast pill (red) + inbox row |
| `youtube.published` | studio | sticky w/ open-on-YouTube link | toast pill + inbox row |
| `tutorial.ready` | studio (learn-anything generator, `v2/04`) | sticky w/ open CTA | toast pill + inbox row |
| `level.update` | signals.reeloo.ai | low priority, batched daily | inbox row only |
| `streak.broken` | signals.reeloo.ai | low priority | inbox row only |
| `flag.acknowledged` | studio (operator replied) | sticky | inbox row only |

### 1.3 Surfaces

- **Per-app toast.** Top-right pill on desktop, top-center on mobile. Auto-dismiss after 6s for non-sticky types, manual dismiss for sticky. Component: `<NotificationToast>` (new in `v2/05` §2).
- **Header bell + badge.** In the shared app header, between the language selector and the user avatar. Badge shows unread count, capped at "9+".
- **Inbox route.** `*/me/notifications` lists every notification newest-first, grouped by day. Filters: All / Exports / Tutorials / System. Bulk mark-as-read. This is on `practice.reeloo.ai/me/notifications`, with a deep-link from any app's header bell.

### 1.4 YouTube OAuth dance (theme I, part of)

YouTube upload needs `https://www.googleapis.com/auth/youtube.upload`, which NextAuth (currently used for sign-in, `v1/products/learn-stations.md` §2 / readalong §2) handles as an additional scope grant.

```
First export-to-YouTube tap
        │
        ▼
Studio: POST /api/export/youtube/start?materialId=…
        │ checks userYouTubeToken existence
        │
        ├─ NO token  → 200 {needsAuth: true, authUrl: "/api/auth/youtube/grant?return=…"}
        │              client redirects user → Google consent screen
        │              callback writes token to depot users.youtubeRefreshToken
        │              redirect back to /me/notifications with state=youtube_linked
        │
        └─ YES token → 202 {jobId, statusUrl}
                        client opens notification with export.started type
                        SSE / polling updates the toast until export.done
                        on success, kicks off the upload sub-job
                        emits youtube.published when YT returns video id
```

The consent screen is the only blocking UI; everything else is async and surfaces through the notification center. The user can navigate away mid-export and come back via the bell.

### 1.5 Component contract

```ts
type Notification = {
  id: string;
  userId: string;
  type: NotificationType;  // see §1.2
  createdAt: ISO8601;
  readAt: ISO8601 | null;
  payload: {
    title: string;        // "Your podcast video is ready"
    body?: string;        // "Karotten · 12 min · 1080p"
    progress?: number;    // 0..1, only for export.progress
    ctas?: Array<{label: string; href: string; emphasis?: "primary" | "secondary"}>;
    iconKey?: string;     // maps to a sprite in the design system
    severity?: "info" | "success" | "warning" | "error";
  };
};
```

### 1.6 Backing service

`notifications.reeloo.ai` is a thin facade over the depot notes table (`category: "notification"`) with a small `unreadCount` cache per user. Same auth (NextAuth + `X-App-Token` for the studio worker writes). Server-Sent Events at `notifications.reeloo.ai/api/stream?userId=…` push the toast updates; the inbox surface falls back to polling.

---

## 2. Video export pipeline (theme I, part 2)

### 2.1 Lift from readalong

Readalong's Cloud Run MP4 pipeline (`v1/products/readalong.md` §5 Flow F, §8 row 4–6) is the only working video export today:

```
Client (VideoExportTab.tsx)
    │ POST /api/export/start
    ▼
@google-cloud/run kicks Cloud Run Job
    │ writes Postgres row (postgres-export-store.ts)
    │
Worker renders frames + audio → MP4 → uploads to gs://readalong-exports/
    │
SSE updates ExportHistoryItem in client
```

v2 promotes this to `export.reeloo.ai`, a standalone service with one client SDK and one Cloud Run worker pool. Postgres stays as the job store.

### 2.2 Generic input shape

All four "exportable" surfaces produce the same intermediate representation:

```ts
type ExportSource = {
  kind: "podcast" | "readalong-deck" | "flashcard-deck" | "book-chapter";
  sourceUrl: string;       // canonical content URL
  language: string;
  duration_s?: number;
  // The pipeline pulls timed text + audio from the canonical source:
  segments: Array<{
    start_ms: number;
    end_ms: number;
    text: string;
    speakerLabel?: string;
    audioUrl?: string;     // pre-rendered mp3 from audio.reeloo.ai
    annotations?: Array<{word: string; gloss: string; cefr?: string}>;
  }>;
  style: {
    template: "karaoke-vertical" | "deck-presentation" | "flashcard-flip" | "book-reading";
    theme: "vanilla" | "cobalt-dark" | "high-contrast";
    aspectRatio: "9:16" | "16:9" | "1:1";
    showAnnotations: boolean;
    showTranslation: boolean;
  };
  destination: "download" | "youtube" | "both";
};
```

### 2.3 Per-app integration

| App | Source | Template | New entry point |
|---|---|---|---|
| `podcast.reeloo.ai` | one episode (`/podcast/<slug>`) | `karaoke-vertical` (9:16 default) | "Make a video" item in the player overflow menu |
| `book.reeloo.ai` | one chapter (`/read/<slug>/ch/<n>`) | `book-reading` (16:9 default) | "Export chapter as video" in chapter menu |
| `practice.reeloo.ai/library/cards` | one deck | `flashcard-flip` (9:16 default) | "Export deck as video" on deck detail |
| `practice.reeloo.ai/library/decks/<id>` (readalong deck) | full deck | `deck-presentation` (16:9 default) | "Export" CTA, same as today |

Each per-app entry point opens the same `<VideoExportSheet>` (a new shared component in `v2/05` §2). The sheet hosts: aspect-ratio toggle, theme toggle, annotations on/off, translation on/off, destination (download / YouTube / both), and a preview thumbnail.

### 2.4 Job state model

Lifted from readalong:

```
status: queued → rendering → encoding → uploading → done | failed
progress: 0..1, monotonic
artifactUrl: gs://reeloo-exports/<jobId>.mp4 when done
youtubeVideoId: when youtube.published fired
errorMessage: when failed
```

State changes push notifications via §1's center.

### 2.5 Where it lives

- Client SDK: `@reeloo/export-client`
- Service: `export.reeloo.ai/api/{start,status,cancel}`
- Worker: Cloud Run Job, image `eu.gcr.io/reeloo-prod/exporter:latest`
- Bucket: `gs://reeloo-exports/` (rename from `readalong-exports`)
- Postgres: `export_jobs` table (lift from `readalong/src/lib/postgres-export-store.ts`)

The MP4 lives behind a 7-day signed URL; the inbox row caches the URL but the bucket lifecycle rule deletes after 30 days. The YouTube upload is the permanent destination.

---

## 3. Issue-flagging UI (theme E)

### 3.1 What exists

`learn-stations/components/FlagIssue.tsx` is mounted in `app/layout.tsx:106` as a floating chip — clicked → modal → POST `/api/flag-issue` → appended to `data/flag-issues.jsonl` (`v1/products/learn-stations.md` §5 Flow F). The `inspect` skill consumes these.

v2 promotes this to a shared component mounted in every app's root layout.

### 3.2 Two modes

The user's spec said: *quick triage (one tap) OR detailed comments + optional screenshot*. Mocked as:

```
┌─────────────────────┐
│  🚩  Flag issue    ▾│   ← collapsed chip, bottom-right, opacity 0.5 until hover
└─────────────────────┘

         ▼ click

┌──────────────────────────────────────────────────┐
│  Something off?                                  │
│                                                  │
│  Quick triage — one tap:                         │
│  [😖 Confusing] [🐛 Bug] [📝 Wrong content]      │
│  [🎙️ Audio off] [⚡ Slow]                         │
│                                                  │
│  …or describe it:                                │
│  ┌──────────────────────────────────────────┐   │
│  │                                          │   │
│  └──────────────────────────────────────────┘   │
│                                                  │
│  [📷 Attach screenshot]  ← uses html2canvas      │
│                                                  │
│  Auto-attached: route, route params, level,      │
│  last 5 signals (for debug; user can untick)     │
│                                                  │
│              [ Send ]    [ Cancel ]              │
└──────────────────────────────────────────────────┘
```

One-tap mode: tapping any chip immediately sends the flag with type + auto-attached context — no further screen.
Detailed mode: free-text + optional screenshot upload.

### 3.3 Component spec

```tsx
<FlagIssue
  // Always-on positioning override
  placement="bottom-right" | "bottom-center" | "header-action"
  // Auto-context the chip should attach
  context={{
    app: AppName;
    surface: string;       // current route
    surfaceParams: Record<string, string>;
    levelSnapshot?: UserLevelSnapshot;
    recentSignalIds?: string[];
  }}
  // Allow turning off screenshot in environments without html2canvas
  screenshot?: boolean;
  // Override the endpoint per app if needed; default below
  endpoint?: string;       // default "https://signals.reeloo.ai/api/flag-issue"
/>
```

The endpoint is the same as the signal sink (`v2/01` §2.4) so a flag is just one signal type (`flag.issue`) with a richer payload (text, screenshotUrl, triage).

### 3.4 Per-app integration map

| App | Where mounted | Placement |
|---|---|---|
| `podcast.reeloo.ai` | `app/layout.tsx` | `bottom-right`, hidden during fullscreen playback |
| `book.reeloo.ai` | `app/layout.tsx` | `bottom-right`, hidden in Deck mode (use header-action instead) |
| `practice.reeloo.ai` | `app/layout.tsx` (already there — adopt v1 surface) | `bottom-right` |
| `studio.reeloo.ai` | `app/layout.tsx` | `header-action` (operator uses it more; promote to chrome) |

### 3.5 Operator-facing surface

Flags are read by the operator at `studio.reeloo.ai/admin/flags`. The operator can reply (`flag.acknowledged` notification → user). The reply is plain text + optional resolution status. No threading; one round-trip per flag.

---

## 4. Analytics funnel (theme F)

> This is a meta-design constraint, not a feature. It defines what events the surfaces must emit *before* design locks. Designers must check whether each screen's interactions are nameable as events; if not, redesign.

### 4.1 Four funnels worth measuring

| Funnel | Top → Bottom | Why it matters |
|---|---|---|
| **F1. Onboarding** | landing → signup → language picked → first lesson started → first lesson passed → returned next day | retention proxy |
| **F2. Daily loop** | open app → started a session → completed ≥3 stations (or ≥5 min) → saved ≥1 word OR rated ≥3 cards → closed gracefully | stickiness proxy |
| **F3. Vocab save → review** | dict.lookup → dict.save → card.shown (next day) → card.rated `good` (within 3 days) | learning effectiveness |
| **F4. Export → publish** | open material → tap "Make a video" → finished export → published to YouTube → returned to view stats | creator-loop conversion |

### 4.2 Event names per step (cited to v2/01 §1.1 taxonomy)

| Funnel.step | eventType | emitted by |
|---|---|---|
| F1.1 | `nav.landing_view` | marketing site (out of scope, but log it) |
| F1.2 | `auth.signup_complete` | NextAuth callback |
| F1.3 | `onboarding.language_picked` | practice onboarding flow |
| F1.4 | `drill.station_start` (first) | practice |
| F1.5 | `drill.station_pass` (first) | practice |
| F1.6 | `nav.return_day_2` | derived (no direct event; computed from session opens) |
| F2.1 | `session.open` | any app on tab visible |
| F2.2 | `session.activity_start` | first meaningful interaction |
| F2.3 | `drill.station_pass` count ≥3 OR `audio.replay_word` count + read events ≥X | derived |
| F2.4 | `dict.save` ≥1 OR `card.rated` ≥3 | already in taxonomy |
| F2.5 | `session.close` (graceful) | beforeunload |
| F3.1 | `dict.lookup` | already in taxonomy |
| F3.2 | `dict.save` | already in taxonomy |
| F3.3 | `card.shown` | already in taxonomy |
| F3.4 | `card.rated` with `outcome="good"` and `ts-card.shown < 3d` | derived |
| F4.1 | `export.cta_open` | any per-app "Make a video" tap |
| F4.2 | `export.cta_confirm` | sheet "Start" button |
| F4.3 | `export.done` | export service callback |
| F4.4 | `youtube.published` | export service callback |
| F4.5 | `nav.return_to_export_stats` | when user opens `/me/notifications` for that job |

### 4.3 Where each fires — the design constraint

Every funnel step has a **specific UI element** that must exist for the event to fire. The design system locks the affordance.

- F1.3 needs a `<LanguagePicker>` component on onboarding — must be a single primary choice, not a multi-select, so the event fires cleanly.
- F2.1 implies every app must mount a session-start sentinel in root layout — design must accept a non-visible component in `app/layout.tsx`.
- F2.5 needs `beforeunload` instrumentation, which is hostile to SPAs that route in-place. Design must accept that the "close" event is approximate.
- F3.4 needs the `card.rated` UI to expose a four-state outcome (`again`/`hard`/`good`/`easy`, matching SM-2 — `v1/products/learn-stations.md` §4 "Cards (SRS deck)"). Two-state designs lose this funnel.
- F4.1 needs every exportable surface to put the "Make a video" CTA in a discoverable place (overflow menu is fine, but it must have a stable `data-testid` — see `v2/05` §2).

### 4.4 Storage

All funnel events land in the same `signals.reeloo.ai/api/events` sink. The materializer also computes per-funnel rolling rates (1d / 7d / 28d) and exposes them at `signals.reeloo.ai/api/funnels` for the `studio.reeloo.ai/admin/analytics` dashboard.

The user is not the consumer of funnels — the operator is. But because the funnels constrain UI element design, design must know they exist.

---

## 5. Avatar primitive (theme J)

### 5.1 Resolution of v1 open question 7

v1 §9 question 7: *"Pip is the learn-stations mascot. Does it travel to book and podcast, or stay as practice.reeloo.ai's only?"*

**v2 answer: Pip travels, but as a *primitive* with structured modes, not as a free-form character.** The avatar appears in every app, but its behaviour is constrained — proactive (does a thing) or passive (waits to be tapped).

### 5.2 Proactive mode

Pip *initiates* attention. Three sub-affordances:

- **Gesture.** Pip points at the screen element the user should look at. Implemented as a positioned SVG with a directional arm. Triggered by: first-lesson tutorial cues, the level-mismatch dialogue (`v2/01` §4.2), export completion celebrations.
- **Facial expression.** Pip's face has 6 states: `idle`, `encouraging`, `concerned`, `proud`, `thinking`, `apologetic`. The state is driven by the parent surface (e.g. `<DrillStation>` sets `<Avatar mood="proud" />` after a pass-with-perfect-score).
- **Whiteboard.** Pip holds a small whiteboard with a hint string. Triggered when the surface has a `hintText` prop. The whiteboard is `<Avatar mood="thinking" whiteboard={hintText} />`. Used in drill stations when the user has been stuck for >20s.

```
   Whiteboard variant                Gesture variant
   ┌──────────────────┐                 (●—●)
   │ try "der Hund"   │                   /│
   │ — masculine!     │                  / │
   └─────┬────────────┘                ●—  ⬇  ← arrow points
        🐦                            (Pip points at element)
         (Pip holding it)
```

### 5.3 Passive mode

Pip waits. One sub-affordance:

- **Info button.** A small circular `(?)` chip with Pip's face inside, anchored next to a question (e.g. a quiz answer). Tap → popover with explanation. The popover content is a content-author field (`infoExplain` on each station / quiz item); when missing, the chip is hidden.

```
   Q: Wie heißt das?  [der Hund]  ✓  (?🐦)
                                       │
                                       ▼ on tap, popover:
                                   ┌─────────────────────────┐
                                   │ Why "der"?              │
                                   │ Masculine animate nouns │
                                   │ take "der" in nominative│
                                   └─────────────────────────┘
```

### 5.4 Trigger taxonomy

| Trigger | Mode | Visual state |
|---|---|---|
| `levelMismatchDialog.open` | proactive | mood=thinking, gesture toward options |
| `drillStation.stuck_20s` | proactive | mood=encouraging, whiteboard=hint |
| `drillStation.pass_perfect` | proactive | mood=proud, no whiteboard, brief 1.5s celebration |
| `drillStation.fail_repeated` | proactive | mood=apologetic, whiteboard="let's slow down" |
| `quizItem.shown` with `infoExplain` | passive | mood=idle, chip rendered |
| `tutorial.first_lesson_step_1` | proactive | mood=encouraging, gesture to start button |
| `export.done` | proactive | mood=proud, whiteboard="video's ready!" (also drops a notification) |

### 5.5 Visual specification

- Pip is a small bird (yellow + orange) — already established in `learn-stations/`. Re-use the existing asset.
- Container: a 64×64 SVG with a transparent background.
- Position: bottom-left of the surface (away from `<FlagIssue>` bottom-right); 16px from edges; safe-area-aware.
- Z-index: above content, below modals.
- Animation: enter slides up + fades in (`--dur-med` 240ms, `--ease-out`); exits fade out (`--dur-fast` 120ms).
- Mute toggle: `*/me/settings/avatar` — `Always`, `Proactive only`, `Passive only`, `Off`. Default: `Always`.

### 5.6 Component

`<Avatar mood mode whiteboard onDismiss>` as one of the v2 components (`v2/05` §2). Implementations of moods live in the shared package, not per-app.

### 5.7 Anti-patterns

- **No talking heads.** Pip's whiteboard is the only text affordance. No speech bubbles, no chat. Keeps the surface scannable, not chatty.
- **No surprise narration.** Pip doesn't speak unless the user has explicitly opted into a voice tutor. The avatar is visual; voice is a separate primitive.
- **No randomized cuteness.** Mood is set by app logic, not by an idle-jiggle timer. The avatar respects user attention.
- **No avatar on top of `<FlagIssue>`.** They live in opposite corners.
