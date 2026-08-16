# HeadState — Functional Requirements

**Scope:** build steps 0–4. Written as **testable acceptance criteria**, not "the system shall" prose, because a coding agent can execute an assertion and cannot execute an aspiration.

**Companion documents.** `tech-spec.md` describes how. This describes what must be true. `decision-log.md` governs both. `CLAUDE.md` carries the invariants.

**Format.** Each requirement has a stable ID, a Given/When/Then statement, and a note where the requirement encodes something non-obvious. IDs are permanent — supersede rather than renumber.

**Priority.** **[MUST]** blocks the step. **[SHOULD]** ships in the step unless it forces a delay. **[LATER]** is recorded here so it is not lost, but is out of scope for steps 0–4.

---

## 0. Scaffold

**FR-0.1 [MUST]** — Given a clean checkout on a toolchain meeting the floors below, when `npm install && npm run build && npx cap sync` runs, then both `ios/` and `android/` projects build without manual intervention.
*Toolchain floors, restated here because this document is the one built against: **Node ≥22, Xcode ≥26.0, iOS deployment target ≥15.0, Android Studio Otter 2025.2.1, minSdk 24**, Capacitor 8 (`@capacitor/core` 8.5.0 as of 31 Jul 2026). iOS defaults to Swift Package Manager under Capacitor 8. Framework is **React + Vite**; Vite's `build.outDir` must match the Capacitor `webDir`.*

**FR-0.2 [MUST]** — Given a production build, when the bundle is inspected, then `server.url` is absent from `capacitor.config`. *A shipped build containing a dev machine's IP is a release-blocking defect.*

**FR-0.3 [MUST]** — Given the repo, when CI runs, then the clinical-constants lint passes: every constant in `packages/clinical-config` carries a citation string, and no clinical threshold appears as an inline literal anywhere in `apps/` or `packages/derivation`.

**FR-0.4 [MUST]** — Given `packages/clinical-config`, when the build runs, then a content hash is computed and upserted into `clinical_config`, and the hash changes if and only if a constant value changes.

**FR-0.5 [MUST]** — Given the bundle identifier **`health.headstate.app`** (D14), when the projects are generated, then that value is used and never changed. A debug variant may suffix it (`health.headstate.app.dev`); the release identifier does not move. *App Intents registration, HealthKit entitlement association and future extensions all key off it.*

**FR-0.6 [SHOULD]** — Given a push to main, when CI runs, then the Android APK builds on a Linux runner and the iOS build runs on a macOS runner.

**FR-0.7 [MUST]** — Given **PostgreSQL 16**, matching the Fly.io production target (D14) and the 16.13 the DDL was executed against, when CI runs, then the CI gate first creates the prerequisite objects the schema DDL references but does not itself define — the `btree_gist` extension, the `headstate_app` and `headstate_ops` roles, and the `users`, `clinical_config` and `consent_policy_versions` tables — then the full DDL executes against a disposable Postgres instance and the migration test suite passes.
*Catches the class of defect where a statement in the spec does not actually execute — checked at spec-review time, not discovered at deploy time. Without creating the prerequisite objects first, the gate fails on a self-containment problem rather than a real defect.*

---

## 1. Capture and the observation store

### 1.1 Immutability

**FR-1.1 [MUST]** — Given an existing observation, when the application role attempts `UPDATE` or `DELETE`, then the database raises an exception and the row is unchanged.
*Test: execute both statements as `headstate_app`; assert both throw.*

**FR-1.2 [MUST]** — Given a user correcting a previously logged observation, when the correction is saved, then a new observation row is inserted and a row is inserted into the separate `supersessions` table linking the old id to the new one. There is no `superseded_by` column on `observations` — the immutability trigger would make it unsettable by the application role, so supersession is a table, not a column. The original row's content is byte-identical to before.

**FR-1.3 [MUST]** — Given an observation with a supersession, when any read path or derivation runs, then the superseded observation is excluded. Reads go through the `observations_live` view — `observations` anti-joined against `supersessions` — never the raw table directly.
*Test: supersede an observation that would change a day classification; assert the classification reflects only the replacement.*

**FR-1.4 [MUST]** — Given any observation created from speech or text, when it is written, then the verbatim utterance is retained in `provenance`.

### 1.2 Tri-state fields

**FR-1.5 [MUST]** — Given a write where a symptom field has `ask_status: 'not_asked'` and a non-null `value`, when submitted to any endpoint, then the server rejects it with `VALIDATION_ERROR`.
*Test directly against the API, the MCP tool and `/ingest/transcript`. All three must reject. Schema descriptions are hints a model can ignore; this check is not.*

**FR-1.6 [MUST]** — Given a user who did not mention nausea, when the observation is stored, then nausea is recorded as `ask_status: 'not_asked'`, `value: null` — not as absent and not as `no`.
*"No nausea" and "nausea was never discussed" are different clinical facts.*

**FR-1.7 [MUST]** — Given a user who explicitly said "no nausea", when stored, then `ask_status: 'asked'`, `value: 'no'`.

### 1.3 Extraction

**FR-1.8 [MUST]** — Given any extracted field, when it is committed, then it carries an evidence span whose quoted text matches the source transcript by exact or tight fuzzy match.

**FR-1.9 [MUST]** — Given an extraction returning a field whose evidence span does not match the transcript, when verification runs, then the field is dropped and not committed. It is not repaired and not retried with the same call.
*Test: adversarial fixtures with plausible but absent quotes. Assert zero fabricated fields survive.*

**FR-1.10 [MUST]** — Given the extraction pipeline, when it runs, then verification includes a structurally independent entailment check: a separate API call, using a different prompt template, whose result is not derived from the extraction call's output. Self-verification within the same call does not satisfy this.
*Test: assert the entailment call is actually made as a distinct call, and that its verdict gates the commit — a fixture where the entailment call would reject must not commit.*

