# CLAUDE.md — HeadState

## Start here

**Status: design complete for build steps 0–4. No source code exists yet.** The
next task is step 0 — scaffold the Capacitor project — followed by step 1,
capture and the observation store.

**Read in this order, depending on what you are doing.**

*Writing code:*
1. This file, in full. It is the contract — invariants, clinical constants, and
   anti-patterns. Everything here is binding.
2. `tech-spec.md` — the schema, the derivation engine, the capture pipeline, the
   MCP server, the native surface, privacy. All its SQL has been executed against
   PostgreSQL 16.13.
3. `frd.md` — acceptance criteria in Given/When/Then form, scoped to steps 0–4.
   Most of it is directly executable as tests. Build against these, not against
   prose.
4. `decision-log.md` — why things are the way they are. Read it before proposing
   a change to any of them.

*Understanding the product rather than building it:*
`README.md`, then `headstate-strategy-brief.md`.

*Resolving a clinical question:*
`Research/continuous-timeline-verification.md` first — it holds the verified
counting rules and states plainly what is and is not novel here.

**Four things that will bite you if you skip them.**

- **Day counting is the single most consequential arithmetic in this codebase**,
  and it has been implemented wrong twice during design. Read the
  `DURATION_QUALIFICATION`, `DAY_OCCUPANCY` and `DAY_CLASS_PRECEDENCE` constants
  and the worked cases below before touching classification. The five standard
  cases must give **1 / 1 / 2 / 5 / 21**.
- **Every count is scoped to a single `derivation_id`** — one computed reading of the record; see the glossary below. An unscoped count
  inflates by the number of competing readings — verified, not theoretical.
- **Nothing is populated without a traceable source.** Extraction is
  span-grounded and independently verified; a field that cannot be traced is
  dropped, never guessed.
- **The clinical constants are not yours to change.** They are versioned config
  with citations. Where no published rule exists, the decision log records it as
  a house decision — add to that record rather than inventing a rule inline.

**Before writing code that depends on them, resolve these** — the full list is in
`tech-spec.md` §13:
- Does HealthKit work under free iOS provisioning? If not, the Apple Developer
  Program is a forced early purchase and HealthKit is priority one.
- Verify the ICHD-3 constants against the printed classification, and retrieve
  the 2026 Delphi Correction. Both are currently unverified.
- Re-check the MCP SDK line — 1.30.0 implements the superseded protocol revision,
  deliberately, and that calculus may have changed.


## What HeadState is

HeadState (headstate.health) is a migraine and symptom record where **individual
timestamped observations are the atomic unit**, and attacks, phases, episodes and
day-classifications are **derived, versioned, recomputable projections** over the
observation stream. It is built by Eric, a migraine patient, for his own use first;
commercialisation is second. It has three jobs: **capture** (frictionless logging),
**recall** (on-demand summaries for a clinical encounter or ER visit), and
**documents** (structured output supporting prior authorisation). The repository
is scaffolded as a Capacitor project from commit one, wrapping the web app —
there is no PWA-only period — plus a remote MCP server so external assistants
(Claude, ChatGPT, Gemini, Siri) can read and write the record.

## Why the model is inverted

Migraine is continuous, and its structure is only knowable in retrospect. A
three-week episode with medication-induced gaps is one thing, not several. Nausea
on a day with no head pain is data, and whether it was premonitory is only
determinable after what follows. A drug that produces a multi-day remission
followed by recurrence is evidence the episode did not end. None of that is
representable by a schema whose primary key is "an attack." For the verified
evidence behind this, and for what is and is not novel here, read
`Research/continuous-timeline-verification.md` first.

## Precedence when sources disagree

When this file and a document in `Research/` disagree: this file wins for
implementation decisions, and the Research file wins for evidence and citations.
Do not silently reconcile a discrepancy — report it.

**Full precedence order, highest first:** `decision-log.md` → `tech-spec.md` and
`frd.md` (jointly, for implementation detail) → this file → `Research/`. This
file carries invariants and constants; where it and `tech-spec.md` describe the
same mechanism at different levels of detail, the specification's detail governs
and this file's summary should be corrected rather than followed. Research files
govern evidence and citations only, never implementation.

`decision-log.md` sits above all of them. Where it disagrees with any other
document, the decision log wins: it is the binding record of settled
decisions, and research documents are the evidence behind a decision, not an
alternative to it.

## Glossary — terms this file uses before defining them

- **observation** — one immutable, timestamped fact. The atomic unit. Never updated, never deleted.
- **derivation** — a parent row identifying one computed reading of the record, keyed `(user_id, variant, algorithm_version, config_hash)`. Episodes and day classifications hang off it. **Every count is scoped to exactly one `derivation_id`**; an unscoped count inflates by the number of competing readings.
- **variant** — which reading a derivation represents. `primary` when nothing is ambiguous; `relapse_collapsed` and `relapse_split` where ICHD-3 declines to resolve a sub-48-hour recurrence; `status_migrainosus` where that classification competes with a distinct-attack reading.
- **qualification** — deciding whether an attack meets a migraine threshold. Evaluated **once per attack**, on total duration across midnight.
- **occupancy** — applying an attack's qualification class to **every calendar date it overlaps**. One exception, scoped to the attack: a waking date where the patient slept through and woke headache-free.
- **attenuation** — a day falling inside a **user-recorded** medication course, flagged as pharmacologically confounded. A boolean, not a class and not a rule. An attenuated `none` day does not count toward day totals; the flag is what stops that being silent.
- **natural_key** — an episode's stable identity across recomputation: the id of its earliest member observation, resolved back through any supersession chain to the original id.
- **`observations_live`** — the observation stream anti-joined against `supersessions`. Derivation reads this view, never the raw table.
- **`review_status`** — `unreviewed | timed_out | user_reviewed`. Orthogonal to `confirmed_by`: that records who set a value, this records whether a human ever looked at it.
- **correction corpus** — the accumulated record of user overrides and confirmations. The project's most durable asset, and the reason `review_status` exists.

