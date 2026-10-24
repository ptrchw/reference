# NumLah → React: Architecture Decisions

Reference doc for Claude Code, not a spec — the two prototypes (`numlah-chitchat-vanilla.html`, `numlah-tehtarik-prototype.html`) and their style guides are the source of truth for visuals and copy. This covers the decisions needed *before* scaffolding: what's shared vs. themed, how theming actually works given the two prototypes are structurally different (not just re-skinned), the Lucide swap, state shape, and screen flow.

**v0.1 MVP scope (Oct 2026): Chit Chat only.** Teh Tarik, the theme switch and birth-year onboarding (§2) move to the backlog. Keep the `/core` vs `/web` split and token-file structure so Teh Tarik can be added later without restructuring, but don't build the `tehtarik` tree, the theme toggle or onboarding in v0.1. v0.1 adds the AI features and guardrails in §6.

**Single app, single service.** This is one codebase and one deployable with a birth-year-driven onboarding step that auto-assigns a theme (user can override later in Preferences) — not two forked products. Everything in §1's "Shared Core" column, plus the state/reducer layer in §4, is the actual application; the two component trees in §2 are a presentation layer on top of it, not separate apps.

## 0. Platform Sequencing: Web Now, Native Later

Building in React (web) first — faster iteration at this stage, no store submission overhead. React Native (Expo) stays the plan for later, per D2's own prototype comments, so the structure needs to keep that port *cheap*, not deferred-and-forgotten. Everything below is already implied by the decisions in §2/§3/§5 — this just makes the resulting boundary explicit for scaffolding:

