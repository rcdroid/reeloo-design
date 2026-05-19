# Reeloo — Unified Design Brief (v2, integrated)

> This is the canonical brief. It supersedes `v1/00-unified-brief.md` and integrates the 11 cross-cutting themes the v1 pass underweighted. Per-app source-grounded specs (`v1/products/*.md`) and the token system (`v1/design-system/*.md`) remain authoritative for "what each app does today" and "what hex was sourced where" — v2 cites them and doesn't restate them.
>
> Reading order, again: this file → `v2/01..v2/05` → `v1/products/*.md` for source-of-truth, `v1/00-unified-brief.md` for historical baseline.

---

## 1. Executive summary

Reeloo is **one product line, three domains, six shared services, twenty-four components**.

The product line is the user's daily learning loop: read → listen → speak → review → grow. v1 found that this loop sits across **three physical postures** (eyes-first read on `book.reeloo.ai`, ear-first listen on `podcast.reeloo.ai`, output-first drill on `practice.reeloo.ai`) and recommended keeping them as three independently-deployable Next.js apps — defended at v1 §1 ("Defence of the recommendation"). v2 keeps that decision but answers the question v1 didn't address: **how do the three apps feel like one product?**

The v2 answer is six shared services that span the apps:

| Service | What it is | Where it lives | v1 status |
|---|---|---|---|
| `dictionary.reeloo.ai` | one dictionary contract, all apps consume | already promoted in v1 | ✓ v1 |
| `audio.reeloo.ai` | content-addressed mp3 + timing.json | already promoted in v1 | ✓ v1 |
| `depot.reeloo.ai/api/llm` | one LLM client, gateway-quotaed | already promoted in v1 | ✓ v1 |
| `signals.reeloo.ai` | event sink + user-level model | new in v2 (`v2/01`) | new |
| `notifications.reeloo.ai` | long-running task center | new in v2 (`v2/02` §1) | new |
| `export.reeloo.ai` | generalised MP4 → YouTube pipeline | new in v2 (`v2/02` §2) | new |

And a shared shell: one mascot (Pip, `v2/02` §5), one account (`account.reeloo.ai`), one issue-flagging chip (`v2/02` §3), one notification bell, one personal-surface route family (`*/me`, `v2/03`).

The user's challenge to v1 was *"separate vs linked?"* The v2 position:

> **Separate deploys, linked product.** Three Next.js apps, six shared services, one identity, one signal model, one mascot, one design system.

That's the line.

---

## 2. The eleven themes — where each is addressed

| Theme | Short name | v2 home | One-sentence outcome |
|---|---|---|---|
| A | Separate vs linked | this file §1 + `v2/00` §3 | three deploys, six shared services, one shell |
| B | Signal → level model | `v2/01` §1–§3 | per-language CEFR scalars + 18-type event taxonomy, served by `signals.reeloo.ai` |
| C | Level-mismatch UX | `v2/01` §4 | named pattern + reusable `<LevelMismatchDialog>` with condensed-view + free-nav escape hatches |
| D | Synthesis pass | this file | done by integration; the brief reads as one product |
| E | Issue-flagging UI | `v2/02` §3 | promote learn-stations' `<FlagIssue>` to a global chip in every app |
| F | Analytics funnel | `v2/02` §4 | 4 named funnels (onboarding, daily-loop, vocab-save, export-publish) defined *before* design locks |
| G | Settings + dashboard | `v2/03` | first-class `/me/{progress,history,map,diary,settings,privacy,notifications}` on `practice.reeloo.ai` |
| H | Test-id contract | `v2/05` | `<surface>:<component>:<element>[:<modifier>]` enforced in design, CI, and gallery |
| I | Video export + YouTube + notifications | `v2/02` §1–§2 | one export service, one notification center, YouTube as a second OAuth scope |
| J | Avatar primitive | `v2/02` §5 | Pip as a global primitive with proactive (gesture / face / whiteboard) and passive (info button) modes |
| K | Learn-anything sandbox | `v2/04` | `practice.reeloo.ai/learn-anything/<subject>` with Pyodide / Sandpack / iframe embeds |

Each theme has a single point of design ownership and a clear contract.

