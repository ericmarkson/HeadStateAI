# HeadState — Design System

**Scope:** the visual identity and component vocabulary behind the screen mockups (recall summary, mid-attack capture, morning checkpoint — see `decision-log.md` D15). Extracted so every future mockup, and eventually every Storybook component, draws from one source instead of being re-derived per screen.

**Status:** covers eight screens built so far — recall summary, mid-attack capture, morning checkpoint, Onboarding, Home, the Timeline (`mockups/timeline.html`, which also carries the episode-boundary override drawer, Task 4), the Review Queue (`mockups/review-queue.html`, Task 5), and Settings (`mockups/settings.html`, Task 6). Observation correction (D16) is built against; the episode-boundary override (D17/D18) is now built too, and the Tier-2 batched-confirmation surface (FR-3.26–3.31) alongside it. Settings closes the loop on the consent architecture Onboarding opened — same `ConsentScope`/`ConsentReceipt` components, same scope names, now showing live current state plus the `research_aggregate` scope Onboarding deliberately deferred. The Report's page order is now decided (D20) though not yet mocked — see §7 task 7/8. §7 is the ordered path to mocking every remaining screen, including the Report. Insights alone stays gated on a scope decision only Eric can make.

---

## 1. Three worlds, not one theme

HeadState's screens don't share a single visual world, and that's deliberate, not inconsistent:

- **Chrome** — page shell, navigation, captions. Follows the viewer's own light/dark preference, per the standard three-state token pattern (bare `:root` = light, `prefers-color-scheme: dark` guarded, `[data-theme="dark"]` for an explicit toggle).
- **Device world** — the capture and checkpoint screens. **Fixed dark**, regardless of viewer theme. These are mockups of a phone screen used mid-attack, in a dark room, by someone photophobic — the point is what the screen *actually* looks like, not what the viewer's OS prefers. A phone-frame device chrome (rounded bezel, notch) marks it explicitly as a "screenshot," not a themed page section.
- **Doc world** — the recall summary. **Fixed paper-toned**, regardless of viewer theme, because it's meant to be printed on white paper and handed to a stranger. Warmer and cooler-neutral than the device world on purpose — paper and glass-in-the-dark are different materials and should not share a palette.

Every color in the device and doc worlds is declared explicitly in `:root`, not inside a media query — they don't react to the viewer's theme by design, which is different from an omission. Only the chrome tokens follow the three-state pattern.

**Reviewer-only controls live in chrome, never inside a device- or doc-world frame.** A state toggle, a "play scenario" trigger, a mode switch that lets one mockup demonstrate two states side by side — none of these exist in the real product; a device/doc frame is a screenshot of one specific, derived state. Putting a reviewer control inside the frame depicts a control that must not exist there. Corrected 16 Aug: the checkpoint mockup's Normal/Open-episode toggle and its play button originally sat inside the phone frame, unlabelled, before this convention was made explicit — both were moved to chrome, matching the pattern the Onboarding and Home mockups already used correctly.

## 2. Color

Named, not defaulted — each was chosen against what the surface actually is (a dark room, a lamp, paper, a clinical flag), not picked from a generic palette.

