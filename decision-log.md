# HeadState — Decision Log

Decisions that are settled. Each entry states what was decided, why, and what it changes.
Where a decision reverses or closes something recorded elsewhere, that is noted so the other document can be reconciled.

**Convention:** a decision here is binding until explicitly superseded by a later entry. If a research document disagrees with a decision recorded here, this file wins and the research document should be read as the evidence behind the decision, not as an alternative to it.

---

## D1 — Capacitor is scaffolded from commit one
**Date:** 13 August 2026 · **Status:** Adopted

The repository starts as a Capacitor project wrapping the web app. There is no PWA-only period.

**Why.** Three of the five open-question recommendations depend on native capability: passive sleep and HRV import via HealthKit and Health Connect, scheduled local notifications for dose-triggered follow-ups (a PWA has no local-notification scheduling API), and the only documented route into ChatGPT Health, which runs through Apple Health. Scaffolding native from the start removes a port and a round of platform bugs arriving exactly when integrations are being shipped. For personal use no App Store review is needed to install on your own device.

**Changes.** Supersedes the build-order note in `headstate-strategy-brief.md` §6 that deliberately declined to pull Capacitor forward. Capacitor is now repository setup rather than a build step; the numbered steps are otherwise unchanged. The web app remains the core codebase — this is a wrapper, not a rewrite.

---

## D2 — TypeScript end to end, Postgres
**Date:** 13 August 2026 · **Status:** Adopted

React or Svelte with Vite for the app, Node and TypeScript for the API and the MCP server, Postgres for storage.

**Why.** One language across the web app, the Capacitor wrapper, the MCP server and the backend, which matters disproportionately for a solo builder. Postgres handles an append-only observation log plus derived projections well, with JSONB for observation bodies and normal relational structure for the derived layer.

**Narrowed by D14:** the framework is **React**, not Svelte.

**Open.** If correlation work later needs the scientific-Python ecosystem, that is a separable service reading the same database rather than a reason to split the stack now.

---

## D3 — Managed platform with managed Postgres
**Date:** 13 August 2026 · **Status:** Adopted

Fly.io, Railway or Render, with managed Postgres. Encrypted at rest and in transit, backups handled by the platform.

**Why.** Fastest to running and adequate for a personal build and early direct-to-consumer use. Because it is plain Postgres, migration later is a database move rather than a re-platform. A hyperscaler's main advantage — an available path to a business associate agreement — only matters if the product goes clinical, which is a separate one-way-door decision.

**Narrowed by D14:** the host is **Fly.io**, running **PostgreSQL 16** — matching the 16.13 the specification's DDL was executed against.

**Constraint carried forward.** Server-side sync is mandatory regardless of platform: Safari's LRU storage eviction means client storage cannot be treated as durable.

---

## D4 — A third review state is added to the schema
**Date:** 13 August 2026 · **Status:** Adopted

`review_status: unreviewed | timed_out | user_reviewed` is added alongside `confirmed_by` on derived objects.

**Why.** `confirmed_by` has two values, so an inference that sat in the review queue for two weeks and lapsed is indistinguishable in the data from one computed a second ago — both read `system`. Without a third state, the intended distinction between "a human reviewed this" and "this lapsed unreviewed" cannot be rendered on a clinician export, and the correction corpus cannot tell genuine confirmation from silence. Small contract change now; painful to retrofit once real data exists.

**Changes.** Amends the derived-layer contract in `CLAUDE.md`. `confirmed_by` retains its two values and its meaning; `review_status` is orthogonal to it. Timeout sets `review_status: timed_out` and does not touch `confirmed_by`.

---

## D5 — Late backfill always counts, flagged by confidence
**Date:** 13 August 2026 · **Status:** Adopted

Backfilled entries always count toward headache-day and migraine-day totals regardless of how late they are. They carry low confidence, and that confidence prints on any export.

**Why.** No published rule supports a backfill-latency exclusion — not ICHD-3, not the 2026 Delphi consensus — and inventing one violates the standing anti-pattern against inventing counting rules rather than citing published ones. The concrete harm is specific: a status migrainosus patient incapacitated for three weeks is simultaneously the person most likely to backfill very late and the person most damaged by exclusion, because the missing days are the ones that establish a Botox or CGRP threshold. A reviewer seeing a flagged low-confidence day is strictly better than a reviewer seeing a silent gap.

**Changes.** Rejects the hard-flag exclusion proposed in the first draft of the granularity recommendation. The mechanism used instead already exists: per-field provenance and confidence, carried into the export.

