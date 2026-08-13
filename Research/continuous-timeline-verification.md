# Research — Continuous-Timeline Model: Verification & Clinical Grounding

**Independent verification · August 2026**
Status: findings verified where marked; conducted by an independent Opus reviewer tasked with verifying or refuting HeadState's central architectural claim and gathering clinical grounding. See `qa-verification-report.md` for the broader independent-check process.

---

> ### Headline
> The central claim — that every existing migraine app models data as **discrete attacks** while HeadState uniquely models a **continuous symptom timeline** with attacks/episodes as derived, recomputable views — is **OVERSTATED and, in its strongest form, WRONG.**
>
> **Three counterexamples defeat the claim as stated:**
> 1. **N1-Headache (Curelator)** is a mandatory daily diary — symptoms logged every day, attack or not.
> 2. **Migraine Insight** ships an explicit continuous "pain level tracking" mode for chronic migraine, with open-ended episodes.
> 3. **The LUMC/Leiden e-headache diary** (Cephalalgia 2025, psychometrically validated) already IS this architecture: daily entry regardless of headache status, with attacks/days derived by an automatic algorithm.
>
> The claim **survives only in a narrowed form**: covering retrospective phase linkage (attaching an earlier observation to a later attack) and pharmacological attenuation modelling (a symptom-free period may be suppressed, not genuinely clear). No app for which a data model could be documented, and no reviewed literature, does either of these — 8 of 21 apps surveyed are unverifiable on phase linkage and 5 of 21 on attenuation modelling; see §1 and §4.

---

## 1. Per-app verdict matrix

Legend: **S** = supports · **P** = partial · **N** = does not support · **U** = unverifiable

Capability columns:
(a) non-attack-day standalone symptom logging
(b) one multi-day/multi-week record as ONE record, or an "ongoing" concept
(c) retrospective reclassification of an earlier symptom as a phase of a LATER attack
(d) modelling that a symptom-free period may be pharmacologically suppressed rather than genuinely clear

| App | (a) | (b) | (c) | (d) |
|---|---|---|---|---|
| Migraine Buddy | P | P–S | P | N |
| N1-Headache | S | N | N | N |
| Bearable | S | N (no episode object at all) | N | N |
| Migraine Insight | S | S | U | N |
| Manage My Pain | U | U | U | U |
| iHeadache | N | U | N | N |
| Migraine Monitor | U | U | U | U |
| Ctrl M Health | U | U | U | U |
| M-sense Migräne | P–S | U | U | N |
| Migraine Trail | N | N | N | N |
| Claru | P | U | N | N |
| Migra | U | U | U | U |
| MigrAid | N | P | N | N |
| MigraPulse | N | N | N | N |
| heyvie | S | U | U | N |
| CEFALY CeCe | U | U | U | U |
| Guava | S | N | N | N |
| Flaredown | S | N | N | N |
| Visible | U | N | N | N |
| Daylio | S (generic) | N | N | N |
| Apple Health | S | N | N | N |

---

## 2. Migraine Buddy — what the features ACTUALLY do

Marketing does not survive contact with the help center. Both flagship "continuous" features are more discrete record types than the strongest form of the claim assumed — but more capable than the premise assumed.

**"Chronic migraine" mode is a long-duration attack record, not a timeline.** Help text: "Open a daily, weekly or monthly record: you can test different record durations and adopt the most comfortable for you," then "add notes in the Notes section with the time stamp feature, to account for changes in pain intensity," and "press 'Remind me later' so that your ongoing record is saved and kept open for later changes."

→ **Migraine Buddy CAN represent a multi-week continuous episode as ONE record with an open/ongoing state.** The premise that "a multi-week continuous migraine gets recorded as several separate attacks" is **NOT true** of Migraine Buddy if the user uses this mode.

**The real weakness is that intra-episode variation lives in free-text timestamped notes, which are unstructured and unqueryable. That is the defensible criticism, not "it splits the episode."**

**Prodrome/postdrome are attack TYPES, not phases of an attack.** "Select the type of attack you wish to record, such as migraine, tension-type headache, cluster headache, prodrome and postdrome," and users can "add your own attack types." So a prodrome record on a non-attack day is possible (hence **P** on (a)) — but it is a **sibling record**, not a child of a later attack. No linkage, no phase graph.

