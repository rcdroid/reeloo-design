# v2/05 — Test-id design contract

> Resolves theme **H**: UX components must be designed with `data-testid` hooks baked in. Every interactive element ends up with a stable id, named to a convention.
>
> This is a *design* constraint, not a test-engineering retrofit. If a design proposes an "anonymous" icon button with no semantic name, that's a bug — fix the design.

---

## 1. Naming convention

```
data-testid="<surface>:<component>:<element>[:<modifier>]"
```

Each segment is kebab-case, ASCII, ≤32 chars. The full id is ≤120 chars.

Definitions:

- **`<surface>`** — the route / page identity. Top-level routes only; nested routes inherit the parent surface. Examples: `podcast-catalog`, `podcast-player`, `book-reader-page`, `book-reader-deck`, `practice-station`, `me-progress`, `me-map`, `level-mismatch-dialog`.
- **`<component>`** — the component name in PascalCase converted to kebab. Examples: `media-card`, `focus-pin-column`, `avatar`, `flag-issue`, `notification-toast`.
- **`<element>`** — the element within the component. Examples: `play-button`, `speed-slider`, `option-quick-condensed`, `whiteboard`.
- **`<modifier>`** — optional disambiguator when the same element appears multiple times. Examples: row index (`row-3`), slot name (`slot-vocab`), state (`state-loading`).

### 1.1 Examples

| Element | `data-testid` |
|---|---|
| Podcast catalog tile for slug "karotten" | `podcast-catalog:media-card:karotten` |
| The 0.85× speed button in the player bar | `podcast-player:media-player-bar:speed-button:speed-0-85` |
| The "Quick condensed review" CTA in the level-mismatch dialog | `level-mismatch-dialog:level-mismatch:option-quick-condensed` |
| The Pip whiteboard hint inside a drill station | `practice-station:avatar:whiteboard` |
| The bell icon in the shared header | `app-header:notification-bell` |
| The 3rd notification row in the inbox | `me-notifications:notification-row:row-3` |
| The "save word" button in the annotation rail (book) | `book-reader-deck:annotation-rail:save-button:row-2` |
| The Pyodide sandbox "Run" button on card 5 of pytorch tutorial | `learn-anything:sandbox-card:run-button:card-5` |

### 1.2 Rules

- **Stable across runs.** Don't bake in random keys, timestamps, or session ids. The 3rd notification row stays `row-3` regardless of which notification fills it.
- **Stable across themes.** Theme toggles don't change ids.
- **No localisation in ids.** English-only ASCII; the UI label can be `「快速回顧」` but the id stays `option-quick-condensed`.
- **No tag-name overloading.** The `data-testid` describes the *role*, not the HTML element. `<div data-testid="play-button" role="button">` is fine.
- **One id per element.** Don't chain `data-testid` with `data-testid-2`; if you need disambiguation, use `:modifier`.
- **No id on layout-only wrappers.** A flex container that exists for spacing doesn't need an id. Only addressable elements (clickable, focusable, asserted in tests) get ids.

---

## 2. Per-component test-id checklist (extends v1 §5 to 24 components)

> v1 inventoried 18 components in `v1/00-unified-brief.md` §5. v2 adds 6 cross-cutting components. Each entry below lists required ids that every implementation must ship.

### v1 components (18) — required ids

| Component | Required ids |
|---|---|
| `<ReelooLogo>` | `:logo-link` |
| `<MediaCard>` | `:` (the card itself, modifier = slug); `:cover-image`; `:title`; `:lang-chip`; `:cefr-chip`; `:save-button` (if save action present); `:meta` |
| `<MediaLibrary>` | `:language-tab:<lang>`; `:recent-row`; `:filter-pill:<pill>`; `:grid`; `:empty-state` |
| `<RecentRow>` | `:row`; `:item:row-<n>` |
| `<ProgressModal>` | `:modal`; `:status-text`; `:cancel-button` (if cancellable) |
| `<MediaPlayerBar>` | `:play-button`; `:back-15`; `:forward-15`; `:scrub-bar`; `:speed-button:speed-<v>`; `:cc-toggle`; `:replay-button` |
| `<FocusPinColumn>` | `:column`; `:segment:row-<n>`; `:active-line` |
| `<ReadingColumn>` | `:column`; `:paragraph:row-<n>`; `:translation-toggle` |
| `<DeckCard>` | `:card`; `:annotation-rail`; `:audio-bar` (inherits `<MediaPlayerBar>` ids) |
| `<AnnotatedSpan>` | `:span:slot-<word>` (the word is the modifier); `:popover` (when open) |
| `<AnnotationRail>` | `:rail`; `:note:row-<n>`; `:save-button:row-<n>` |
| `<WordKaraoke>` | `:word:slot-<idx>`; `:active-word` (single active highlight) |
| `<LanguageTabs>` | `:tab:<lang>`; `:active-tab` |
| `<StationRunner>` | `:runner`; `:stage`; `:cta-next`; `:cta-skip`; `:progress-strip` |
| `<SpeakingDrill>` | `:drill`; `:tts-button`; `:mic-button`; `:transcript`; `:score-pill` |
| `<Flashcard>` | `:card`; `:front`; `:back`; `:flip-button`; `:rate-again`; `:rate-hard`; `:rate-good`; `:rate-easy` |
| `<DictionaryCard>` | `:card`; `:headword`; `:audio-button`; `:examples`; `:senses` |
| `<ThemeProvider>` | n/a (no interactive surface) |

