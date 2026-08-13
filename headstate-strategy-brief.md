# HeadState — Strategy & Build Brief

**headstate.health** · Continuous-timeline migraine tracking and clinical reporting
Current as of 13 August 2026

**Direction:** Build for Eric first — a tool he needs and uses. Commercialize second. Patient-first DTC. Three jobs: **capture**, **recall**, **documents**.

---

## 1. What HeadState Is

A migraine record where **individual observations are the atomic unit, and attacks, phases and episodes are derived from them** — computed, versioned, and recomputable as new information arrives.

This exists because migraine is continuous and its structure is only knowable in retrospect. A three-week episode with medication-induced gaps is one thing, not several. Nausea on Wednesday is data whether or not there's head pain, and whether it was premonitory is only determinable after Thursday. A drug that produces sixty hours of quiet followed by recurrence tells you the episode didn't end. None of that can be represented by a schema whose primary key is "an attack."

On top of that record sit three deliverables: frictionless capture, on-demand recall for clinical encounters, and structured documents for prior authorization.

---

## 2. The Defensible Position

Ranked by how durable each actually is. The algorithm is the engine, not the fence.

### 2.1 The correction corpus — the only asset that compounds
Every time a user overrides an episode boundary, confirms that Wednesday's nausea was premonitory, or rejects a proposed phase label, that is ground truth on migraine structure that exists nowhere else. The algorithm is the mechanism; **the accumulated human corrections are the moat.** It can't be copied, it gets more valuable the longer it runs, and it's the only advantage here that survives a well-funded competitor deciding to enter.

Design consequence: corrections must be captured as **first-class, persisted, attributable facts** — not as silent edits to a row. Every derived object needs to know whether it was computed or human-confirmed, and by whom, and when.

### 2.2 The reporting platform — the part with a buyer
The algorithm produces numbers. The report is what gets a prior authorization approved, what an ER physician actually reads, and what a neurologist trusts enough to change a prescription. Numbers have a spec sheet; reports have a purchaser. This is the larger business and should be treated as a product in its own right, not as an export feature.

### 2.3 Correct by default, not correct if you find the setting
Competitors do have relevant capabilities buried in them — Migraine Buddy's chronic mode can hold an open monthly record; Migraine Insight supports open-ended episodes. But a feature that requires reading a help-center article is functionally not a feature, particularly in an app opened by someone photophobic and cognitively impaired. The population that most needs continuous modeling — chronic, refractory, status migrainosus — is least likely to go looking for a hidden mode in month two of a bad stretch.

**The claim is not "we can do what they can't." It's "the clinically correct model is the default."** Defaults are a legitimate strategy.

### 2.4 Medication response as a phenotyping signal
The genuinely novel inference. Treatment response is normally recorded as an *outcome*. Here it's used as *evidence about structure*: a triptan working implies you were in the headache phase, not prodrome. A steroid producing a multi-day remission followed by recurrence implies one continuous episode rather than two. **Whether a drug helped is diagnostic information about what was happening and where the boundaries fall.**

No implementation of this was found. One caveat before it becomes a pitch line: *"triptan response as diagnostically informative"* has prior art in the clinical literature. What appears unclaimed is using it **computationally, per episode, to resolve boundaries and assign phases.** Confirm that distinction holds before leaning on it.

### 2.5 Retrospective phase linkage
No app for which a data model could be documented links an earlier logged observation to a phase of a later attack — 8 of 21 apps surveyed are unverifiable on this rather than confirmed absent. Migraine Buddy has prodrome and postdrome as *attack types* — sibling records with no linkage to a subsequent attack and no phase graph — and constrains records not to overlap, confirming an interval-exclusive schema. It does allow retroactively changing a record's type, but not attaching it to a later attack.

The evidence that this matters is strong. Across 229 attacks in 34 patients: 69% had premonitory symptoms, but **only 27.5% were recognized before pain onset.** When patients did predict, positive predictive value was **85.1%** with a mean lead time of **10 hours 38 minutes**. Most predictive symptoms were photophobia, drowsiness, yawning, increased thirst, blurred vision and nausea.

**Roughly 72% of premonitory data is only recoverable retrospectively.** That is the entire argument for letting Thursday's headache reach back and relabel Wednesday's nausea — and the reason phase should never be asked at capture time.