## Non-negotiable invariants

These are hard rules. Violating any of them is a bug, not a style issue.

1. **Observations are immutable and append-only.** Corrections never delete or
   mutate an observation — they append a new observation and record a
   supersession. The verbatim user utterance is always retained.
2. **Derived objects are versioned.** Every episode, phase annotation and day
   classification carries `algorithm_version` and `computed_at`. A report generated
   in March must be reproducible in June after the algorithm has been reinterpreted.
3. **Two clocks, always.** `observed_at` (when it happened) and `recorded_at` (when
   it was logged) are distinct fields. Never collapse them into one timestamp.
4. **Symptom fields carry two orthogonal fields, never a bare boolean.** "No
   nausea" and "nausea was never discussed" are different clinical facts, and
   collapsing to a boolean corrupts the provider output. The prohibition is on a
   bare boolean, not on recording an affirmative. Every symptom field carries
   `ask_status: not_asked | asked` and `value: yes | no | unknown` (null when
   `ask_status = not_asked`).
5. **Per-field provenance and confidence on every extracted field.** Record whether
   a value came from an explicit lexical match ("8 out of 10", "left side") or a
   paraphrase ("really bad" → severe), and whether the source was the user, a
   passive feed, or an inference. Provenance travels all the way into the provider
   report.
6. **No field may be populated without a traceable source.** Extraction must be
   schema-constrained against the transcript. Run a second-pass check that every
   populated field traces to a specific transcript span or a named passive feed
   before commit. Never free-generate clinical detail.
7. **The algorithm proposes; the human confirms; the output distinguishes the two.**
   Every derived claim records whether it was computed or human-confirmed. This is
   simultaneously a correctness requirement, a regulatory boundary, and the
   mechanism that produces the project's most valuable asset (the correction
   corpus).
8. **User overrides are first-class persisted facts and must survive
   recomputation.** An overridden episode boundary is not a patch applied on top of
   the algorithm's output — it is data.
9. **Payer-facing exports report DAY COUNTS, never attack counts.** See the
   clinical constants table below. This is the single most consequential rule in
   the codebase.
10. **Never ask for phase at capture time.** Phase is assigned retrospectively.
    Asking "is this prodrome?" while logging is wrong on evidence (only ~27.5% of
    premonitory symptoms are recognised prospectively) and wrong on design.
11. **Clinical thresholds are versioned config, never inline constants.** ICHD-4 is
    in development and MOH's survival as a diagnosis is under debate.
12. **Text input is first-class, not a fallback.** Voice is unusable on transit, in
    shared offices, and around family. Any feature that only works by voice is
    incomplete.
13. **Backfilled observations always count toward `headache_days` and
    `migraine_days`, regardless of latency.** No published rule supports a
    backfill-latency exclusion. A backfilled observation carries low confidence
    via the standard per-field mechanism (invariant 5), and that confidence
    travels into every export — flag it, never drop it.

## Clinical constants — implement these, do not invent rules

All values below are sourced from published clinical criteria, not house rules.
All must live in versioned config (see Repository layout), never as inline
literals.

