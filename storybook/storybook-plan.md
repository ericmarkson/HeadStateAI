# HeadState — Storybook Component System: Build Plan

**Status:** Step 1 (style guide) complete. Step 2 onward (the actual Storybook/React build) not started — no application source code exists anywhere in the repo yet.

## Why this exists

`../design-system.md` documents ~30 named components across three visual "worlds" (chrome/device/doc) and two breakpoints (mobile/desktop), extracted from the 10 self-contained static HTML mockups in `../mockups/`. That document always deferred real component extraction to "the phase after full mockup coverage... gated on build step 0 existing." Full mockup coverage — including the desktop-responsive retrofit pass, §7a — finished in this project's most recent design session. This is that next phase.

Everything for this phase lives inside this `storybook/` folder — deliberately separate from the repo root, which is otherwise pure design/spec documents (`../CLAUDE.md`, `../tech-spec.md`, `../frd.md`, `../decision-log.md`, `../design-system.md`) plus the mockups. This isn't yet the binding monorepo `../CLAUDE.md` describes for build step 0 (that requires `apps/app` at the repo root, alongside `apps/api`, `packages/derivation`, `packages/clinical-config`, `ios/`, `android/`) — it's a self-contained npm project that gets *promoted* into that layout once step 0 actually happens, rather than prematurely scaffolding root-level directories a design-only repo doesn't need yet.

Building Storybook this way, rather than jumping straight from mockups to React, exists to solve one specific risk: extracting ~30 components ad hoc from 10 large, screen-specific HTML files (each with reviewer-only demo cruft, embedded vanilla JS, and base64-embedded fonts) invites exactly the kind of cross-screen drift the mockup phase already caught real bugs from — `BatchBar`'s `status_migrainosus` eligibility, `ConnectionCard`'s fail-open/fail-closed divergence, both real, both QA-caught, both a consequence of "shared" logic that wasn't actually being verified as shared. A single canonical style guide, checked component-by-component against the spec once, removes that risk before any React code gets written.

## Scope boundary

**Building:** `apps/app` (React + Vite + TypeScript + Storybook), a minimal `packages/schema` (types only, no runtime), root workspace plumbing.

**Not building:** Capacitor/`ios/`/`android/`, CI beyond Storybook's own test run, Postgres, `apps/api`, the MCP server, real data or routing, or `ConflictNotice` (marked "an open proposal, not settled spec" in `../design-system.md` — do not implement until that changes).

`../CLAUDE.md`'s binding repository layout locks a full monorepo shape as part of build step 0 (Capacitor, CI, a Postgres DDL gate, the API/MCP server — see `../frd.md` FR-0.1–0.7). None of that is needed to build Storybook; only `apps/app` needs to exist. This plan bootstraps just enough skeleton to host `apps/app`, contained inside this `storybook/` folder for now, so it can be lifted into the repo root wholesale once step 0's other pieces (Capacitor, CI, Postgres) actually get built, rather than restructuring it then or prematurely creating root-level `apps/`/`packages/` directories now.

## Step 1 — The style guide (done)

`styleguide.html`, in this same `storybook/` folder, is the canonical, component-by-component design reference this whole build is verified against. It did not need to be built from scratch: `../mockups/headstate-styleguide.html` was already a living gallery — token sections (Color, Type, Radius) plus a `.gallery` of one `.cell` per named component — kept in sync through most of the project's life, already covering the §7a desktop-era components (`TabBar`, `SideNav`, `SectionSubnav`, `HomeActionCard`) and the post-contrast-fix token values.