### v2 new components (6) — required ids

| Component | Required ids |
|---|---|
| `<AppHeader>` | `app-header:logo`; `app-header:nav-link:<app>`; `app-header:language-selector`; `app-header:notification-bell`; `app-header:user-avatar`; `app-header:user-menu` (when open) |
| `<Avatar>` (`v2/02` §5) | `:avatar`; `:whiteboard` (when present); `:info-button` (passive mode); `:info-popover` (when open); `:dismiss` |
| `<NotificationToast>` (`v2/02` §1) | `notification-toast:toast:<id>`; `:cta-primary`; `:cta-secondary`; `:dismiss`; `:progress-bar` (when applicable) |
| `<FlagIssue>` (`v2/02` §3) | `flag-issue:chip`; `flag-issue:modal`; `flag-issue:triage:<kind>` (e.g. `:triage:confusing`); `flag-issue:textarea`; `flag-issue:screenshot-attach`; `flag-issue:send-button` |
| `<LevelMismatchDialog>` (`v2/01` §4) | `level-mismatch-dialog:dialog`; `:option-quick-condensed`; `:option-free-nav`; `:option-stay`; `:dont-ask-again` |
| `<SandboxCard>` (`v2/04` §4) | `sandbox-card:card:card-<n>`; `:editor`; `:run-button`; `:reset-button`; `:hint-button`; `:output`; `:completed-marker` (visible only when expectedOutput matched) |

Plus three personal-surface components from `v2/03`:

| Component | Required ids |
|---|---|
| `<MeLayout>` | `me-layout:nav-tab:<tab>`; `me-layout:content` |
| `<SkillRadar>` (`v2/03` §4) | `me-map:skill-radar:chart`; `:axis:<skill>` |
| `<CEFRGrid>` (`v2/03` §4) | `me-map:cefr-grid:cell:<skill>-<cefr>` |

---

## 3. How this constrains design

### 3.1 No anonymous icons

Every clickable icon must have a name in the component spec. If a designer ships an "icon row" of 6 mystery icons, the test-id rule forces them to name each one. If they can't articulate the name, the icon shouldn't exist — it's a usability tell.

### 3.2 Modifier discipline forces decisions

The "is this row identified by index or by slug?" question (modifier choice) makes the designer commit:
- If a notification row's identity is "the 3rd one shown right now" → `row-3` (index).
- If a tile's identity is "the karotten podcast" → `karotten` (slug).
- You can't have both. Picking one forces the team to agree on what the addressable identity is.

### 3.3 Component shapes pop into focus

When a designer documents the test-ids for a component, the component's affordance count becomes obvious. A `<MediaPlayerBar>` with 7 required ids is a 7-affordance bar. If the design has 9 affordances and only 7 ids, the design has either over-shot or the spec is under-defined. Either way, conversation.

### 3.4 No string-based icon switching

Anti-pattern: `<button data-testid={`btn-${kind}`}>` where `kind` is content-driven. This bypasses the contract. Use `:slot-<kind>` instead (`<button data-testid={`media-card:save-button:slot-${kind}`}>`) so the surface + component is fixed.

### 3.5 Mandatory in Figma

Every component in the Figma library (v1 §8 deliverable 2) has a "Test ID" field on its properties panel. The field is autopopulated when known, editable when context-dependent. The Code Connect map (v1 §8 deliverable 4) reads the field to produce the React `data-testid` prop.

---

## 4. Enforcement

### 4.1 Lint rule

A custom ESLint rule (`@reeloo/eslint-plugin/require-testid`) flags any JSX element matching `[role=button], button, [onClick], input, select, textarea, [tabIndex]` without a `data-testid` attribute (or a parent prop that forwards one). It runs in CI on every PR.

### 4.2 Smoke contract

Each app's smoke test (existing `inspect` skill, `data/verify-routes.json`) asserts that the top-12 ids per surface render. A missing id fails the smoke run.

### 4.3 Documentation

The component-gallery service (`http://localhost:18106`, per the user's "Inventory First" memory note) gets a new column: "Test IDs" listing the contract ids. A component shipped without the column populated is rejected at PR review.

---

## 5. Test-id evolution

When a component grows a new affordance:

1. The new affordance gets a draft id in the component spec.
2. The spec PR is reviewed; id is approved.
3. Implementation lands.
4. Any test relying on prior ids continues to pass — ids are additive, never renamed.

**Renaming an id is a breaking change** — must be coordinated with the test suite and the design-system version bump. Strongly prefer to deprecate-and-add (keep the old id around for one release).

---

## 6. Open questions for design

1. Should the per-app surface name (`podcast-catalog`, `book-reader-page`) be derived from the route or hand-named? My vote: hand-named in the surface manifest, because route trees change but surface identity rarely does.
2. For dynamically generated lists (notification rows, recent items), modifier = index. Is that resilient enough, or do we need a stable per-item id (notification-id) as the modifier? My vote: index for assertions about "the N-th", id for assertions about "this specific item".
3. Spaces of components that aren't in v2's 24 — when the readalong studio's 134-component sprawl (`v1/products/readalong.md` §5 Flow F) lands in `studio.reeloo.ai`, do we extend the contract to them? My vote: yes, but with relaxed enforcement (warnings, not failures) until that surface stabilises.