---

## D6 — The prodrome pattern surface is deferred entirely
**Date:** 13 August 2026 · **Status:** Adopted

No forward-looking prodrome alert, and no passive prodrome pattern display, in any form, for now.

**Narrowed by D21:** a retrospective, non-phase-dependent, unscored correlation surface (Insights) is unblocked — the phase-dependent, forward-looking deferral this entry describes stands untouched.

**Why.** Three reasons, any one sufficient. It may not be definable for the primary user: "present before 6 of your last 9 attacks" presupposes discrete attacks with clean interictal baselines, which a continuous-headache patient may not have, and the premonitory literature it rests on is entirely episodic migraine. A conditional base-rate shown at decision time is arguably the thing the no-prediction guardrail exists to prevent, regardless of past-tense phrasing. And it depends on retrospective phase annotation, which is build step 5 — so deferring costs nothing that is not already deferred.

**What this closes.** Two escalated questions are now moot: whether the no-prediction guardrail permits a conditional base-rate display, and whether such a surface should be default-on. The guardrail in `CLAUDE.md` is unchanged and unamended.

**What answers this later.** Retrospective phase labelling is being built anyway. Once it has run for some months, the data will show whether a computable "before an attack" exists for this user. Revisit then, with evidence rather than argument.

---

## D7 — Capture is event-anchored, with a next-morning daily backstop
**Date:** 13 August 2026 · **Status:** Adopted

**Tier 2, event-anchored capture, is the primary mechanism.** Onset and resolution timestamped at or near the moment they are noticed via an open/close affordance. Medication doses auto-trigger targeted follow-ups timed to pharmacology — roughly 30–60 minutes and 2 hours after a triptan or NSAID, next day for a corticosteroid taper — asking specifically about attenuation and recurrence.

**Tier 1, the daily checkpoint, is a backstop, not the primary mechanism.** Next morning only. Two questions are unconditional regardless of branching: aura, and whether the user fell asleep with a migraine and woke without one. On any day inside an open episode the checkpoint is capped short.

**Why event-anchored is load-bearing.** Three fixed constants require sub-day timestamp resolution that a once-daily date-level checkpoint structurally cannot supply: the 12-hour maximum remission inside a status migrainosus episode, the 48-hour distinct-attack separation and the sub-48-hour ambiguous window, and the 30-minute treated-day threshold. A daily checkpoint cannot reliably distinguish a 10-hour gap from a 14-hour one, or 46 hours from 50.

**Why next morning only.** The Delphi overnight rule — asleep with a migraine, awake without one, counts as one migraine day — can only be observed by a morning prompt. An evening checkpoint is asked before sleep and cannot see the waking state, nor an attack starting after the prompt.

**Why aura is unconditional.** An aura day counts as a migraine day regardless of head pain, and migraine days drive the CGRP payer threshold. Gating the aura question behind "did you have a headache today" silently drops days from that count.

**Why the checkpoint is capped short during an episode.** An adaptive checkpoint that grows longer on worse days is heaviest exactly when capacity is lowest. Detail moves into small real-time event asks instead.

---

## D8 — The recall summary is exempt from the confirmation gate
**Date:** 13 August 2026 · **Status:** Adopted

Unconfirmed inferences are force-surfaced for review before the prior-authorisation packet is generated. The recall summary is **exempt**; unconfirmed items are labelled visibly on the page instead.

**Why.** The confirmation gate was specified as mandatory before any "clinician summary" is produced. The recall summary is a clinician summary, and it must render in seconds, offline, one tap from app open, for a photophobic patient in an emergency department. An unskippable confirmation flow on that path is the same category of harm as a paywall on it. Per-field provenance already supports labelling unconfirmed items in place, which preserves the honesty the gate was protecting without blocking the crisis path.

---

## D9 — The document gate is not relaxed for the primary user
**Date:** 13 August 2026 · **Status:** Adopted

The mandatory confirmation gate before generating a prior-authorisation packet applies to Eric as it would to anyone else.

**Why.** The draft recommendation would have skipped it on the grounds that his own discipline substitutes for guardrails. But he is the ground-truth generator the correction corpus depends on, and the corpus is the project's most durable asset. Relaxing his gate removes friction at precisely the one source that produces the asset.

---

## D10 — Specification scope is build steps 0–4
**Date:** 13 August 2026 · **Status:** Adopted

