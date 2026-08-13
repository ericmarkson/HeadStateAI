# Research — Independent QA & Verification Report

### Independent adversarial review, August 2026 — 31 claims checked, 6 corrections required

> **Status note:** The corrections documented below have **already been applied** to [`../headstate-strategy-brief.md`](../headstate-strategy-brief.md) and to [`clinical-regulatory.md`](clinical-regulatory.md). This document is the **audit trail** for those corrections and the **source of record for the URLs** behind each one — it is not an outstanding to-do list. Nothing below requires further action on its own; where action items remain, they are tracked in the strategic critique and overall verdict sections at the end.

See also: [`competitive-landscape.md`](competitive-landscape.md), [`assistant-integration.md`](assistant-integration.md), [`clinical-regulatory.md`](clinical-regulatory.md), [`product-architecture.md`](product-architecture.md), [`continuous-timeline-verification.md`](continuous-timeline-verification.md).

---

## Reviewer's bottom line

The competitive-landscape and product-architecture work was good. The clinical/regulatory section — the part the brief was most confident about — contained two errors that would cause real harm if acted on, and the research missed a randomised controlled trial that directly attacks one core differentiator. The brief's own "Confidence and Gaps" section listed PROMIS status, the FDA guidance characterisation, and the RTM codes under "Well established." All three were wrong or materially incomplete. That is a laundering problem, not a rounding error.

---

## Verification table — 31 claims