| Token | Light chrome | Dark chrome | Why |
|---|---|---|---|
| `--surface` | `#F6F7F9` | `#15141C` | Cool near-white / deep ink-slate with a whisper of violet — a considered near-black, not tech-black |
| `--surface-raised` | `#FFFFFF` | `#1D1B26` | Cards, the switcher |
| `--ink` / `--ink-muted` / `--ink-faint` | `#1B1C22` / `#5B5E68` / `#6A6C74` | `#EDEAF2` / `#A9A5B8` / `#8A8699` | Three-step text hierarchy. `--ink-faint` corrected 16 Aug — the original values (`#8A8D97` / `#726D82`) measured 3.31:1 and 3.42:1 against their own surfaces, below the 4.5:1 body-text minimum |
| `--accent` | `#B5652A` | `#D98A46` | Warm amber-clay — the color of a single dim lamp in a dark room, or an amber pill bottle. Not blue (generic health-tech), not teal (generic wellness-app), not saturated red (explicitly prohibited by `CLAUDE.md`'s content guidance for symptom severity) |
| `--focus` | `#2B57B0` | `#7DA2FF` | Deliberately separate from `--accent` — a focus ring must never be confused with a "confirmed" or "flagged" state |
| `--good` | `#3F6B62` | `#7FA79D` | Dusty teal-slate. Used sparingly for confirmed/system-calm states — a second, quieter semantic role, never the primary accent |
| `--flag` | `#9C4A3A` | `#C97A68` | Muted brick, not alarm-red. Used for MOH-threshold and red-flag chips — "flagged," not "emergency" |

| Token | Value | World | Why |
|---|---|---|---|
| `--app-bg` | `#121019` | Device (fixed) | Near-OLED-black — "let the screen stay off" energy |
| `--app-surface` | `#1C1927` | Device (fixed) | Cards, receipts |
| `--app-good-dim` | `#233330` | Device (fixed) | Added with Home (task 2), documented here after the fact — the dim well behind a `--app-good` foreground, same relationship `--app-accent-dim` has to `--app-accent`. Used for the calm/confirmed pill background and the `resolved` confidence tag |
| `--app-accent` | `#D98A46` | Device (fixed) | Same amber family as chrome dark — the one warm light source |
| `--app-faint` | `#87839C` | Device (fixed) | Corrected 16 Aug — the original `#5E5A6E` measured 2.60–2.84:1 against `--app-surface`/`--app-bg`, below the 4.5:1 floor, on copy that includes the FR-4.2 ambiguity line and the "no records" state — exactly the text that most needs to stay legible on a screen built for a photophobic reader |
| `--doc-bg` | `#FCFAF5` | Doc (fixed) | Soft paper, not pure white — reads as a real form, still prints cleanly |
| `--doc-accent` | `#A85A26` | Doc (fixed) | Deepened clay for print-safe contrast; reads as a mid-grey in B&W print |
| `--doc-flag` | `#9C4A3A` | Doc (fixed) | Shared with chrome's `--flag` — the MOH/red-flag language should look the same whether it's on screen or on paper |
| `--doc-faint` | `#756C5B` | Doc (fixed) | Corrected 16 Aug alongside `--app-faint` — the original `#A39A8E` measured 2.66:1 against `--doc-bg` |
| `--app-focus` | `#7DA2FF` | Device (fixed) | Reuses dark-chrome's `--focus` value, pinned rather than theme-reactive. 7.62:1 against `--app-bg` (corrected 16 Aug — originally stated as "~9:1," a measurement error; the value itself was never wrong) |
| `--doc-focus` | `#2B57B0` | Doc (fixed) | Reuses light-chrome's `--focus` value, pinned. ~5.5:1 against `--doc-bg` |
| `--app-border` | `#332F42` | Device (fixed) | Added with Home (task 2), documented here after the fact — the hairline used for card and rail borders. `--rail-none` (below) aliases it |
| `--app-flag` | `#C97A68` | Device (fixed) | Added with the boundary-override drawer (task 4), documented here after the fact — a pinned device-world flag/error color for BoundaryAdjuster's inline validation state (invalid span). Deliberately its own token rather than reusing chrome's reactive `--flag` — reusing a theme-reactive token inside the fixed-dark device world is exactly the class of bug the focus-ring fix (below) already corrected once |
| `--rail-migraine` / `--rail-headache` / `--rail-aura` / `--rail-none` | aliases `--app-accent` / `#B98A52` / `#8E86D6` / aliases `--app-border` | Device (fixed) | **Provisional — added with Timeline (task 3), not yet cross-checked against a full palette pass.** DayRail border-left color, keyed to `day_classifications.class` per `DAY_CLASS_PRECEDENCE` (migraine > headache > aura_only > none). Each passes 4.5:1 against `--app-surface` individually (measured 5.36–6.30:1; `--rail-none`/`--app-faint`-adjacent text is the tightest margin at ≈4.73:1), but migraine/headache sit close together as amber hues — every rail states its class in text too, never by color alone, so this is not currently a legibility risk, but a future pass should widen the hue separation |

**Bug this fixes.** The first three mockups shared one global `:focus-visible{outline-color:var(--focus)}` rule, but `--focus` is a *chrome* token that flips with the viewer's OS theme. On a light-OS viewer, that resolved to the light-mode blue (`#2B57B0`) inside the always-dark device world — 2.77:1 against `--app-bg`, below the WCAG 3:1 non-text minimum. Fixed and doc worlds need their own pinned focus color for the same reason every other fixed-world token is pinned (§1): **scope the rule**, don't rely on the global one inside a world that doesn't react to viewer theme:

```css
.device-screen :focus-visible { outline-color: var(--app-focus); }
.doc-sheet :focus-visible     { outline-color: var(--doc-focus); }
```

Any future mockup with a device- or doc-world container must include this scoped override, not just the global `:focus-visible` rule.

## 3. Type

Three roles, paired deliberately rather than defaulted to a single system sans:

- **Display — Fraunces** (400/600, italic 400). Warm, slightly idiosyncratic serif. Used with restraint, at headline scale only: the recall summary's "Day 12" hero line, screen titles, and — in italic — the provenance line, which reads like a handwritten margin note rather than body copy. It carries the "this is a personal, documented record" half of the product's identity.
- **Body/UI — IBM Plex Sans** (400/500/600). Humane and legible at small sizes in low light; carries the "clinically precise" half. Used for everything else: labels, buttons, running text.
- **Data — IBM Plex Mono** (400/500), `font-variant-numeric: tabular-nums`. Every timestamp, dose, and day count, so columns of numbers actually line up like a real chart. This is the face that makes the recall summary's medication table and the "9 of 10" MOH stat read as *measured*, not typeset.

All three are self-hosted as `@font-face` data URIs (CSP blocks font CDNs inside a published artifact) — see §5 for the source files. `IBM Plex Sans` 400/500/600 and `Fraunces` 400/600 are each served from one variable-font file with multiple pinned-weight `@font-face` rules; `IBM Plex Mono` 400/500 are genuinely separate static files.

Scale: `--fs-eyebrow` (0.6875rem, uppercase, tracked) · `--fs-caption` (0.8125rem) · `--fs-body` (0.9375rem) · `--fs-body-lg` (1.0625rem) · `--fs-h3` (1.1875rem) · `--fs-h2` (1.5rem) · hero uses `clamp(2.1rem, 6vw, 3.1rem)`, not a fixed step, since it has to hold at phone width and full-document width.

## 4. Component patterns established so far

Each is named for what a future Storybook component would be called. None are built as components yet — this is the naming and behavior contract to build against once step 0 exists.

| Pattern | What it is | World(s) used in |
|---|---|---|
| **StatusPill** | Small pill with a dot indicator — "In a migraine · Day 12" | Device |
| **Badge** | Pill-shaped label, neutral or flagged variant | Doc, chrome |
| **FlagChip** | Badge variant with a leading dot, `--flag` colored — MOH/red-flag language | Doc |
| **Card / Receipt** | Rounded container; the Receipt variant holds labelled field rows, each with a confidence/provenance tag and a one-tap "not quite?" correction affordance that turns the value into an inline text input. **The tag vocabulary is closed, not extensible per-mockup** — see the enumerated list below the table before introducing a new tag value | Device |
| **MedTable** | Tabular data with `tabular-nums`, most-recent-first, muted italic effect column | Doc |
| **TimelineList** | Vertical list, fixed-width date column + body, arrow glyph for "led to" transitions | Doc |
| **SegmentedControl** | Two/three-way pill toggle, one option active. **Reclassified 16 Aug from Device to Chrome** — every use built so far (the checkpoint's Normal/Open-episode switch, Home's episode/context-card/capture-demo toggles) is a reviewer affordance for comparing states side by side, never a control that exists in the product itself. Has no confirmed in-product Device-world use yet | Chrome |
| **BinaryChoice** | Two large tap targets with a title + helper subline, for a single yes/no decision that matters (e.g. "Still going" / "Something changed") | Device |
| **Annotation** | Dashed-border callout, small text — used in mockups to call out a spec requirement being demonstrated (e.g. "no follow-up questions during an attack") | Device — mockup-only, likely doesn't ship in the real app |
| **UnconfirmedMark** | Dotted underline + hover/focus label, for any field carrying `review_status: unreviewed` shown outside the confirmation gate | Doc |
| **MomentCard** | A Receipt variant grouping every observation from one capture moment (shared `provenance.transcript_id`) into one Timeline row, so a compound utterance stays one card rather than fragmenting into several | Device (Timeline reuses the device world) |
| **PassiveStrip** | Quieter, non-card row for `source: passive_feed` observations — deliberately lower visual weight than a MomentCard, since nothing was asserted | Device |
| **DayRail** | Thin per-day header strip above a day's MomentCards, colored by `day_classifications.class`. Carries episode/day-class context so MomentCards never have to fork per derivation variant | Device |
| **HistoryMarker** | Small marker on a corrected row; expands inline to the superseded value without navigating away. Default state is collapsed — current value only | Device |
| **CorrectableField** | The generalised "not quite?" pattern from the Receipt component, used identically everywhere a field is shown. Typed by the field's own shape (§3.7 of `tech-spec.md`): chip-picker for enums and tri-state symptoms, stepper for severity, datetime input for timestamps — never one universal text box | Device, Doc |
| **BoundaryPicker** | Tier-1 override interaction: shown for a sub-48-hour recurrence, presents `relapse_collapsed` and `relapse_split` side by side with their governing rule, one tap picks. **Built** (task 4) — a bottom-sheet drawer off the Timeline's DayRail; a real picked override always governs display over the separate reviewer-only relapse toggle, never the reverse | Device (drawer off DayRail) |
| **BoundaryAdjuster** | Tier-2 override interaction, one level deeper than BoundaryPicker: manual start/end edit for a boundary matching neither computed reading, with a plausible-length edit check (no clinical threshold — a plausibility guard only) per the npj spec requirement in `CLAUDE.md`. **Built** (task 4) | Device (drawer off DayRail) |
| **ConsentScope** | One card per consent scope: scope name, the `consent_current` key shown verbatim, plain-language explanation, an explicit "if you decline" line naming what still works, and an Allow/Decline pair of identical geometry and resting treatment — nothing pre-selected, nothing focused (FR-7.3). Granted state uses `--app-good`, declined uses neutral, never `--flag`; declining is a legitimate path, not an error. One card = one `consent_events` row. **Built in Settings** (task 6): the same six-scope set as onboarding plus `research_aggregate` (deliberately deferred there), split into "what I keep" / "what can leave" per FR-7.4, now showing a live current decision rather than a first-ask — `sale` still renders no control anywhere | Device (onboarding, Settings) |
| **StepRail** | Segmented progress bar plus a "name · Step N of M" label, at the top of a bounded linear flow. Segments encode done/current/remaining only — it is a position indicator, not navigation. **Deliberately absent from Settings** (task 6) — Settings isn't a bounded flow, so forcing a step count onto it would misrepresent the screen | Device |
| **ConsentReceipt** | Closing summary of every consent scope with its decision, timestamp and policy version, mono-set and including scopes left undecided. Renders the append-only log's own shape rather than a settings-style summary. **Built in Settings** (task 6) as a full consent-history view — every grant/withdraw across time, live-synced to the ConsentScope cards above it, with one scope's fixture demonstrating grant→withdraw→re-grant→withdraw to prove the append-only claim rather than just asserting it | Device (onboarding close, Settings) |
| **ConnectionCard** | Settings-only "current state" variant of the onboarding `.perm` permission card: name, a state pill (connected / ambiguous / off — never a bare boolean), and a detail line. **Built** (task 6) for health-data (5 demo-switchable states including FR-4.2's ambiguous zero-records state and FR-4.3's Android<14 install branch), notifications (including FR-4.5's exact-alarm-revoked case, surfaced honestly rather than silently re-requested), and weather (gated by the `collect_location` scope exactly as onboarding demonstrated) | Device (Settings) |
| **ExportLinkRow** | One row per `export_links` record: id, a status badge (active / expired / revoked), expiry and view-count-against-`max_views` metadata, and — active links only — a Revoke action carrying the FR-7.12 disclosure permanently in its own copy, not just at click time. Revoke is one-way: `status` moves to `revoked` and the row goes inert, never disappears and never re-revokes. **Built** (task 6) | Device (Settings) |
| **ContextCard** | Home's "today's context" container: a stack of PassiveStrip rows plus one explanatory footer, with four required states — connected-with-data, zero-records, not-connected, and past-the-read-window. The footer carries the honesty load: it must state that reads happen on foreground only (FR-4.9), and in the zero-records state must present the result as ambiguous rather than asserting denial (FR-4.2) | Device |
| **OpenEpisodeCard** | Accent-ruled card naming what is currently open, when it opened, how long it has run, and the affordance to close it. Satisfies the npj "a summary view shows which events remain open" requirement, and is the capture-layer close half of open/close. Capture-layer only — never shows competing derived readings | Device |
| **BackfillConfirmation** | The second capture-confirmation variant (FR-1.27b), structurally distinct from the one-line real-time confirm rather than a reworded version of it: names the past date, names the backfill flag and reduced confidence, and states that the day still counts. Appears at capture, not only on export | Device |
| **ConflictNotice** | Dashed-border interrupt shown when a backfilled observation lands in a slot that already holds one. Presents the existing entry and offers add-alongside or inspect-first — never auto-merge, since merging would mutate an observation. **Behaviour is an open proposal, not settled spec** | Device |
| **ReviewQueueItem** | Tier-2 queue row (48-hour-rule or status migrainosus) distinguishing `confirmed_by`/`review_status` explicitly, never collapsing them. The 48-hour-rule variant reuses BoundaryPicker/BoundaryAdjuster verbatim as its "choose a different reading" path — one override write, two entry points (Timeline's DayRail drawer and this queue), never two competing interactions for the same decision. A `timed_out` item stays fully confirmable (FR-3.28) and is labelled, not disabled. **Built** (task 5) | Device |
| **BatchBar** | Select-all + "Confirm selected (N)" bar for accepting each selected item's current system reading in bulk. **Eligibility, not just the UI, is scoped**: only kinds with a single "system's current reading" to honestly accept are selectable at all — a status migrainosus item's checkbox is permanently disabled, because ICHD-3 doesn't decide between it and the ordinary-episode reading, so there is no default a bulk action could accept without silently resolving that ambiguity for the user (invariant 7). This was a QA-caught blocking bug in the first build (bulk confirm unconditionally forced `status_migrainosus`); fixed by excluding the kind from batch eligibility at the data layer, not just hiding a control. **Built** (task 5) | Device |

**Note on rows marked as open proposals** (ConflictNotice above is currently the only one). This table's stated purpose is the build contract — what a future component should do. An explicitly-unsettled row sitting in it without visual distinction from the settled rows around it is a real risk for a reader (human or agent) scanning it for what to implement. Until this table is split into settled/proposed sections, any row carrying a bolded "not settled" caveat must be treated as informational only, never as something to implement as specified.

### Card/Receipt confidence-tag vocabulary — closed list

Every tag value any mockup's Receipt component may render, with what it means. Do not invent a new one in a mockup without adding it here first.

| Tag | Meaning |
|---|---|
| `lexical` | An explicit lexical match in the transcript ("8 out of 10", "left side") |
| `paraphrase` | Mapped through a vocabulary from non-clinical language ("really bad" → severe) |
| `user` | The value was corrected by the user via CorrectableField, superseding an extracted value |
| `no source` | `ask_status: not_asked` — nothing to tag, because nothing was asked (invariant 4) |
| `resolved` | Populated by a deterministic lookup or resolver, not stated directly — e.g. a drug name resolving to its `drug_class`, or chrono-node resolving "Tuesday" to an absolute date. Distinct from `system`: a `resolved` value still traces to something the user said |
| `system` | Populated by the pipeline with no user utterance behind it at all — e.g. `recorded_at`. A field carrying this tag is never correctable; if a field has a "not quite?" affordance, it cannot be tagged `system` |
| `reduced` | Confidence lowered per D19's `BACKFILL_WINDOW` rule — always paired with the backfilled-copy treatment, never shown alone |

## 5. Using this system

**Where the mockups live.** `mockups/` at the repo root — alongside `tech-spec.md`, `frd.md`, this file, the lot. Not scratch output; these are the project's actual design deliverables and persist the same way everything else does. Seven published files at the top level (`headstate-mockups.html`, `headstate-styleguide.html`, `onboarding.html`, `home.html`, `timeline.html`, `review-queue.html`, `settings.html`), each mirrored to a Claude Artifact URL for viewing — the repo file is the source, the URL is a rendering of it, re-published whenever the file changes. `mockups/templates/` holds the pre-font-splice source for each (placeholders intact, useful as a starting point for a new mockup). `mockups/fonts/` holds the five font files already fetched and base64-encoded — reuse these `.b64.txt` files for any new mockup rather than re-fetching from Google Fonts.

**Tokens.** The full `:root` custom-property block, as currently implemented, lives in any published mockup's `<style>` block — `headstate-mockups.html` is the shortest reference. Copy it verbatim as the starting point for any new screen; do not re-derive colors by eye.

**Fonts.** Five self-hosted files, fetched from Google Fonts and re-served as data URIs (never linked — CSP blocks the CDN):
- Fraunces italic 400, Fraunces normal 400+600 (variable, shared file)
- IBM Plex Sans 400+500+600 (variable, shared file)
- IBM Plex Mono 400, IBM Plex Mono 500 (separate static files)

Once build step 0 exists, these still belong in a proper shared assets location inside the app package rather than base64-duplicated per mockup file — that gap is about the *product*, not about where the mockups themselves live, which is settled above.

**Spacing/radius.** Two radius scales by world, not one: `--radius-doc` (3px — paper has sharp corners) for the doc world, `--radius-card` (14px) and `--radius-pill` (999px) for the device world — native app UI convention, not decorative variety. Layout uses flex/grid `gap`, never per-element margins.

## 6. Resolved (D16–D19)

These four were blocked as of the first pass; each now has a settled answer, recorded in `decision-log.md`. D16 (Timeline row model) and D17/D18 (episode-boundary override) are now built against — `mockups/timeline.html`, tasks 3 and 4 of §7. D19 (retrospective backfill) is decided and demonstrated within the Timeline's existing capture-pipeline fixtures (a backfilled entry, not a standalone screen — see D19 itself, which specifies this was never meant to be a separate surface).

- **Timeline row model (D16).** One row = one capture moment (MomentCard, grouped by `provenance.transcript_id`), not an observation, a day, or an episode. Episode/day-class context is a DayRail overlay, not the row unit — which is what lets `relapse_collapsed` and `relapse_split` coexist without two competing row sets. A corrected observation shows current value only by default, with a HistoryMarker to expand it.
- **Observation correction interaction (D16, component in §4).** CorrectableField, typed per field shape, used identically on the Timeline, the recall summary, and the review queue — not a bespoke pattern per surface. The UI presents a single-field edit; the write it produces is a whole new observation row plus a supersession link (FR-1.2) — that gap between UI abstraction and data write is deliberate, not an oversight to fix.
- **Episode-boundary override interaction (D17, D18).** Two tiers, both reached from a drawer off the DayRail, not a dedicated route: BoundaryPicker (choose between the two computed candidates — light, inline) and BoundaryAdjuster (manual edit when neither computed reading fits — one tap deeper, more deliberate). Per D18, an override sets the affected episode's `confirmed_by` to `user` even where other fields on that row were computed.
- **Retrospective backfill capture (D19).** Not a separate screen. Flow C's relative-date resolution already produces a past `observed_at` through the same capture pipeline; the only additions are a same-slot conflict check and, when `recorded_at − observed_at` exceeds `BACKFILL_WINDOW` (6 hours, binary — no decay curve), a confidence flag surfaced in the capture confirmation itself, not only on a later export.
- **Report page order (D20).** Content and information architecture only — not a build. Single-criterion packet, seven pages, pre-generation confirmation gate. See §7 task 7 and `decision-log.md` D20.

Insights remains out of scope per D6. The prior-authorisation Report's *build* remains out of scope per D10 — D20 settled only its page order, as content the way D16–D19 settled interaction design, not as authorisation to implement step 7.

## 7. Roadmap to full coverage

Ordered by dependency, not by page name — each step either needs nothing further or needs only the step(s) above it.

| # | Task | Status |
|---|---|---|
| 1 | Mock **Onboarding** — consent screens (separate collection/sharing, equally weighted, FR-7.3/7.4) + conversational baseline capture | No decision needed. First because Settings (6) reuses its consent components |
| 2 | Mock **Home**, full surface — symptom-free-day state, passive context card, retrospective-entry affordance, both capture-confirmation variants (real-time vs. D19's backfilled line) | No decision needed |
| 3 | Mock **Timeline** — MomentCard feed, PassiveStrip, DayRail, HistoryMarker | **Done** — `mockups/timeline.html`. Also demonstrates the `relapse_collapsed`/`relapse_split` dual reading (D17 scalar threshold, chrome-world toggle) and the DAY_CLASS_PRECEDENCE merge case, without building the boundary-override drawer itself (that's task 4) |
| 4 | Mock the **episode-boundary override drawer** — BoundaryPicker (Tier 1) + BoundaryAdjuster (Tier 2), off the Timeline's DayRail | **Done** — built into `mockups/timeline.html`'s Aug 13 DayRail. Independently QA'd; one blocking desync bug (override display disagreeing with the reviewer-only toggle) found and fixed |
| 5 | Mock the **review queue** — Tier-2 batched confirmation | **Done** — `mockups/review-queue.html`. Covers the 48-hour-rule and status-migrainosus portions of Tier 2 only (medication-response inference is step-5/out of scope per FR-3.27). Independently QA'd; one blocking bug (bulk confirm silently forcing `status_migrainosus` with no user-visible default to accept) found and fixed by excluding that kind from batch eligibility entirely |
| 6 | Mock **Settings** — integrations + consent/sharing | **Done** — `mockups/settings.html`. Reuses (1)'s ConsentScope/ConsentReceipt components verbatim; adds ConnectionCard and ExportLinkRow. Independently QA'd; two low-severity confirmed bugs found and fixed — a live revoke stamping the real system clock against the mockup's fixed demo clock (would misrender "revoked · today" for any reviewer opening the file on a later date), and `syncWeather()`'s null-handling silently diverging from onboarding's identical gate (fail-closed vs. onboarding's permissive default on an undecided scope) |
| 7 | Draft the **Report's page order** (D20) | **Done** — see `decision-log.md` D20. Single-criterion packet (one of `PAYER_CGRP_EPISODIC` / `PAYER_CHRONIC_BOTOX` / `PAYER_BOTOX_CONTINUATION` per generation, not an omnibus document); seven-page order: cover, criterion checklist, day-count detail, treatment/preventive-failure history, MOH flag status, supporting clinical context, confirmation/provenance footer. The FR-3.30 confirmation gate is pre-generation (reuses task 5's `ReviewQueueItem`/`BatchBar`, filtered to the reporting window, including `timed_out` items), not a page inside the document itself |
| 8 | Mock the **Report** | Uses (7)'s layout + (5)'s gate pattern + the MedTable/FlagChip already built for the recall summary |
| 9 | Resolve **Insights'** scope | Eric's call, not a house decision to draft — touches the no-prediction guardrail directly. Either deliberately narrow D6 for a specific bounded surface, or leave shelved pending the neurologist review. Blocks nothing above it |
| 10 | Mock **Insights** | Only once (9) has an actual answer to mock against |

**Explicitly not on this list:** extracting any of this into real Storybook components. That's the phase after full mockup coverage, and it's gated on build step 0 (the Capacitor scaffold) existing — there is no `packages/` yet to put a component in.