Corollary: any prodrome *alert* must be low-recall and high-confidence. When it fires it's probably right; it will stay silent for most attacks. Never present it as coverage.

### 2.6 Attenuation-aware day classification
No app for which a data model could be documented — and no reviewed paper — models the possibility that a symptom-free window is pharmacologically attenuated rather than genuinely clear; 5 of 21 apps surveyed are unverifiable on this rather than confirmed absent. Real white space, with one important constraint on how it's framed.

From a 65-year systematic review of corticosteroids in migraine and status migrainosus: dexamethasone reduced 24-hour recurrence by a median **30% ARR (NNT 3.3)** and 72-hour recurrence by **11% ARR (NNT 9.1)**, and *"recurrent headaches become milder than pretreated headaches."* **Steroids reduce recurrence rather than merely deferring it, and post-steroid recurrences are typically milder.** Recurrence still happens in a large minority, but a "suppress, then restore unchanged" model won't survive clinical review.

Model this as **attenuated symptom expression under active pharmacology** — a confounder on severity and observability — not as a suppressed-then-restored attack.

### 2.7 The standards gap — the strongest argument, and it isn't about competitors
ICHD-3 supplies three thresholds and deliberately declines to resolve a fourth:

| Threshold | Rule |
|---|---|
| **12 hours** | Maximum remission tolerated *within* one status migrainosus episode: "Remissions of up to 12 hours due to medication or sleep are accepted" |
| **48 hours headache-free** | Separates two distinct attacks: "count as distinct attacks only those that are separated by at least 48 hours headache-free" |
| **Under 48 hours** | **Deliberately ambiguous** — recurrence "may represent a relapse of the same attack or a new attack." ICHD-3 stops there |
| Overnight | ICHD-3 reckons duration "until the time of awakening." The 2026 Delphi consensus departs from this at 100% agreement: sleep with migraine, wake headache-free = one migraine day |

**ICHD-3 does not resolve the sub-48-hour recurrence question, and the 2026 Delphi consensus — the most authoritative current statement on counting migraine days — is silent on multi-day continuity, status migrainosus, midnight-spanning attacks, and same-day recurrence after treatment.**

A system that stores raw observations and can **recompute both readings** — relapse-collapsed and relapse-split, each labeled with the rule that produced it — isn't inventing a house rule. It's implementing a standards-compliant answer to a question the standards leave open. That is arguable in front of a neurologist in a way that "our timeline is nicer" is not.

### 2.8 The execution gap
The continuous model is standard in academic headache research, not in products. The **LUMC/Leiden e-headache diary** (validated in *Cephalalgia* 2025) already does daily entry regardless of headache status, with "an automatic algorithm [that] processed all data entries and classified each day either as a non-headache day, a headache day or a migraine day." ***npj Digital Medicine* 2025** published best practices for event-driven electronic diaries using migraine as the worked example. The Delphi rules were explicitly written for "automated e-diary labeling in digital headache calendars."

**Nobody has shipped the clinical-trial-grade model to a patient who needs it, with conversational capture in front and assistant-based recall on top.** The specification is public; the product doesn't exist. That's an execution position rather than an architectural one, and it compounds with §2.4–2.6, which nobody has at all.

---

## 3. Design Principle: The Algorithm Proposes, The Human Confirms

Every derived claim carries whether it was **computed** or **human-confirmed**, and the report shows which is which.

This one decision does three jobs at once:

- **Correctness.** You will disagree with the boundary algorithm sometimes. Override has to be a persisted first-class fact, not a patch.
- **Regulatory.** Day-counting arithmetic against published rules is safe. An algorithm *asserting* "you were in prodrome" or "this was one episode" drifts toward diagnostic interpretation, which is the FDA line. Proposals that a human confirms, with the distinction visible in the output, stays on the right side of it.
- **Credibility.** A clinician trusts a document more, not less, when it distinguishes patient-confirmed from machine-inferred. Provenance is a feature in health, not a disclaimer.

And it is the mechanism that generates §2.1.

---

## 4. The Data Model