**FR-1.11 [MUST]** — Given a field extracted from an explicit lexical match such as "8 out of 10", when stored, then `match_type: 'lexical'`. Given a field mapped through a vocabulary such as "really bad" → severe, then `match_type: 'paraphrase'`.
*Computed deterministically from the span against a canonical vocabulary. Never asked of the model — verbalised self-confidence is documented as systematically miscalibrated.*

**FR-1.12 [MUST]** — Given a time expression in a transcript, when resolved, then resolution is performed by chrono-node against an explicit reference timestamp and IANA zone, not by the model.

**FR-1.13 [MUST]** — Given a time expression that cannot be resolved unambiguously, when the observation is written, then `observed_at` and `observed_date` are both null, the observation is excluded from day classification, and it is surfaced for follow-up. No timestamp is guessed.

**FR-1.14 [MUST]** — Given a compound utterance describing events at different times, when extracted, then multiple observations are produced, each with its own timestamp and evidence span.

**FR-1.15 [MUST]** — Given the same transcript POSTed twice within the idempotency window, when processed, then the second call returns the original result and creates no duplicate observations.
*Keyed on a hash of user, transcript and reference timestamp. A flaky Shortcut will do this.*

**FR-1.16 [MUST]** — Given a medication observation, when committed, then `drug_class` is populated from the taxonomy — `triptan | ergot | opioid | combination_analgesic | non_opioid_analgesic | corticosteroid`. An unclassifiable medication is rejected with `MISSING_REQUIRED_FIELD` rather than stored without a class.
*Every MOH threshold depends on it. `corticosteroid` is in the taxonomy: without it, a steroid write is rejected, which breaks FR-1.18's next-day taper follow-up and leaves the attenuation model (FR-3.10, FR-3.10a) with nothing to derive from.*

### 1.4 Capture model

**FR-1.17 [MUST]** — Given a user beginning an episode, when they log onset, then the timestamp is recorded at or near the moment noticed via the open/close affordance — not inferred from a later daily checkpoint.

**FR-1.18 [MUST]** — Given a logged medication dose, when it is committed, then follow-up prompts are scheduled within tolerance windows: the first within **[25, 65] minutes**, the second within **[110, 130] minutes** for a triptan or NSAID, and, for a corticosteroid taper, on the **next local calendar day**. Each prompt asks specifically about attenuation and recurrence.

**FR-1.19 [MUST]** — Given the daily checkpoint, when it is presented, then it is presented **in the morning** and asks about the prior day and night. An evening or end-of-day checkpoint does not satisfy this requirement.
*Only a morning prompt can observe the Delphi overnight rule, and an evening prompt cannot capture an attack starting after it fires.*

**FR-1.20 [MUST]** — Given the daily checkpoint, when it is presented, then the aura question and the overnight question are asked **unconditionally**, regardless of any branching on whether a headache occurred.
*Test: complete a checkpoint answering "no headache"; assert both questions were still presented. An aura day counts as a migraine day regardless of head pain, and migraine days drive the CGRP payer threshold — gating aura behind a headache question silently drops days from that count.*

**FR-1.21 [MUST]** — Given a day inside an open episode, when the daily checkpoint is presented, then it is capped short — the two unconditional questions plus a binary "still going / anything changed" — and does not lengthen.
*An adaptive checkpoint that grows on worse days is heaviest exactly when capacity is lowest.*

**FR-1.22 [MUST]** — Given an open episode with no observation for 12 hours, when the threshold passes, then one lightweight nudge is delivered. Not more than one per 12-hour window.

**FR-1.23 [MUST]** — Given the user is mid-attack and logs by voice, when the record is written, then the system confirms and asks **no** follow-up questions.
*Test: assert the mid-attack flow emits exactly one confirmation interaction and zero question prompts.*

**FR-1.24 [MUST]** — Given the daily checkpoint, when the user does not respond, then the day is **not** silently inferred as symptom-free. An active "no events" confirmation is required for a `none` classification to be `confirmed_by: user`.

**FR-1.25 [MUST]** — Given any question wording used in the checkpoint, then it is a documented pre-release review checklist item, with a named reviewer, confirming the wording is independently authored and contains no PROMIS, HIT-6, MIDAS, MSQ or MPFID item wording.
*The question wording is a licensing surface. String-matching against licensed instruments is not reliable, so this is a review gate rather than an automated test.*

### 1.5 Backfill

**FR-1.26 [MUST]** — Given an observation backfilled any number of days late, when day classification runs, then the affected day **counts** toward `headache_days` and `migraine_days`.
*Test: backfill an observation 30 days late; assert the day count increases. No published rule supports a backfill-latency exclusion, and the harm lands hardest on exactly the patient the product exists for.*

**FR-1.27 [MUST]** — Given a late-backfilled observation, when it is committed, then it carries reduced confidence, and that confidence appears on any export containing the affected day.

**FR-1.27a [MUST]** — Given an observation where `recorded_at − observed_at > BACKFILL_WINDOW` (6 hours), when committed, then it is flagged backfilled and written with reduced confidence. Given the same gap **at or under** `BACKFILL_WINDOW`, then ordinary real-time confidence applies. The flag is binary — there is no intermediate tier between "real-time" and "backfilled."
*Test both sides of the boundary. D19: a graduated decay schedule would invent a specific curve with no published rule behind it.*

**FR-1.27b [MUST]** — Given an observation flagged backfilled under FR-1.27a, when the capture confirmation is shown, then the confirmation states the entry was logged for a past date at reduced confidence — not only downstream on an export.
*Test: backfill an entry for two days prior; assert the confirmation text differs from the real-time confirmation ("Logged. Feel better.") and names the flag. D19: honesty about a backfill belongs at the moment of capture, to the person doing the backfilling, not only to a later reader of the export.*