---

## 3. Information architecture (v2 unified)

The v1 route tree (v1 §2) stands. v2 adds the shared shell and new routes:

### 3.1 Public surfaces

```
reeloo.ai/                                  # marketing landing (TBD; out of scope)
account.reeloo.ai/                          # NEW — NextAuth + youtube grant; .reeloo.ai cookie scope
│
├── podcast.reeloo.ai/                      # 4 v1 routes + the shared shell
│   ├── /                                   # catalog (theme F funnel F1 entry)
│   ├── /podcast/<slug>                     # focus-pin player; "Make a video" CTA (theme I)
│   ├── /wizard                             # on-demand dialogue gen
│   └── /library                            # bookmarks + recent
│
├── book.reeloo.ai/                         # ~5 v1 routes + the shared shell
│   ├── /                                   # DropZone + curated BookCards (level-aware chips, v2/01 §2.3)
│   ├── /read/<slug>                        # TOC
│   ├── /read/<slug>/ch/<n>?mode=&target=   # Page / Deck reader; "Make a video" CTA (theme I)
│   ├── /search?q=&slug=                    # flexsearch
│   └── /library                            # IndexedDB-uploaded books
│
└── practice.reeloo.ai/                     # the gravity well
    ├── /                                   # course picker (Pip mascot)
    ├── /<course>                           # lesson list, level-aware
    ├── /<course>/l<N>?s=<station>          # 18-station player; level-mismatch dialog at entry
    ├── /<course>/l<N>/condensed            # NEW (v2/01 §4.4) condensed-view escape hatch
    ├── /library/                           # cards + decks + games (v1 §2)
    ├── /games/                             # word-rain, scenario sim
    ├── /daily-review                       # SRS
    ├── /learn-anything                     # NEW (v2/04) input surface
    ├── /learn-anything/<subject>           # NEW generated tutorial player
    ├── /me/                                # NEW (v2/03) personal surfaces
    │   ├── /progress                       # dashboard
    │   ├── /history                        # raw event log
    │   ├── /map                            # learning-map visualization
    │   ├── /diary                          # timeline + weekly summaries
    │   ├── /notifications                  # inbox (v2/02 §1.3)
    │   ├── /settings                       # per-language + per-feature
    │   └── /privacy                        # signal off-switch + data export/delete
    └── /api/                               # STT, TTS, cards-state, telemetry (proxy to signals), dictionary proxy
```

### 3.2 Operator surfaces

```
studio.reeloo.ai/                           # internal, oauth2-proxy gated
├── /deck-generator                         # URL → deck (from readalong)
├── /admin/flags                            # NEW — operator triage of <FlagIssue>
├── /admin/analytics                        # NEW — funnel dashboards (theme F)
├── /admin/level-thresholds                 # NEW — tune level-mismatch deltas (v2/01 §4.7)
├── /admin/prompts/<id>                     # NEW — edit learn-anything outline prompt (v2/04 §2.2)
├── /admin/tutorials                        # NEW — pin/regenerate cached tutorials
├── /admin/*                                # rest of readalong's 8 admin pages
└── /test/*                                 # drill-engine + unit-generate tests
```

### 3.3 Shared services

```
dictionary.reeloo.ai/api/dictionary?word=&lang=        # v1
audio.reeloo.ai/<hash>.mp3 + <hash>.timing.json        # v1
depot.reeloo.ai/api/{llm,tts,wizard/generate,podcast-audio/<hash>.mp3}   # v1
signals.reeloo.ai/api/{events,level/<u>,history/<u>,funnels}            # v2 (v2/01 §2.4, v2/02 §4)
notifications.reeloo.ai/api/{list,unread-count,stream,mark-read}        # v2 (v2/02 §1.6)
export.reeloo.ai/api/{start,status,cancel}                               # v2 (v2/02 §2.5)
```

`account.reeloo.ai` is auth-only; it issues NextAuth cookies for `.reeloo.ai` so the user signs in once and the cookie propagates.

---

## 4. Personas & journeys, revisited with v2 systems

The four v1 journeys (v1 §3) still load-bear. v2 adds the cross-cutting wiring they were missing.