### Observation layer — immutable, append-only
```
observation_id, user_id
observed_at (timestamp, timezone)
recorded_at (timestamp)
kind: symptom | pain | medication | medication_response | sleep |
      food | activity | stress | cycle | environment | care_event | note
body: { type, value, laterality?, region?, quality?, severity?, dose?, route?, drug_class? }
source: user_voice | user_text | user_tap | passive_feed | imported
provenance: verbatim_span? | feed_name?
confidence: float
superseded_by: observation_id?          # corrections never delete
```

### Derived layer — recomputable, versioned, overridable
```
episode_id, algorithm_version, computed_at
member_observations[]
boundary_rule: icd_48h | status_migrainosus_12h | med_response_inference |
               user_override
open: bool
phase_annotations[]{ observation_id, phase, confidence,
                     assigned_at, assigned_by: system | user }
day_classifications[]{ date, class: none | headache | migraine | aura_only,
                       rule: delphi_untreated_4h | delphi_treated_30min |
                             delphi_aura | attenuated,
                       attenuation_flag: bool, active_pharmacology[],
                       confirmed_by: system | user }
```

Derivation rules, since `class` is not mutually exclusive by count:
- `headache_days = class ∈ {headache, migraine}`
- `migraine_days = class ∈ {migraine, aura_only}`

An aura day counts as a migraine day regardless of whether headache was present (Delphi rule 3) — that's why `aura_only` is a distinct class rather than a variant of `headache`. A naive `count(class=='headache')` under-counts by every migraine day.

### Five properties that follow
1. **Symptoms exist without a parent attack.** An observation is valid on its own.
2. **Phases are retrospective annotations, never capture-time questions.** A new headache triggers a recompute that *offers* to relabel earlier observations. Never ask "is this prodrome?" while logging — the user only knows 27.5% of the time.
3. **Two clocks.** `observed_at` and `recorded_at`, plus `algorithm_version` on every derived object. Without this, a report generated in March isn't reproducible in June after reinterpretation — and for anything going to a payer, reproducibility isn't optional.
4. **Episodes are opinionated but overridable**, with overrides surviving recomputation.
5. **Dual readings are a feature.** For sub-48-hour recurrence, compute and display both counts with their governing rule.

### Named constants, sourced not invented
`STATUS_MIGRAINOSUS_MAX_REMISSION = 12h` · `STATUS_MIGRAINOSUS_MIN_DURATION = >72h unremitting and debilitating (ICHD-3 1.4.1)` · `DISTINCT_ATTACK_SEPARATION = 48h headache-free` · `RECURRENCE_AMBIGUOUS_WINDOW = <48h → compute both` · `DELPHI_TREATED_DAY_MIN = 30min` · `DELPHI_UNTREATED_DAY_MIN = 4h with ≥2/4 characteristics, or ≥1/4 plus one associated symptom` · `DELPHI_AURA_DAY = aura regardless of headache` · `DELPHI_OVERNIGHT = one migraine day` · `MOH_GENERAL_CRITERION = headache ≥15 days/month in a patient with a pre-existing headache disorder, over >3 months (ICHD-3 8.2)`

`MOH_GENERAL_CRITERION` is what per-class and combined-class MOH counting sit inside — without it, MOH detection would fire on drug-days alone rather than on drug-days within a qualifying headache pattern.

Version these as config. **ICHD-4 is in active development** and MOH's survival as a diagnosis is under debate.

### Borrow the npj event-diary specification
Free, published, and directly applicable: events opened and closed with explicit start/end times; events must be able to span midnight; **only one migraine event active at a time** (close the previous before opening a new one); a summary view showing which events remain **open**; and end-of-day prompts capturing an **active "no events" confirmation** rather than inferring from silence.

That last point resolves the "does silence mean fine?" problem — the trial-grade approach is a cheap explicit negative, not an inference.

### Keep the free-text escape hatch
Structure serves the prior auth. **Free text serves the questions you didn't model.** Retain the verbatim utterance on every observation and support full-text search and free-form query over the raw stream. A structured app answers only what was anticipated; that flexibility is the one real advantage of a NotebookLM-style setup and it's worth not losing.

---

## 5. The Three Jobs

### Capture — frictionless or nothing else matters
One utterance to structured observations, with a visible receipt and one-tap correction. No wizard. Mid-attack: write the record, confirm in four words, **ask nothing**. Re-engage after resolution for richer detail. Text must be genuinely first-class — voice is unusable on transit, in shared offices, and around family.