**FR-1.28 [SHOULD]** — *Superseded by FR-1.26; a backfill cutoff must never gate `headache_days` or `migraine_days`.* A backfill cutoff may exist as a **display** preference only — a user-visible, changeable setting affecting what renders by default — and must never affect the underlying day counts. Per D5 and invariant 13, backfill always counts.

---

## 2. Recall

**FR-2.1 [MUST]** — Given no network connectivity, when the user opens the recall summary, then it renders completely from local data.
*Test: airplane mode on device; Playwright `setOffline` on web. Assert full render.*

**FR-2.2 [MUST]** — Given the app is open, when the user reaches the recall summary, then it is reachable in **one tap**.

**FR-2.3 [MUST]** — Given the recall summary, when rendered, then the first section, in the largest type on the page, states where the current episode stands and how many prior emergency presentations occurred during it.
*Test: assert section order. The motivating failure was each ED visit being triaged as a new acute event; that must be corrected in the first line read.*

**FR-2.4 [MUST]** — Given the recall summary, when rendered, then the second section lists every medication and dose in the preceding 72 hours with times, most recent first, plus the acute-medication day count for the last 30 days with the MOH threshold flagged if met.

**FR-2.5 [MUST]** — Given the recall summary, when rendered, then it contains no request for a specific drug or dose. It states what was tried and what happened; the prescribing conclusion is the clinician's.
*A patient naming their own medication is the single biggest trigger for drug-seeking suspicion.*

**FR-2.6 [MUST]** — Given the recall summary, when rendered, then it carries a provenance line stating the record was generated from a continuously time-stamped log rather than reconstructed from memory, and a "verify with patient" instruction.

**FR-2.7 [MUST]** — Given unconfirmed inferences feeding the recall summary, when it is generated, then **no confirmation flow blocks generation**, and the unconfirmed items are labelled visibly on the page.
*Test: seed unconfirmed inferences; assert the summary generates with zero interstitial prompts and that the labels are present. The confirmation gate applies to the prior-auth packet only. An unskippable flow on the crisis path is the same category of harm as a paywall on it.*

**FR-2.8 [MUST]** — Given the recall summary, when the user chooses to print, then the page is handed to the native print pipeline via a print plugin, reaching AirPrint or Android's print framework. `window.print()` alone does not satisfy this.

**FR-2.9 [MUST]** — Given no printer is available, when the user chooses to share, then a locally generated PDF is offered through the OS share sheet, produced with no network.

**FR-2.10 [SHOULD]** — Given first-run onboarding, when it completes, then the user is prompted once to print a wallet card.
*The only paper reliably present in an emergency department is paper already in a wallet.*

**FR-2.11 [MUST]** — Given the on-screen recall view, when rendered, then it is screenshot-friendly: no auto-refreshing elements, no content behind an interaction.
*Users will screenshot it whether or not it is a feature.*

---

## 3. Derived episodes and day classification

### 3.1 Determinism and parity

**FR-3.1 [MUST]** — Given identical inputs — observations, overrides, config, timezone — when `derive()` runs twice, then output is byte-identical.

**FR-3.2 [MUST]** — Given a golden fixture set, when run through the server derivation call site and the client derivation call site, then output is **byte-identical across both**.
*This is a first-class deliverable, not a nice-to-have. It is the mechanism that prevents two implementations of clinical logic from disagreeing on a payer document.*

**FR-3.3 [MUST]** — Given any derived object, when written, then it carries `algorithm_version`, `config_hash`, `computed_at`, `computed_from`, `confirmed_by` and `review_status`.

**FR-3.4 [MUST]** — Given a report generated at time T, when regenerated later after reinterpretation, then the original is reproducible from `computed_from` and `config_hash`.

### 3.2 Day classification

**FR-3.5 [MUST]** — Given an attack with any acute medication taken during it and total duration, measured across midnight, ≥30 minutes, when classified, then the attack is migraine-qualifying with `rule: 'delphi_treated_30min'`, and every local calendar date the attack occupies takes `class: 'migraine'` — regardless of drug class, characteristics or associated symptoms.
*Qualification is evaluated once, against the attack's total duration — not per calendar day. A day-scoped implementation that sums duration within a single date is the defect FR-3.6a exists to prevent.*

**FR-3.6 [MUST]** — Given an untreated attack with total duration, measured across midnight, ≥4 hours and either ≥2 of 4 characteristics, or ≥1 characteristic plus ≥1 associated symptom, when classified, then the attack is migraine-qualifying with `rule: 'delphi_untreated_4h'`, and every local calendar date the attack occupies takes `class: 'migraine'`.
*Test explicitly that a 4-hour attack with **one** characteristic and **no** associated symptom does **not** qualify. A naive `duration >= 4h` implementation is the expected failure — as is measuring duration per calendar day rather than per attack.*

**FR-3.6a [MUST]** — Given an untreated attack from 22:00 Monday to 03:00 Tuesday with two headache characteristics, **where the patient slept through the end of the attack and woke headache-free**, when classified, then it produces **one** migraine day attributed to Monday. *Qualification uses the attack's total 5-hour duration, not 2 hours on Monday and 3 on Tuesday — a per-calendar-day measurement reaches the 4-hour threshold on neither and yields zero migraine days. The wake-headache-free precondition is what suppresses Tuesday's occupancy; without it the same attack yields two migraine days (FR-3.8a).*
*Duration is measured per attack, across midnight — not per calendar day. A per-calendar-day implementation gives 2 hours Monday and 3 hours Tuesday, reaches the 4-hour threshold on neither, and produces zero migraine days: a systematic under-count of nocturnal onset that harms the CGRP payer threshold directly.*