| Constant | Value | Source |
|---|---|---|
| `STATUS_MIGRAINOSUS_MAX_REMISSION` | 12 hours — max remission tolerated within one status migrainosus episode ("Remissions of up to 12 hours due to medication or sleep are accepted") | ICHD-3 1.4.1 |
| `STATUS_MIGRAINOSUS_MIN_DURATION` | >72 hours — unremitting, debilitating headache, typical of the patient's previous attacks except in duration and severity | ICHD-3 1.4.1 |
| `DISTINCT_ATTACK_SEPARATION` | 48 hours headache-free — "count as distinct attacks only those that are separated by at least 48 hours headache-free" | ICHD-3 Definition of Terms |
| `RECURRENCE_AMBIGUOUS_WINDOW` | <48 hours (scalar threshold only). See the separate dual-reading behavioural rule below the table — the requirement to compute both readings is not part of this constant's value. | ICHD-3 Definition of Terms |
| `DELPHI_TREATED_DAY_MIN` | 30 minutes — a treated day counts at ≥30 min regardless of drug class, characteristics or associated symptoms (94% consensus) | Delphi 2026 |
| `DELPHI_UNTREATED_DAY` | Compound predicate, not a scalar — do not implement as `if duration >= 4h then migraine_day`: duration ≥4 hours **AND** (≥2 of 4 headache characteristics **OR** (≥1 characteristic **AND** ≥1 associated symptom)) (83% consensus) | Delphi 2026 |
| `DELPHI_AURA_DAY` | aura meeting ICHD-3 criteria counts regardless of whether headache is present (83% consensus) | Delphi 2026 |
| `DURATION_QUALIFICATION` | **Qualification is evaluated once per attack**, against the attack's total duration measured across midnight (an attack continuing into sleep is reckoned until awakening, per ICHD-3 1.1). This is what prevents a 22:00→03:00 attack being split into 2 h and 3 h and reaching neither the treated nor the untreated threshold. First match wins: (1) acute medication taken during the attack **and** duration ≥30 min → migraine-qualifying; (2) duration ≥4 h **and** the `DELPHI_UNTREATED_DAY` predicate → migraine-qualifying; (3) otherwise → headache-qualifying | **house decision, D11** — ICHD-3 and Delphi are both silent on midnight-spanning attacks; the per-attack duration measurement itself is ICHD-3 1.1 |
| `DAY_OCCUPANCY` | **Every local calendar date the attack overlaps takes the attack's qualification class.** A migraine-qualifying attack spanning four dates produces four migraine days — this is ICHD-3's own headache-day definition ("any day affected by headache for any part or the whole of the day") applied literally. **One exception, Delphi rule 4 (100% consensus):** where the patient fell asleep with the attack ongoing and woke headache-free, **that attack's occupancy of the waking date is suppressed** — the suppression is scoped to the attack, not to the date, so a separate attack occupying the same date still counts | ICHD-3 headache-day definition; exception per Delphi 2026 rule 4, **house decision D11** for the attack→date mapping itself |
| `DAY_CLASS_PRECEDENCE` | Where two attacks occupy one date, `day_classifications` holds one row per date, so occupancy merges: `class` takes the highest-ranking occupancy — `migraine` > `headache` > `aura_only` > `none` — `rule` comes from the attack that produced the winning class, and `attributed_episode` from the **earliest** occupying attack | **house decision, D11** |
| `ACUTE_MEDICATION_DAY` | a calendar day on which any acute/abortive agent was taken, counted per `drug_class`, with the day attributed to each class taken that day. This is the input to every MOH threshold below. | derived from ICHD-3 8.2 |
| `DRUG_CLASS` (taxonomy) | `triptan \| ergot \| opioid \| combination_analgesic \| non_opioid_analgesic \| corticosteroid` — required field on every medication observation; referenced by every MOH constant | ICHD-3 8.2 |
| `MOH_GENERAL_CRITERION` | headache on ≥15 days/month in a patient with a pre-existing headache disorder, plus regular overuse of one or more acute/symptomatic drugs for >3 months. This is the gating criterion — the per-class and combined-class constants below refine it, they do not replace it. Do not fire an MOH flag on drug-days alone without also checking this. "Pre-existing headache disorder" is satisfied by a **user-attested settings flag**, never inferred from the record — inferring it would be a diagnostic assertion the guardrails prohibit. | ICHD-3 8.2 |
| `MOH_PER_CLASS` | map, not a scalar: `triptan → 10 days/month`, `ergot → 10 days/month`, `opioid → 10 days/month`, `combination_analgesic → 10 days/month`, `non_opioid_analgesic → 15 days/month`, each sustained over >3 months | ICHD-3 8.2 |
| `MOH_COMBINED_CLASS` | **distinct calendar days** on which ergotamine, triptans, non-opioid analgesics **or** opioids were taken, ≥10/month, with no single class overused. **Not** the sum of per-class day counts, and **excludes combination analgesics** — ICHD-3 handles those at 8.2.5. A per-class counter alone under-detects, but the combined count is over distinct days, not summed days: 6 triptan days + 6 NSAID days that are **disjoint** is 12 distinct days and fires; the same 6 + 6 overlapping on 3 dates is 9 distinct days and does **not** fire. | ICHD-3 8.2.6 |
| `MOH_EVALUATION_WINDOW` | calendar month | **house decision, D12 — ICHD-3 says "per month"; a rolling 30-day count may be displayed as a labelled non-clinical early warning but is never the ICHD-3 determination** |
| `PAYER_CGRP_EPISODIC` | ≥4 migraine days/month, plus documented failure of two preventive drug classes | payer criteria — see `Research/clinical-regulatory.md` |
| `PAYER_CHRONIC_BOTOX` | ≥15 headache days/month with ≥8 migraine days among them (Cigna wording: "headache lasting 4 hours per day or longer"), plus failure of ≥3 prophylactics drawn from ≥2 classes | payer criteria — see `Research/clinical-regulatory.md` |
| `PAYER_BOTOX_CONTINUATION` | ≥50% reduction in monthly migraine frequency from baseline | payer criteria — see `Research/clinical-regulatory.md` |
| `REVIEW_TIMEOUT` | 14 days — the review-tier timeout that sets `review_status: timed_out` without touching `confirmed_by` | house decision — product |
| `BACKFILL_WINDOW` | 6 hours — binary threshold, not a decay curve. `recorded_at − observed_at` beyond this flags the observation backfilled (reduced confidence, per invariant 13); below it, ordinary real-time confidence applies | house decision, D19 — product, not clinical |

**Worked cases for `DURATION_QUALIFICATION` / `DAY_OCCUPANCY`.** These are the cases the rule exists to get right, and each is a `[V]` fixture:

- Attack Mon 20:00 → wakes headache-free Tue 07:00 → qualifies under whichever rule applies — treated at ≥30 min, or untreated with the ≥4 h compound predicate — occupies Mon and Tue, step 3a excludes Tue → **1 migraine day (Monday)**; Tuesday is `none` and contributes **0** headache days.
- Untreated 22:00 Mon → 03:00 Tue, 2 characteristics, asleep through the end and wakes free → 5 h total qualifies, step 3a excludes Tue → **1 migraine day**, not 0.
- Same attack, patient awake at 03:00 → step 3a does not apply, Tuesday had 3 h of headache → **2 migraine days**.
- Qualifying attack Mon 09:00 → Fri 09:00, awake throughout → occupies five dates → **5 migraine days**, not 1.
- Continuous three-week episode → **21 headache days**.

**Recurrence within `RECURRENCE_AMBIGUOUS_WINDOW` — separate behavioural rule.**
ICHD-3 says recurrence under 48 hours "may represent a relapse of the same attack
or a new attack" and stops there. **Compute and expose BOTH readings, each
labelled with its governing rule. Do not pick one silently.** This is a
derived-layer requirement, not a capture-layer one — see "Reconcile the
one-active-event rule" under Borrow this published spec, below.

**Status migrainosus vs. MOH — precedence when both could apply.** ICHD-3 notes
that status migrainosus "may often be caused by medication overuse." If
`MOH_GENERAL_CRITERION` (or a per-class/combined-class constant) is met, code MOH
plus the underlying migraine type — not status migrainosus. This file surfaces
that the distinction exists; it does not decide it in patient-facing copy.

**ICHD-3's own day counts are opinion, not measurement.** ICHD-3 states its
day-count and duration thresholds are "based on expert opinion rather than on
formal evidence." Never present a threshold (MOH days, CGRP eligibility days,
etc.) to a patient as settled fact — present it as the criterion a specific named
guideline or payer uses.

**ICHD-3 1.1 migraine criteria the schema must support:** ≥5 lifetime attacks;
duration 4–72 h untreated; ≥2 of 4 (unilateral · pulsating · moderate-to-severe ·
aggravated by **or causing avoidance of** routine physical activity); and **at
least one of** nausea and/or vomiting **OR** photophobia **AND** phonophobia
(both together). This last logical structure is commonly implemented wrong — it is
`nausea/vomiting OR (photophobia AND phonophobia)`, not a flat OR of four symptoms.

**Caveat:** ICHD-3 quotations in the research documents were retrieved
electronically, not from the printed classification, and the Delphi paper has a
published Correction that has not been reviewed. **Verify against printed ICHD-3 /
Cephalalgia 2018;38(1):1–211 and the corrected Delphi paper before any clinical or
regulatory use.**

## Data model contract

Field names below are indicative, not binding — the point is the shape and the
invariants, not the exact names.

Observation layer — immutable, append-only:

```
observation_id, user_id
observed_at (timestamp, timezone), recorded_at (timestamp, timezone)
kind: symptom | pain | medication | medication_response | sleep | food | activity
      | stress | cycle | environment | care_event | note
body: { type, value, laterality?, region?, quality?, severity?,
        dose?, route?, drug_class?, perceived_relief? }
  - for kind: symptom — body carries ask_status: not_asked | asked and
    value: yes | no | unknown (null when ask_status = not_asked). See invariant 4.
  - for kind: medication — body carries dose, route, and
    drug_class: triptan | ergot | opioid | combination_analgesic |
    non_opioid_analgesic | corticosteroid (see clinical constants table), plus
    perceived_relief and, for a taper, course_start / course_end.
source: user_voice | user_text | user_tap | passive_feed | imported
provenance: verbatim_span? | feed_name?
confidence: float
```

Supersession is a separate table, not a column on the observation: the
immutability trigger makes a `superseded_by` column unsettable by the
application role, so putting it on the observation itself would be a
contradiction.

```
supersessions: superseded_id, superseded_by, reason, created_at
```

Derivation reads an `observations_live` view — the observation stream
anti-joined against `supersessions` — never the raw table.

Derived layer — recomputable, versioned, overridable. All three derived object
types use the same `confirmed_by: system | user` field — there is one shape for
"the algorithm proposed this / a human confirmed it," not three — and the same
`review_status: unreviewed | timed_out | user_reviewed` field alongside it. The
two are orthogonal: `confirmed_by` records who set the value, `review_status`
records whether a human has actually looked at it. A timeout sets
`review_status: timed_out` and does not touch `confirmed_by`. This is what lets a
clinician export distinguish a reviewed inference from one that lapsed, and lets
the correction corpus tell genuine confirmation from silence:

```
episode_id, algorithm_version, computed_at,
confirmed_by: system | user, review_status: unreviewed | timed_out | user_reviewed
computed_from: observation_id[]   # exact observation versions used this run
member_observations[]
boundary_rule: icd_48h | status_migrainosus_12h | med_response_inference
open: bool
phase_annotations[]{ observation_id, phase, confidence, assigned_at,
                      confirmed_by: system | user,
                      review_status: unreviewed | timed_out | user_reviewed }
day_classifications[]{ date, class: none | headache | migraine | aura_only,
                        rule: delphi_untreated_4h | delphi_treated_30min | delphi_aura | null,
                        attenuation_flag: bool, active_pharmacology[],
                        confirmed_by: system | user,
                        review_status: unreviewed | timed_out | user_reviewed }
```