| # | Claim | Verdict | Note |
|---|-------|---------|------|
| 1 | ICHD-3 1.1 duration 4–72h untreated | CONFIRMED | Verbatim match |
| 2 | ICHD-3 "≥2 of 4" (unilateral / pulsating / mod-severe / aggravated by activity) | CONFIRMED | Verbatim; ICHD-3 wording is "aggravation by **or causing avoidance of**" routine physical activity — the brief's paraphrase dropped the avoidance clause, minor schema implication |
| 3 | ICHD-3 nausea/vomiting OR (photophobia AND phonophobia — both) | CONFIRMED | The tricky logical structure was stated correctly. Criterion D: "at least one of the following: nausea and/or vomiting; photophobia and phonophobia." Commonly botched; noted as done well |
| 4 | ≥5 lifetime attacks | CONFIRMED | Criterion A |
| 5 | MOH: triptans/ergots/opioids/combination ≥10 days/mo; simple analgesics ≥15; >3 months | PARTIALLY CORRECT | Numbers right, three gaps, see Correction 3 |
| 6 | CGRP mAb PA ≥4 migraine days/month | CONFIRMED | Cigna IP0050: "4 or more migraine headache days per month (prior to initiating a migraine-preventative medication)"; also requires ≥3-month failure of two preventives from different classes — the brief omitted the trial requirement for CGRP while stating it for Botox |
| 7 | Botox: ≥15 headache days with ≥8 migraine days + 3 failed prophylactics | CONFIRMED | Verbatim in Centene/Coordinated Care WA.PHAR.232 |
| 8 | Botox continuation ≥50% reduction | CONFIRMED | "maintained a 50% reduction in monthly migraine headache frequency from baseline" |
| 9 | PROMIS public domain, free with attribution, "the safe path" | WRONG | Highest materiality item, see Correction 1 |
| 10 | HIT-6 proprietary, licensed via QualityMetric | CONFIRMED | © 2001, 2015 QualityMetric Inc.; licensing gate is real |
| 11 | FDA issued revised General Wellness + CDS guidance January 2026 | CONFIRMED | Both final, 6 Jan 2026, issued without public comment |
| 12 | Brief's characterisation of the device/non-device line | WRONG IN TWO WAYS | See Corrections 2 and 4 |
| 13 | CMS added RTM codes eff. 1 Jan 2026; 98985 = 2–15 days; 98979 = first 10 min | PARTIALLY CORRECT | 98979 confirmed; 98985 is musculoskeletal-specific, see Correction 5 |
| 14 | RTM accommodates self-reported non-physiologic data | CONFIRMED | Structural point holds |
| 15 | RTM device-supply codes are respiratory (98976) / MSK (98977) only | PARTIALLY CORRECT | Omits 98978 (cognitive behavioural therapy monitoring), arguably the closest existing fit and worth a specific look |
| 16 | FTC HBNR: 60 days to individuals; simultaneous FTC notice at ≥500 | CONFIRMED | Refinement: fewer than 500 → report to FTC within 60 days of calendar year end; media notice triggered at 500 residents of a single state |
| 17 | Not HIPAA-covered as pure DTC; become a BA on contracting with a clinic | CONFIRMED | Legally sound as stated |
| 18 | HealthKit / Health Connect inaccessible from a PWA | CONFIRMED BUT BADLY UNDERSTATED | See What Was Missed item 4 |
| 19 | MCP spec revision 2026-07-28, streamable HTTP, RFC 8707 | CONFIRMED | Current spec |
| 20 | Claude: any user can add arbitrary remote MCP server by URL, no review | CONFIRMED | Free/Pro/Max/Team/Enterprise. Nuances: Free tier capped at one custom connector; Team/Enterprise adds are Owner-only — so the "power-user soft launch" audience is effectively paid individual users |
| 21 | ChatGPT Health exists with named launch connectors | CONFIRMED, LIST INCOMPLETE | Announced 7 Jan 2026; US-wide rollout 23 Jul 2026, 18+, web + iOS. Named: Apple Health, Function, MyFitnessPal, Weight Watchers, AllTrails, Instacart, Peloton, plus b.well, Epic, Oracle Health, One Medical. The brief's five are all real |
| 22 | "A real, named pathway for exactly your category" into ChatGPT Health | UNVERIFIABLE, STATED AS FACT, IS INFERENCE | No documented third-party health-connector submission path exists. Launch connectors are bespoke brand partnerships. General Apps SDK guidelines list PHI as Restricted Data: "Do not collect, solicit, or process… Protected health information" |
| 23 | Apps SDK guidelines say not to request health records | CONFIRMED, ACTUALLY STRICTER THAN STATED | Restricted-Data prohibition, not a soft discouragement |
| 24 | Migraine Buddy owned by Aptar Digital Health | CONFIRMED | Aptar acquired Healint, announced 21 Feb 2024 |
| 25 | Migraine Buddy: 1M+ installs, 4.8★/66k, 3.5M community, Oura Jan 2026 | CONFIRMED | Play Store shows 4.8★, 66K reviews, 1M+ installs, developer "Aptar Digital Health," "3.5 million migraine sufferers." Oura collaboration announced 27 Jan 2026 |
| 26 | Migraine Trail — voice-first, iOS, closest competitor | CONFIRMED | Live; marketing copy matches the brief's quote |
| 27 | Claru (Reelpop) — iOS + Android, AI voice, "#1" is paid PR | CONFIRMED | Both stores live. "Halstead Research 2026 Rankings" is an openpr.com press release; the brief's skepticism is correct |
| 28 | Migra — on-device Apple Foundation Models, altimeter, Watch, MIDAS/HIT-6 | CONFIRMED | Also confirms Apple Health bidirectional sync and no-account/no-cloud posture |
| 29 | N1-Headache / Curelator exists; corporate health doubtful | PARTIALLY VERIFIABLE | App live on both stores. No post-2019 company news found; "possibly stagnant" is fair but remains inference — the brief labelled it correctly |
| 30 | Retention: 70–80% never return after day one; 3–4% day-30; 70% within 100 days | CONFIRMED AS RESTATEMENT | D1 health-app retention 20–30% (→70–80% complement); fitness D30 3–4%; scoping review median 70% discontinuation <100 days. Note the 3–4% figure is *fitness*, and Business of Apps' underlying number is "3% by day 30 in 2023" — imprecise but not misleading |
| 31 | Digital-therapeutics-for-migraine market $2.09B→$4.82B | UNVERIFIABLE | Brief already flagged it; fine |

---

## Corrections, ranked by materiality

### 1. PROMIS is NOT public domain and NOT free for commercial use. CRITICAL.

The brief called PROMIS "Public domain, NIH-developed, free with attribution. The safe path," put it in the MVP ("PROMIS-based impact scoring"), and listed its status under "Well established." All wrong.

HealthMeasures Terms of Use: Copyright "©2006–2017 PROMIS Health Organization" — the phrase "public domain" does not appear anywhere in the terms. Free "without licensing or royalty fees" only for individual research or individual clinical use. "Commercial Users must seek permission to use, reproduce, or distribute any HealthMeasures Measurement Systems and Instruments, regardless of purpose, at all times." Electronic administration requires HealthMeasures Electronic Administration Permission (HEAP), with a review fee; only non-commercial single research studies are exempt. "Integration into proprietary technology requires written permission."

Consequence: the central licensing recommendation inverts. PROMIS is not meaningfully safer than HIT-6 for a commercial app — all four instruments require permission, PROMIS included. Eric would have shipped an unlicensed instrument believing he was on the conservative path. The "legal budget for two specific questions" must become three, and PROMIS/HEAP should be the FIRST letter written, not MIDAS.