**Partial (c) exists.** Via Migraine Impact Reports you can "Change Migraine Attack by prodrome or postdrome" on an existing calendar record — retrospective reclassification of a record's TYPE. What is missing is attachment to a specific later attack.

**Records are constrained not to overlap:** "Be careful of creating overlapping records when editing the Start time or End time of your attacks." This is the clearest confirmation of an interval-exclusive, attack-centric schema.

---

## 3. Where the claim genuinely holds

- **(c) Retrospective phase reassignment — linking an earlier observation to a later attack: no app for which a data model could be documented does this.** 8 of 21 apps surveyed are unverifiable on this capability (column (c) in the matrix above), not confirmed absent. A 2026 comparison of nine trackers concludes no tracker addresses prodrome/postdrome phases or ongoing multi-day attack logging. **CONFIRMED** as a real gap among the apps that could be checked.
- **(d) Pharmacological masking / "suppressed not absent" days: no app for which a data model could be documented models this, and no reviewed literature proposes it.** 5 of 21 apps surveyed are unverifiable on this capability (column (d) in the matrix above), not confirmed absent. Zero positive hits across every app and diary-methodology paper that could be checked. **The single most defensible novelty claim in the architecture, among what could be verified.**
- **General-purpose consumer trackers have the timeline but no episode layer.** Bearable, Guava, Flaredown, Daylio and Apple Health all support arbitrary standalone symptom observations, but none derive episodes. Note: Bearable is **not** freely timestamped — it buckets into four fixed periods: "Symptom and/or Factor time periods (pre, am, mid, and pm)." So, restricted to this class of general-purpose trackers, the true differentiator is **timeline + derived episodes in one system**, not the timeline alone. That does not extend to clinical-grade e-diaries: the LUMC/Leiden e-headache diary already combines a daily observation timeline with algorithmically derived days, so "timeline + derived episodes" is not itself unclaimed against that comparator — see §2.8 of the strategy brief on the resulting execution-gap framing.

---

## 4. Evidence gap — flag prominently

Reddit/forum user complaints could **NOT** be retrieved (search proxy rejected reddit-scoped queries; direct HTTP fetch blocked by proxy policy). **Negative user evidence is therefore UNVERIFIED, not absent.**

Manage My Pain, Migraine Monitor, Ctrl M Health, CeCe, Migra and Visible need hands-on testing — their public docs do not describe data models. **Do not report the U rows as N.**

---

## 5. Clinical grounding

### ICHD-3 1.4.1 Status migrainosus — CONFIRMED

Description: "A debilitating migraine attack lasting for more than 72 hours."

Criteria: A. A headache attack fulfilling criteria B and C. B. Occurring in a patient with 1.1 Migraine without aura and/or 1.2 Migraine with aura, and typical of previous attacks except for its duration and severity. C. Both of: 1. unremitting for >72 hours; 2. pain and/or associated symptoms are debilitating. D. Not better accounted for by another ICHD-3 diagnosis.

**The critical note: "Remissions of up to 12 hours due to medication or sleep are accepted."** Comment: status migrainosus "may often be caused by medication overuse" — if MOH criteria are met, code MOH + migraine type, NOT status migrainosus.

### ICHD-3 already has the medication-interruption rule — adopt it, don't invent one. CONFIRMED. Highest-value clinical finding.