### 4.1 J1 — Morning podcast on a phone, weak hands (revisited)

User opens `podcast.reeloo.ai`. The shared `<AppHeader>` shows Pip's avatar (idle), the notification bell (1 unread: yesterday's export is ready), and the user chip with B1.4 inferred level. The catalog tile order is biased toward the user's level (`signals.reeloo.ai/api/level/<u>?lang=de`).

User taps a German tile. The focus-pin player auto-plays. They hit a word they don't know, tap it — `seekToWord` rewinds 0.5s (`v1/products/podcast.md` §5 Flow B), the annotation rail slides in. They tap "save" — `dict.save` fires to `signals.reeloo.ai`, the word lands in their SRS queue at `practice.reeloo.ai/library/cards`. The save action also emits `card.shown` for tomorrow's review (funnel F3.3, `v2/02` §4.2).

In the player's overflow menu they see "Make a video" (theme I). Tap → `<VideoExportSheet>` (`v2/02` §2.3) opens with aspect-ratio 9:16, theme `cobalt-dark`, destination `download`. Confirm → `export.cta_confirm` event, `<NotificationToast>` shows export progress. User puts phone away; the export finishes; the bell badges; the `export.done` notification appears in `/me/notifications` with a download link.

### 4.2 J2 — Evening reading on a tablet, eyes-only (revisited)

User opens `book.reeloo.ai/read/atomic-habits.../ch/1`. The chapter loads with three subtle chips (`v2/01` §2.3): "challenge / fit / easy" — the current chapter is *fit*. They begin reading; `read.paragraph_dwell_ms` events stream silently.

Halfway in, the level model (which updates on a 5-min materializer cadence) ticks B1 → B1+ on reading. No surfacing — the diary will mention it next week.

They save three glossary entries. Each `dict.save` flows to `signals.reeloo.ai`; tomorrow's `practice.reeloo.ai/library/cards?lang=en&recent=24h` will include them.

End of chapter, they tap "Export this chapter as a video" (theme I). The `<VideoExportSheet>` opens with template `book-reading` 16:9, destination `youtube`. Because YouTube isn't linked yet, the export service responds `needsAuth: true` and routes them through the OAuth grant (`v2/02` §1.4). Consent screen, then back to `/me/notifications` with `state=youtube_linked` and the export already queued.

### 4.3 J3 — Active speaking drill before lunch (revisited)

User opens `practice.reeloo.ai/minna/l8?s=1`. Before the station mounts, the level-mismatch check fires (`v2/01` §4.1). Their inferred Japanese level is N5 (≈A1), the lesson target is also A1 — no dialogue. They proceed.

The sequential-drill station runs (`v1/products/learn-stations.md` §5 Flow B). Each attempt emits `drill.attempt` with STT confidence + score. Pip is in *encouraging* facial state. On the third missed attempt of station 4, Pip switches to proactive mode and lifts a whiteboard: "watch the long vowel ー" (`v2/02` §5.2). The hint comes from the station's `hintText` field.

After station 8 the user is stuck on a substitution drill. Their `quiz.struggle` and `substitution.score` events fire; the materializer notes `weakTopics: ["particle-を"]`. The lesson continues; the next time they open `/learn-anything` and type "Japanese particles", the outline will weight wo-particle (`v2/04` §3).

### 4.4 J4 — Weekly SRS review (revisited)

User opens `practice.reeloo.ai/library/cards?lang=de`. SM-2 picks 30 cards from the union of saved-from-podcast (J1), saved-from-book (J2), and vocab introduced in drills (J3). Each `card.shown` and `card.rated` fires. The rating UI exposes the four-state outcome (`again`/`hard`/`good`/`easy`) — required by funnel F3.4 (`v2/02` §4.3).

At the end of the session, the `<DailyLoopSummary>` (`v2/03` §3.1) congratulates: "17 cards, 1 chapter, 4 stations". The streak bumps to 13.

### 4.5 J5 (NEW) — Learn PyTorch on a Tuesday evening

User opens `practice.reeloo.ai/learn-anything`. Types "PyTorch", taps "Make a tutorial". The outline call fires (`v2/04` §2.2); a `<NotificationToast>` says "Generating your PyTorch tour…". The user keeps reading their book in another tab.