URLs:
- https://www.healthmeasures.net/images/PROMIS/Terms_of_Use_HM_approved_1-12-17_-_Updated_Copyright_Notices.pdf (mirror: https://promishungary.org/wp-content/uploads/2019/07/Terms_of_Use_HM_approved_1-12-17_-_Updated_Copyright_Notices.pdf)
- https://www.healthmeasures.net/explore-measurement-systems/promis/obtain-administer-measures

### 2. The FDA CDS exclusion does not apply to patient-facing software at all. CRITICAL.

The brief built the entire regulatory posture — the five "tipping" items, the "clinician must be able to independently review the basis," the "transparent publicly accepted sources" test — on the CDS guidance. But: "The Non-Device CDS exclusion applies only to software intended for use by healthcare professionals." HeadState as confirmed (patient-first DTC) is outside it. The statutory carve-out at FD&C §520(o)(1)(E) is HCP-directed by construction.

Consequence: the operative framework is the General Wellness guidance plus the plain device definition and existing digital-health enforcement policy — not CDS. The conclusion ("almost certainly non-device as a structured-capture-and-summarisation tool") probably survives, but the reasoning doesn't, and the reasoning is what a lawyer or investor will interrogate. The CDS analysis becomes relevant only if and when the clinician export becomes a clinician-facing product — which is exactly the "one-way door" the brief flagged elsewhere without connecting the two.

URLs:
- https://meddeviceguide.com/blog/fda-clinical-decision-support-cds-guide
- https://www.mwe.com/insights/fda-issues-long-awaited-final-clinical-decision-support-software-guidance/

### 3. Three gaps in the MOH threshold spec — one is a functional bug. HIGH.

Numbers are right, but the brief called the thresholds "published and unambiguous" and proposed a "rolling per-drug-class counter." A per-class counter will systematically miss ICHD-3 8.2.6: "Regular intake of any combination of ergotamine, triptans, non-opioid analgesics and/or opioids on a total of ≥10 days/month for >3 months without overuse of any single drug or drug class alone." A patient at 6 triptan days + 6 NSAID days has MOH and the counter says clear. Fix: track per-class AND a combined-classes ≥10 rolling total. Also:

- ICHD-3's term is "non-opioid analgesic"-overuse headache, not "simple analgesics." Use ICHD-3's label if claiming ICHD-3 fidelity in a provider report.
- ICHD-3 itself states the day counts "are based on expert opinion rather than on formal evidence." "Unambiguous" is the wrong word for a number surfaced to patients; the brief's own epistemic-honesty principle demands softer framing.
- ICHD-4 is in active development (ICHD-4 alpha 2020; "Work in Progress 1," Goadsby/Lipton/Terwindt et al., Cephalalgia 2024), and whether MOH survives as a diagnosis is explicitly under debate. Build thresholds as versioned config, not constants.

URLs:
- https://ichd-3.org/8-headache-attributed-to-a-substance-or-its-withdrawal/8-2-medication-overuse-headache-moh/8-2-6-medication-overuse-headache-attributed-to-multiple-drug-classes-not-individually-overused/
- https://ichd-3.org/8-headache-attributed-to-a-substance-or-its-withdrawal/8-2-medication-overuse-headache-moh/
- https://journals.sagepub.com/doi/10.1177/03331024241233937

### 4. The brief described the pre-2026 FDA posture while citing the 2026 revision. HIGH.

Brief: "outputs must be grounded in transparent, publicly accepted sources — not opaque proprietary logic," red line 1 "predicting attack risk," red line 4 "opaque proprietary scoring." The January 2026 revision moved in the opposite direction. The 2022 guidance said software that "identifies a risk probability or risk score" fails Criterion 3; the 2026 revision REMOVES that language and adds enforcement discretion for single recommendations, with a worked example of "software functions that predict risk of future cardiovascular events." The General Wellness guidance simultaneously expanded to cover non-invasive physiologic monitoring outputs intended solely for wellness uses.

Consequence: the strategic payoff the brief drew — "item 1 is a feature Migraine Buddy, Claru and Migraine Trail all ship. Your competitors are operating closer to that line than you would be… a reason not to feel behind" — is unsupported. FDA moved the line toward the competitors. "We don't guess" may still be a good product and trust position; it is no longer a regulatory-arbitrage position. Note analysts disagree on the transparency direction (Ropes & Gray and the FDA Law Blog read Criterion 4 as relaxed; others read the disclosure element list as expanded) — flag as contested, get counsel, don't restate either version as settled.

URLs:
- https://www.ropesgray.com/en/insights/alerts/2026/01/fda-adapts-with-the-times-on-digital-health-updated-guidances-on-general-wellness-products
- https://www.thefdalawblog.com/2026/01/a-busy-day-in-the-cdrh-neighborhood-updates-to-the-cds-and-general-wellness-guidance-documents/