The technical specification and the functional requirements cover step 0 (repository scaffold) plus: capture and the observation store; recall; derived episodes and day classification; and integrations, meaning the Capacitor native surface plus sleep and weather.

Steps 5–7 — phase annotation and retrospective relabelling including medication-response inference, correlation, and the prior-authorisation packet — are deliberately left unspecified. They will change based on what steps 1–4 reveal.

**Document order.** Technical specification first, because the data model is the product and the requirements depend on it rather than the reverse. Functional requirements second, written as **testable acceptance criteria** rather than "the system shall" prose — a coding agent can execute `given 6 triptan days and 6 NSAID days in a 30-day window, the combined-class MOH rule fires`; it cannot execute "the system shall support medication overuse detection."

**No business requirements document for now.** There is no second stakeholder to align, its content already exists across the README and strategy brief, and the monetisation research concluded that no viable paid product has yet been identified. When a specific person needs to read something — a clinical advisor, a contract developer, a potential co-founder — the right artifact is a short positioning memo, not a BRD.

---

## D11 — Qualification is per attack; occupancy is per date
**Date:** 15 August 2026 · **Status:** Adopted · **House decision — no published rule exists for the attack-to-date mapping**

Day classification is two separate operations, and collapsing them produces a counting error in one direction or the other.

**Qualification** is evaluated **once per attack**, against the attack's total duration measured **across midnight**. Per ICHD-3 1.1, an attack continuing into sleep is reckoned until awakening.

**Occupancy**: **every local calendar date the attack overlaps takes the attack's qualification class.** A migraine-qualifying attack spanning four dates produces four migraine days.

**One exception, Delphi rule 4 (100% consensus):** where the patient fell asleep with the attack ongoing and woke headache-free, **that attack's occupancy of the waking date is suppressed.** The suppression is scoped to the attack, not to the date — a separate attack occupying the same date still counts.

**Where two attacks occupy one date**, the date takes the **highest-ranking class** — `migraine` > `headache` > `aura_only` > `none` — with `rule` taken from the attack that produced the winning class and `attributed_episode` from the earliest occupying attack.

**Why both halves are load-bearing.**

*Qualification must be per attack.* Measuring duration per calendar date under-counts nocturnal onset: an untreated 22:00→03:00 attack with two characteristics gives 2 hours on Monday and 3 on Tuesday, reaches the 4-hour threshold on neither, and yields **zero** migraine days where the answer is one. Migraine days drive the CGRP payer threshold.

*Occupancy must span dates.* Attributing an attack only to its onset date under-counts every multi-day attack: a four-day status migrainosus episode yields **one** headache day where the answer is four, and a three-week episode yields one where the answer is 21. That would make the ≥15-day Botox threshold and the `MOH_GENERAL_CRITERION` gate structurally unreachable for a continuous-headache patient — the patient this product exists for. Occupancy is also ICHD-3's own definition of a headache day, applied literally: *"any day affected by headache for any part or the whole of the day."*

**Why this is a house decision.** ICHD-3 and the 2026 Delphi consensus are both silent on midnight-spanning attacks — `continuous-timeline-verification.md` records the gap. The per-attack duration measurement is ICHD-3 1.1; the attack-to-date mapping is not published anywhere and is chosen here.

**Worked cases**, each a required fixture:

| Case | Result |
|---|---|
| Attack Mon 20:00 → wakes headache-free Tue 07:00 | 1 migraine day (Monday); Tuesday `none`, 0 headache days |
| Untreated 22:00 Mon → 03:00 Tue, 2 characteristics, asleep through the end, wakes free | 1 migraine day, not 0 |
| Same attack, awake at 03:00 | 2 migraine days |
| Qualifying attack Mon 09:00 → Fri 09:00, awake throughout | 5 migraine days, not 1 |
| Continuous three-week episode | 21 headache days |

---

## D12 — Medication-overuse is evaluated on a calendar month, with a rolling window as display only
**Date:** 15 August 2026 · **Status:** Adopted · **House decision on window; thresholds are ICHD-3**

The ICHD-3 medication-overuse thresholds are expressed as "days per month." The clinical evaluation uses a **calendar month**, which is the literal reading. A rolling 30-day count may additionally be shown as a **non-clinical early-warning display**, clearly labelled as such, and never as the ICHD-3 determination.