`rule` is nullable — an ordinary `none` day has no governing rule. Attenuation
is carried entirely by `attenuation_flag`, a boolean that can be true on any
class (including `none`); it is not a `rule` value and does not change `class`.
A day carrying twenty minutes of headache under an active medication window is
`class: headache` with `attenuation_flag: true` — the flag is orthogonal to
both `class` and `rule`, never a third reading of either.

Derived objects hang off a `derivations` parent row carrying `variant`,
`algorithm_version`, `config_hash` and `computed_at`, with
`UNIQUE (user_id, variant, algorithm_version, config_hash)`. This is what makes
every count scoped by construction and every report reproducible — competing
readings (e.g. `relapse_collapsed` and `relapse_split` for a sub-48-hour
recurrence) coexist as distinct derivations rather than colliding in one
unscoped table. Every count is scoped to a single `derivation_id`.

Day-count derivation (invariant 9's payer exports depend on this):

```
-- ALWAYS scoped to a single derivation.
headache_days = count(*) FILTER (WHERE class IN ('headache','migraine'))
migraine_days = count(*) FILTER (WHERE class IN ('migraine','aura_only'))
   FROM day_classifications WHERE derivation_id = $1 AND local_date BETWEEN $2 AND $3
```

An `aura_only` day counts as a migraine day regardless of whether headache was
present (Delphi rule 3). `class` is not mutually exclusive with these derived
counts — `migraine` days are also `headache` days for the purpose of
`headache_days`; do not count classes as a partition.

**Two different headache-day counts exist and must never be conflated.** The
Delphi classification count above (`headache_days` / `migraine_days`) is one
number; a payer's own criterion is a different one computed under different
rules — Cigna's Botox criterion requires headache **≥4 hours per day**, and
Delphi explicitly disclaims reimbursement contexts. **No API, tool or export
may emit an unlabelled `headache_days`** — every count carries the rule that
produced it.

Override entity — persisted separately from `boundary_rule`, not embedded in it.
`boundary_rule: user_override` is removed: an override is a fact, not a rule tag,
and embedding it in `boundary_rule` gives recomputation nowhere to preserve it.

```
override_id, overrides_type: episode_boundary | phase | day_classification
overrides_id, overriding_value, set_by: user_id, set_at
```

An episode's `overrides_id` — its natural key — is the id of its earliest
member observation, **resolved through any supersession chain back to the
original id**, not the id of whichever row is currently live. Recompute is
keyed to that resolved id so it survives both a boundary recompute (episode
start moves) and a correction to the first observation itself (the
observation's own id changes hands via `supersessions`). Resolving only one
hop, or resolving to the live row instead of walking the chain, silently
orphans every override on that episode the moment its first observation is
corrected.

Recomputation reads overrides as inputs and must never discard or silently
overwrite them. Recomputation must also **exclude** any observation with a
supersession row, and every derived object must record (via `computed_from`)
which specific observation version(s) it was computed from.

## Borrow this published spec

npj Digital Medicine 2025 published best practices for event-driven electronic
diaries, using migraine as the worked example. It is free and directly applicable:

- Events open and close with explicit start/end times.
- Events must be able to span midnight, with edit checks for plausible length.
- Only one migraine event is active at a time — close the previous before opening
  a new one. **This applies to user-facing capture only.** It does not extend to
  the derived layer: `RECURRENCE_AMBIGUOUS_WINDOW` requires relapse-collapsed and
  relapse-split readings to co-exist as competing candidate episodes. Do not build
  an interval-exclusive schema at the derived layer to satisfy this bullet — that
  is the incumbent design this project criticises, not the target one.
- A summary view shows which events remain open.
- End-of-day prompts capture an **active "no events" confirmation** rather than
  inferring from silence.

## Capture model

Event-anchored capture is the **primary** mechanism. Onset and resolution are
timestamped at or near the moment they are noticed, via the open/close
affordance above. Medication doses auto-trigger targeted follow-ups timed to
pharmacology — roughly 30–60 minutes and 2 hours after a triptan or NSAID, next
day for a corticosteroid taper — asking specifically about attenuation and
recurrence.

The daily checkpoint is a **backstop**, not the primary mechanism. Next morning
only. Two questions are unconditional regardless of branching: aura, and whether
the user fell asleep with a migraine and woke without one. On any day inside an
open episode, the checkpoint is capped short.

**Why, compactly:**
- The 12-hour, 48-hour and 30-minute clinical constants need sub-day timestamp
  resolution that a once-daily, date-level checkpoint cannot supply.
- A morning prompt is needed to capture the waking state and overnight
  resolution of an attack that began the previous evening — that observation is
  what `DAY_OCCUPANCY`'s one exception (Delphi rule 4: asleep with the attack
  ongoing, woke headache-free) needs to be applied at all, and it is what makes
  `DURATION_QUALIFICATION`'s across-midnight measurement possible. An evening
  checkpoint is asked before sleep and cannot see the waking state.
- Gating the aura question behind "did you have a headache today" silently drops
  days from the migraine-day count that feeds the CGRP payer threshold — aura
  counts as a migraine day regardless of head pain.
- An adaptive checkpoint that grows longer on worse days is heaviest exactly when
  capacity is lowest; detail moves into small real-time event asks instead.

## Confirmation gate

Unconfirmed inferences are force-surfaced for review before a
prior-authorisation packet is generated.

**The recall summary is exempt.** It must render in seconds, offline, one tap
from app open, for a photophobic patient in an emergency department. Unconfirmed
items are labelled visibly on the page instead — per-field provenance already
supports this without blocking the crisis path.

**The gate is not relaxed for the primary user.** He is the ground-truth
generator the correction corpus depends on; relaxing his gate removes friction at
the one source that produces the asset.

## Regulatory guardrails that constrain code

These are not legal advice — they are coding constraints.

- **Do not ship diagnostic assertions.** No "you have migraine," no differential,
  no headache-subtype suggestion.
- **Do not ship treatment recommendations.** No "take a triptan now," no "ask
  about CGRP."
- **Do not ship time-critical decision support** (e.g. real-time dosing guidance).
- **Do not ship attack-risk prediction or forecasting, and do not present trigger
  correlation as prediction.** No risk scores, no "your migraine risk today is
  elevated." This is a **product and trust choice, not regulatory arbitrage** —
  FDA's January 2026 revision removed the risk-score language from the CDS
  criteria, so competitors shipping prediction are not obviously offside; we
  simply choose not to. See the M-sense RCT note under Working conventions for
  why trigger-correlation claims specifically need care.
- **The prodrome pattern surface is deferred entirely — no forward-looking
  alert and no passive pattern display, in any form.** The no-prediction
  guardrail above is unchanged and unamended; this is a separate, narrower scope
  decision. Retrospective phase labelling is still built (build step 5) and will
  generate the data to revisit this with evidence rather than argument.
- **Do not ship red-flag secondary-headache detection with diagnostic framing.**
  Surfacing that a clinical distinction exists is acceptable; deciding it is not.
- **Any LLM feature that *interprets* rather than *summarises* requires legal
  review before shipping.** Summarising a conversation into a structured
  ICHD-3-formatted record is very likely fine. Generating clinical impressions is
  the highest-uncertainty area in the project.
- **Do not embed PROMIS, HIT-6, MIDAS, MSQ or MPFID item wording in anything that
  will be distributed or sold.** Commercial use requires permission — PROMIS
  included, despite common belief: PROMIS is free only for individual research or
  individual clinical use, and electronic administration requires a paid
  HealthMeasures Electronic Administration Permission (HEAP). MIDAS is a
  different, narrower case — its current commercial rights-holder is
  **unconfirmed**, which is why it needs legal confirmation before use rather than
  an assumed prohibition. See `Research/clinical-regulatory.md`.
- **Consent architecture:** Washington's My Health My Data Act requires separate
  opt-in for *collection* and separate authorisation for *sharing*, plus a
  distinct consumer health data privacy policy, and is enforceable by private
  action. Build the consent model for this from the start — retrofitting is
  painful. Sharing includes routing data through a third-party AI assistant.
- **The assistant connector is a sharing event.** It needs its own consent flow
  and disclosure, not blanket coverage under general terms. **Anthropic
  documents that MCP connectors are not zero-data-retention-eligible and are
  retained under standard policy** — that half is documented, not unknown.
  **OpenAI's position on data passed through third-party MCP tool calls is the
  part that remains undocumented.** Disclose the difference accurately rather
  than treating both as unknown, and confirm before exposing the connector to
  anyone but the developer.

## Anti-patterns — specific mistakes to avoid in this codebase

- Nesting symptoms under an attack record as child fields.
- Treating "migraine status" as a container rather than a derived property.
- A boolean symptom field.
- Asking phase, or any clinically retrospective question, at capture time.
- Silently choosing one interpretation where ICHD-3 is ambiguous.
- Attributing a multi-day attack only to its onset date — it under-counts every
  day after the first, and the days lost are exactly the ones that reach a
  payer threshold.
- Measuring attack duration per calendar date rather than per attack — it
  under-counts nocturnal onset.
- Silently annotating a pharmacologically attenuated day rather than surfacing
  it — the anti-pattern is the silence, not the exclusion itself. An
  attenuated `none` day correctly does not count toward headache or migraine
  days, but every range export must enumerate attenuated days within it.
- Excluding late-backfilled days from a day count — no published rule supports a
  backfill-latency exclusion, and the harm lands hardest on exactly the patient
  the product exists for.
- Inventing an episode-boundary or migraine-day rule instead of citing a published
  one.
- Exporting attack counts to a payer-facing document.
- Mutating an observation in place.
- Any mid-attack flow with more than one confirmation beat — write the record,
  confirm in a few words, ask nothing.
- Building the correction UI as a settings screen rather than an inline affordance.
- An evening or end-of-day checkpoint — it cannot observe the waking state and
  overnight resolution a morning prompt needs, both to measure attack duration
  across midnight and to detect the one occupancy exception (Delphi rule 4).
- Gating the aura question behind a headache question.
- Any blocking confirmation flow on the recall/ED path.
- An unscoped day count — always parameterise by a single derivation.
- Emitting a day count without the rule that produced it.

## Anchor on attenuation — get the framing right

No app or paper models the possibility that a symptom-free window is
pharmacologically attenuated rather than genuinely clear. This is genuine
white space — but the framing matters. Corticosteroid evidence (pooled:
dexamethasone reduced 24 h recurrence by median 30% ARR, NNT 3.3; 72 h recurrence
by 11% ARR, NNT 9.1; and recurrent headaches after steroid treatment were *milder*)
shows steroids **reduce** recurrence rather than merely deferring it. Model this as
**attenuated symptom expression under active pharmacology** — a confounder on
severity and observability — **not** as a suppressed-then-restored attack. A
"suppress then resume unchanged" model will not survive clinical review.

A day is flagged attenuated only when it falls inside a medication window the
**user recorded** — a `course_start`/`course_end` on a medication observation.
No half-life model is inferred; inventing per-agent pharmacokinetic windows
would be the invented-clinical-rule anti-pattern. Two properties to hold
plainly: the flag applies to **any** day inside the window, not only days
classifying `none`; and an attenuated `none` day **does not count** toward
headache or migraine days — the flag is annotation, made non-silent by every
range export enumerating attenuated days within the range.

## Platform notes

- The repository is scaffolded as a Capacitor project from commit one, wrapping
  the web app — there is no PWA-only period. The web app remains the core
  codebase; Capacitor is a wrapper, not a rewrite. This gives HealthKit, Health
  Connect, background audio capture, native push, App Store presence, iOS App
  Intents and Android AppFunctions from the start. For personal use, no App Store
  review is needed to install on your own device.
- Stack is TypeScript end to end: React with Vite for the app (D14), Node and
  TypeScript for the API and the MCP server, Postgres for storage, deployed on a
  Fly.io with managed Postgres 16 (D14). Postgres
  suits an append-only observation log with derived projections well — JSONB for
  observation bodies, normal relational structure for the derived layer.
- Safari's LRU storage eviction makes **server-side sync mandatory, not
  optional**, regardless of platform.
- **One remote MCP server**, built against `@modelcontextprotocol/sdk` 1.30.0,
  which implements the protocol revision the 2026-07-28 rewrite superseded — a
  deliberate compatibility choice, not a claim of conformance to 2026-07-28,
  because real clients still speak 1.x. OAuth 2.1 + PKCE, RFC 8707 resource
  indicators, exposing log and query tools over the same backend the app uses.
  Reuse those tokens for the iOS Shortcuts path — do not build two auth systems.
- iOS Shortcuts: "Ask for Input" becomes a spoken prompt when invoked via Siri.
  Shortcuts cannot split one sentence into multiple parameters — irrelevant,
  because the backend extracts. Capture the whole utterance as one string and
  POST it.
- Offline queue that syncs on reconnect. Attacks do not wait for wifi.
- See `Research/assistant-integration.md` for the full platform matrix.

## Repository layout

**This is binding, not a suggestion** — FR-0.3 and FR-0.4 hardcode `packages/clinical-config` by path, so a different layout fails step 0.

```
headstate/
├── apps/
│   ├── app/              # React + Vite; dist/ is the Capacitor webDir
│   └── api/              # Node API + MCP server
├── packages/
│   ├── derivation/       # shared clinical engine — imported by both call sites
│   ├── clinical-config/  # versioned constants, content-hashed, citation-linted
│   └── schema/           # shared types, Zod schemas
├── ios/  android/        # generated by `cap add`, committed
├── docs/                 # architecture decision records
├── CLAUDE.md  decision-log.md  tech-spec.md  frd.md  README.md
└── Research/
```

Bundle identifier is **`health.headstate.app`**, locked (D14). Framework is **React + Vite** (D14). Managed Postgres is **Fly.io, PostgreSQL 16** (D14), matching the 16.13 the DDL was verified against.

## Build order

Capacitor is repository scaffolding, not a numbered step. Steps 1–7 are
otherwise sequenced for the developer as the primary user, not for a market
launch.

| # | Step | Notes |
|---|---|---|
| 0 | Capacitor scaffold, repo setup, CI | TypeScript end to end, Postgres, managed platform |
| 1 | Capture and the observation store | Immutable log, schema with the third review state, event-anchored capture with the next-morning backstop |
| 2 | Recall | One-page summary. Printable, shareable, offline. Exempt from the confirmation gate |
| 3 | Derived episodes and day classification | ICHD-3 constants, Delphi rules, dual readings, override. Backfill always counts |
| 4 | Integrations | HealthKit and Health Connect for sleep, weather |
| 5 | Phase annotation and retrospective relabelling | Includes medication-response inference. Also the point at which the prodrome-surface deferral can be revisited with data |
| 6 | Correlation | A research bet, not a claim — see the M-sense trial note above |
| 7 | Prior-authorisation packet | Gated by the mandatory confirmation gate |

See `decision-log.md` for the reasoning behind this order.

## Working conventions for agents on this project

- **Do not assert a clinical fact without a citation traceable to a document in
  `Research/`.** If it is not there, mark it unverified rather than asserting it.
- Distinguish verified from inferred in anything you write. The project has
  already been burned once by a confidence section that listed unverified claims
  as well established.
- Several research findings are explicitly marked UNVERIFIABLE (not absent). Do
  not upgrade them by restating them as fact.
- Write current-state documentation. Do not add "previously we thought" sections
  or version-comparison narratives.
- When in doubt between a null and a guess, choose the null.
- **Trigger-correlation features are a research bet, not a claim.** The largest
  RCT of trigger analytics layered on a diary (M-sense Migräne, n=477, 12 weeks)
  compared the full-feature app against a control app containing only a headache
  and medication diary: migraine days/month fell 7.74→5.79 in the full-feature arm
  versus 7.76→5.41 in the diary-only control — a 0.38-day difference that was not
  statistically significant. Investigators concluded self-monitoring itself drove
  the improvement in both arms. Do not write copy that asserts a clinical benefit
  for correlation or trigger-identification features; treat them as unproven
  pending stronger evidence.

## Perishable facts — re-check before relying on these

Some facts referenced in this file have a shelf life and must be re-verified
before being relied on for a shipping or clinical decision:

- The MCP SDK line — `@modelcontextprotocol/sdk` 1.30.0 implements the revision
  superseded by 2026-07-28; re-check at build time whether real clients have
  moved onto the 2026-07-28 line before it does.
- CPT codes referenced anywhere in the payer/reimbursement material are effective
  1 January 2026 — confirm against the current year's code set before use.
- ICHD-4 is in development and MOH's survival as a distinct diagnosis is under
  debate — the MOH constants above are ICHD-3 and may not carry forward
  unchanged.
- The ChatGPT Health rollout and its submission paths are moving targets —
  reconfirm before building or documenting against them.
- The Delphi 2026 paper has an outstanding published Correction that has not yet
  been retrieved or reviewed — reconfirm the `DELPHI_*` constants once it is.
- Version numbers cited for named packages carry an as-of date, not a
  verification marker — package ecosystems move faster than this document.

## File index

- `tech-spec.md` — **the build document.** Schema, derivation engine, capture
  pipeline, MCP server, native surface, sync, privacy. Scoped to steps 0–4. All
  SQL executed against PostgreSQL 16.13.
- `frd.md` — **the acceptance criteria.** Given/When/Then, scoped to steps 0–4,
  written to be executable as tests rather than read as prose.
- `design-system.md` — visual identity extracted from the screen mockups: color
  and type tokens, the three-world theming model (chrome / device / doc), and
  the component vocabulary a future Storybook library would build against.
  Covers three screens; records which remaining screens are ready to mock and
  which are blocked on an undecided interaction pattern.
- `storybook/` — a standalone Storybook/React/Vite/TypeScript project,
  self-contained with its own `package.json`, deliberately **not** part of
  the monorepo layout below and not connected to `app/` (see next) in any
  way — no shared workspace, no shared `node_modules`. It exists to verify
  `design-system.md`'s ~30-component vocabulary as real, working components
  before the real app is built, not to become that app. `storybook/styleguide.html`
  is the canonical, component-by-component design-system reference: every
  token and every named component/state from `design-system.md` §4, in one
  self-contained HTML gallery. Moved out of `mockups/` once it stopped being
  one mockup among ten and became the page the Storybook build is verified
  against directly — keep it current with §4, a new component pattern gets a
  gallery cell added here in the same pass. `storybook/storybook-plan.md` is
  the phased plan for that build.
- `app/` — reserved at the repo root for the real application (React + Vite
  per D14) whenever real build-step-0/1 work starts. Currently empty;
  `storybook/` does not populate it and is not a preview of its structure.
- `mockups/` — the actual mockup HTML files `design-system.md` was extracted
  from, self-contained and openable directly in a browser. `mockups/templates/`
  and `mockups/fonts/` hold the reusable pre-font-splice source and the
  already-encoded font files a new mockup should build from.
- `decision-log.md` — the binding record of settled decisions; wins over this
  file and over research documents where they disagree.
- `README.md` — project orientation and status.
- `headstate-strategy-brief.md` — current strategy, defensible position, data
  model, build order, risks.
- `HeadState-Clinical-Review.docx` — document prepared for a neurologist to
  pressure-test the clinical assumptions.
- `Research/continuous-timeline-verification.md` — **read this first.** What is
  and is not novel, per-app capability matrix, ICHD-3 counting rules, premonitory
  and corticosteroid evidence, prior-auth counting.
- `Research/clinical-regulatory.md` — ICHD-3 fields, MOH thresholds, instrument
  licensing, FDA posture, privacy regimes, payer criteria, reimbursement.
- `Research/qa-verification-report.md` — independent adversarial QA: 31 claims
  checked, 6 corrections, audit trail.
- `Research/assistant-integration.md` — MCP, Claude/ChatGPT/Gemini surfaces, Siri
  Shortcuts, auth, PWA vs native.
- `Research/competitive-landscape.md` — incumbents, the 2026 AI wave, adjacent and
  clinical players, gap analysis.
- `Research/product-architecture.md` — product principles, information
  architecture, core user flows, conversational extraction discipline.
- `Research/migraine-buddy-features.md` — full feature inventory of the incumbent.
- `Research/tech-postgres-temporal-model.md` — **largely superseded.** The
  schema-design exploration `tech-spec.md` was built from. Its DDL is **not**
  current: it puts `superseded_by` back as a column, uses `config_version`
  instead of `config_hash`, fixes the recompute window at ±72h, and lists
  `attenuated` as a rule value — all four reversed in the specification. Read it
  for reasoning, never for schema.
- `Research/open-question-recommendations.md` — the researched recommendations
  behind the granularity, recall-format, prodrome and automation-level decisions.