What Step 1 actually did:
1. `git mv mockups/headstate-styleguide.html styleguide.html` (initially to the repo root, then relocated again into `storybook/` alongside this plan once that folder was created) — moved, not copied, so there's exactly one source of truth.
2. Added the three components that were genuinely missing when cross-checked against `../design-system.md` §4's full table: `DayCountTable` (modeled on `../mockups/report.html`'s real `.doc-daycount-table` markup), `DeviceClose`, and `DocOpen` (the latter two built this session, in §7a tasks 4–5, after the gallery was last touched).
3. Ran an independent verification pass (a fresh-eyes subagent diff against `../design-system.md`'s token tables, the §4 component table, the closed confidence-tag vocabulary, and the scoped `:focus-visible` requirement) to catch any other drift before treating the file as ground truth.
4. Updated cross-references: `../design-system.md` §5 now describes `styleguide.html`'s canonical-reference status instead of listing it as one of the mockup files; `../CLAUDE.md`'s File index carries entries for `storybook/`, this document, and the style guide.

Every batch in Step 2+ below is verified against `styleguide.html` directly — not against a scattered list of mockup files — because that consolidation is the entire reason to build it first.

## Step 2 — Minimal monorepo skeleton

```
HeadStateAI/                     # repo root — design/spec docs only, untouched by this build
├── CLAUDE.md  tech-spec.md  frd.md  decision-log.md  design-system.md
├── mockups/                      # unchanged
└── storybook/                    # this whole build lives here, self-contained, until step 0 promotes it
    ├── storybook-plan.md         # this file
    ├── styleguide.html
    ├── package.json              # private, workspaces: ["apps/*","packages/*"], npm — not pnpm/turborepo, nothing here needs a task graph yet, and FR-0.1 already names npm
    ├── tsconfig.base.json
    ├── .gitignore                # node_modules/, dist/, storybook-static/, .vitest-cache/
    ├── apps/app/
    │   ├── package.json
    │   ├── vite.config.ts
    │   ├── .storybook/{main.ts, preview.ts, vitest.setup.ts}
    │   ├── public/fonts/
    │   └── src/
    │       ├── styles/{tokens.css, fonts.css}
    │       ├── lib/worldDecorator.tsx
    │       └── components/<Name>/{Component.tsx, .module.css, .stories.tsx}
    └── packages/schema/           # types only, see Step 7
```

`ios/`, `android/`, `apps/api/`, `packages/derivation/`, `packages/clinical-config/` are **not created** anywhere — an empty placeholder would invite the wrong assumption that step 0 partially happened. When step 0 actually happens, `storybook/apps/app` and `storybook/packages/schema` move to the repo root wholesale (`apps/app`, `packages/schema`) alongside the new `apps/api`/`packages/derivation`/`packages/clinical-config`/`ios`/`android` — a `git mv`, not a rebuild.

Storybook lives **inside** `apps/app` (`apps/app/.storybook/`, stories colocated under `src/components/**/*.stories.tsx`), not as a separate `apps/storybook` package. A separate package is the right call for an independently-versioned/published design system; this one's whole point is "components that literally ship in the app," which colocation guarantees and a separate package doesn't.

**A practical note for whoever runs this on Windows:** if the repo lives inside a live-synced cloud-storage folder (Google Drive, OneDrive, Dropbox), `npm install` can fail reproducibly with `EPERM`/`ENOTEMPTY`/`EBADF` errors — the sync client locks files mid-write while npm extracts hundreds of small package files into `node_modules`. Pause sync (or otherwise exclude `storybook/**/node_modules` from it) before running installs in this folder.

## Step 3 — Storybook install and configuration

Run `npm create storybook@latest` from `apps/app` — it auto-detects Vite and installs `@storybook/react-vite` (Storybook 10.x, CSF3 scaffolding: `satisfies Meta<typeof X>` + `StoryObj<typeof meta>`). Add:

- `@storybook/addon-docs` — autodocs (a separate install again in v10, no longer bundled).
- `@storybook/addon-a11y` — axe-core-backed accessibility checks per story.
- `@storybook/addon-vitest` (+ `vitest`, `@vitest/browser`, `playwright`) — converts stories into real Vitest tests, the current consolidated path for both play-function interaction tests and a11y-in-CI.
- `vite-tsconfig-paths` — wired into both `apps/app/vite.config.ts` and Storybook's `viteFinal`, so path aliases resolve identically in both.

**Do not install** `@storybook/addon-essentials`, `-interactions`, `-links`, or `@storybook/blocks` — dead/deprecated packages as of Storybook 10; controls, actions, viewport, backgrounds, and toolbars are core now. Play functions import `@storybook/test` directly (`within`, `userEvent`, `expect`, `fn`), no separate addon.

### `.storybook/preview.ts` responsibilities
1. Import `../src/styles/fonts.css` then `tokens.css` once, globally.
2. Define exactly two named viewports via `parameters.viewport.options`: `mobile` (390×844) and `desktop` (1280×800) — matching the project's real two-breakpoint strategy (D22), not Storybook's default device presets.
3. Define `globalTypes.colorScheme` (light/dark/system) — the **only** legitimate toolbar toggle, since chrome is the only world that's actually theme-reactive.
4. Register the world decorator (below) as the sole `decorators` entry.
5. Set `parameters.a11y.test = 'error'` globally, so any story failing an axe check fails the Vitest run, not just shows a warning in the Storybook UI.

### The world × breakpoint decorator (`src/lib/worldDecorator.tsx`)

This is the one genuinely novel piece of config this project needs, because `../design-system.md` §1's three-world model doesn't map onto Storybook's built-in theme-toggle idiom.

- **`world` is a story-meta `parameter`** (`parameters: { world: 'device' }`), never a `globalTypes` toolbar control. A toolbar toggle would let a reviewer render `SideNav` inside `.doc-sheet`, which cannot happen in the real product — modeling world as a parameter set once per component (twice, for the handful that genuinely appear in two worlds, like `Badge`) rules that out structurally rather than by convention.
- The decorator wraps by world: `device` → `.device-screen` (inside a phone bezel when `parameters.deviceFrame` calls for it, see below); `doc` → `.doc-sheet`; `chrome` → a bare wrapper reading `context.globals.colorScheme` and setting `data-theme`.
- **`colorScheme` is structurally inert for device/doc** — those branches never read it at all, not "read and ignore under a condition" (which could regress silently). Device/doc tokens in `tokens.css` are hardcoded literals scoped to their container class, never `var()` re-reads of chrome tokens, and never sit inside a `prefers-color-scheme` query — mirroring the real bug `../design-system.md` §2 documents (`--focus`, a chrome token, once leaked into the always-dark device world and rendered a light-mode-blue focus ring at 2.77:1 contrast, below the WCAG 3:1 floor).
- Bezel-vs-no-bezel at desktop is per-component, via `parameters.deviceFrame: 'tier1' | 'bounded'` — Tier-1 screens drop the bezel and go full-chrome at desktop (real `SideNav`); bounded flows (Onboarding, mid-attack capture, morning checkpoint) keep the bezel and render `DeviceClose` instead, per D22.
- A component that only makes sense at one breakpoint (`SideNav`, `SectionSubnav`) sets `globals: { viewport: { value: 'desktop' } }` at its own meta — this locks (doesn't hide) the toolbar viewport control for that story.
- `.device-screen`, `.doc-sheet`, and `.sidenav` in `tokens.css` all carry their scoped `:focus-visible` overrides verbatim from `../design-system.md` §2 / `styleguide.html`.

## Step 4 — Token and font migration

1. Copy `styleguide.html`'s `:root` block verbatim into `apps/app/src/styles/tokens.css` — same names, same values, same three-state chrome pattern, same fixed device/doc blocks, same two radius scales (`--radius-doc` vs `--radius-card`/`--radius-pill`). No renaming — every project doc references these exact names.
2. Fonts: `../mockups/fonts/*.woff2` are already decoded, real binary files sitting next to the `.b64.txt` sources — copy them into `apps/app/public/fonts/`, no re-decoding needed.
3. Before hand-rolling `@font-face`, check whether `@fontsource-variable/fraunces`, `@fontsource-variable/ibm-plex-sans`, and `@fontsource/ibm-plex-mono` are published with the exact weight/style axes this project pins (Fraunces 400/600 + italic 400; Plex Sans 400/500/600; Plex Mono 400/500 static). Prefer them if so (versioned, lockfile-pinned, no CDN dependency); fall back to manual `public/fonts` + hand-written `@font-face` (referencing files via Vite `?url` imports so hashed filenames track the build) otherwise.
4. `font-display: swap` everywhere; preload only the 1–2 most above-the-fold weights. Never base64-inline — the CSP constraint that motivated the mockups' data-URI approach doesn't apply inside a normal Vite app.
5. Carry `font-variant-numeric: tabular-nums` into each component's own CSS Module on the specific selectors that had it (timestamps, doses, day counts), not as one blanket global rule.

CSS approach overall: **one global `tokens.css`**, then **CSS Modules per component** referencing `var(--token-name)`. No Tailwind, no vanilla-extract — the existing plain-CSS-custom-property system already works, and both alternatives would mean re-deriving values into new syntax for no functional gain.

## Step 5 — Component build order

Decompose **component-by-component off `styleguide.html`** (= `../design-system.md` §4's inventory), not screen-by-screen. Build leaf-first, verify each component against its `styleguide.html` cell before moving on.

| Batch | Components |
|---|---|
| A — Primitives | `Badge`, `StatusPill`, `FlagChip`, `SegmentedControl` (chrome-world per its reclassification), `UnconfirmedMark` |
| B — Field blocks | `CorrectableField` (3 typed variants — chip-picker, stepper, datetime — one component, not three, typed per `../tech-spec.md` §3.7), `HistoryMarker` |
| C — Card composites | `Card`/`Receipt` (7-value closed confidence-tag vocabulary as a discriminated-union prop, never a free string), `MomentCard`, `PassiveStrip`, `ContextCard` (4 required states), `OpenEpisodeCard`, `BackfillConfirmation` |
| D — Nav shell | `TabBar`/`SideNav` (one `NAV_ITEMS` source, one render path — the React equivalent of the mockups' single `renderNav(container)` called twice), `HomeActionCard`, `SectionSubnav`, `DeviceClose`, `DocOpen` |
| E — Timeline | `DayRail`, `TimelineList`, `BoundaryPicker`, `BoundaryAdjuster` |
| F — Review Queue | `ReviewQueueItem`, `BatchBar` — re-verify the `status_migrainosus` batch-ineligibility rule as an actual test, not a manual check, since it was a real blocking bug once already |
| G — Settings | `ConsentScope`, `ConsentReceipt`, `ConnectionCard`, `ExportLinkRow` — build/verify `ConsentScope`/`ConsentReceipt` against Onboarding's usage first (upstream per `../design-system.md` §7's own ordering), then confirm Settings' additional `research_aggregate` scope and live-state requirements are satisfied by the same component, not a fork |
| H — Report | `MedTable`, `CriterionPicker`, `CriterionChecklist`, `GenerationRecord`, `DayCountTable` — one `CriterionChecklist` story per payer criterion (args), not three components |
| I — Onboarding / bounded flows | `BinaryChoice`, `StepRail` |
| J — Insights | `ScopeNote`, `CorrelationCard` — the "never displays a raw count against its minimum-sample threshold" rule enforced at the type level (the below-threshold variant has no numerator prop slot), not just documented |

Sequencing: A → B → C → D, then E–J roughly in the order above (matches `../design-system.md` §7's own dependency ordering) and are otherwise parallelizable once D lands, except F depends on E.

`ConflictNotice` is explicitly **not built** — open proposal, not settled spec.

## Step 6 — Page-level composition

**Thin composition stories, not full "page" entries.** Storybook's unit is the component; a real page needs routing and data-fetching this project deliberately doesn't build yet, and building either to satisfy a "Home page story" would silently smuggle in step-0/step-1 scope this plan explicitly excludes.

Concretely: one composition story per screen, under `src/components/_pages/<Screen>Composition.stories.tsx` (the `_pages` prefix keeps these visually separated in the Storybook sidebar from real reusable components), assembling real components with mock data matching `styleguide.html`'s own fixture values — proving the components fit together at both viewports, not a routed/clickable page. Turn `a11y.test` on for compositions only after their constituent leaf components already pass individually (a composed page surfaces every child's issue at once, which is noisy before each child is clean). No `react-router`/`TanStack Router`.

## Step 7 — Testing strategy

- **a11y in CI**: `@storybook/addon-vitest` runs stories as Vitest tests (`vitest --project=storybook`); `parameters.a11y.test: 'error'` fails the run on any axe violation.
- **Play-function tests** (`@storybook/test`) targeted at the state-machine-bearing components:
  - `BoundaryPicker`/`BoundaryAdjuster` — picking `relapse_collapsed` vs `relapse_split` updates display; the plausible-length guard rejects an implausible manual edit without a hard clinical threshold (per `../CLAUDE.md`'s npj-spec plausibility-guard-only requirement).
  - `ReviewQueueItem`/`BatchBar` — a `status_migrainosus` item's checkbox is disabled and excluded from batch count; a `timed_out` item stays confirmable and labelled, not disabled (FR-3.28).
  - `ConsentScope` — neither Allow nor Decline starts focused or pre-selected (FR-7.3); declining never applies `--flag` styling.
  - **Focus-visible-per-world regression test** — the exact bug `../design-system.md` §2 documents happening once. Per world container (`.device-screen`, `.doc-sheet`, `.sidenav`): drive focus via `userEvent.tab()` (not `.focus()` directly — the focus-visible heuristic is keyed to input modality), assert `getComputedStyle(el).outlineColor` matches that world's pinned token, run with `colorScheme` toggled both ways and assert the outline color does **not** change. Reliable on Chromium; best-effort elsewhere.
- **Visual regression, no-cost path**: `@storybook/addon-vitest`'s browser mode + Playwright's `toHaveScreenshot()` against committed baseline PNGs under `src/components/**/__screenshots__/`. Zero recurring cost, no SaaS dependency. Chromatic noted as a later option if hosted review links become worth paying for — not needed now.

## Step 8 — `packages/schema` scope

Create it now, but **types only** — no Zod, no runtime validation, no database awareness. Transcribe `../tech-spec.md` §3.6–3.7's observation body shapes and `../design-system.md`'s closed confidence-tag vocabulary (`export type ConfidenceTag = 'lexical' | 'paraphrase' | 'user' | 'no source' | 'resolved' | 'system' | 'reduced'`, etc.) into `packages/schema/src/index.ts` as plain types. Cheap — a few hundred lines, no new tooling decision — and it buys real value: every component's props type-check against the same shapes the eventual real backend will produce, so `apps/app` doesn't need re-typing when step 1 (the real one — capture and the observation store) builds the observation store for real. If this drifts from what that step actually needs, the fallback is cheap: fold the types back into local `apps/app` types and let step 0 build the real package from scratch.

## Milestones

1. **Skeleton boots** — workspaces + Storybook scaffolded, world decorator wired, one throwaway story per world proving the three frames and two viewports render distinctly. *Done when:* `npm run storybook` shows three correctly framed placeholder stories; `npm run build-storybook` succeeds.
2. **Tokens/fonts land** — `tokens.css`/`fonts.css` migrated. *Done when:* the placeholder stories now show real colors/type per world, and the focus-visible regression test passes against a plain `<button>` even before real components exist.
3. **Batch A+B** — every primitive/field-block component has a story per documented state, passes a11y, matches its `styleguide.html` cell on manual side-by-side.
4. **Batch C+D** — Receipt's tag vocabulary is type-enforced; ContextCard's 4 states each have a story; TabBar/SideNav work at both viewports from one data source, including Review Queue's all-inactive case.
5. **Batches E–J** — each verified against `styleguide.html`; every previously-QA-caught behavior (BatchBar's exclusion, ConnectionCard's fail-open/fail-closed split, CorrelationCard's no-raw-count rule) has an explicit test, not just a story.
6. **Page compositions** — all 10 screens have a composition story, each passing a11y once its children do.
7. **Test suite hardening** — all Step 7 tests written, visual baselines committed, `npm install && npm run test && npm run build-storybook` succeeds clean from a fresh checkout — the Storybook-scoped analogue of FR-0.1, without the Capacitor/native half.

## Verification

- After milestone 1/2: `npm run storybook` shows correctly framed/themed placeholder stories at both viewports; `npm run build-storybook` succeeds.
- Per component batch: manual side-by-side comparison against the matching `styleguide.html` cell in a browser, plus `npm run test` (Vitest + addon-vitest) green for that batch's a11y and any play-function tests.
- End state: clean checkout → `npm install && npm run test && npm run build-storybook` completes with no manual steps.

### Critical files

- `styleguide.html` (this folder) — the component inventory, states, and world/tier assignments every batch is built and verified against.
- `../design-system.md` — the underlying spec `styleguide.html` is kept faithful to; consult directly for any behavior nuance a gallery cell's short annotation doesn't fully capture.
- `../tech-spec.md` §3.6–3.7 — observation body shapes and provenance structure behind `packages/schema` and every `CorrectableField`/`Receipt`/`DayCountTable` prop shape.
- `../CLAUDE.md` (Repository layout) and `../frd.md` FR-0.1–0.7 — the binding constraints this plan scopes around without implementing, and the layout `storybook/apps/app` gets promoted into once step 0 happens for real.