**Supporting definitions, also settled here:**
- **`ACUTE_MEDICATION_DAY`** — a calendar day on which any acute or abortive agent was taken. Counted per drug class, with the day attributed to each class taken that day.
- **`MOH_COMBINED_CLASS` "total"** means **distinct calendar days on which any listed agent was taken**, not the sum of per-class day counts. ICHD-3 8.2.6 counts days of intake. Six triptan days and six NSAID days that overlap on three dates is nine distinct days, not twelve, and does **not** fire.
- **The 8.2.6 agent list is taken literally from ICHD-3**: ergotamine, triptans, non-opioid analgesics, opioids. It does **not** include combination analgesics, which ICHD-3 handles separately at 8.2.5. Generalising to "any combination" would over-fire.
- **`MOH_GENERAL_CRITERION`'s "pre-existing headache disorder"** is satisfied by a **user-attested settings flag**, never inferred from the record. Inferring it would be a diagnostic assertion, which the regulatory guardrails prohibit.

---

## D13 — Attenuation derives from recorded pharmacology, not an inferred half-life model
**Date:** 15 August 2026 · **Status:** Adopted

A day is flagged as pharmacologically attenuated only when it falls inside a medication window the **user recorded** — a corticosteroid taper logged with a start and end date, for example. The system does **not** compute an attenuation window from an inferred half-life model.

**Why.** No published rule defines an attenuation window for this purpose, and inventing per-agent pharmacokinetic windows would be exactly the invented-clinical-rule anti-pattern. Deriving the flag from a fact the user stated keeps it honest and needs no fabricated constant.

**Two consequences to build correctly.** The flag applies to **any** day inside a recorded window, not only days that would otherwise classify `none` — a day with twenty minutes of headache under dexamethasone is also confounded, and restricting the flag to `none` days would hide the more common case. And **an attenuated `none` day does not count** toward headache days or migraine days; the flag is annotation. That is defensible because it is not silent, but the documents must say it plainly rather than implying the flag protects the count.

**`corticosteroid` is added to the drug-class taxonomy.** The capture model schedules a next-day follow-up after a steroid taper and the attenuation model depends on steroids being recordable; without the class, the medication write is rejected.

---

## D14 — Step-0 scaffold choices
**Date:** 15 August 2026 · **Status:** Adopted

Three values that FR-0.1 and FR-0.5 require before a scaffold can be generated.

| Choice | Value | Why |
|---|---|---|
| **Bundle identifier** | **`health.headstate.app`** | Reversed-domain from the owned domain, which is the convention both platforms expect and which matters for domain-verified associations. **Locked — never change it.** App Intents registration, the HealthKit entitlement, and any future widget or Watch extension all key off it. A debug variant may suffix (`health.headstate.app.dev`); the release identifier does not move. |
| **Framework** | **React** with Vite | Largest ecosystem and the deepest Capacitor community coverage. The deciding factor is agentic tooling: this is built primarily through coding agents, and React has substantially more training data behind it, so native-integration work needs less hand-holding. Nothing in this app is performance-bound, so Svelte's bundle-size advantage buys nothing here. |
| **Managed Postgres** | **Fly.io** | Ships PostgreSQL 16, matching the **16.13** the specification's DDL was actually executed against — so no re-verification is needed and the spec's verification claim stays true. Machines-based deploys suit a low-traffic personal build, and Fly has the better documented story for hosting the MCP server alongside the API. |

**Consequence for FR-0.7.** The CI gate provisions Postgres 16 to match production. If the host is ever changed to one shipping a different major version, the full DDL must be re-executed and the specification's verification claim re-checked before that change ships.

---

## D15 — A design system is extracted from the first mockups
**Date:** 16 August 2026 · **Status:** Adopted

Three screens were mocked up (recall summary, mid-attack capture, morning checkpoint) and their tokens, type pairing and component vocabulary were extracted into `design-system.md` rather than left inline in each mockup.

**Why.** The goal is high-fidelity mockups of every screen, eventually feeding Storybook components. Re-deriving a palette or a component shape by eye on each new mockup would drift and would not translate into a component library. Extracting now, while the system is small, is cheap; doing it after five more screens exist would mean reconciling five inconsistent palettes.

**What it settles.** Three visual "worlds" rather than one theme — a viewer-themed chrome, a fixed-dark device world (capture and checkpoint screens, deliberately not theme-reactive because they represent a specific dark-room screen state), and a fixed-paper doc world (the recall summary, meant to print). Full color, type and component-pattern rationale is in `design-system.md`.