### 5. CPT 98985 is a musculoskeletal code. MEDIUM–HIGH.

Official descriptor: "…device(s) supply for data access or data transmissions to support monitoring of musculoskeletal system, 2–15 days in a 30-day period." The brief presented 98985 as generically "materially lowering the bar," then two paragraphs later noted the respiratory/MSK body-system problem — without noticing that 98985 is itself one of those body-system codes. The lowered floor does not reach headache. So the reimbursement section's one piece of good news evaporates: there is still no neurology/headache RTM device-supply code, and the 2026 additions did not create one. 98979 (RTM treatment management, first 10 minutes) is confirmed and is not body-system-restricted. Also note companion codes 98984/98986 in the same family, and existing 98978 (CBT monitoring) which the brief omits.

Net: downgrade the RTM thesis from "promising, unverified" to "unavailable as described." That strengthens rather than weakens the brief's own conclusion, but the stated reason was wrong.

URLs:
- https://www.forcetherapeutics.com/blog/cms-updates-remote-therapeutic-monitoring-rtm-codes-in-the-2026-physician-fee-schedule
- https://www.nixonlawgroup.com/resources/cms-finalizes-2026-remote-monitoring-reimbursement-updates-what-changed-for-rpm-and-rtm

### 6. Downgrade "a real, named pathway for exactly your category" (ChatGPT Health). MEDIUM.

State as inference. Evidence: ChatGPT Health exists with named connectors, and general app submissions opened 17 Dec 2025. Absent: any documented route for a third party to become a *Health* connector. And the general guidelines bar processing PHI as Restricted Data. What OpenAI did say points somewhere the brief won't like — see What Was Missed item 4.

URLs:
- https://openai.com/index/introducing-chatgpt-health/
- https://developers.openai.com/apps-sdk/app-submission-guidelines
- https://openai.com/index/developers-can-now-submit-apps-to-chatgpt/

---

## What was missed — 8 items

### 1. The M-sense RCT — the single most important omission. Attacks differentiator "interictal capture" head-on.

