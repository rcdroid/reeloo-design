# v2/03 — Personal surfaces (`*/me`)

> Resolves theme **G**: settings, progress dashboard, learning map, diary.
>
> Status: new. v1 mentions `/settings` and `/profile` on practice (v1 §2) but does not specify them.

---

## 1. Domain pick — where `/me` lives

**Decision:** all personal surfaces live at `practice.reeloo.ai/me/*`. The other two domains expose `*/me` as a redirect to the practice surface.

Defence:
- `practice.reeloo.ai` is the gravity well (v1 §1 bullet 2). It's where the user already has SRS state (`v1/products/learn-stations.md` §4 "Cards (SRS deck)") and the operator workflow culminates.
- Logging in once on `practice.reeloo.ai` and propagating cookies to `book.reeloo.ai` + `podcast.reeloo.ai` via a shared `account.reeloo.ai` auth subdomain (NextAuth domain cookie scope `.reeloo.ai`) means the user only sees the same `/me/*` surfaces from one place.
- Avoids duplicating four surfaces in three codebases.

The shared `<AppHeader>` (in `v2/05` §2) shows a single user avatar + dropdown that links to `/me/history`, `/me/progress`, `/me/map`, `/me/diary`, `/me/notifications`, `/me/settings`, `/me/privacy`, `Sign out`. The dropdown items always link to `practice.reeloo.ai/me/<tab>`.

---

## 2. The `/me/*` route family

```
practice.reeloo.ai/me                  redirect → /me/progress
practice.reeloo.ai/me/progress         progress dashboard
practice.reeloo.ai/me/history          full activity history (raw event log, paginated)
practice.reeloo.ai/me/map              learning map (visualization of the level model)
practice.reeloo.ai/me/diary            timeline of activity + auto weekly summaries
practice.reeloo.ai/me/notifications    inbox (v2/02 §1)
practice.reeloo.ai/me/settings         settings (per-language, per-feature)
practice.reeloo.ai/me/privacy          signal sink off-switch + data export / delete
```

A single `<MeLayout>` renders left-rail tabs + content. On mobile the tabs become a horizontal scroll under the page header.

---

## 3. `/me/progress` — progress dashboard

The summary surface — what would I want to see in a 5-second glance?

Layout (desktop):

```
┌────────────────────────────────────────────────────────────────────┐
│  AppHeader                                                          │
├──────────────┬─────────────────────────────────────────────────────┤
│              │  ┌─────────────────────────────────────┐             │
│   /me        │  │ Streak: 12 days  (longest: 27)      │             │
│              │  │ ████████████ █▀                      │             │
│  · Progress  │  └─────────────────────────────────────┘             │
│  · History   │  ┌──────────────┐ ┌────────────────────────┐         │
│  · Map       │  │ Today's loop │ │ Inferred level: B1.4   │         │
│  · Diary     │  │ 17 cards     │ │ DE — based on 1,247 ev │         │
│  · Notif.    │  │ 4 stations   │ │ [why?]                  │         │
│  · Settings  │  │ 1 chapter    │ └────────────────────────┘         │
│  · Privacy   │  └──────────────┘                                    │
│              │  ┌─────────────────────────────────────────────┐    │
│              │  │ This week                                    │    │
│              │  │ • 84 words saved   • 156 cards rated         │    │
│              │  │ • 3 lessons passed • 1 chapter finished      │    │
│              │  │ • Word-rain best: 23s                         │    │
│              │  └─────────────────────────────────────────────┘    │
│              │  ┌─────────────────────────────────────────────┐    │
│              │  │ Pick up where you left off                   │    │
│              │  │ ▷ Minna L8 — substitution-pattern (3/12)     │    │
│              │  │ ▷ Atomic Habits — chapter 4                  │    │
│              │  │ ▷ Karotten (DE) — 3:42 in                    │    │
│              │  └─────────────────────────────────────────────┘    │
└──────────────┴─────────────────────────────────────────────────────┘
```

Each "Pick up" row is one click → routes to the right app's deep link. The streak header uses the `--anno-cultural #cb4b16` token for the active days, dimming inactive ones. The level chip is a link to `/me/map`.

### 3.1 Components used