### Recall — the ER and appointment brief
A **query** workload, not an export, and the thing that pays back in week one.

One page, generated on demand:
- **Where you are now:** "Day 12 of a continuous episode. Two prior ER visits this episode."
- **What's on board:** every medication and dose in the last 72 hours with times — the most consequential fact in an ER and the hardest to recall while symptomatic.
- **Baseline pattern:** typical duration, typical presentation, aura status, known red flags.
- **Current preventives** and recent acute-medication day count.
- **What was tried this episode and what happened** — including temporary remission followed by recurrence.

Two hard constraints. **An ER physician will not browse an app** — they'll take a printed page, a link, or a verbal summary. And emergency medicine defaults to treating each visit as a fresh acute presentation, which is the same discrete-versus-continuous failure, relocated into the clinical encounter. Arriving at visit three with "day 12, two prior visits, here's the drug history and the recurrence curve" is a materially different conversation.

Requirements: works offline, one tap from the lock screen, printable, shareable with someone who isn't the patient.

### Documents — the prior-authorization packet
**Payer exports default to DAY COUNTS, never attack counts.**

Cigna's Botox criteria, verbatim: *"greater than or equal to 15 headache days per month with headache lasting 4 hours per day or longer."* CGRP criteria are also day-based. **No payer policy reviewed contains any provision for status migrainosus, refractory migraine, or continuous headache.**

A 21-day episode with ≥4h of headache daily is **21 headache days** and clears the ≥15 threshold outright. Reported as one attack — or three — it fails every threshold in every policy checked. Attack counts are clinically interesting and reimbursement-toxic.

**These are two different counts, computed under two different rules, and the export must report both separately rather than substitute one for the other.** The 2026 Delphi consensus defines a **migraine day** for clinical trials, real-world practice and automated e-diary labelling — and it explicitly disclaims regulatory, reimbursement and prior-authorisation contexts. Cigna's Botox criterion is a **headache day**, defined independently as "headache lasting 4 hours per day or longer," with no reference to Delphi at all. HeadState must compute and label each: a Delphi-rule migraine-day count and a payer-definition headache-day count, each tagged with the rule that produced it.

Under the Delphi rule, a treated day needs only **≥30 minutes** of headache regardless of drug class or characteristics, so medication-induced gaps mostly don't erase migraine days. The payer's own headache-day rule carries no such floor. Under either rule, the days at genuine risk are those *entirely* symptom-free due to medication, and neither rule recovers them on its own. **That is precisely the value of §2.6 — attenuation modeling is what stops drug-suppressed days from silently deleting themselves from either count.**

Also required: MOH counters, per-class **and** combined-class. Per-class thresholds are ≥10 days/month for triptans, ergots, opioids and combination analgesics, ≥15 for non-opioid analgesics. But **ICHD-3 8.2.6** covers any combination totaling **≥10 days/month** with no single class overused — so 6 triptan days plus 6 NSAID days is MOH while a per-class counter reports clear. Track both.

Show the work: a payer or clinician should be able to see why day 14 counted, and which rule counted it.

**Note on status migrainosus:** ICHD-3 **1.4.1** is a debilitating attack unremitting beyond 72 hours, typical of the patient's prior attacks except in duration and severity. ICHD-3 also warns it "may often be caused by medication overuse," and if MOH criteria are met, code MOH instead. Surface that distinction rather than asserting either.

---

## 6. Build Order

Sequenced for you as the user, which differs from a go-to-market sequence.

1. **Capture and the observation store.** Immutable log, schema, conversational extraction with receipts and correction. Everything depends on this.
2. **Recall.** The one-page current-state brief. Printable, shareable, offline. **Pays back before there's enough data for anything else** — which also answers the cold-start problem.
3. **Derived episodes and day classification.** ICHD-3 constants, Delphi rules, dual readings, override. This is where a three-week episode becomes representable.
4. **Integrations.** Sleep and weather first — highest signal, lowest effort. Then cycle and HRV. **This is where the platform question binds** (§7).
5. **Phase annotation and retrospective relabeling**, including medication-response inference. Needs a few months of history to be interesting, and it operates on the past, so it can wait.
6. **Correlation.** Needs the most data and is the least proven. Build it for yourself, not as a claim.
7. **Prior-authorization packet.** Needs 30–90 days of clean data and needs to be right. Last, despite being the most valuable — it's only as good as everything above it.