*A Prescribed Digital Health App and Number of Migraine Days: A Randomized Clinical Trial* (2025; PMID 40591361). n=477, 12 weeks. Full M-sense Migräne app (trigger identification, correlation analysis, medication tracking — essentially the brief's insight layer) versus a control app containing only a headache and medication diary.

Result: 7.74→5.79 migraine days/month (full app) vs 7.76→5.41 (diary-only control). Between-group difference 0.38 days, not significant. Most secondary outcomes null. Investigators' interpretation: self-monitoring itself drove the improvement in both arms.

Why it matters most: the brief's interictal-capture differentiator rests on enabling trigger inference. The largest RCT of trigger-inference-on-top-of-a-diary found no incremental clinical benefit over a bare diary. You can still build it; you cannot claim clinical value without addressing this trial, and a headache-specialist advisor will raise it in the first meeting. It also reframes the paywall question constructively: if the diary is the active ingredient and the correlations are unproven, paywalling correlations is defensible and paywalling logging is not — the brief reasoned its way to nearly the opposite instinct. And it cuts against "would acquiring N1-Headache's trigger/protector methodology be faster" — probably don't pay for methodology whose incremental benefit failed to replicate.

URLs:
- https://pubmed.ncbi.nlm.nih.gov/40591361/
- https://migrainecollaborative.org/a-failed-clinical-trial-of-a-digital-app-for-migraine-where-to-from-here

### 2. A well-funded European competitor and an entire reimbursement pathway never mentioned.

M-sense Migräne (Newsenselab, Berlin) — VC-backed (High-Tech Gründerfonds, IBB Ventures), CE-marked, and Germany's 7th DiGA, i.e. prescribable and statutory-insurance-reimbursed. sinCephalea (Perfood) is a second migraine DiGA. The brief's "Clinical/enterprise" tier named Belgium and Estonia but omitted Germany entirely — while spending its whole reimbursement analysis on US RTM codes that turn out not to apply to headache. DiGA is the only jurisdiction on earth where a migraine app is already reimbursed at scale. A strategy-level omission, not a landscape gap. (It cuts both ways: M-sense holds a DiGA listing AND produced the null RCT above.)

URLs:
- https://healthcare-startups.de/die-siebte-diga-ist-da-m-sense-migraene-app/
- https://www.ibbventures.de/en/news/newsenselab-financing

### 3. There is now a published expert-consensus definition of a "migraine day."

The brief cited only van der Arend 2023. Better: De Loose, Paemeleire, Deligianni et al., "What's a migraine day? Towards a consensus definition for clinical care and research – a Delphi panel-based study," J Headache Pain 2026;27:135. Consensus positions: untreated day = headache ≥4h with ≥2/4 characteristics, or ≥1/4 characteristic plus one associated symptom (83%); treated day = ≥30 min regardless of drug class or characteristics (94%); aura day = any day with ICHD-3-compliant aura regardless of headache (83%). Explicitly intended for implementation in digital headache calendars. Resolves the brief's open question, and the ≥30-minute treated-day rule has direct schema consequences.

URL:
- https://link.springer.com/article/10.1186/s10194-026-02383-2

### 4. Apple already ships the zero-app version of the assistant differentiator — and the PWA decision blocks the only realistic route into ChatGPT Health.

HealthKit has a first-class `headache` category type and a documented Symptom Type Identifiers family. Shortcuts has a "Log Health Sample" action. So a user can already say "Hey Siri, log a headache," write it to Apple Health, and — because ChatGPT Health's flagship connector is Apple Health — query it conversationally in ChatGPT. Zero third-party app. Fierce Healthcare quotes OpenAI: "if an app integrates with Apple Health, ChatGPT can use the data it makes available there, with permission."

Two consequences the brief never drew: the "nobody is assistant-reachable" white space is narrower than claimed — a crude but free substitute ships in the OS. And the documented path into ChatGPT Health for a small developer runs through HealthKit — which requires native — which the PWA decision forecloses. The brief said the assistant framing "dissolves most of the technical blockers" caused by the PWA; on this specific path it does the reverse. Migra already has bidirectional Apple Health sync and is therefore already inside that pipe.

URLs:
- https://developer.apple.com/documentation/healthkit/hkcategorytypeidentifier/headache
- https://www.fiercehealthcare.com/ai-and-machine-learning/openai-makes-health-chatgpt-widely-available-moving-deeper-consumer-health

### 5. Android's real assistant surface is AppFunctions, not Gemini Spark — and it's native-only.

The brief treated Android as "Tasker + AutoVoice, poor" and deferred Gemini Spark. It missed Android AppFunctions — an Android 16 platform feature plus Jetpack library, positioned by Google as the on-device counterpart to MCP, letting Gemini invoke app capabilities directly; broadening in Android 17. Requires a native Android app. (Gemini Spark did gain third-party/MCP support 30 Jun 2026, which the brief roughly got right.) So on both platforms, first-class assistant reachability is converging on native app manifests (App Intents / AppFunctions), with remote MCP as the web-only fallback. The Phase-2 Capacitor deferral is therefore later than the platform trend supports.

URLs:
- https://9to5google.com/2026/02/25/android-appfunctions-gemini/
- https://9to5google.com/2026/06/30/gemini-spark-apps-more/

### 6. A legal regime with a private right of action, omitted entirely.

The brief treated FTC HBNR + FTC Act §5 as the whole DTC picture and called it "materially lighter than HIPAA." Missing: state consumer health data laws — Washington's My Health My Data Act (in effect since Mar 2024, applies to WA residents' consumer health data, requires separate opt-in consent for collection AND separate authorisation for sharing, mandates a distinct Consumer Health Data Privacy Policy, and — uniquely — is enforceable by private action under the WA Consumer Protection Act), plus Nevada SB 370 and Connecticut's CTDPA health provisions. For a DTC symptom tracker these are more operationally binding and more litigation-exposed than HBNR, and they shape onboarding consent UX in v1. HBNR is a breach-response obligation; MHMDA is a product-design obligation.

URLs:
- https://www.dwt.com/blogs/privacy--security-law-blog/2024/04/my-health-my-data-act-washington-nevada
- https://fpf.org/wp-content/uploads/2023/06/NV_WA-Comparison-Chart-FINAL.pdf

### 7. Naming and TLD — the brief was silent.

Good news: no trademark registration or application matching "HEADSTATE" or "HEAD STATE" turns up in a Trademarkia search of major registries. The mark looks available.

Risk to clear: HEADSPACE / HEADSPACE HEALTH is a registered family in overlapping classes (health/wellness apps, mental health). "HeadState" at headstate.health is a head-prefixed, two-syllable health-app mark in adjacent goods. Plausible likelihood-of-confusion exposure, and Headspace maintains an active enforcement posture (TTAB oppositions on file). Not a blocker; a $1–2k clearance opinion before spending on brand.

`.health` is open registration under Registry Services LLC / dotHealth — no eligibility restriction, no credential verification, and its ICANN approval was opposed by multiple governments and ALAC precisely because the string implies legitimacy it doesn't verify. Practical implications: `.health` confers no trust signal, renewal pricing is premium-tier, and new-gTLD deliverability/link-filtering friction is real for a product whose acquisition depends on email and shared PDF links.