Three minutes later the bell badges: `tutorial.ready`. They tap → `/learn-anything/pytorch` opens with 11 cards. Card 2 is a Pyodide sandbox; `<ProgressModal>` shows "Bringing Python into your browser…". After 3s the sandbox is warm. They edit `print(x.shape)` to `print(x.dtype, x.shape)`, hit Run, see `torch.float32 torch.Size([3])`. `<SandboxCard>` notices the expectedOutput match and emits `drill.station_pass`. Pip flashes *proud*.

The user finishes 7 of 11 cards before bed. `deck.complete` doesn't fire (incomplete), but `card.shown × 7` and `card.rated × 7` do. Tomorrow morning the diary surfaces "Started PyTorch tour, 7 of 11 cards done. Want to keep going?" with a deep-link.

### 4.6 J6 (NEW) — Avatar in-context assistance

User opens a quiz card in J3. The Pip info button `(?🐦)` is anchored next to the answer (passive mode, `v2/02` §5.3). They tap; popover shows "Why 'wo' here? — it marks the direct object." The content came from the station's `infoExplain` field.

Without the avatar primitive, this affordance would be a per-station icon button with no consistent identity. With the avatar primitive, the user sees the same Pip face in the same role across podcast, book, and practice. One mental model, three apps.

---

## 5. The signal model and what it does for design

(Full spec: `v2/01`. Summary here.)

Every meaningful interaction emits an event to `signals.reeloo.ai`. The materializer rebuilds a per-language `UserLevelModel` with six skill scalars + windowed lookups + weak topics + vocab gaps + an inferred CEFR with confidence.

**For design, the consequence is:** every surface that shows content has to know how to *use* the model. Not "decorate with it" — *use* it.

| Surface | What the model changes |
|---|---|
| podcast catalog | tile order biased toward inferred CEFR |
| podcast tile detail | "for your level" chip if `inferredCEFR ≈ targetCEFR` |
| book chapter list | challenge / fit / easy chips (settings-gated) |
| practice course list | lesson sorting by closeness to inferred CEFR |
| practice lesson entry | level-mismatch dialogue (`v2/01` §4) |
| practice library/cards | weakTopics gets a bonus station every 5 cards |
| learn-anything outline | conditions on level + weakTopics |
| me/map | visualizes the whole model (radar + grid + path) |
| me/diary weekly summary | LLM is conditioned on the week's signal delta |

The design language for "level-aware" surfaces uses the `<LevelChip>` component consistently, with three visual states (`challenge` ring tinted `--anno-grammar #859900`, `fit` tinted `--anno-vocab #268bd2`, `easy` tinted `--color-muted`). Never named "easy" or "hard" in the user-visible copy — it reads `Comfortable` / `In your range` / `A stretch`. We're describing the relationship, not labeling the user.

The signal model is *opt-out*, not *opt-in* — defaulting to ON, with a single toggle at `/me/privacy` (`v2/03` §6.2). The toggle is one of the few user-visible "Reeloo learns from you" surfaces; copy must be honest, not coy.

---

## 6. Visual system

v1's token system (`v1/design-system/proposed-tokens.md`) is canonical. v2 adds tokens for the new components; nothing is removed.

### 6.1 v2 token additions

```
--color-avatar-bg              : surface tint behind Pip's body (light/dark variants)
--color-avatar-whiteboard-bg   : the small whiteboard card surface
--color-avatar-whiteboard-fg   : whiteboard text color
--color-notification-info      : --color-primary at 90% (default)
--color-notification-success   : --color-state-correct-solid-bg
--color-notification-warning   : --color-state-warning-solid-bg
--color-notification-error     : --color-state-wrong-solid-bg
--color-level-chip-challenge   : alias of --anno-grammar
--color-level-chip-fit         : alias of --anno-vocab
--color-level-chip-easy        : alias of --color-muted

--radius-toast                 : 12px (matches --radius-md)
--shadow-toast                 : 0 8px 24px rgba(0,0,0,0.18)
--shadow-mismatch-dialog       : 0 24px 64px rgba(0,0,0,0.30)

--motion-toast-enter           : 240ms cubic-bezier(0.22, 1, 0.36, 1)
--motion-mismatch-enter        : 320ms cubic-bezier(0.22, 1, 0.36, 1)
--motion-avatar-bounce         : 360ms cubic-bezier(0.34, 1.56, 0.64, 1)
```