- `<StreakStrip>` — new in `v2/05` §2
- `<DailyLoopSummary>` — new
- `<LevelChip>` — new, also reused in app headers and the level-mismatch dialogue
- `<RecentRow>` — reused from v1 §5 (`podcast-app/RecentRow.tsx`)

---

## 4. `/me/map` — learning map (visualize the level model)

The user asked: *"the learning map visualizes the user-level model from B"*.

### 4.1 Metaphor pick

Three candidates considered:

| Metaphor | Pros | Cons |
|---|---|---|
| **Grid** (CEFR × skill matrix) | factual, dense, scannable | reads like a transcript |
| **Path** (Duolingo-style linear) | warm, motivational, journey-feel | hides multi-skill nuance |
| **Radar / spider** | shows the 6 skill scalars at once | hard to compare over time |

**Decision: a hybrid** — radar at the top (snapshot of the six skills as a chart), grid below (CEFR row × skill column, cells coloured by mastery), and an expandable path beneath (chronological milestones the user hit). Three views of the same data; each answers a different question:

- *How am I doing right now?* → radar.
- *Where am I strong vs weak?* → grid.
- *How did I get here?* → path.

### 4.2 Mock

```
┌──────────────────────────────────────────────────────────────────┐
│  Learning map — Deutsch                                           │
│                                                                  │
│                   Listening                                       │
│                        ●●●●○○                                      │
│              Vocab ●●●●●○      Reading ●●●●○○                     │
│                                                                  │
│              Grammar ●●●○○○      Speaking ●●●○○○                  │
│                        Writing                                    │
│                        ●●○○○○                                      │
│                                                                  │
│  ────────────────────────────────────────────────────────────    │
│         A1     A2     B1     B2     C1     C2                    │
│  Listen ████  ████  ████  ██░░  ░░░░  ░░░░                       │
│  Read   ████  ████  ██░░  ░░░░  ░░░░  ░░░░                       │
│  Vocab  ████  ████  ████  ██░░  ░░░░  ░░░░                       │
│  Gram.  ████  ███░  ░░░░  ░░░░  ░░░░  ░░░░                       │
│  Speak  ████  ███░  ░░░░  ░░░░  ░░░░  ░░░░                       │
│                                                                  │
│  Top movers this week (from /level/why):                          │
│  • Listening +0.3 — 6 podcast sessions (Karotten, Karotten-2…)   │
│  • Vocab     +0.2 — 47 dict.save events across podcast + book    │
│  • Grammar   -0.1 — 3 missed substitution-pattern stations       │
│                                                                  │
│  ▷  Suggested next: Minna L12 (B1 grammar push)                   │
│  ▷  Or:             Atomic Habits ch. 4 (B1 reading)              │
└──────────────────────────────────────────────────────────────────┘
```

- Radar uses `--color-primary` for the filled poly with 30% opacity, `--color-border-strong` for the outline.
- Grid cells: full=`--color-state-correct-bg`, half=`--color-state-warning-bg`, empty=`--color-bg-elevated`.
- Suggestions come from the materializer (`v2/01` §2.2 — same "why" endpoint feeds both).

### 4.3 Per-language switcher

Top of `/me/map` has a language tabset matching v1's `<LanguageTabs>` (v1 §5). Switching tabs re-fetches `signals.reeloo.ai/api/level/<u>?lang=ja` and re-renders.

---

## 5. `/me/diary` — activity timeline + weekly summaries

The diary is a chronological list of everything that happened, with a generated summary at the top of each week.

### 5.1 Layout

```
┌──────────────────────────────────────────────────────────────────┐
│  Diary                                                            │
│                                                                  │
│  ▼ Week of May 12–18, 2026                                        │
│    ┌────────────────────────────────────────────────────┐        │
│    │ Generated summary                                   │        │
│    │ You finished the Karotten German series (6 eps),    │        │
│    │ saved 84 words, and pushed B1 listening up 0.3.    │        │
│    │ Stuck point: substitution-pattern on dative.        │        │
│    │ Win: first complete chapter of Atomic Habits.       │        │
│    └────────────────────────────────────────────────────┘        │
│                                                                  │
│    Tue May 17                                                    │
│    • 07:31 — opened podcast Karotten (DE), played 12 min        │
│    • 07:43 — saved 4 words: "Möhre, Karotte, knackig…"          │
│    • 08:02 — practice: Minna L8 station 3 — passed              │
│    • 21:14 — book: Atomic Habits ch.2 → ch.3 (Deck mode, 22min) │
│                                                                  │
│    Mon May 16                                                    │
│    • 06:50 — game/word-rain best score 23s                       │
│    • …                                                            │
│                                                                  │
│  ▶ Week of May 5–11, 2026  (click to expand)                      │
└──────────────────────────────────────────────────────────────────┘
```