**FR-3.6b [MUST]** — Given a qualifying attack from Monday 09:00 to Friday 09:00 with the patient awake throughout, when classified, then **five** migraine days are produced, one per occupied date — not one.
*Attributing a multi-day attack only to its onset date under-counts every day after the first, and makes the ≥15-day Botox and MOH general-criterion thresholds structurally unreachable for a continuous-headache patient.*

**FR-3.6c [MUST]** — Given a continuous three-week episode, when day counts are computed, then **21** headache days are reported.

**FR-3.7 [MUST]** — Given aura meeting ICHD-3 criteria with no headache, when classified, then `class: 'aura_only'`, `rule: 'delphi_aura'`.

**FR-3.8 [MUST]** — Given an attack beginning Monday 20:00 and continuing until the patient wakes headache-free Tuesday 07:00, when classified, then exactly **one** migraine day is produced, attributed to **Monday** and carrying whichever qualifying rule applied (`delphi_treated_30min` or `delphi_untreated_4h`); Tuesday classifies `none` and contributes **zero** headache days.
*This departs from ICHD-3, which reckons duration until awakening. Delphi 2026, rule 4, 100% consensus. Tuesday's exclusion is the occupancy exception (Delphi rule 4) applied on top of the attack's qualifying class — there is no `delphi_overnight` rule value. A generative implementation (classify Monday from the 4-hour rule, then separately classify Tuesday because the patient woke from a migraine) produces **two** migraine days and leaks a headache day onto Tuesday, a day with no headache. Test both halves: exactly one migraine day, and Tuesday's zero.*

**FR-3.8a [MUST]** — Given the same 22:00 Monday → 03:00 Tuesday attack but with the patient **awake** at 03:00, when classified, then **two** migraine days are produced.
*The wake-headache-free exception does not apply, and Tuesday had three hours of headache.*

**FR-3.6d [MUST]** — Given two attacks occupying the same local date, one headache-qualifying and one migraine-qualifying, when classified, then exactly **one** `day_classifications` row is written for that date with `class = 'migraine'`, `rule` taken from the attack that produced the winning class, and `attributed_episode` set to the **earliest** occupying attack's episode.
*`day_classifications` is `PRIMARY KEY (derivation_id, local_date)` — one row per date. Under `relapse_split`, and on any date carrying a resolved attack plus a new one, two attacks routinely occupy one date. Without a merge rule a literal implementation either raises a primary-key violation or silently last-write-wins. Precedence is `migraine` > `headache` > `aura_only` > `none`.*

**FR-3.8b [MUST]** — Given a waking date whose occupancy by attack A is suppressed under the wake-headache-free exception, and a separate attack B also occupying that date, when classified, then the date **still counts** with attack B's class.
*The exception is scoped to the attack, not to the date. Suppressing the date itself would zero a day on which a genuinely separate attack occurred.*

**FR-3.9 [MUST]** — Given a set of day classifications, when day counts are derived, then `headache_days` counts `class IN ('headache','migraine')` and `migraine_days` counts `class IN ('migraine','aura_only')`.
*Test: assert that a naive `count(class = 'headache')` produces a **different, lower** number, and that the test fails if someone implements it that way. This arithmetic feeds the payer export.*

**FR-3.9b [MUST]** — Given a month containing a sub-48-hour recurrence, which emits `relapse_collapsed` and `relapse_split` derivations in addition to `primary`, when `headache_days` is computed, then the count is parameterised by a single `derivation_id` and does not multiply by variant.
*An unscoped count returns 2 headache days for one calendar date when two derivations exist over it — executed and confirmed.*

**FR-3.9a [MUST]** — Given a request for a payer-facing headache-day count, when produced, then it is computed under the payer's own criterion (e.g. Cigna Botox: headache ≥4 hours per day) and labelled as such, and is never substituted with the Delphi classification count from FR-3.9.
*Delphi explicitly disclaims reimbursement contexts. The two counts are different numbers computed under different rules and must never be conflated.*

**FR-3.10 [MUST]** — Given a day that would classify `none` but falls inside a **user-recorded** medication window (`course_start`/`course_end` on a medication observation — not an inferred half-life model), when classified, then `attenuation_flag: true` and `active_pharmacology` lists the agents. **The class remains `none`.**

**FR-3.10a [MUST]** — Given a day inside a user-recorded medication window that classifies `headache` or `migraine`, when classified, then it also carries `attenuation_flag: true`.
*Attenuation is not restricted to `none` days. A day with twenty minutes of headache under dexamethasone is also confounded, and that is the more common case.*

**FR-3.10b [MUST]** — Given an attenuated `none` day, when day counts are computed, then it does **not** count toward headache days or migraine days.
*The flag is annotation, not protection. This requirement exists so that is explicit rather than assumed.*

**FR-3.11 [MUST]** — Given any export reporting a date range, when generated, then it enumerates the attenuated days within that range, flagged, whatever their class.
*An attenuated day is class `none`, and a headache-day export lists only qualifying days — so without this requirement, the attenuated day never appears on the export the flag most needs to protect. Test: assert no export path silently drops an attenuated day from the enumerated range.*

### 3.3 Episode boundaries

**FR-3.12 [MUST]** — Given two head-pain observations separated by ≥48 hours headache-free, when derived, then two distinct episodes.

**FR-3.13 [MUST]** — Given a remission of ≤12 hours inside an otherwise unremitting run **that meets status migrainosus criteria (>72 hours, debilitating, typical of prior attacks)**, when derived, then the episode is **not** broken.

**FR-3.13a [MUST]** — Given the same ≤12-hour remission **outside** a status migrainosus assessment, when derived, then it does **not** extend an ordinary episode.
*The 12-hour allowance applies only within a status migrainosus assessment — it is not a general episode rule, and applying it generally would contradict FR-3.12's 48-hour distinct-attack separation.*