**Divergence from independent QA, recorded rather than accidental:** an independent QA pass recommended pulling the Capacitor wrapper to Phase 1 and promoting the prior-authorization document to the lead position. This brief keeps Capacitor at step 4 and the prior-auth packet at step 7. That sequencing is a deliberate choice for a personal-use-first build, not an oversight: Capacitor arrives when integrations bind (§7), and the prior-auth packet is last because it is only as good as the data beneath it. Both recommendations are noted here rather than silently overridden.

---

## 7. Platform

The two must-haves pull against each other. Easy daily logging wants a web app openable anywhere. Correlation wants sleep, HRV and cycle — and **HealthKit and Health Connect are native-only, no exceptions.**

**For personal use, no App Store review is needed to run a Capacitor build on your own phone.** That's the unlock: go native early and privately, with none of the distribution overhead. Capacitor wraps the same web codebase; it isn't a rewrite.

**Interim measure available today:** a Shortcuts automation dumping Apple Health data to the API on a schedule. Unpolished, needs no wrapper, unblocks correlation work immediately.

**Recommendation:** PWA as the core codebase, **Capacitor wrapper at step 4.** It unlocks HealthKit and Health Connect, background audio, native push, App Intents and Android AppFunctions — and, if commercialized, App Store presence plus the only documented route into ChatGPT Health, which runs via Apple Health.

**Assistant access:** one remote MCP server with OAuth 2.1 + PKCE, exposing log and query tools over the same backend. Claude accepts arbitrary MCP servers by URL with no review, so this can be running in days. The Free tier caps at one custom connector. This is where recall becomes genuinely good — conversational query over your own history, without the file shuffling.

---

## 8. Regulatory — Now Versus Commercialization

| Concern | Building for yourself | Gate before selling |
|---|---|---|
| FDA device status | Not in the picture — no distribution, no claims | Real. The operative framework is **General Wellness**, not the CDS carve-out: CDS covers only software for healthcare professionals, so a patient-facing app sits outside it entirely |
| **PROMIS** | **Free** — HealthMeasures permits individual clinical use | **Commercial use requires permission "regardless of purpose, at all times,"** plus paid HEAP permission for electronic administration. First letter to write |
| MIDAS / HIT-6 | Likely fine individually | HIT-6 is licensed by QualityMetric, paid. MIDAS rights-holder unconfirmed |
| HIPAA | Not a covered entity | Triggered by contracting with clinics or pushing to an EHR — a one-way door |
| FTC Health Breach Notification Rule | N/A | Applies to DTC health apps |
| **WA My Health My Data Act** | N/A | **Product-design obligation with a private right of action.** Separate opt-in for collection, separate authorization for sharing. Design onboarding for it early even while it doesn't bind |
| Assistant data retention | Your risk to accept | Undocumented whether Anthropic or OpenAI retain health data through MCP calls. Confirm before shipping to others |
| Trademark | N/A | HEADSTATE appears unregistered, but Headspace Health is a registered family in overlapping classes with active TTAB enforcement. $1–2k clearance before brand spend |

**Worth doing early even though nothing requires it:** recruit a headache specialist as an advisor. Cheap, buys real credibility, and they'll pressure-test the attenuation model before it's built on.

---

## 9. Risks