**What it does not settle.** The remaining screens split into three buckets, recorded in `design-system.md` §6: ready to mock against this system now (Settings, Onboarding, the review queue, the full Home surface); blocked on a real design decision first (the Timeline row model, the observation-correction interaction, the episode-boundary override interaction, retrospective backfill capture); and out of scope regardless (Insights per D6, the prior-authorisation Report per D10). D16–D19 below resolve the blocked bucket.

---

## D16 — The Timeline is observation-level, grouped by capture moment, not attack-level
**Date:** 16 August 2026 · **Status:** Adopted

The Timeline's row is one **capture moment** — observations grouped by `provenance.transcript_id` when present, otherwise one row per observation — reusing the Receipt card component already built for mid-attack capture rather than inventing a new row type. Episode boundaries and day classification render as a thin per-day rail above each day's rows, not as the row unit itself. Passive-feed observations (sleep, weather) get a quieter, non-card strip, since they weren't asserted by anyone and shouldn't carry the same visual weight as something the user said.

A corrected observation shows its **current value only by default**; a small history marker on the row expands inline to show what it was corrected from, without navigating away.

**Why.** The failure mode that motivated this whole project was organising the record around discrete attacks. The Timeline can't repeat that at the presentation layer even though it fixed it at the schema layer. But raw observations don't fork per derivation variant — only the interpretive overlay does — so a sub-48-hour recurrence's two competing readings (`relapse_collapsed` / `relapse_split`) never need two competing sets of rows: one observation feed, one togglable overlay. Current-value-only-by-default keeps the feed scannable, which the correction corpus depends on people actually using; full inline history on every row would bury the signal invariant 7 exists to capture.

**Changes.** Retires `product-architecture.md`'s `/timeline` and `/event/:id` routes as specified. No new route for episode/day detail — see D17.

---

## D17 — Episode-boundary override is a two-tier interaction off the Timeline
**Date:** 16 August 2026 · **Status:** Adopted

Overriding an episode boundary splits into two interactions of different weight, reached from a drawer/sheet off the Timeline's episode rail — not a dedicated `/episode/:id` route.

**Tier 1 — pick a computed candidate.** For a sub-48-hour recurrence, the system already computes both `relapse_collapsed` and `relapse_split` (§5.3 of `tech-spec.md`). Overriding here is choosing which reading is correct — a light, inline, one-tap action.

