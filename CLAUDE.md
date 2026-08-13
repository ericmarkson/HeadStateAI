# CLAUDE.md — HeadState

## What HeadState is

HeadState (headstate.health) is a migraine and symptom record where **individual
timestamped observations are the atomic unit**, and attacks, phases, episodes and
day-classifications are **derived, versioned, recomputable projections** over the
observation stream. It is built by Eric, a migraine patient, for his own use first;
commercialisation is second. It has three jobs: **capture** (frictionless logging),
**recall** (on-demand summaries for a clinical encounter or ER visit), and
**documents** (structured output supporting prior authorisation). It ships as a PWA
with a Capacitor native wrapper, plus a remote MCP server so external assistants
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

## Non-negotiable invariants

These are hard rules. Violating any of them is a bug, not a style issue.

1. **Observations are immutable and append-only.** Corrections never delete or
   mutate an observation — they append a new observation and set the old one's
   `superseded_by`. The verbatim user utterance is always retained.
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
| `DELPHI_UNTREATED_DAY_MIN` | Compound predicate, not a scalar — do not implement as `if duration >= 4h then migraine_day`: duration ≥4 hours **AND** (≥2 of 4 headache characteristics **OR** (≥1 characteristic **AND** ≥1 associated symptom)) (83% consensus) | Delphi 2026 |
| `DELPHI_AURA_DAY` | aura meeting ICHD-3 criteria counts regardless of whether headache is present (83% consensus) | Delphi 2026 |
| `DELPHI_OVERNIGHT` | sleep with migraine, wake headache-free → one migraine day (100% consensus; deliberately departs from ICHD-3, which reckons duration until awakening) | Delphi 2026 |
| `ACUTE_MEDICATION_DAY` | a calendar day on which any acute/abortive agent was taken, counted per `drug_class`, with the day attributed to each class taken that day. This is the input to every MOH threshold below. | derived from ICHD-3 8.2 |
| `DRUG_CLASS` (taxonomy) | `triptan \| ergot \| opioid \| combination_analgesic \| non_opioid_analgesic` — required field on every medication observation; referenced by every MOH constant | ICHD-3 8.2 |
| `MOH_GENERAL_CRITERION` | headache on ≥15 days/month in a patient with a pre-existing headache disorder, plus regular overuse of one or more acute/symptomatic drugs for >3 months. This is the gating criterion — the per-class and combined-class constants below refine it, they do not replace it. Do not fire an MOH flag on drug-days alone without also checking this. | ICHD-3 8.2 |
| `MOH_PER_CLASS` | map, not a scalar: `triptan → 10 days/month`, `ergot → 10 days/month`, `opioid → 10 days/month`, `combination_analgesic → 10 days/month`, `non_opioid_analgesic → 15 days/month`, each sustained over >3 months | ICHD-3 8.2 |
| `MOH_COMBINED_CLASS` | ≥10 days/month total across any combination with no single class overused. **A per-class counter alone under-detects: 6 triptan days + 6 NSAID days is MOH and a per-class counter reports clear. Implement both.** | ICHD-3 8.2.6 |
| `PAYER_CGRP_EPISODIC` | ≥4 migraine days/month, plus documented failure of two preventive drug classes | payer criteria — see `Research/clinical-regulatory.md` |
| `PAYER_CHRONIC_BOTOX` | ≥15 headache days/month with ≥8 migraine days among them (Cigna wording: "headache lasting 4 hours per day or longer"), plus failure of ≥3 prophylactics drawn from ≥2 classes | payer criteria — see `Research/clinical-regulatory.md` |
| `PAYER_BOTOX_CONTINUATION` | ≥50% reduction in monthly migraine frequency from baseline | payer criteria — see `Research/clinical-regulatory.md` |

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
    non_opioid_analgesic (see clinical constants table), plus perceived_relief.
source: user_voice | user_text | user_tap | passive_feed | imported
provenance: verbatim_span? | feed_name?
confidence: float
superseded_by: observation_id?
```

Derived layer — recomputable, versioned, overridable. All three derived object
types use the same `confirmed_by: system | user` field — there is one shape for
"the algorithm proposed this / a human confirmed it," not three:

```
episode_id, algorithm_version, computed_at, confirmed_by: system | user
computed_from: observation_id[]   # exact observation versions used this run
member_observations[]
boundary_rule: icd_48h | status_migrainosus_12h | med_response_inference
open: bool
phase_annotations[]{ observation_id, phase, confidence, assigned_at,
                      confirmed_by: system | user }