1. **Extraction quality is the whole bet.** One sentence to clinically valid structure, with no form to fall back on. **You are the test case and can answer this in 60 days for free** — the single best reason to build for yourself first.
2. **The attenuation model needs the §2.6 framing** to survive clinical review. Get it right before building on it.
3. **Derived boundaries will sometimes be wrong.** Override must be a first-class persisted fact from day one.
4. **Structured versus easy capture stays in tension.** Prior-auth-grade data wants ICHD-3 fields; easy logging wants one sentence. You may discover you'd rather tap four things. Stay open.
5. **Retrospective reinterpretation breaks report reproducibility** without the two-clock design in from the start. Retrofitting is painful.
6. **The med-response inference may have more prior art than it appears.** Verify before it becomes a differentiator claim.
7. **Distribution, if commercializing.** No App Store presence as a PWA, near-zero organic discovery in assistant directories, and a noisy floor of near-identical AI trackers to be confused with.
8. **The algorithm is the most copyable component.** Position on the correction corpus and the report, not on inference logic.
9. **Correlation is a research bet, not a claim.** The largest randomised trial of trigger analytics layered on a diary (M-sense Migräne, n=477, 12 weeks) compared the full app — trigger identification, correlation analysis, medication tracking — against a control app containing only a headache and medication diary. Result: migraine days per month fell from 7.74 to 5.79 in the intervention arm versus 7.76 to 5.41 in the control, a between-group difference of 0.38 days that was not statistically significant, with most secondary outcomes null. Investigators concluded self-monitoring itself drove the improvement in both arms. Consequence: correlation features are a research bet, not a marketing claim, and this is the evidential basis for the paywall position — gating correlation features is defensible on the evidence; gating logging or the recall/documents report is not.

---

## 10. Open Questions

1. **How much intra-episode granularity do you actually want?** Every observation timestamped, or a few checkpoints daily? This sets how heavy capture feels, and it's the main lever on adherence.
2. **The ER page — printed, or a link you display?** Determines whether offline printing is a v1 requirement.
3. **Do you want prodrome alerts at all?** 85% precision, ~10-hour lead, poor sensitivity. Useful when it fires, silent most of the time. Some people would rather not be warned.
4. **Does medication response resolve boundaries automatically, or only propose?** Per §3 it should propose — but confirm that a proposal you have to approve doesn't become the friction that stops you using it.

---

## 11. Confidence and Gaps

**Verified:** ICHD-3 1.4.1 criteria and the 12-hour remission allowance · the 48-hour distinct-attack rule and the deliberate sub-48-hour ambiguity · all four 2026 Delphi migraine-day definitions with consensus percentages, including that the Delphi definitions are scoped to trials, real-world practice and e-diary labelling and explicitly disclaim regulatory/reimbursement/prior-auth use · Cigna Botox day-count criteria and the absence of continuous-headache provisions across payer policy · premonitory prediction statistics (69% / 27.5% / 85.1% PPV / 10h38m) · dexamethasone recurrence ARRs and the milder-recurrence finding · npj event-diary best practices · LUMC e-diary derived-day architecture · Migraine Buddy's chronic-mode, prodrome/postdrome and non-overlapping-record behavior from its own help center · N1-Headache's mandatory daily model · Migraine Insight's open-ended episodes · the M-sense Migräne RCT result (n=477, 0.38-day non-significant between-group difference) · that no migraine app currently exposes an MCP server · that no app for which a data model could be documented performs retrospective phase linkage or attenuation modeling, with 8 of 21 apps surveyed unverifiable on phase linkage and 5 of 21 unverifiable on attenuation modeling.

**Unverified — treat as open:**
- ICHD-3 quotations came through a summarizing fetch layer, consistent across two independent sources. **Check against printed ICHD-3 / *Cephalalgia* 2018;38(1):1–211 before quoting clinically.**
- The Delphi paper **has a published Correction** that was not retrieved. Verify figures against the corrected version.
- Manage My Pain, Migraine Monitor, Ctrl M Health, Migra, CeCe and Visible data models are **undocumented publicly** — unverifiable rather than absent. Hands-on testing needed before claiming any lack a capability.
- **No user-forum evidence retrieved** (proxy blocked Reddit). Negative user sentiment is unverified rather than absent.
- Prior art on medication response as a computational phenotyping signal.
- Whether Anthropic or OpenAI retain health data passed through MCP tool calls.
- MIDAS commercial rights-holder.
- Whether FDA's January 2026 CDS revision relaxed or expanded the transparency criterion — analysts disagree.
- Migraine Buddy's reported Novartis publications and Lipton association.

**Precedence when sources disagree:** where this brief and `CLAUDE.md` disagree, `CLAUDE.md` governs implementation. Where this brief and a `Research/` document disagree on evidence, the Research document governs.

---

*Research base: competitive landscape · assistant-integration feasibility · clinical and regulatory posture · product architecture · adversarial QA (31 claims checked) · focused verification of the continuous-timeline claim. Full source lists retained per stream.*