**Tier 2 — manual adjustment.** For a boundary that matches neither computed reading (bad data, an edge case the 48-hour rule doesn't handle), a genuinely manual start/end edit is reached one level deeper behind an explicit "Adjust boundary" action — more deliberate than Tier 1, not hidden in Settings.

**Why.** Conflating these was the risk. The common case is cheap because the system already proposed both principled readings — the algorithm-proposes/human-confirms design already does most of the work. The rare case is a real departure from what was computed and deserves more friction, not less. A dedicated Episode page was considered and rejected: it would have revived a standalone detail route in the same shape as the retired `/event/:id`, for a page that doesn't need its own space when a drawer off the Timeline already has the context.

---

## D18 — An episode-boundary override sets that episode's `confirmed_by` to `user`
**Date:** 16 August 2026 · **Status:** Adopted

When a recompute honours an episode-boundary override, the resulting `episodes` row is written with `confirmed_by: 'user'`, even though other fields on that same row (e.g. `computed_from`, unrelated day classifications hanging off it) were computed, not entered.

**Why.** A human set that boundary. `confirmed_by` recording `system` on a row a human directly overrode would be false, and would corrupt exactly the signal the correction corpus (invariant 7) exists to capture. The alternative — leaving `confirmed_by` at `system` and relying solely on the separate `overrides` table to signal human involvement — was considered and rejected as needlessly indirect: anything reading the episode row alone (a clinician export, the recall summary) would see a computed-looking boundary that a human actually set.

**Changes.** `tech-spec.md` §5.4 (Recomputation) and §5.7 (Review tiers) are amended: a recompute that finds a live override for an episode's `natural_key` writes `confirmed_by: 'user'` on that episode, not the default `system` FR-3.26 specifies for newly computed rows. `frd.md` gains FR-3.25b.

---

## D19 — Backfill confidence is a binary threshold, not a decay curve
**Date:** 16 August 2026 · **Status:** Adopted · **House decision — product, not clinical**

An observation is flagged backfilled — and written with reduced confidence per D5/invariant 13 — when `recorded_at − observed_at` exceeds **`BACKFILL_WINDOW` = 6 hours**. Below that threshold it carries ordinary real-time confidence. There is no graduated schedule (same-day / this-week / older); the flag is binary.

**Why.** D5 and FR-1.27 established that a backfilled observation "carries reduced confidence" but never defined how much, or how that scales with lateness — a real gap, found while designing the retrospective-capture screen, not a style choice. A graduated decay schedule would invent a specific curve with no published rule behind it, which is exactly the anti-pattern `CLAUDE.md` prohibits for clinical thresholds — and this number isn't even a clinical one, so it doesn't belong dressed up as more precise than it is. Binary is honest about that and is what a single-user product actually needs: a confirmation line that says "logged for Tuesday — backfilled, lower confidence" doesn't need three tiers of hedging.

**Consequence for the capture flow.** Retrospective/backfilled logging is **not a separate screen or flow**. Flow C's "I had a bad one Tuesday, forgot to log it" already resolves "Tuesday" to a past `observed_at` through chrono-node (FR-1.12) — the same capture pipeline (mic/text → extraction → receipt card) runs unchanged. The only additions are a same-slot conflict check and, when `BACKFILL_WINDOW` is exceeded, the confidence flag and an honest line in the confirmation itself, not only downstream on an export.

**Changes.** `BACKFILL_WINDOW` is added to `CLAUDE.md`'s clinical constants table (alongside `REVIEW_TIMEOUT`, the other product house decision that sits in that table) and to `tech-spec.md` §3.5. `frd.md` §1.5 gains a threshold-testable FR.

---

## D20 — The Report is a single-criterion packet, gated before generation rather than gated inside it

**Date:** 18 August 2026 · **Status:** Adopted · **House decision — content and information architecture; no new clinical rule**

**Scope note.** D10 marked the prior-authorisation Report out of scope "regardless," deliberately, pending what steps 1–4 reveal. Drafting a page order now is a narrower act than that decision anticipated blocking — nothing here commits to a build, and every element below is already specified elsewhere (`CLAUDE.md`'s clinical constants, `tech-spec.md` §3.3/§4.3/§5.5–5.7, `frd.md` FR-3.30, the recall summary's already-built `MedTable`/`FlagChip`/`TimelineList`). This decision is content layout, not new build scope, and D10's caution about the *behaviour* of steps 5–7 remains fully in force — nothing here authorises building step 7's backend.

**The decision.** A generated Report targets **one named payer criterion at a time** — `PAYER_CGRP_EPISODIC`, `PAYER_CHRONIC_BOTOX`, or `PAYER_BOTOX_CONTINUATION` — selected before generation, not a single omnibus document covering all three. Page order, top to bottom:

1. **Cover.** States the target payer and named criterion in plain language (e.g. "Cigna — chronic migraine, Botox initiation"), patient/provider identity fields, the reporting window, and generation metadata — `algorithm_version`, `config_hash`, `derivation_id`, generated-at timestamp. Reproducibility (invariant 2) starts here: a packet generated in March must show, on its own face, exactly what would need to match to reproduce it in June.
2. **Criterion checklist.** The named payer's own criterion, restated with citation, as a literal per-requirement checklist — met / not met — each line backed by the specific computed number satisfying it, and that number carrying its governing rule. Never a bare "eligible: yes." This is where `CLAUDE.md`'s "ICHD-3's own day counts are opinion, not measurement" caveat and the ban on presenting a payer threshold as settled fact actually surface to the reader, not buried in a footnote.
3. **Day-count detail.** The `day_classifications` feeding the checklist, over the reporting window (`D12`'s calendar month, or the criterion's own stated window), with every attenuated day in that range enumerated per D13/tech-spec §5.5 — never silently excluded. Where a criterion's own headache-day definition differs from the Delphi migraine-day count (Cigna's "≥4 hours per day" is not the Delphi ≥30-minute treated-day rule), both appear, separately labelled — `CLAUDE.md`'s "two different headache-day counts... must never be conflated" applies to this page by name.
4. **Treatment and preventive-failure history.** Named drug, class, dose, dates tried, and why discontinued or judged a failure — reuses the recall summary's `MedTable`/`doc-timeline` components rather than inventing new ones. This is prose-with-dates about what was tried, which is different in kind from an attack **count** — invariant 9 prohibits reporting attack counts as a metric, not describing a treatment history narrated by date. Task 8 must not read invariant 9 as license to omit this section, and must not compute or display anything labelled as an attack count while building it.
5. **MOH flag status**, shown only if a flag is live. Presented as the criterion a named guideline uses, per the constants table, with the status-migrainosus-vs-MOH precedence **surfaced, not decided** — matching the same guardrail already applied to the recall summary's red-flag language.
6. **Supporting clinical context.** Baseline pattern, typical presentation, what's different this time — reuses the recall summary's "Baseline & red flags" section close to verbatim rather than re-deriving new copy for the same facts.
7. **Confirmation and provenance footer.** States plainly that every derived claim behind this packet was human-confirmed before generation (not merely computed), and lists the citation for every named threshold used. This is where the gate's *result* is attested on the document itself — the gate's *mechanism* lives before generation, not on this page (see below).