**FR-3.13b [MUST]** — Given a status migrainosus reading that competes with a distinct-attack reading (FR-3.14), when derived, then the status migrainosus reading is emitted as an **additional** derivation variant, not as a replacement for the competing reading.

**FR-3.14 [MUST]** — Given a recurrence within <48 hours, when derived, then **both** readings are emitted as separate `derivations` rows, each carrying `variant` of `relapse_collapsed` or `relapse_split` along with `algorithm_version` and `config_hash`, with episodes and day classifications hanging off them, each labelled with its governing rule.
*Test: assert both rows exist and neither is silently preferred. ICHD-3 says recurrence within 48 hours "may represent a relapse of the same attack or a new attack" and does not resolve it.*

**FR-3.15 [MUST]** — Given competing readings for the same period, when stored, then no exclusion constraint prevents their coexistence.

**FR-3.16 [MUST]** — Given an episode unremitting >72 hours, debilitating per recorded severity, and containing no remission exceeding 12 hours (FR-3.13), when derived, then it is flagged as candidate status migrainosus, and any recorded characteristics are surfaced alongside the flag for the clinician to judge whether the episode is typical of the patient's prior attacks.
*"Typical of prior attacks" is not computable from the schema and is not evaluated by the system — it is surfaced for clinician judgment rather than evaluated. The requirement is restated against what is computable: duration >72 hours, debilitating per recorded severity, and remissions ≤12 hours.*

**FR-3.17 [MUST]** — Given an episode meeting both status migrainosus and MOH criteria, when surfaced, then **the distinction is surfaced, not decided**. The output states both apply and that ICHD-3 directs coding MOH.

### 3.4 Medication overuse

**FR-3.18 [MUST]** — Given 10 triptan days in a calendar month, sustained over more than 3 months, when MOH is evaluated, then the per-class threshold fires.

**FR-3.18a [MUST]** — Given an MOH evaluation, when performed, then it uses a **calendar month**. A rolling 30-day count may be displayed only as a labelled non-clinical early warning and is never the ICHD-3 determination.
*Resolves an inconsistency where the window was described as "rolling 30-day" while the constants table said "days/month." D12 settles it: calendar month is the clinical determination.*

**FR-3.19 [MUST]** — Given 15 non-opioid-analgesic days, when evaluated, then the per-class threshold fires. Given 14, it does not.

**FR-3.20 [MUST]** — Given 6 triptan days and 6 NSAID days that are **disjoint** (12 distinct dates) in a calendar month, and the general criterion is met, when evaluated, then the combined-class rule (ICHD-3 8.2.6) fires.
*Test this case by name. A per-class counter alone reports clear, and that is the expected defect.*

**FR-3.20a [MUST]** — Given 6 triptan days and 6 NSAID days overlapping on 3 dates (9 distinct dates) in a calendar month, and the general criterion is met, when evaluated, then the combined-class rule **does not** fire.
*`MOH_COMBINED_CLASS` counts distinct calendar days, not the sum of per-class counts (D12) — this is what makes FR-3.20 and FR-3.20a produce different outcomes from the same per-class totals. The combined-class rule covers ergotamine, triptans, non-opioid analgesics and opioids, and **excludes combination analgesics** (ICHD-3 handles those at 8.2.5).*

**FR-3.21 [MUST]** — Given drug-day counts meeting a threshold but the general criterion — headache ≥15 days/month with a pre-existing headache disorder over >3 months — is not met, when evaluated, then **no MOH flag fires**. "Pre-existing headache disorder" is satisfied by a **user-attested settings flag**, never inferred from the record.
*Test: assert MOH does not fire on drug-days alone. The flag makes the general criterion evaluable — there is no diagnosis field, and inferring one would be a diagnostic assertion the guardrails prohibit.*

**FR-3.22 [MUST]** — Given any MOH flag surfaced to the user, when displayed, then it is presented as the criterion a named guideline uses, not as settled fact.
*ICHD-3 states its own day counts are "based on expert opinion rather than on formal evidence."*

### 3.5 Overrides

**FR-3.23 [MUST]** — Given a user override, when recomputation runs, then the override is read as an input and its value is preserved in the output.
*Test: set an override, trigger recompute, assert the override still applies.*

**FR-3.24 [MUST]** — Given an override, when stored, then it is keyed to a stable natural key — a local date for a day classification override, or an episode's `natural_key` for an episode boundary or phase override — never to a derived row id and never to the episode's start timestamp. The episode's `natural_key` is the id of its earliest member observation, resolved through any supersession chain back to the original id, not the id of whichever row is currently live.
*Derived rows are replaced on recompute; an override keyed to a row id is orphaned on the first recompute. A start timestamp is equally unsafe — a recompute that moves the start orphans it. Keying to the first observation's id survives recompute because observations are immutable, but only if the resolution walks the supersession chain: without it, correcting the first observation of an episode silently orphans every override on it.*

**FR-3.25 [MUST]** — Given an override write, when committed, then a recompute is enqueued in the same transaction — identically to an observation write.
*Otherwise a second device can pull the override while still displaying derived state that does not account for it.*

**FR-3.25a [MUST]** — Given a recompute that replaces an episode by `(derivation_id, natural_key)`, when it runs, then day classifications referencing that episode survive and their `attributed_episode` reference is nulled rather than the delete failing.
*`day_classifications.attributed_episode` is declared `ON DELETE SET NULL`; without it the replace fails with a foreign-key violation — verified against PostgreSQL 16.13.*

**FR-3.25b [MUST]** — Given a recompute that finds a live `overrides` row keyed to an episode's `natural_key`, when the episode row is written, then it carries `confirmed_by: 'user'` — not the `system` default FR-3.26 specifies — even where other fields on that same row (`computed_from`, etc.) were computed rather than entered.
*Test: set an episode-boundary override, trigger recompute, assert `confirmed_by: 'user'` on the resulting episode row. D18: a human set the boundary; the field should say so rather than reporting the row as machine-computed.*