Not in the 1.1 chapter (which is why it's easy to miss) — it is in ICHD-3 "Definition of Terms":

- **Frequency of attacks:** IHS Guidelines "recommend … to count as distinct attacks only those that are separated by at least 48 hours headache-free."
- **Duration of attack:** "If an attack of migraine is successfully relieved by medication but symptoms recur within 48 hours, these may represent a relapse of the same attack or a new attack."
- **Attack of headache (or pain):** "Headache (or pain) that builds up, remains at a certain level for minutes, hours or days, then wanes until it has resolved completely."
- **Headache days:** "Number of days during an observed period of time (commonly one month) affected by headache for any part or the whole of the day."
- **1.1 Note:** "When the patient falls asleep during migraine and wakes up without it, duration of the attack is reckoned until the time of awakening."

**Engineering implication:** three distinct already-standardised thresholds; implement all three as named constants rather than one house rule:

- **12 h** — max remission tolerated WITHIN one status migrainosus episode
- **48 h headache-free** — the boundary separating two DISTINCT attacks
- **<48 h recurrence** — deliberately AMBIGUOUS in ICHD-3 ("may represent a relapse … or a new attack"). ICHD-3 does not resolve it. **This is where HeadState can legitimately add value: store the observation stream, then expose BOTH counts (relapse-collapsed and relapse-split) as recomputable derived views. Standards-compliant rather than invented.**

⚠️ These quotes were obtained through a summarising fetch layer (direct HTTP to ichd-3.org blocked in that environment). Content is confirmed and consistent across two independent sources; **check exact wording against the printed ICHD-3 / Cephalalgia 2018;38(1):1–211 before quoting in a regulatory or clinical document.**

### Migraine phases and prodrome predictability — CONFIRMED, but temper the claim

Best direct evidence (Journal of Pain Research 2018, 34 patients, 229 attacks):

- 69% of attacks (158) had premonitory symptoms; only **27.5% were recognised before pain onset**
- **67.6%** of patients predicted ≥1 attack, but only **35.3% were "good predictors"** (>50% of attacks)
- **Positive predictive value 85.1%** (only 11 false positives)
- Mean lead time **10 h 38 min** before pain onset
- Most predictive: "Photophobia and drowsiness … other significant symptoms were yawning, increased thirst, blurred vision, and nausea"

Authors' caveat: patients rarely mis-identify premonitory symptoms, but only a minority can predict most attacks — limiting real-world prodromal-treatment potential.

Foundational: Giffin et al., Neurology 2003, electronic diary study of premonitory symptoms. Mechanism review: Karsan & Goadsby, Nat Rev Neurol 2018.

**Defensible tracker claim:** high specificity (PPV ~85%) with ~10 h lead time means prodrome detection is actionable WHEN IT FIRES, but sensitivity is poor — so it must be a LOW-RECALL, HIGH-PRECISION alert, never a coverage metric. **This asymmetry strengthens the case for retrospective phase linkage: if only 27.5% of premonitory symptoms are recognised prospectively, retrospective reclassification is the only way to recover the other ~72%. That is the best clinical argument for the continuous model — lead with it.**

### Corticosteroids / masking — CONFIRMED, and it CUTS AGAINST the original framing

Woldeamanuel, Rapoport & Cowan, Cephalalgia 2015 (65-year systematic review + pooled analysis):

- Parenteral dexamethasone most common (56%), median single dose 10 mg (range 4–24 mg)
- **24 h recurrence: median ARR 30%** (range 6–48.2%), **NNT 3.3**
- **72 h recurrence: median ARR 11%** (range 6–48.6%), **NNT 9.1**
- Meta-analysis: "single-dose parenteral dexamethasone was associated with 11.1% ARR, NNT of 9, and 26% RRR for 72-hour headache recurrence"
- Status migrainosus specifically: 16 mg IV dexamethasone non-inferior to 900 mg IV valproate
- "with corticosteroid treatment, recurrent headaches become milder than pretreated headaches and later respond to nonsteroidal therapy"
- Outpatient bridge dosing: "oral dexamethasone at 4 mg twice daily for 4 days"

**Flag plainly:** steroids REDUCE recurrence (they do not merely defer it), and post-steroid recurrent headaches are MILDER, not rebound-amplified. Recurrence still occurs in a large minority — pooled ARR is 11–30%, not 100% — so a "steroid clears → migraine returns unchanged" masking model is **NOT supported by the evidence.** Model attenuation as **attenuated symptom expression under active pharmacology** (a confounder on severity/observability), NOT as a suppressed-then-restored attack. **Overclaiming here is the most likely place for a clinical advisor to reject the design.**

### E-diary data models — CONFIRMED, and the "ongoing event" pattern is already standard

npj Digital Medicine 2025, best practices for event-driven electronic diaries (EDeDs) in trials, describes the design treated as novel:

- Events with duration (migraine, sickle cell crisis) are captured via "opening" and "closing" mechanisms with start and end times
- The system must allow events to span across midnight, with edit checks confirming plausible event length
- **Migraine is the named example** of a condition where "only one event can occur simultaneously" — "the reporter must close the previous event before reporting a new one"
- Summary screens display which events remain OPEN
- End-of-day reminders capture an active "no events" confirmation — i.e. explicit negative observations

Also relevant but UNVERIFIABLE: the 2014 JMIR systematic review of commercial headache diary apps (text could not be retrieved — ScienceDirect robots-disallowed, JMIR returned empty). Cited for completeness only: https://mhealth.jmir.org/2014/3/e36/

---

## 6. Prior-auth counting under a continuous model

**Answer: 21 headache days, not N attacks. Counting as attacks harms the patient.**

Botox (chronic migraine), Cigna criteria — verbatim: "greater than or equal to 15 headache days per month with headache lasting 4 hours per day or longer." No definition of "headache day" is supplied, no diary is required, and there is **no mention of continuous headache, status migrainosus, or refractory/intractable headache.** Benefit is judged as "a reduction in the overall number of migraine days per month or a reduction in number of severe migraine days per month."

CGRP mAbs/gepants: episodic ≥4 (some payers ≥6/≥8) migraine days/month; chronic ≥15 headache days/month with ≥8 meeting migraine criteria. Diaries recommended, not mandated — "Reviewers should not have to calculate frequency from clinical notes." **No payer policy reviewed contains special counting provisions for status migrainosus, refractory migraine, or continuous headache.**

A 21-day continuous episode where each day has ≥4 h of headache = **21 headache days**, clearing the ≥15 threshold outright. Reporting it as "1 attack" (or even "3 attacks") would fail every threshold in every policy reviewed. **HeadState must default its payer-facing export to day-counts. Attack counts are clinically interesting and reimbursement-toxic.**

---

## 7. 2026 Delphi consensus — how it collapses a continuous episode

De Loose, Paemeleire, Deligianni et al., *J Headache Pain* 2026;27:135. Final consensus definitions of a migraine day:

1. **Without acute treatment (83%, 15/18):** "Headache lasting ≥ 4 h having at least two out of four headache characteristics, regardless of the presence of any associated symptom or at least one out of four headache characteristics combined with one associated symptom."
2. **With acute treatment (94%, 17/18):** "Lasting ≥ 30 min, regardless of drug (class) taken, the presence of any headache characteristic or associated symptom."
3. **In presence of aura (83%, 15/18):** "An aura, fulfilling the ICHD-3 criteria for migraine with aura, is experienced regardless of the presence of headache."
4. **Overnight remission (100%):** if a patient "goes to sleep with a migraine and wakes up headache-free the day after, this should be counted as one migraine day" — deliberately DEPARTING FROM ICHD-3's "reckoned until the time of awakening."

**Applied to the 21-day case:** Delphi and the payer criterion are two different counts, computed under two different rules, and must be reported separately rather than treated as the same number. Delphi yields a **migraine-day** count: the 30-minute treated-day threshold is decisive, so medication-induced gaps mostly do not erase migraine days — a treated day needs only 30 minutes of headache to count, regardless of drug class. The Cigna payer criterion in §6 is expressed instead in **headache days**, independently defined as headache lasting **≥4 hours per day**, with no reference to Delphi's 30-minute floor at all. A day that clears the Delphi treated-day threshold does not automatically clear the payer's 4-hour qualifier, so the two counts can diverge on the same underlying data. Under either rule, days that are ENTIRELY symptom-free because of medication are the ones at risk of dropping out, and neither rule recovers them on its own — that is exactly what attenuation modelling is for. So the Delphi collapse is **21 migraine days (minus any fully symptom-free days)**, computed and reported alongside, not instead of, the §6 headache-day count.

**Rationale and scope:** the panel's driver was that "substantial variability remains across clinical trials" — **23 phase III trials used 12 different definitions.** Intended uses include clinical trials, real-world practice, and "automated e-diary labeling in digital headache calendars" — i.e. the rules are explicitly written to be executed by software like HeadState. They do NOT address regulatory, reimbursement or prior-auth contexts. Open questions the panel left: feasibility testing in trials and practice; e-diary design (diagnostic vs simplified follow-up modes); whether aura-only days mask treatment effects in composite endpoints.

**Genuine gap — CONFIRMED:** the Delphi paper does NOT address headaches persisting >24 h across consecutive calendar days, status migrainosus, midnight-spanning attacks, or same-day recurrence after treatment. **The most authoritative 2026 statement on counting migraine days is silent on exactly the case HeadState is built for. This is the strongest standards-level argument for the product — stronger than the app-differentiation argument, which does not hold.**

⚠️ A **Correction** to this paper exists (10.1186/s10194-026-02407-x) — it was NOT retrieved. Verify the numbers against the corrected version before quoting.

---

## 8. Bottom line

**Drop:**
- "No app lets you log symptoms on non-attack days" — false (N1-Headache, Bearable, Migraine Insight, heyvie, M-sense, Apple Health)
- "No app can hold a multi-week episode as one record" — false (Migraine Buddy's monthly record, Migraine Insight's open episodes)
- "Derived episodes over a timeline is a new model" — false (LUMC e-diary, npj EDeD best practices, and the Delphi paper's own automated-labeling use case)

**Keep and lead with:**
- No app for which a data model could be documented links an earlier observation to a phase of a later attack (8 of 21 apps surveyed unverifiable on this, not confirmed absent) — and the 27.5% prospective-recognition figure proves retrospective reclassification is the ONLY way to capture most premonitory data
- No app or paper for which a data model could be documented models pharmacologically attenuated days (5 of 21 apps surveyed unverifiable on this, not confirmed absent)
- ICHD-3 leaves <48 h recurrence deliberately ambiguous while the 2026 Delphi consensus is silent on multi-day continuity — so a system that stores raw observations and recomputes BOTH readings is filling a real standards gap

**Fix before build:**
- Default payer exports to DAY COUNTS (21, not N)
- Implement 12 h / 48 h / <48 h as named ICHD-3-sourced constants rather than house rules
- Reframe attenuation as severity/observability attenuation, because the steroid evidence (ARR 11–30%, milder recurrences) contradicts a suppress-then-restore masking model

---

## See also

- `competitive-landscape.md` — full app-by-app market survey feeding the verdict matrix above
- `clinical-regulatory.md` — broader clinical and regulatory grounding
- `qa-verification-report.md` — independent verification process this document is part of
- `product-architecture.md` — how the narrowed claim is implemented in the data model
- `assistant-integration.md` — how derived views surface in the assistant layer
- `../headstate-strategy-brief.md` — top-level strategy synthesis

---

## Sources

https://ichd-3.org/1-migraine/1-4-complications-of-migraine/1-4-1-status-migrainosus/ · https://ichd-3.org/definition-of-terms/ · https://ichd-3.org/1-migraine/1-1-migraine-without-aura/ · https://link.springer.com/article/10.1186/s10194-026-02383-2 · https://link.springer.com/article/10.1186/s10194-026-02407-x · https://journals.sagepub.com/doi/10.1177/0333102414566200 · https://assets.bmctoday.net/practicalneurology/pdfs/PN0522_CF6_StatusMigrainosus.pdf · https://www.dovepress.com/to-what-extent-are-patients-with-migraine-able-to-predict-attacks-peer-reviewed-fulltext-article-JPR · https://www.neurology.org/doi/10.1212/01.WNL.0000052998.58526.A9 · https://www.nature.com/articles/s41582-018-0098-4 · https://www.nature.com/articles/s41746-025-02165-1 · https://journals.sagepub.com/doi/10.1177/03331024251359222 · https://static.cigna.com/assets/chcp/pdf/resourceLibrary/prescription/Botox.pdf · https://www.artificerhealth.com/blog/cgrp-prior-authorization-guide/ · https://n1-headache.com/patients/faq/ · https://migrainebuddy.com/help-center/category/how-to-guides/ · https://migrainebuddy.com/help-center/category/using-migraine-buddy/ · https://apps.apple.com/us/app/migraine-insight-tracker/id1364291643 · https://bearable.app/support/common-questions/track-multiple-times-per-day-in-the-health-check-in/ · https://guavahealth.com/article/tracking-and-visualizing-symptoms-in-guava · https://www.halsteadresearch.com/insights/best-migraine-trackers-2026 · https://migrainetrail.com/blog/best-migraine-tracker-apps-2026 · https://www.getmigra.com/blog/best-migraine-tracker-app-2026/ · https://migrapulse.com/ · https://apps.apple.com/us/app/migraid-migraine-tracker/id6757091931 · https://apps.apple.com/us/app/heyvie-migraine-coach/id1591462187 · https://www.claruapp.com/ · https://m-sense.de/kopfschmerztagebuch/ · https://www.imedicalapps.com/2012/07/iheadache-app-clinical-diary-patients/ · https://migrainemonitor.com/faq/ · https://mhealth.jmir.org/2014/3/e36/