**The confirmation gate is pre-generation, not a page inside the document.** FR-3.30 requires every unconfirmed inference fed into the packet to be force-surfaced before generation proceeds, and `CLAUDE.md` is explicit the gate is not relaxed for the primary user. The natural implementation is: generation is requested → the set of derived objects with `computed_from` timestamps inside the reporting window is checked for `confirmed_by: 'system'` → if any exist, the flow routes into Task 5's `ReviewQueueItem`/`BatchBar` components, filtered to just that set, and the "Generate" action stays disabled until the filtered queue is empty. **A `review_status: 'timed_out'` item blocks generation identically to a fresh `unreviewed` one** — FR-3.28 is explicit that timing out never touches `confirmed_by`, so an item that timed out is still `system`-confirmed and still unconfirmed by the gate's own definition; nothing about the gate treats a timeout as resolved. This was worth stating explicitly because it is the one place a plausible-looking implementation (skip items already "reviewed" in the loose sense) would silently violate FR-3.30.

**Why single-criterion, not omnibus.** The three payer constants want structurally different supporting data — `PAYER_CGRP_EPISODIC` wants two failed preventive classes; `PAYER_CHRONIC_BOTOX` wants a specific ≥15/≥8/≥4h-per-day combination plus three failed prophylactics from two classes; `PAYER_BOTOX_CONTINUATION` isn't an initiation packet at all, it's a before/after comparison against a stored baseline. A single document trying to hold all three either bloats with sections irrelevant to whichever payer actually receives it, or grows conditional logic inside the document to hide the ones that don't apply — worse than asking the question once, up front, the way the README already frames it: "reports have a purchaser," and a purchaser is asking about one specific thing.

**Changes.** `design-system.md` §7 task 7 is marked done, referencing this entry. Task 8 (mocking the Report) builds against the page order above and the pre-generation gate description; it does not need a new design decision to proceed. No changes to `CLAUDE.md`, `tech-spec.md` or `frd.md` — every constant, component and FR cited above already exists there unchanged.

---

## D21 — Insights ships as a retrospective, descriptive correlation surface — never predictive, never scored

**Date:** 21 August 2026 · **Status:** Adopted · **Eric's call, per `design-system.md` §7 task 9 — not a house decision to draft. Narrows D6.**