- The summary is generated by the depot LLM gateway (`depot.reeloo.ai/api/llm`, v1 §1) every Monday morning, conditioned on the previous week's events.
- The prompt is one of the few user-visible LLM surfaces; copy must be friendly + factual, not motivational-coach-tone.
- The raw timeline is paginated, 20 days per page, lazy-loaded on scroll.
- Each row links to its deep route — the `/library/cards` rating, the podcast tile, the book chapter.

### 5.2 Auto weekly summary contract

```
POST depot.reeloo.ai/api/llm
{
  model: "gemini-3.1-flash-preview",
  prompt: "<system + the user's last-7d event aggregate JSON>",
  outputSchema: {
    headline: "string ≤ 90 chars",
    body: "string ≤ 300 chars",
    stuckPoints: "string[]",
    wins: "string[]"
  }
}
```

Cached per user per week. Generated once, surfaceable forever.

---

## 6. `/me/settings` and `/me/privacy`

### 6.1 Settings groups

| Group | Settings |
|---|---|
| **General** | display name, timezone, default language, theme (vanilla / cobalt-dark / high-contrast), Pip avatar mode (Always / Proactive only / Passive only / Off — `v2/02` §5.5) |
| **Per language** | UI of the language, TTS voice preference, STT confidence floor, audio playback default speed, translate-to (zh-TW / en) |
| **Practice** | drill ordering (linear / shuffled), free-nav unlock per course, condensed-view default (Ask / Always / Never), session length goal |
| **Read** | reader font size override (default 19px), annotation density (all / mid / minimal), karaoke highlight on/off |
| **Listen** | default speed (0.7 / 0.85 / 1.0 / 1.15 / 1.25), auto-replay-on-segment-end on/off |
| **Notifications** | email digest (off / weekly), per-type push allowlist |
| **Account** | sign out everywhere, link YouTube (status from `v2/02` §1.4), connected accounts |

### 6.2 Privacy

`/me/privacy` is a dedicated tab because the signal sink (`v2/01` §3) is a meaningful trust surface:

```
┌──────────────────────────────────────────────────────────────────┐
│  Privacy                                                          │
│                                                                  │
│  Send signals to my level model     [ ON  ▮▯ ]                    │
│  (When off: your level stops updating; we stop logging events.)  │
│                                                                  │
│  Include screenshots when I flag    [ OFF ▯▮ ]                   │
│  (Off by default. Adds an html2canvas render to the flag.)       │
│                                                                  │
│  Auto-attach last 5 signals to flag [ ON  ▮▯ ]                    │
│  (Helps the operator reproduce.)                                 │
│                                                                  │
│  ─────────────────────────────────────────────────               │
│  My data                                                          │
│                                                                  │
│  ▷  Download all my signals  (.jsonl)                             │
│  ▷  Download my level snapshots (.json)                           │
│  ▷  Delete everything   ← double-confirm                          │
└──────────────────────────────────────────────────────────────────┘
```

All three buttons call documented endpoints from `v2/01` §2.4.

---

## 7. Component dependencies (new in v2)

- `<MeLayout>` — left-rail nav + content; mobile-friendly
- `<StreakStrip>`
- `<DailyLoopSummary>`
- `<LevelChip>`
- `<SkillRadar>` — 6-axis radar chart
- `<CEFRGrid>` — 5×6 mastery grid
- `<TimelineRow>`
- `<WeeklySummaryCard>`
- `<SettingsRow>` — label + control + optional help text
- `<PrivacyToggleRow>` — same shape, but tied to the signal-sink off-switch wiring

All inherit v1's typography + spacing + radius tokens (`v1/design-system/proposed-tokens.md`).