### 3.6 Review and the gate

**FR-3.26 [MUST]** — Given a new derived object, when created, then `confirmed_by: 'system'` and `review_status: 'unreviewed'`. **Exception: FR-3.25b.**

**FR-3.27 [MUST]** — Given a medication-response boundary inference, a 48-hour-rule application or a status migrainosus classification, when computed, then it is **displayed immediately** and **queued** for confirmation — it does not prompt at the time. **[LATER — step 5]** *Medication-response inference is outside this document's scope; the 48-hour-rule and status migrainosus portions of this requirement apply now, under step 3.*

**FR-3.28 [MUST]** — Given a queued item untouched past the timeout, when the timeout elapses, then `review_status` becomes `'timed_out'` and `confirmed_by` is **unchanged**.
*A timeout must never be recorded as human confirmation.*

**FR-3.29 [MUST]** — Given a candidate phase label, when applied under Tier 1, then it is applied **only retrospectively**, triggered by a later attack opening — never at capture time. **[LATER — step 5]**
*Premonitory status is only determinable after what follows. Phase annotation is step 5, outside this document's scope.*

**FR-3.30 [MUST]** — Given unconfirmed inferences feeding the prior-authorisation packet, when generation is requested, then each is force-surfaced for confirmation before generation proceeds. This gate is non-configurable and **is not relaxed for the primary user**. **[LATER — step 7, recorded here for completeness]**

**FR-3.31 [MUST]** — Given the system in operation, when metrics are collected, then correction rate, backlog size and **timeout rate** are all instrumented.
*Timing out is informationally identical to never reviewing. Timeout rate is the failure signal the other two miss.*

### 3.7 Recomputation

**FR-3.32 [MUST]** — Given a new observation, when committed, then a recompute is enqueued covering at minimum `[observed_at − 72h, observed_at + 72h]`, widened to cover any overlapping episode.

**FR-3.33 [MUST]** — Given a config hash change, when it is deployed, then affected derived rows are recomputed and stored under the new hash without destroying rows computed under the prior hash.

**FR-3.34 [MUST]** — Given a recompute, when it runs, then superseded observations are excluded and `computed_from` records the exact ids used.

---

## 4. Integrations

**FR-4.1 [MUST]** — Given the user grants health permission, when the app foregrounds, then sleep data is read from HealthKit or Health Connect and written as observations with `source: 'passive_feed'` and the feed named in `provenance`.

**FR-4.2 [MUST]** — Given HealthKit returns zero records, when the app handles it, then the state is treated as **ambiguous** between "denied" and "no data exists", and the UI does not assert denial.
*Apple deliberately makes read-denial indistinguishable from absence. A confident "access denied" prompt would be wrong.*

**FR-4.3 [MUST]** — Given Android 13 or below without Health Connect installed, when the user attempts to connect, then an explicit install-or-open branch is presented.

**FR-4.4 [MUST]** — Given Health Connect's 30-day rolling read window, when historical data beyond it is needed, then the app either requests extended history or states plainly that earlier data is unavailable.

**FR-4.5 [MUST]** — Given a scheduled dose follow-up, when Android's exact-alarm permission is revoked, then the app detects this on foreground and re-requests, and surfaces that pending reminders were lost.
*Test: revoke the permission, foreground the app, assert detection. This is the most likely silent field failure.*

**FR-4.6 [MUST]** — Given an Android device reboot between a dose being logged and its follow-up firing, when the app next foregrounds, then outstanding follow-ups are re-derived from persisted state and rescheduled.
*AlarmManager alarms do not survive reboot and the plugin does not handle this.*

**FR-4.7 [MUST]** — Given location permission for weather, when weather is fetched, then only coarse location is sent, and `collect_location` consent is verified first.

**FR-4.8 [SHOULD]** — Given weather data, when stored, then it is written as observations of kind `environment` with the source named — not merged silently into an attack record.

**FR-4.9 [MUST]** — Given the app promises health data sync, when documented in the UI, then it does **not** claim near-real-time background updates.
*Foreground polling is the mechanism; iOS background execution is opportunistic and can be starved entirely.*

---

## 5. Sync

**FR-5.1 [MUST]** — Given a client with watermark N, when it pulls, then it receives observations with `seq > N` in `seq` order and advances the watermark to the maximum received.

**FR-5.2 [MUST]** — Given a push batch whose response is lost after the server committed, when the client retries the identical batch, then **zero duplicate rows** exist server-side and the client reconciles to `pushed`.
*Test with a commit-then-500 server. This is the core reliability test.*

**FR-5.3 [MUST]** — Given a batch containing one malformed row among N valid, when pushed, then the N−1 valid rows are accepted and only the malformed row is marked `failed`.
*One bad row must never stall the queue.*

**FR-5.4 [MUST]** — Given repeated sync failures, when retrying, then intervals grow with exponential backoff and jitter, capped, and reset on success.
*Test with an injected fake clock, not real sleeps.*

**FR-5.5 [MUST]** — Given unsynced observations, when the UI renders, then pending count and last successful sync time are visible, and individual unsynced records are marked.

**FR-5.6 [MUST]** — Given unsynced observations, when the recall summary is generated, then it renders **complete from local data** and does not imply incompleteness because of sync state.
*Local completeness and cloud backup are different concerns; conflating them would mislead a user in an emergency department.*

**FR-5.7 [MUST]** — Given a client-computed derived object whose `algorithm_version` is unrecognised, when it reaches the server, then it is rejected or flagged, not silently accepted.