- **`/core` vs `/web` split from day one**, even with only one platform right now. `/core` = reducer/store, analysis engine, i18n, data model, theme-resolution logic — plain TypeScript, zero DOM/browser APIs. `/web` = the two component trees, tokens, anything DOM-specific. When native happens, `/native` sits alongside `/web` and reuses `/core` untouched.
- **Storage behind one interface, not scattered `localStorage` calls** — same instinct D1's `window.storage` polyfill already uses. `core/storage.ts` exports `get`/`set`; the web implementation wraps `localStorage` today, a later RN implementation wraps `AsyncStorage`. One file changes, not every component that reads state.
- **Tokens as plain JS objects, not CSS files** (§2's `tokens/chitchat.ts`/`tokens/tehtarik.ts`) — objects can back `className`/CSS variables on web today and `StyleSheet.create()` on native later; CSS files can't port at all.
- **No URL router** (§5) — the screen-stack-in-the-reducer pattern has no web-only routing dependency to strip out later.
- **Icon names stay identical** (§3) — `lucide-react` now, `lucide-react-native` later, same component names, different package, near-zero-diff swap.

What *not* to set up yet: no Expo/RN scaffolding, no cross-platform UI kit (Tamagui, React Native Web, etc.) until the native port is actually scheduled — deciding that now is guessing with worse information than you'll have later. TypeScript now, though — typing `Entry`/`CheckSession`/`AppState` once (§4) pays for itself directly at the port.

## 1. Component Inventory

| Component | Shared Core | D1 (Chit Chat) only | D2 (Teh Tarik) only |
|---|---|---|---|
| Analysis engine (`fourDAnalysis`, `totoAnalysis`, insight generators, seeded-weight RNG) | ✅ identical logic in both files today | | |
| i18n (`STRINGS`/`APP_STRINGS`, `t()`, EN/ZH) | ✅ same pattern, different copy per theme | | |
| Icon set | ✅ once migrated (see §3) | | |
| Entry input (4D keypad, TOTO grid) | ✅ same behaviour, different chrome (D1: numeric-keypad panel; D2: digit-box overlay + grid overlay) | | |
| Setup / Ledger (home screen) | ✅ same job, different layout (ticket-stub cards vs. ledger rows) | | |
| Check Results screen | ✅ | | |
| Menu drawer + Preferences + Notifications + About subscreens | ✅ | | |
| Shake Me Lah (random number ribbon) | ✅ | | |
| Onboarding / birth-year capture | ✅ new — not in either prototype, needs its own design (see §2) | | |
| History screen | | ✅ | — no dedicated screen; D2 has a vestigial, uncalled `openHistoryEntry()` stub worth deleting rather than porting |
| Favourites screen | | ✅ (derived view over History — see §4) | — D2 has inline heart/like toggles but no favourites list |
| Reality Check screen | | ✅ | — not in D2 at all |
| Snackbar / undo-delete | | | ✅ (D1 has no undo pattern) |

Net: the *engine* (analysis, i18n, entry/check data model) is already near-identical and should port to shared core almost verbatim. The *screens* are where the two diverge — three screens exist only in Chit Chat.

## 2. Theme Assignment & Mechanism

**Onboarding flow (new — not in either prototype):** Splash → birth-year capture → theme auto-assigned from it → mount that theme's tree → Setup/Ledger. The birth-year screen itself has no design in either file and shouldn't borrow either theme's chrome since theme isn't known yet when it renders — use the shared brand shell only (logo mark + "NumLah" wordmark + tagline are already visually identical between the two splash screens, so that neutral shell already exists; the birth-year input itself needs a plain, theme-agnostic treatment).

Two things this doc can't settle and should be flagged back to you rather than guessed at:
- **The actual age cutoff.** Directionally, Teh Tarik's kopitiam/"routine player" persona and larger Atkinson-Hyperlegible-driven type scale reads older; Chit Chat's "casual player" persona reads younger — but the exact birth-year threshold is a product call, not a design inference.
- **What happens right at the boundary / on ambiguous input** (e.g. user skips the field) — needs a default.

**Manual override still exists:** both prototypes already have a Theme rocker in Preferences that's currently a visual-only stub — this is where it gets wired for real. Track *why* the current theme is set so a later onboarding-logic change doesn't silently overwrite someone's manual choice:
```
theme: 'cc' | 'tt'
themeSource: 'auto' | 'manual'
birthYear: number | null
```

**Why this still isn't a single shared component tree:** D1's ticket-stub/perforated-card language and D2's ledger-row/hand-ruled language remain structurally different layouts, not a token reskin of one layout (different fonts too: D1 body is Inter, D2 body is Atkinson Hyperlegible; both share Zilla Slab display + IBM Plex Mono). A component branching on `if (theme === 'tt')` for every card would get unreadable fast. The "single app, single service" goal is better met by pushing everything *except* visual layout into one shared layer than by forcing the two visual languages into one component:

- **Core service (shared, single source of truth):** the reducer/store (§4), analysis engine, i18n, entry/check/history data model, notification logic, preferences persistence, theme-resolution logic (birth year → theme, manual override handling) — none of this differs by theme today and none of it should.
- **Presentation only (forked by necessity):** `components/chitchat/*` and `components/tehtarik/*` — the two component trees, each consuming the same hooks (`useEntries`, `useCheck`, `useHistory`, `useFavourites`, `usePreferences`, `useTheme`) and each reading its own token file (`tokens/chitchat.ts` / `tokens/tehtarik.ts` — colour, radius, shadow, font-stack per style guide §01–02 of each). `ThemeProvider` reads the resolved `theme` once at the app shell and mounts the matching tree; theme doesn't get threaded through every component's props.
- **Feature gating:** History / Favourites / Reality Check components live only in the `chitchat` tree and simply aren't routed to from `tehtarik`. The underlying data (history log, starred flags) stays in the shared store regardless of theme, so switching theme via Preferences doesn't lose anything — a user who switches from Teh Tarik to Chit Chat immediately sees their existing check history reflected in the newly-available History/Favourites screens.

## 3. Icon Migration

D1 already hand-rolls dependency-free SVGs that reproduce `lucide-react` paths exactly (`Ico()`/`icoHTML()`/`ICON_PATHS`, 17 icons in active use — `Sparkles` is defined but never called, drop it). Migrating D1 is mechanical: delete the hand-rolled helpers, `import` the real `lucide-react` components, keep the same names.

D2 uses the Material Icons webfont with a ~130-line font-load-detection + inline-SVG-fallback system (`buildSVG`/`swap`/`fontActive` in the second `<script>` block). That entire system should be deleted, not ported — every `<span class="material-icons(-outlined)">name</span>` becomes a Lucide component call.

| Material name (D2) | → Lucide | Notes |
|---|---|---|
| `arrow_back_ios_new` | `ChevronLeft` | Unifies with D1's back icon; confirm the slightly different visual weight is acceptable |
| `home` | `Home` | |
| `close` | `X` | |
| `menu` | `Menu` | |
| `chevron_right` | `ChevronRight` | |
| `favorite` / `favorite_border` | `Heart` | Toggle via `fill` prop, same pattern D1 already uses for `Star`. Note: D2's SVG fallback map is missing `favorite_border` entirely today (bug) — Lucide fixes this for free |
| `delete` | `Trash2` | Matches D1 |
| `backspace` | `Delete` | Verify glyph against Lucide's rendering before committing |
| `expand_more` / `expand_less` | `ChevronDown` / `ChevronUp` | |
| `add` | `Plus` | Matches D1 |
| `tune` | `SlidersHorizontal` | Matches D1 |
| `notifications` | `Bell` | Matches D1 |
| `info` | `Info` | Matches D1 |
| `schedule` | `Clock` | Matches D1 |
| `search`, `waving_hand` | — | Defined in D2's fallback map but **not actually used anywhere in markup** — confirm they're dead before deciding whether to port (`waving_hand` has no clean Lucide equivalent if it does turn out to be needed) |

Once both themes render from `lucide-react`, icon sizing/stroke-width should be centralized (D1 defaults to `strokeWidth={2}`; D2's fallback SVGs used `1.6` for the heart specifically — worth normalizing to `2` for consistency unless that was a deliberate aesthetic call).

## 4. State Shape & Data Flow

D1's `state` object is the more complete model (D2 uses scattered globals + direct DOM writes) and should be the basis for the shared store. Recommend a single reducer (Context + `useReducer`, or Zustand) rather than D1's imperative `render()` dispatch or D2's `getElementById` mutations:

```
AppState {
  theme: 'cc' | 'tt'
  themeSource: 'auto' | 'manual'
  birthYear: number | null
  onboarded: boolean                                // gates whether onboarding is shown on launch
  language: 'en' | 'zh'
  game: '4d' | 'toto'
  entries: { '4d': Entry[], 'toto': Entry[] }      // in-progress, unsaved check list
  checkResult: CheckSession | null
  history: CheckSession[]                           // theme-agnostic storage; only 'cc' UI surfaces it
  notifications: Notification[]
  preferences: { sound: boolean, haptic: boolean, weeklyDigest: boolean /*default false, §6*/ }
  drawWatcher: { lastRunAt: string | null }         // §6 — tracks which draws the digest has covered
  screen, menuScreen, screenOrigin                  // see §5
}
Entry        = { id, value?: string /*4D*/, numbers?: number[] /*TOTO*/, starred: boolean }
CheckSession = { id, game, timestamp, entries: Entry[] }
```

Note a real divergence to resolve, not just port: D1 stores entries as structured data (`value: '4821'`, `numbers: [3,12,19,27]`); D2 stores a pre-formatted display string (`'48 21'`, `'03 12 19 + 27'`) and infers 4D-vs-TOTO by checking `value.length > 11`. Normalize to D1's structured shape and derive display formatting per-theme at render time — don't carry D2's string-based model forward.

**Favourites is a derived selector, not stored state** — D1's `getStarredNumbers()` walks `history`, collects starred entries, dedupes by `game+value` keeping the most recent. Port this as a memoized selector, not a separate favourites array.

**Persistence:** only `game` selection is currently persisted in the prototypes (via a `window.storage` polyfill in D1, itself just a wrapper the artifact environment provides) — but `theme`/`themeSource`/`birthYear`/`onboarded` are no longer optional to persist now that onboarding depends on them: without it, every app open re-triggers the birth-year prompt. Route all of it through the `core/storage.ts` interface from §0 (web-backed today, swappable later) rather than calling `localStorage` directly from components. `language`, `history`, and `favourited` flags are still **not** persisted in either prototype and remain an open product decision — likely yes for history/favourites given they're the point of the feature, but design that now rather than retrofit it.

## 5. Routing

Neither prototype uses URL routing — both are single-page screen-state-machines (D1: `state.screen` ∈ `setup | checkResult | realityCheck`, `state.menuScreen` ∈ `null | preferences | notifications | history | favourites | info`; D2: overlay `.open` classes on `ledger | entry | toto | results` + menu drawer with 3 subscreens). Keep this pattern — a simple screen-stack in the reducer, no router library — rather than introducing one now: a mobile utility app with no deep-linking need doesn't benefit from route trees, both prototypes already encode the exact transition logic to replicate, and per §0 this is also what keeps the eventual React Navigation swap for native a clean drop-in rather than a rip-and-replace.

Keep D1's `screenOrigin` field — it's what makes "back" from a Reality Check opened via History return to History rather than Setup, and is the one piece of navigation-stack logic worth preserving exactly as-is.

**Full screen set:**
- First launch only *(backlog, not v0.1)*: Splash → Birth Year (shared, theme-neutral chrome) → theme resolves → Setup/Ledger
- Shared: Setup/Ledger → Entry (4D/TOTO) → Check Results → Menu (Preferences, Notifications, About)
- Chit Chat only: → History, → Favourites, → Reality Check (reachable from Setup, History, or Favourites)

## 6. AI Features & Guardrails (v0.1)

**Positioning:** NumLah is a results checker and record-keeper. AI makes recording and reading results easier. It never suggests, ranks or "improves" numbers, and nothing it says implies a past result predicts a future draw. The AI layer is model-agnostic: one `core/ai` interface with provider adapters behind a server-side route, so no vendor is hard-wired and no API key reaches the browser.

**In scope for v0.1:**

| Feature | Type | What it does |
|---|---|---|
| Ticket scan (OCR) | AI-powered | Photo of a 4D/TOTO ticket → vision model extracts numbers → user confirms or edits before anything is added. Images are processed and discarded, never stored. |
| AI insight copy | AI-powered | LLM rewrites the stats the engine already computes into fresh Chit Chat-voice copy (EN/ZH). Falls back to the existing static templates on any failure. |
| Voice readout | Accessibility | Reads results, insights and the weekly digest aloud via the browser's speech API. |
| Draw-watcher weekly digest | Agentic | Opt-in weekly loop: finds the week's draws → checks the user's favourites against them → decides what to report → writes the digest → validates it → delivers it to Notifications. |

**Weekly digest:** off by default, opt-in toggle in Settings (Preferences). Weekly only, never per draw. On the web MVP it lands in the in-app Notifications inbox (no web push). If the user has no favourites, no digest is produced. Draw results are simulated in v0.1 (no real data source yet; scraping Singapore Pools is ruled out by their terms).

**Guardrails (apply to every AI output, not just the digest):**

1. **No number suggestion, ever.** No "AI picks," recommended, lucky, hot/cold or "smarter" numbers. Shake Me Lah stays plain unweighted random and must never be AI-driven.
2. **Outcomes only in the digest.** Each favourite: hit or no hit, and the prize tier if hit. No statistics, gaps or frequencies in the digest.
3. **No near-miss framing.** Never "so close," "almost," "3 of 4 digits." A non-hit is reported plainly.
4. **No forward-looking language.** Never "due," "overdue," "streak," "next time," "likely," "chance," "about to." Past stats are described as past only.
5. **No promotion.** No next-draw dates, jackpot amounts, "play now" prompts or anything that encourages buying a ticket, in the digest or in AI insight copy.
6. **Quiet by default.** The digest is opt-in, weekly, and easy to switch off from the digest itself.
7. **Restraint built into the agent.** It reduces output rather than adding to it: one digest per week maximum, no catch-up bursts (a missed week becomes one digest, not several), no re-sending.
8. **Grounded generation.** The LLM only receives structured facts the engine computed and may only restate them. Every number in its output must appear in the facts.
9. **Validate, then fall back.** Each AI output is checked against a banned-phrase list (EN and ZH) and the facts. If it fails, use the deterministic template instead. The app works fully with AI unavailable.
10. **Responsible-play line.** The digest and Settings carry a short, non-judgemental link to NCPG support resources (confirm current contact details before any public release).

**Backlog (not v0.1):** Teh Tarik theme and theme selection; birth-year onboarding; "Ask NumLah" questions over saved history; responsible-gambling pattern detection (needs clinical/NCPG input); per-draw notifications; real results feed (licensed source, not scraping); native app. Singapore's gambling-promotion rules need a proper check before any public release.