URLs:
- https://trademark.justia.com/976/99/headspace-97699638.html
- https://icannwiki.org/.health

### 8. Smaller landscape items.

MigraPulse (migrapulse.com, App Store id6758339023) markets "calm, voice-first migraine tracking" — a sixth entrant in the exact category, not named. Neura Health raised an $11.4M Series A (Sept 2025, backed by the AHA's Go Red for Women fund) — the brief described it as a competitor without noting it is now the best-funded US-side player adjacent to this lane. Migraine Buddy still has no Apple Health sync (top request on its public feedback board) — a real, current, exploitable gap the brief didn't spot, and one only a native build can exploit.

URLs:
- https://apps.apple.com/us/app/migrapulse/id6758339023
- https://www.globenewswire.com/news-release/2025/09/29/3157649/0/en/Neura-Health-Secures-11-4-Million-Series-A-to-Fuel-National-Expansion-of-Comprehensive-Tech-Enabled-Neurology-Care-Model.html
- https://131xmm.nolt.io/287

---

## Strategic critique

Central claim verified: no migraine tracker ships an MCP server or assistant connector. The reviewer could not find one. Searches across MCP registries, app listings and 2026 comparison roundups surface Oura and Whoop MCP servers (all community-built, none official) and zero headache/migraine servers. Migra's own competitive roundup confirms no app mentions Siri, ChatGPT, Claude or MCP. So that differentiator is CONFIRMED as currently uncontested — with the large caveat in Missed item 4 that the OS-level substitute already exists.

Is "three uncontested things combined" sound? It is three positions of very unequal quality, and the brief's presentation flattened that.

- **Assistant-reachable:** genuinely uncontested, genuinely cheap, and genuinely small. The brief already concedes near-zero directory discovery, so by its own analysis this is a delight feature for users acquired elsewhere. A retention feature is not a market position.
- **Interictal capture:** the strongest position on first principles (the denominator argument is correct) and now the weakest on evidence, because the M-sense RCT tested essentially this and found nothing beyond a bare diary. It also carries the brief's own hardest risk (a daily ask is where retention dies) and is the only differentiator that requires the user to do more work.
- **Prior-auth-grade output:** the best of the three and the brief under-plays it. Payer criteria are pure arithmetic on cited public thresholds, verified above; the output has immediate dollar value; it works on day 30 rather than day 90; it is defensible without any causal claim; and it pairs naturally with Evidation's 41%-untreated / 43%-unmedicated finding. Critically, it is the only one that solves the cold-start problem rather than depending on it being solved. Reviewer's read: this should be the headline, with assistant-reachability as a feature and interictal capture as a research bet.

Stacked, they don't compound the way the brief implies — they're three separate value props for three different moments (day 1, day 30, day 90), which is a sequencing asset but not a moat. The brief's own honesty about "no technology moat" is the correct conclusion; it just arrives after three pages of framing that reads like a moat.

Is the assistant-connector strategy a wedge or a distraction? Mostly a distraction from the thing that determines success — with one exception worth keeping. The brief itself establishes that directories provide no discovery, the PWA must be first-class because voice is unusable in most contexts, and the backend LLM does the extraction regardless of front door. Those three facts mean the connector is a thin adapter over an endpoint being built anyway: days of work, near-zero risk, keep it. What it cannot do is substitute for a standalone product that works, and elevating it to differentiator #1 is a framing that will misallocate attention. The exception: connectors are the cheapest available credibility and press asset ("the first migraine tracker you can talk to Claude about"), which in a market where competitors buy syndicated "#1" placements is a real, if unglamorous, use.

One risk materially under-rated, one over-rated.

- **UNDER-RATED: platform data-retention.** The brief listed this as a pre-launch confirmation item. It is closer to a gating condition. OpenAI's Apps SDK guidelines already classify PHI as Restricted Data; Anthropic's own docs warn that custom connectors reach "arbitrary services that have not been verified." Routing migraine data through a third-party assistant, then telling users it's private, is exactly the FTC Act §5 deceptive-privacy-claim fact pattern the brief cites two sections earlier — and under MHMDA it is a *sharing* event requiring separate written authorisation, with a private right of action attached. This isn't "confirm before launch"; it's "the connector may need a distinct consent flow and a distinct privacy disclosure, designed in v1."
- **OVER-RATED: fabrication reaching a clinician.** The brief's mitigations (tri-state fields, per-field provenance, span-traceability, verbatim retention, provenance printed on the report) are unusually strong — genuinely the best-engineered part of the document. Ranked #5, defended at #1 quality. Meanwhile distribution risk has no mitigation proposed at all.

Two fundamentally better strategies the brief didn't consider.

1. **Lead with the prior-authorisation document, not the tracker.** Position as "the thing that gets your CGRP or Botox approved" — a goal, with a deadline (the appointment), where the required data collection window is ~30 days rather than 90, and where the user has a financial reason to log that survives a bad week. This inverts the retention problem from "build a daily habit and hope value emerges" to "here is a 30-day job with a payoff." It also gives an acquisition channel the brief says it lacks: patient advocacy orgs, headache clinic waiting rooms, and the r/migraine population that discusses PA denials constantly. Same schema, same MOH flagging, same report — different order of operations and a far better answer to "what does week one feel like" and "how do the first thousand users find you."
2. **Reconsider Germany/DiGA.** The brief's reimbursement analysis concludes US RTM is a dead end (correct, and more dead than it says). It never asks the obvious follow-up: where does a migraine app get paid? Answer: BfArM's DiGA directory, where two migraine apps already sit. That path is slow, evidence-heavy and CE-mark-gated — probably wrong for Eric today — but it should be an explicit rejected option with reasons, not an absence.

Does the sequencing make sense given the PWA/native tension? Not quite, and the brief nearly catches itself. It concedes the PWA costs HealthKit/Health Connect, no App Store presence, no App Intents, and weaker push, then defers Capacitor to Phase 2 ("6–12 months, contingent on traction"). But three of the brief's own priorities depend on native: the realistic ChatGPT Health route is via Apple Health; Android assistant reachability is converging on AppFunctions; and "how do the first thousand users find you" has no answer without store presence. Meanwhile Capacitor is a wrapper, not a rewrite — the cost of doing it in Phase 1 is small and the option value is large. Recommend: pull the Capacitor wrapper forward to Phase 1, and treat the PWA as the cross-platform fallback it actually is rather than the primary strategy. The brief's stated trigger ("when health-data integrations become the constraint") is already met on the day you decide ChatGPT Health matters.

---

## Intellectual honesty assessment

Genuinely good: the per-risk inline caveats in the Principles section ("Risk: all product risk moves into extraction quality…") are excellent practice and rare. "The uncomfortable finding you should sit with" and the "two hard cautions" (especially "assume near-zero organic discovery") argue against the brief's own thesis, which is the right instinct. Flagging Claru's "#1" as paid syndication, the market-size figure as directional, and the missing clinician quote are all correct calls. The tri-state/provenance data-model reasoning is the strongest section in the document.

Where it launders uncertainty: the "Confidence and Gaps" section is the problem, and it's a specific, diagnosable failure mode — the flagged-unverified list is populated with items the research agents couldn't find, while the "well established" list is populated with items they found a source for, regardless of whether the source supported the claim as stated. Everything checked and found wrong (PROMIS, the FDA characterisation, the RTM codes) sat in "Well established." That's not a research gap, it's a verification gap, and it makes the confidence section actively worse than no confidence section — a reader who trusts it will skip verifying exactly the three things that most needed it.

Inference stated as fact, beyond the corrections: "There is a real, named pathway for exactly your category" (ChatGPT Health) — inference from the existence of connectors. "Every competitor is a native app with a microphone. None exposes itself as a tool…" — independently confirmed, but the brief asserts an exhaustive negative over an app-store long tail it elsewhere describes as a "gold rush," without saying how it was established. "MOH thresholds are published and unambiguous" — ICHD-3 says the opposite about its own numbers. "Both native-only, no exceptions" (HealthKit/Health Connect) — true, but stated with a confidence the rest of the brief reserves for verified items. "Migraine Buddy… published peer-reviewed RWE papers with Novartis and academic co-authors including Goadsby" and "named KOL association in Richard Lipton" — NOT verified by the reviewer; asserted flatly and load-bearing for the "well-resourced incumbent with a real evidence base" framing. Add to the verify list.

Where it over-hedges: N1-Headache's corporate status is hedged three separate times ("possibly stagnant," "looks doubtful," "may be available") for a conclusion of low consequence, while PROMIS licensing — which has immediate legal consequence — is asserted without hedge. Hedging is not distributed by materiality.

---

## Overall verdict

Sections on competitive landscape, assistant integration, platform, product shape and sequencing: safe to act on with the amendments above. The competitive read is accurate and independently verifiable — every competitor spot-checked exists and is positioned as described, the incumbent facts are precise to the review count, the platform mechanics are current to the 2026-07-28 MCP spec, and the data-model reasoning is better than what most funded teams produce. The retention and distribution risks are correctly identified and correctly ranked.

Clinical and Regulatory section: do not act on as written. Two errors — PROMIS licensing and the CDS applicability/direction — are the kind that cost money and credibility rather than time: shipping an unlicensed instrument, and building a regulatory narrative on a carve-out that doesn't cover patient-facing software while claiming competitors are on the wrong side of a line FDA has moved. The MOH counter as specified will under-detect. The RTM section's good news doesn't apply to headache. Every one of these sat in the brief's "well established" list.

One omission changes strategy, not just facts: the M-sense RCT. Differentiator "interictal capture" needs re-arguing in light of it, and the paywall question should be reopened with it in hand.

Suggested order of operations: (1) rewrite the clinical/regulatory section with counsel, starting with a HEAP inquiry to HealthMeasures; (2) read the M-sense trial and decide whether interictal capture is a headline claim or a research bet; (3) promote the prior-auth document to the lead position; (4) pull Capacitor into Phase 1; (5) add MHMDA/Nevada consent design to v1 scope; (6) spend $1–2k on trademark clearance before any brand spend. The connector work is cheap, verified-uncontested, and can proceed in parallel — just stop calling it differentiator #1.

---

## Sources

- https://www.healthmeasures.net/images/PROMIS/Terms_of_Use_HM_approved_1-12-17_-_Updated_Copyright_Notices.pdf
- https://promishungary.org/wp-content/uploads/2019/07/Terms_of_Use_HM_approved_1-12-17_-_Updated_Copyright_Notices.pdf
- https://www.healthmeasures.net/explore-measurement-systems/promis/obtain-administer-measures
- https://meddeviceguide.com/blog/fda-clinical-decision-support-cds-guide
- https://www.mwe.com/insights/fda-issues-long-awaited-final-clinical-decision-support-software-guidance/
- https://ichd-3.org/8-headache-attributed-to-a-substance-or-its-withdrawal/8-2-medication-overuse-headache-moh/8-2-6-medication-overuse-headache-attributed-to-multiple-drug-classes-not-individually-overused/
- https://ichd-3.org/8-headache-attributed-to-a-substance-or-its-withdrawal/8-2-medication-overuse-headache-moh/
- https://journals.sagepub.com/doi/10.1177/03331024241233937
- https://www.ropesgray.com/en/insights/alerts/2026/01/fda-adapts-with-the-times-on-digital-health-updated-guidances-on-general-wellness-products
- https://www.thefdalawblog.com/2026/01/a-busy-day-in-the-cdrh-neighborhood-updates-to-the-cds-and-general-wellness-guidance-documents/
- https://www.forcetherapeutics.com/blog/cms-updates-remote-therapeutic-monitoring-rtm-codes-in-the-2026-physician-fee-schedule
- https://www.nixonlawgroup.com/resources/cms-finalizes-2026-remote-monitoring-reimbursement-updates-what-changed-for-rpm-and-rtm
- https://openai.com/index/introducing-chatgpt-health/
- https://developers.openai.com/apps-sdk/app-submission-guidelines
- https://openai.com/index/developers-can-now-submit-apps-to-chatgpt/
- https://pubmed.ncbi.nlm.nih.gov/40591361/
- https://migrainecollaborative.org/a-failed-clinical-trial-of-a-digital-app-for-migraine-where-to-from-here
- https://healthcare-startups.de/die-siebte-diga-ist-da-m-sense-migraene-app/
- https://www.ibbventures.de/en/news/newsenselab-financing
- https://link.springer.com/article/10.1186/s10194-026-02383-2
- https://developer.apple.com/documentation/healthkit/hkcategorytypeidentifier/headache
- https://www.fiercehealthcare.com/ai-and-machine-learning/openai-makes-health-chatgpt-widely-available-moving-deeper-consumer-health
- https://9to5google.com/2026/02/25/android-appfunctions-gemini/
- https://9to5google.com/2026/06/30/gemini-spark-apps-more/
- https://www.dwt.com/blogs/privacy--security-law-blog/2024/04/my-health-my-data-act-washington-nevada
- https://fpf.org/wp-content/uploads/2023/06/NV_WA-Comparison-Chart-FINAL.pdf
- https://trademark.justia.com/976/99/headspace-97699638.html
- https://icannwiki.org/.health
- https://apps.apple.com/us/app/migrapulse/id6758339023
- https://www.globenewswire.com/news-release/2025/09/29/3157649/0/en/Neura-Health-Secures-11-4-Million-Series-A-to-Fuel-National-Expansion-of-Comprehensive-Tech-Enabled-Neurology-Care-Model.html
- https://131xmm.nolt.io/287