**FR-5.8 [MUST]** — Given a client reconnecting, when server-computed derived state exists for the same `computed_from` set, then the server result **overwrites** the local one and any provisional label clears.

**FR-5.9 [MUST]** — Given a schema migration, when it runs against a local database containing `pending` rows, then those rows survive with `pending` status and remain serialisable to a valid push payload.
*Test against a fixture seeded with pending rows. Never gate a migration on sync completion — a user offline for a week must still be able to log.*

---

## 6. MCP server

**FR-6.1 [MUST]** — Given any MCP tool invocation, when handled, then `requireConsent(userId, 'share_assistant_connector', { id: clientId })` is called before any clinical data is read or written.
*Test: revoke consent for one client, assert every tool throws `CONSENT_REQUIRED` before touching data.*

**FR-6.2 [MUST]** — Given consent granted for Claude, when ChatGPT calls a tool, then it is rejected. Consent is per-platform.

**FR-6.3 [MUST]** — Given a bearer token whose `aud` does not match the canonical resource URI, when presented, then the request is rejected.
*Token passthrough is forbidden by the MCP spec.*

**FR-6.4 [MUST]** — Given a request with a missing or invalid `Origin` header, when received, then it is rejected with 403.

**FR-6.5 [MUST]** — Given an unauthenticated request, when rejected, then the 401 carries `WWW-Authenticate: Bearer resource_metadata="…"` pointing at the RFC 9728 document.

**FR-6.6 [MUST]** — Given every registered tool, when inspected, then each declares a `title` and the applicable `readOnlyHint` or `destructiveHint`.
*Defaults are `destructiveHint: true` and `openWorldHint: true`, so an unannotated read tool misrepresents itself to every client, regardless of distribution path. Claude's Connectors Directory submission also requires this, but that only applies if the project later submits under a Team or Enterprise organisation — a solo builder cannot submit at all.*

**FR-6.7 [MUST]** — Given a write tool, when it succeeds, then it returns the full canonical record as persisted plus a one-line human-readable recap.
*The read-back is the primary safety net; no client-side confirmation dialog is guaranteed across assistants.*

**FR-6.8 [MUST]** — Given `get_day_counts`, when it returns, then every number carries the `config_hash` and version of the rule that produced it, and the count of unreviewed inferences.
*A day count without its rule is not honest output.*

**FR-6.9 [MUST]** — Given an error, when returned, then it uses a stable code from the defined set with an actionable `suggestion`, and leaks no stack trace or internal identifier.

**FR-6.10 [MUST]** — Given an `episode_id` returned to an assistant, when presented by a different authenticated principal, then it is rejected.
*A server-minted handle is never authentication.*

**FR-6.11 [MUST]** — Given text returned from any read tool, when processed, then it is treated as data and never as instructions. No free-text observation content can trigger a tool call or alter scope.

**FR-6.12 [MUST]** — Given a personal access token issued for the iOS Shortcut, when used, then it is accepted only at `/ingest/transcript` and rejected at `/mcp` and every read endpoint.
*The iOS Shortcut posts to `/ingest/transcript`, a plain REST endpoint — it does not speak JSON-RPC, so scoping the token to MCP tool names is the wrong surface. Scope it to the endpoint instead.*

**FR-6.13 [MUST]** — Given a personal access token, when stored, then only a salted hash is persisted, and regeneration invalidates the previous hash immediately.

**FR-6.14 [MUST]** — Given any token, when logged, then neither the raw token nor the `Authorization` header appears in any log.

---

## 7. Consent, audit and privacy

**FR-7.1 [MUST]** — Given a consent grant or withdrawal, when recorded, then a `consent_events` row is inserted and `consent_current` is updated **in the same transaction**.
*An async projection creates a real window of un-consented data flow.*

**FR-7.2 [MUST]** — Given any code path that moves data outward — MCP response, export link, health-platform push — when executed, then it calls `requireConsent` inside the handler and fails closed.
*Test: for each outward path, remove the consent row and assert the path throws. Middleware a future route could bypass does not satisfy this.*

**FR-7.3 [MUST]** — Given a consent screen, when rendered, then allow and decline are equally weighted, nothing is pre-checked, and nothing has default focus.

**FR-7.4 [MUST]** — Given onboarding, when consent is collected, then collection consent and sharing consent are separate screens with separate affirmative actions. No single control satisfies both.

**FR-7.5 [MUST]** — Given a user who declines location, assistant and export-link consent, when they use the app, then symptom logging, recall and day classification all work fully.
*Declining must be a complete working path.*

**FR-7.6 [MUST]** — Given the app is published, when the Consumer Health Data Privacy Policy is served, then it is a standalone route, separately linked, distinct from the general privacy policy, containing only what the Act requires.

**FR-7.7 [MUST]** — Given the in-app consent text and the published policy, when compared, then they are generated from the same `consent_policy_versions` rows and provably cannot drift.

**FR-7.8 [MUST]** — Given any write to the clinical record, when it occurs, then an `audit_log` row records actor type and id, auth method, scopes used, action, target ids, outcome and correlation id.

**FR-7.9 [MUST]** — Given an `audit_log` row, when inspected, then `request_summary` contains the **shape** of the change, not clinical content.
*Test: assert no medication name, symptom free-text or severity value appears in any audit row.*

**FR-7.10 [MUST]** — Given an export link, when created, then the token is 256 bits from a CSPRNG, only its hash is stored, `expires_at` defaults to 24 hours, and `max_views` defaults to a small finite number.

**FR-7.11 [MUST]** — Given any access attempt on an export link, when it occurs, then a row is written to `export_link_access_log` regardless of outcome — served, expired, revoked, exceeded, bad PIN or not found.
*Without this the HBNR 60-day discovery clock can never start, which is itself the compliance gap.*