All three themes (`vanilla`, `cobalt-dark`, `high-contrast`) get values; the high-contrast variant tints the avatar surface darker and replaces all `rgba(0,0,0,0.X)` shadows with `--color-border-strong` rings (matching v1's a11y discipline at `v1/00-unified-brief.md` §4).

### 6.2 Theme propagation across domains

A user setting their theme to `cobalt-dark` on `practice.reeloo.ai/me/settings` must propagate to `podcast.reeloo.ai` + `book.reeloo.ai`. Implementation: theme is written to a cookie scoped `.reeloo.ai` (matching the auth cookie), and the pre-hydration script reads the cookie before reading `localStorage` (extends v1 §4 dark-mode strategy).

Defaults remain:
- `podcast.reeloo.ai` → `cobalt-dark`
- `book.reeloo.ai` → `vanilla`
- `practice.reeloo.ai` → `vanilla`

User override is global once set.

### 6.3 Typography

No change from v1 §4. IBM Plex (sans + serif + mono), Noto Sans TC, Noto Serif JP. New components use the existing size scale (11/13/15/19/22/30/34/38) without inventing new sizes.

---

## 7. Component inventory (v2 — 24 components)

v1 §5 inventoried 18 components. v2 adds 6 cross-cutting components. Full per-component test-id contract is in `v2/05` §2.

### 7.1 The 24 components

| Component | Category | Status |
|---|---|---|
| `<ReelooLogo>` | identity | v1 |
| `<AppHeader>` | shell | **new (v2)** |
| `<MediaCard>` | content | v1 |
| `<MediaLibrary>` | content | v1 |
| `<RecentRow>` | content | v1 |
| `<ProgressModal>` | feedback | v1 |
| `<MediaPlayerBar>` | content | v1 |
| `<FocusPinColumn>` | content | v1 |
| `<ReadingColumn>` | content | v1 |
| `<DeckCard>` | content | v1 |
| `<AnnotatedSpan>` | content | v1 |
| `<AnnotationRail>` | content | v1 |
| `<WordKaraoke>` | content | v1 |
| `<LanguageTabs>` | nav | v1 |
| `<StationRunner>` | content | v1 |
| `<SpeakingDrill>` | content | v1 |
| `<Flashcard>` | content | v1 |
| `<DictionaryCard>` | content | v1 |
| `<ThemeProvider>` | shell | v1 |
| `<Avatar>` (Pip primitive) | shell | **new (v2)** |
| `<NotificationToast>` | feedback | **new (v2)** |
| `<FlagIssue>` (global) | feedback | promoted from `learn-stations` (`v1/products/learn-stations.md` §5 Flow F) |
| `<LevelMismatchDialog>` | feedback | **new (v2)** |
| `<SandboxCard>` | content | **new (v2)** |

Plus v1's primitives (`<Button>`, `<Pill>`, `<Toggle>`, `<TextInput>`, `<Select>`, `<Slider>`) and `v2/03` personal-surface components (`<MeLayout>`, `<StreakStrip>`, `<DailyLoopSummary>`, `<LevelChip>`, `<SkillRadar>`, `<CEFRGrid>`, `<TimelineRow>`, `<WeeklySummaryCard>`, `<SettingsRow>`, `<PrivacyToggleRow>`).

The 24 cross-app components ship from one shared package (`@reeloo/ui`); the personal-surface components are app-local to `practice.reeloo.ai` (since `/me` lives there).

---

## 8. The avatar, the chip, and the toast — the shared shell

These three primitives are what make three domains feel like one product.

### 8.1 `<AppHeader>` — top of every page

```
┌──────────────────────────────────────────────────────────────────┐
│  🐦 Reeloo   Podcast · Book · Practice    🇩🇪  🔔₂   👤 B1.4  ▾ │
└──────────────────────────────────────────────────────────────────┘
```

- Reeloo wordmark (links to `*/`).
- Three app nav links — active app gets a `--color-primary` underline.
- Language selector — only shown on apps that are per-language (always shown here for parity).
- Notification bell with unread count (`v2/02` §1.3).
- User chip with inferred level (subtle, optional per `v2/01` §5 open question 2).
- User menu (avatar dropdown): `/me/progress`, `/me/map`, `/me/diary`, `/me/notifications`, `/me/settings`, `/me/privacy`, sign out.

### 8.2 `<Avatar>` — Pip everywhere

(Spec: `v2/02` §5.) Three modes of operation:

- **In the shell**: idle mood, shown in the user menu trigger.
- **Proactive**: gesture / face / whiteboard, lifted by lesson logic.
- **Passive**: info-button (?) chip next to questions.

Across the three apps the Pip identity is the same — same sprite, same six moods, same animations. The reasons it appears differ.

### 8.3 `<FlagIssue>` — bottom-right chip

(Spec: `v2/02` §3.) Mounted in every app's root layout. One-tap triage or detailed comment + screenshot. Auto-attaches the surface + level + last 5 signals so the operator can reproduce.

### 8.4 `<NotificationToast>` — top-right toast

(Spec: `v2/02` §1.5.) Long-running task surface. Same toast component in every app; the inbox lives on practice (`/me/notifications`) and is reachable from any app's header bell.

### 8.5 `<LevelMismatchDialog>` — the level-aware moment

(Spec: `v2/01` §4.) Same dialogue across all three apps when content/level delta is significant. The condensed-view escape hatch is specific to practice; the dialogue surface itself is shared.

---

## 9. Funnels constrain design (theme F as a meta-rule)

Four funnels (`v2/02` §4):

1. **Onboarding** — landing → signup → language picked → first lesson passed → returned day 2.
2. **Daily loop** — open → session active → 3+ stations or 5+ min → ≥1 dict.save or 3+ card.rated → close gracefully.
3. **Vocab save → review** — dict.lookup → dict.save → card.shown → card.rated good within 3d.
4. **Export → publish** — open material → tap Make a video → export.done → youtube.published → return for stats.

For design: every funnel step has a named UI element that must exist. If the design doesn't expose the affordance, the funnel can't measure. This is a constraint, not a feature — every screen must be auditable for the events it can emit.

A handful of concrete consequences (from `v2/02` §4.3):

- The `<LanguagePicker>` in onboarding must be **single-select**, not multi-select.
- The card rating UI must expose **four** outcomes (SM-2 grades), not two.
- "Make a video" must live in a **stable** overflow location across apps so it gets a stable test-id and a stable funnel measurement.
- Session-close detection is best-effort; design accepts it's a noisy event.

---

## 10. Test-id contract (theme H as a design constraint)

(Spec: `v2/05`.)

Format: `<surface>:<component>:<element>[:<modifier>]`. Enforced in:

1. Figma component library (every component has a `Test ID` field).
2. ESLint rule `@reeloo/eslint-plugin/require-testid` in CI.
3. Component-gallery service column "Test IDs".
4. Per-surface smoke contract from the `inspect` skill (`data/verify-routes.json`).

The constraint matters because **anonymous interactive elements are designed-out**. If a button can't be named, it shouldn't exist.

---

## 11. Migration map updates (delta from v1 §6)

The v1 migration table (v1 §6) is preserved verbatim. v2 adds:

```
NEW shared services
  /                                          → account.reeloo.ai             (NextAuth subdomain)
  learn-stations/api/telemetry/learning      → signals.reeloo.ai/api/events  (migrate)
  readalong/api/export/*                     → export.reeloo.ai/api/*        (lift + generalise)
  learn-stations/api/flag-issue              → signals.reeloo.ai/api/flag-issue  (signal sink, same payload shape)
                                             + ALSO shipped globally via shared <FlagIssue> in all apps

NEW practice routes
  /                                          → practice.reeloo.ai/me                (route family)
  /                                          → practice.reeloo.ai/me/{progress,history,map,diary,notifications,settings,privacy}
  /                                          → practice.reeloo.ai/learn-anything
  /                                          → practice.reeloo.ai/learn-anything/<slug>
  /<course>/l<N>                             → /<course>/l<N>/condensed             (condensed-view escape hatch)

NEW studio routes
  /                                          → studio.reeloo.ai/admin/flags
  /                                          → studio.reeloo.ai/admin/analytics
  /                                          → studio.reeloo.ai/admin/level-thresholds
  /                                          → studio.reeloo.ai/admin/prompts/<id>
  /                                          → studio.reeloo.ai/admin/tutorials
```

---

## 12. Anti-patterns (extends v1 §8 "Anti-patterns doc")

In addition to v1's anti-patterns:

- **Don't surface the inferred CEFR as a score** in app chrome. It's a relationship to content, not a grade. Use `<LevelChip>` with `challenge / fit / easy` framing.
- **Don't make Pip chat.** No speech bubbles. Whiteboard text only. The avatar is a visual primitive, not a chatbot.
- **Don't randomize Pip's mood.** Mood is set by app logic. No idle jiggle, no surprise narration.
- **Don't put the avatar on top of `<FlagIssue>`.** They live in opposite corners.
- **Don't show notification toasts for low-priority events.** `level.update` and `streak.broken` go to the inbox only, never to the toast.
- **Don't ship anonymous interactive elements.** Every clickable thing has a test-id.
- **Don't introduce new themes.** v1 locked `vanilla / cobalt-dark / high-contrast`; v2 inherits.
- **Don't pretend the user is opted-in.** The privacy toggle (`v2/03` §6.2) is honest about what the signal sink does.
- **Don't pre-batch tutorials.** `learn-anything` generates per slug on demand and caches; we don't ship 1000 pre-baked tutorials.
- **Don't ship the level-mismatch dialogue without the escape hatch.** Always offer condensed-view OR free-nav OR stay; never strand the user.

---

## 13. In scope / out of scope for the design pass

### 13.1 In scope (v1 §7 items, plus v2 additions)

From v1: tokens, 18-component Figma library, 12 canonical screen mockups, dark + a11y variants for 4 most-used components, iconography contract, empty/loading/error states, onboarding flow.

Newly in scope for v2:

- 6 new component specs (Avatar, NotificationToast, FlagIssue-global, LevelMismatchDialog, SandboxCard, AppHeader) in Figma + Code Connect map.
- Personal-surface mocks: `/me/progress`, `/me/map` (radar + grid + path), `/me/diary`, `/me/settings`, `/me/privacy`, `/me/notifications`.
- 6 avatar moods (idle, encouraging, concerned, proud, thinking, apologetic) — sprite states + animations.
- Notification toast variants per type (info / success / warning / error + export progress).
- Level-mismatch dialogue (overleveled + underleveled variants, mobile + desktop).
- Condensed-view runner mock.
- Learn-anything input + tutorial player + sandbox card mocks.
- `<AppHeader>` with all three apps' active state.
- Test-id contract documentation built into Figma's component properties panel.

### 13.2 Out of scope (v1 §7 list stays, with one addition)

- v1 list unchanged.
- v2 adds: **operator-facing studio surfaces** (`studio.reeloo.ai/admin/*`) — the operator dashboards for flags, analytics, level-thresholds, prompts, tutorials are functional, not user-facing. Spec them later if the operator role grows.

---

## 14. Open questions (extends v1 §9)

v1 questions 1–6, 8–10 remain open and worth design's attention. v2 resolves #7 (Pip travels, as a primitive). v2 adds:

11. **Should `/me` propagate to `book` and `podcast` as redirects or as duplicate surfaces?** v2 picks redirect-to-practice. If product wants offline-first behaviour in `book.reeloo.ai`, this might need revisiting.
12. **CEFR confidence band threshold for dialogue suppression.** v2 picks 0.6 (with a 1-week observation cooldown when confidence < 0.5). Worth user-test.
13. **Sandbox runtime pre-flight notice.** v2's open question (`v2/04` §7) on whether Pyodide's ~5MB / 3s cold start needs a one-time toast or always-show.
14. **Per-language UI rollout for learn-anything.** v2's open question (`v2/04` §7) on English-first MVP vs simultaneous multi-language.
15. **The signal SDK in podcast + book.** Both apps currently emit nothing. The SDK is straightforward to ship, but the design must agree on which interactions emit which events (preferably the same shape as `v2/01` §1.1).
16. **YouTube export as the only paid-side affordance.** If the product later monetises export quotas, the design must accommodate a quota chip near the "Make a video" CTA. v2 doesn't address; design can plan ahead.
17. **Diary tone.** Auto-generated weekly summaries (`v2/03` §5.2) are the only LLM-authored user-facing copy. The voice (encouraging? factual? terse?) is a content-strategy call that design owns the rendering of but copywriting/product owns the prompt for.

---

## 15. Deliverables requested from Claude design (extends v1 §8)

1. **`tokens.json`** — the v1 token contract plus the v2 additions in §6.1 (avatar, notification, level-chip, toast, mismatch-dialog, avatar-bounce). Three themes for all.
2. **Figma library (24 components)** — v1's 18 plus v2's 6, each with state + theme variants. New components must include `data-testid` field in properties (`v2/05` §3.5).
3. **Personal-surface library** — the 10 components from `v2/03` §7 as a sub-library tagged "practice/me".
4. **18 screen mockups (v1's 12 + v2's 6)**:
   - v1: catalog grid, V3 player, wizard form, annotation rail open, book home, TOC, Page mode reader, Deck mode reader, course picker, station player, library cards, daily-review.
   - v2 additions: `/me/progress` dashboard, `/me/map` (radar + grid + path), `/me/diary` (week summary + timeline), `/learn-anything` input, `/learn-anything/<slug>` with sandbox card, the `<LevelMismatchDialog>` overleveled variant.
5. **Dark + a11y variants** for v1's 4 most-used + v2's 6 new components (Avatar in `cobalt-dark` is a different palette than `vanilla`; `<NotificationToast>` needs `high-contrast` with `--color-border-strong` ring instead of shadow).
6. **Code Connect map** — extend v1's 18-entry map to 24 entries, linking each Figma component to the shared `@reeloo/ui` package path or the `practice/me` app-local path.
7. **Migration playbook updates** — `learn-stations/components/FlagIssue.tsx → @reeloo/ui/FlagIssue.tsx` plus per-app mount instructions; the readalong export pipeline lift recipe.
8. **Anti-patterns doc** — v1's plus v2's §12.
9. **Test-id starter sheet** — the per-component checklist from `v2/05` §2 as a Figma sticker sheet.
10. **Avatar sprite sheet** — Pip in 6 moods × 2 lighting (light/dark) at 64×64 + 128×128 + 256×256.

---

## 16. TL;DR (one paragraph)

Reeloo is one product line with three physical postures (read, listen, drill), shipped as three Next.js apps (`podcast.reeloo.ai`, `book.reeloo.ai`, `practice.reeloo.ai`) plus an internal `studio.reeloo.ai`. v1 made that call; v2 keeps it but adds the cross-cutting wiring v1 didn't model: a per-user **signal model** (`signals.reeloo.ai`) that turns every interaction into a CEFR-aligned skill vector, a **level-mismatch UX pattern** that offers condensed-view + free-nav escape hatches, a **notification center** + generalized **video export pipeline** (lifted from readalong's Cloud Run stack) for long-running tasks including YouTube publishing, a global **issue-flagging chip** promoted from `learn-stations`, a **funnel-first analytics contract** that constrains UI affordances before design locks, an **avatar primitive** (Pip) with proactive + passive modes that travels across all three apps, a **personal-surface family** (`*/me/{progress,history,map,diary,settings,privacy,notifications}`) that lives on `practice.reeloo.ai`, a **learn-anything generative-tutorial mode** with Pyodide/Sandpack sandboxes at `practice.reeloo.ai/learn-anything/<subject>`, and a **`data-testid` design contract** so every interactive element ends up with a stable named identity. The 18 v1 components grow to 24; the 3 v1 services grow to 6; the architectural recommendation survives the user's "separate vs linked" challenge in the form: **separate deploys, linked product — one identity, one signal model, one mascot, one design system, across three domains shaped to their postures.**