**Resolving input.** Eric raised this directly with his neurologist: the doctor confirmed there is clinical value in surfacing correlation, on the reasoning that the product's value is returning a *personalised* record — what this particular patient's data actually shows — rather than a *predefined* one imposed on them. That is the resolving input D6 said to wait for ("revisit then, with evidence rather than argument"), recorded here as a direct clinical consultation, not a citation to a paper — distinguish it from the `Research/` evidence base accordingly (per the working-conventions section's own instruction to keep verified, inferred and clinician-consulted inputs distinguishable).

**The decision.** Insights is unblocked, scoped narrowly:

- **Retrospective only.** Every surfaced relationship describes what has already happened in the logged record — "on days following under 6 hours of sleep this quarter, you logged a migraine day the next day in about 40% of cases." Nothing is forward-looking. No alert, no notification, no "today" framing of any kind.
- **Never a score, never a risk framing.** No number that reads as a probability of a future attack, no "elevated," no color-coded risk state. This is not a narrowing of the guardrail — `CLAUDE.md`'s prohibition on attack-risk prediction and on presenting trigger correlation as prediction is **unchanged and unamended**, exactly as D6 stated for itself. The doctor's endorsement is evidence that *correlation* has value: it is not, and could not be, permission to predict.
- **Personalisation over predefinition, concretely.** No fixed "top migraine triggers" checklist and no prompt to log a new field "to find your triggers." Insights only ever computes over data the user already logs for its own reasons (sleep, weather, medication, symptoms); it never expands capture scope to feed itself. This is the literal mechanism behind "brings back the human touch instead of predefining what a migraine can be" — the surface reflects the individual record back, it does not tell the user what to record.
- **Scoped by data dependency, not by a single go/no-go date.** Sleep/weather/medication-day co-occurrence needs nothing beyond what steps 1–4 already capture and can be mocked and built without waiting further. Anything keyed to phase labels (e.g. "symptom X preceded an attack in Y% of cases") stays gated on step 5's retrospective phase annotation actually existing and having run for a while — D6's own condition for revisiting, which this decision narrows but does not waive. A first version of Insights can ship with the former and grow into the latter once step 5 produces the data.
- **Copy discipline.** Correlation is described as co-occurrence, never as cause. "On days you did X, Y happened Z% of the time" is acceptable; "X causes Y" or "X is a trigger" is not — the M-sense RCT (n=477, no significant benefit of trigger-analytics over a bare diary; investigators concluded self-monitoring itself drove the improvement) is the reason this stays a research bet rather than a clinical claim, and that finding is unaffected by the neurologist's endorsement of the *product* value of personalisation. Do not write copy asserting a clinical benefit for this surface.

**Why this counts as narrowing D6, not reopening it.** D6 deferred the *prodrome pattern surface* specifically — forward-looking, phase-dependent, exactly the shape this decision continues to prohibit. D6 also gave itself an explicit reopening condition ("once [phase labelling] has run for some months, the data will show whether a computable 'before an attack' exists"). That condition is not yet met — step 5 doesn't exist yet — so the phase-dependent half of D6's deferral stands untouched. What changes here is narrower: a *non-phase-dependent*, *retrospective*, *unscored* correlation surface was never actually blocked by D6's own reasoning (none of D6's three reasons — undefinability for a continuous-headache patient, the base-rate-at-decision-time risk, and the step-5 data dependency — apply to a backward-looking sleep/weather correlation), and D6 said as much implicitly by scoping itself to "the prodrome pattern surface," not to correlation generally. This decision makes that scope split explicit rather than leaving it to be inferred.

**Changes.** `design-system.md` §7 task 9 is marked resolved, referencing this entry; task 10 (mocking Insights) is unblocked for the retrospective/non-phase-dependent scope only. `decision-log.md` D6 gains a "Narrowed by D21" cross-reference. No change to `CLAUDE.md`'s guardrail language — it was never in question.

---

## Revised build order

Capacitor is repository scaffolding rather than a numbered step (D1). The steps are otherwise unchanged.

| # | Step | Notes |
|---|---|---|
| 0 | Capacitor scaffold, repo setup, CI | TypeScript end to end, Postgres, managed platform (D1–D3) |
| 1 | Capture and the observation store | Immutable log, schema with the third review state (D4), event-anchored capture with the next-morning backstop (D7) |
| 2 | Recall | One-page summary. Printable, shareable, offline. Exempt from the confirmation gate (D8) |
| 3 | Derived episodes and day classification | ICHD-3 constants, Delphi rules, dual readings, override. Backfill always counts (D5) |
| 4 | Integrations | HealthKit and Health Connect for sleep, weather. Unblocked by D1 |
| 5 | Phase annotation and retrospective relabelling | Includes medication-response inference. Also the point at which D6 can be revisited with data |
| 6 | Correlation | A research bet, not a claim — see the M-sense trial |
| 7 | Prior-authorisation packet | Gated by mandatory confirmation (D9) |

---

## Still open

Not decided, and deliberately so.

- **The MCP query-scope tension.** A remote MCP server exposing query tools makes any paywall on derived output architecturally unenforceable, and gating at the query-scope layer would reintroduce the history-depth gating that was ruled out. No clean resolution found. Does not bind for a personal build.
- **What, if anything, becomes a paid product.** Two candidates were identified — maintained payer-policy matching, and a clinician-cosignature workflow — precisely because neither reduces to the user's own data reformatted. Both point toward a clinic or business-to-business direction rather than a consumer SKU.
- **Whether the clinical path is a real ambition.** Contracting with clinics or pushing into an EHR triggers business-associate status. A one-way door, to be walked through deliberately if at all.
- **Instrument licensing.** The daily checkpoint's question wording is a licensing surface. Questions must be independently authored, and any use of PROMIS, HIT-6, MIDAS, MSQ or MPFID item wording needs legal clearance before anything is distributed or sold.