**FR-7.12 [MUST]** — Given the revoke UI, when displayed, then it states plainly that revocation stops future access and does not retract content already viewed, downloaded or screenshotted.

**FR-7.13 [MUST]** — Given a link that is expired, revoked or over its view cap, when accessed, then it returns nothing and logs the outcome.

**FR-7.14 [SHOULD]** — Given an export link, when created, then an optional PIN can be set, checked server-side with attempt limiting.

**FR-7.15 [MUST]** — Given the error-tracking integration, when an error is captured, then request bodies are nulled, query parameters are placeholders only, and any field matching a clinical field name is scrubbed **before the event leaves the process**.
*Test with a deliberate error carrying clinical content; assert the outbound payload is clean. Unverified scrubbing makes error tracking an undisclosed sharing event happening by accident.*

**FR-7.16 [MUST]** — Given the device database, when created, then SQLCipher encryption is enabled with a CSPRNG-generated key stored in iOS Keychain or Android Keystore, never in preferences or config.

**FR-7.17 [MUST]** — Given iOS, when the Keychain item is created, then accessibility is `kSecAttrAccessibleWhenUnlockedThisDeviceOnly`.

**FR-7.18 [MUST]** — Given an encrypted database present with its key absent — a restore onto a new device — when the app launches, then this is detected as a **distinct state from corruption** and the user is offered a server restore. The app does not fail silently and does not regenerate a key, orphaning the file.
*Test: delete the Keychain item, relaunch, assert the specific state is detected.*

**FR-7.19 [MUST]** — Given `drug_class` and any timestamp column, when stored, then they are **not** encrypted at the column level.
*Encrypting them is opaque to the query planner and would break the MOH rolling-window query and every range scan — the core clinical logic, not an optimisation.*

**FR-7.20 [SHOULD]** — Given the schema, when created, then `deletion_requested_at` and per-user DEK wrapping columns exist, even though the destroy-and-tombstone path is not yet implemented.
*The shape is far cheaper to bake in than to migrate onto accumulated rows.*

### 7.x Breach detection

*§11 of the tech spec specifies breach detection and the discovery it enables is what starts the HBNR 60-day clock; these requirements give it acceptance criteria.*

**FR-7.21 [MUST]** — Given access attempts on a single export link, when the count of distinct client IPs exceeds `max_views`, or a view's geolocation is inconsistent with prior views on the same link, then an alert is raised.

**FR-7.22 [MUST]** — Given the audit log, when write volume from a single actor is unusual relative to that actor's baseline, or a write falls outside the actor's normal scopes, or any write is attributed to an MCP client that was never granted consent, then an alert is raised.

**FR-7.23 [MUST]** — Given the export-link and MCP endpoints, when 401 or 403 responses spike relative to baseline, then an alert is raised.

**FR-7.24 [MUST]** — Given `consent_events` and `export_link_access_log`, when a calendar month elapses, then a scheduled review of both is performed and its completion is recorded.
*For a solo operator this calendar-scheduled review is the actual discovery mechanism — without it, the HBNR clock has no trigger.*

---

## 8. Cross-cutting quality gates

**FR-8.1 [MUST]** — Given the CI suite, when it runs, then every clinical constant is asserted against its cited source.

**FR-8.2 [MUST]** — Given the CI suite, when it runs, then cross-runtime derivation parity (FR-3.2) is asserted, and a divergence fails the build.

**FR-8.3 [MUST]** — Given the extraction test suite, when it runs, then **fabrication rate is measured and asserted at zero** against the adversarial fixture set, with the model id, temperature and seed pinned so the gate is deterministic; **or**, if determinism cannot be guaranteed, the fabrication-rate measurement runs as a scheduled non-blocking job rather than a blocking CI gate.
*Fabrication rate is a pipeline-construction target, not an accuracy metric to improve gradually. A flaky blocking CI gate on a stochastic system gets disabled, and the safety property goes with it — so the gate must be made non-flaky by pinning, or demoted to a scheduled job that cannot be silently bypassed by disabling a blocking check. State which of the two is chosen at implementation time.*

**FR-8.4 [MUST]** — Given any user-facing output containing a derived clinical claim, when rendered, then it distinguishes computed from human-confirmed.

**FR-8.5 [MUST]** — Given any user-facing output, when rendered, then a static check against a prohibited-phrase list runs over the output templates and asserts none match. Additionally, as a documented pre-release review checklist item with a named reviewer, output is confirmed to contain no diagnostic assertion, no treatment recommendation, and no attack-risk prediction or forecast beyond what the static phrase list can catch.
*Split into an assertable static check and a documented review item, because "contains no diagnostic assertion" in general is not string-matchable.*

**FR-8.6 [MUST]** — Given the prodrome pattern surface, when a repository-wide search runs for its named identifiers, then zero matches are found in any layer.
*Deferred entirely per D6. Recorded here so its absence is deliberate and testable — via a repository-wide identifier search, which is assertable — rather than an oversight.*

---

## 9. Explicitly out of scope for steps 0–4

| Item | Where it lands |
|---|---|
| Phase annotation and retrospective relabelling | Step 5 |
| Medication-response inference as a shipped user-facing feature | Step 5 — the `boundary_rule` value exists in the schema; the behaviour is unspecified |
| Correlation and trigger analysis | Step 6. A research bet, not a claim — the M-sense RCT found no incremental benefit over a bare diary |
| Prior-authorisation packet | Step 7. FR-3.30 records its gate requirement now so it is not lost |
| Prodrome pattern surface | Deferred entirely, D6. Revisit with data once step 5 has run |
| App Intents and Android AppFunctions | Deferred. FR-0.5 and the tech spec's layout rules keep the path open |
| Menstrual cycle import | Deferred. No maintained plugin exists; needs a custom plugin |
| Multi-profile or caregiver accounts | Not planned |
| Community or social features | Not planned |