day_classifications[]{ date, class: none | headache | migraine | aura_only,
                        rule: delphi_untreated_4h | delphi_treated_30min | delphi_aura | attenuated,
                        attenuation_flag: bool, active_pharmacology[],
                        confirmed_by: system | user }
```

Day-count derivation (invariant 9's payer exports depend on this):

```
headache_days = count(date where class ∈ {headache, migraine})
migraine_days = count(date where class ∈ {migraine, aura_only})
```

An `aura_only` day counts as a migraine day regardless of whether headache was
present (Delphi rule 3). `class` is not mutually exclusive with these derived
counts — `migraine` days are also `headache` days for the purpose of
`headache_days`; do not count classes as a partition.

Override entity — persisted separately from `boundary_rule`, not embedded in it.
`boundary_rule: user_override` is removed: an override is a fact, not a rule tag,
and embedding it in `boundary_rule` gives recomputation nowhere to preserve it.

```
override_id, overrides_type: episode_boundary | phase | day_classification
overrides_id, overriding_value, set_by: user_id, set_at
```

Recomputation reads overrides as inputs and must never discard or silently
overwrite them. Recomputation must also **exclude** any observation with
`superseded_by` set, and every derived object must record (via `computed_from`)
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
  and disclosure, not blanket coverage under general terms. Whether Anthropic or
  OpenAI retain health data passed through MCP tool calls is undocumented —
  confirm before exposing the connector to anyone but the developer.

## Anti-patterns — specific mistakes to avoid in this codebase

- Nesting symptoms under an attack record as child fields.
- Treating "migraine status" as a container rather than a derived property.
- A boolean symptom field.
- Asking phase, or any clinically retrospective question, at capture time.
- Silently choosing one interpretation where ICHD-3 is ambiguous.
- Silently dropping pharmacologically attenuated days from a day count.
- Inventing an episode-boundary or migraine-day rule instead of citing a published
  one.
- Exporting attack counts to a payer-facing document.
- Mutating an observation in place.
- Any mid-attack flow with more than one confirmation beat — write the record,
  confirm in a few words, ask nothing.
- Building the correction UI as a settings screen rather than an inline affordance.

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

## Platform notes

- Core codebase is a PWA. **Apple HealthKit and Android Health Connect are
  native-only, no exceptions** — a PWA cannot read either. Nor can a PWA do
  background audio capture, and Safari LRU storage eviction makes **server-side
  sync mandatory, not optional**.
- Capacitor wrapper (not a rewrite) unlocks HealthKit, Health Connect, background
  audio, native push, App Store presence, iOS App Intents and Android
  AppFunctions. For personal use, no App Store review is needed to install on your
  own device.
- **One remote MCP server** with OAuth 2.1 + PKCE (MCP spec 2026-07-28: streamable
  HTTP, RFC 8707 resource indicators) exposing log and query tools over the same
  backend the PWA uses. Reuse those tokens for the iOS Shortcuts path — do not
  build two auth systems.
- iOS Shortcuts: "Ask for Input" becomes a spoken prompt when invoked via Siri.
  Shortcuts cannot split one sentence into multiple parameters — irrelevant,
  because the backend extracts. Capture the whole utterance as one string and
  POST it.
- Offline queue that syncs on reconnect. Attacks do not wait for wifi.
- See `Research/assistant-integration.md` for the full platform matrix.

## Repository layout

This layout is a suggestion — follow it unless the repository already differs.
Root holds strategy deliverables and, going forward, source. `Research/` holds
research inputs and verification records. Suggested additions: `src/` for source
code, `docs/` for architecture decision records, and the clinical constants kept
in a single versioned config module (e.g. `src/clinical/constants.*`) with a test
suite asserting each constant against its cited source.

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

- The MCP spec revision cited for OAuth 2.1 + PKCE (2026-07-28) — confirm no
  newer revision has superseded it.
- CPT codes referenced anywhere in the payer/reimbursement material are effective
  1 January 2026 — confirm against the current year's code set before use.
- ICHD-4 is in development and MOH's survival as a distinct diagnosis is under
  debate — the MOH constants above are ICHD-3 and may not carry forward
  unchanged.
- The ChatGPT Health rollout and its submission paths are moving targets —
  reconfirm before building or documenting against them.
- The Delphi 2026 paper has an outstanding published Correction that has not yet
  been retrieved or reviewed — reconfirm the `DELPHI_*` constants once it is.

## File index

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
