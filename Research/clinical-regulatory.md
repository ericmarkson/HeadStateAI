# Research — Clinical, Regulatory, Licensing & Reimbursement

**Researched August 2026.** Independently fact-checked; claims independently verified are marked **[QA]**.
Clinical counting rules, ICHD-3 constants, premonitory evidence and corticosteroid data live in `continuous-timeline-verification.md`.

> **Read the licensing and FDA sections before writing code.** These carry consequential, easy-to-get-wrong findings.

---

## 1. What headache clinicians need

> **Sourcing caveat:** the ICHD-3 passages quoted in this section (and the MOH thresholds below) were retrieved electronically rather than from the printed classification. They were consistent across two independent sources, but must be verified against printed ICHD-3 / *Cephalalgia* 2018;38(1):1–211 before any clinical or regulatory use.

### ICHD-3 fields the schema must capture

Migraine without aura (**ICHD-3 1.1**):

| Criterion | Requirement |
|---|---|
| A | ≥5 lifetime attacks |
| B | Duration **4–72 hours** untreated (2–72 h if under 18) |
| C | **≥2 of 4**: unilateral · pulsating · moderate-to-severe intensity · **aggravated by *or causing avoidance of*** routine physical activity |
| D | **At least one of:** nausea and/or vomiting; **or** photophobia **and** phonophobia (both together) |
| E | Not better accounted for by another ICHD-3 diagnosis |

**[QA] Criterion D's logical structure is commonly implemented wrong.** It is `nausea/vomiting OR (photophobia AND phonophobia)`. Do not collapse to "either photophobia or phonophobia."

**[QA] Do not drop the "or causing avoidance of" clause** from criterion C — it changes what needs to be captured. A patient who stops moving to avoid worsening pain meets it.

Aura features are separate: visual, sensory, speech/language, motor, brainstem, retinal, with onset and duration. Aura status changes treatment safety decisions (e.g. oestrogen-containing contraception, some triptan cautions).

### The three counts — not interchangeable
- **Headache days** — any day with head pain for any part of the day. Drives chronic migraine diagnosis (≥15/month) and MOH.
- **Migraine days** — the subset meeting ICHD-3 phenotype. **The number payers ask for**, and the primary endpoint in essentially all modern CGRP and gepant trials.
- **Acute medication days** — tracked separately **by drug class**, because MOH thresholds differ by class.

There is live methodological disagreement on defining a migraine day from e-diary data (van der Arend et al., *Cephalalgia* 2023). **Adopt and cite the 2026 Delphi consensus** rather than inventing a definition: De Loose L, Paemeleire K, Deligianni C, et al. "What's a migraine day? Towards a consensus definition for clinical care and research — a Delphi panel-based study." *J Headache Pain.* 2026;27:135. DOI [10.1186/s10194-026-02383-2](https://link.springer.com/article/10.1186/s10194-026-02383-2). See also `continuous-timeline-verification.md`. **Note:** a published Correction to this paper exists (DOI 10.1186/s10194-026-02407-x) and has not yet been retrieved or reviewed — see Open gaps.

### Medication-overuse headache thresholds (ICHD-3 8.2)
General criteria: headache ≥15 days/month in someone with a pre-existing headache disorder, plus regular overuse over >3 months.

| Class | Threshold |
|---|---|
| Triptans (8.2.2), ergots, opioids, combination analgesics | **≥10 days/month** |
| **Non-opioid analgesics** (8.2.3) — ICHD-3's own term, not "simple analgesics" | **≥15 days/month** |
| **8.2.6 — any combination of ergotamine, triptans, non-opioid analgesics and/or opioids, with no single class overused** | **≥10 days/month total** |

**[QA] A per-class counter alone systematically under-detects.** 6 triptan days + 6 NSAID days is MOH under 8.2.6, and a per-class counter reports clear. **Implement per-class AND combined-class.**

**[QA] ICHD-3 states its own day counts are "based on expert opinion rather than on formal evidence."** Do not describe them as unambiguous, especially when surfacing them to a patient.

**[QA] ICHD-4 is in active development** (alpha 2020; "Work in Progress 1," Goadsby/Lipton/Terwindt et al., *Cephalalgia* 2024), and whether MOH survives as a diagnosis is explicitly under debate. **Thresholds must be versioned config, not constants in code.**

### What is wrong with the diary data clinicians get today
- Paper diaries suffer documented **recall bias** — patients back-filling at the appointment. Measured specifically in paediatric headache literature.
- Digital diaries are an active validation area *because* paper is considered unreliable — see the LUMC validation studies (*Cephalalgia* 2025 and the 2024 tertiary-centre digital headache calendar validation).
- **No first-person clinician quote was sourced.** The structural evidence (recall-bias literature, multiple ongoing e-diary validation efforts) supports the narrative, but a citable quote should be pulled from an AHS/AAN talk or KOL interview before external use.

---

## 2. Validated instruments — licensing

> **[QA] PROMIS is not the safe path.**

| Instrument | Status |
|---|---|
| **PROMIS** (Pain Interference, Pain Intensity, Physical Function) | **NOT public domain.** © **PROMIS Health Organization** — the phrase "public domain" appears nowhere in the HealthMeasures terms. Free **only for individual research or individual clinical use**. *"Commercial Users must seek permission to use, reproduce, or distribute… regardless of purpose, at all times."* Electronic administration requires **HealthMeasures Electronic Administration Permission (HEAP)** with a review fee. *"Integration into proprietary technology requires written permission."* |
| **HIT-6** | Unambiguously proprietary. © 2001, 2015 **QualityMetric Inc.** Paid licence required for commercial/app deployment |
| **MIDAS** | Developed by Richard Lipton et al. (*Neurology* 2001). Widely reproduced in literature and on payer PA forms. **Current commercial rights-holder not confirmed** — historically associated with Innovative Medical Research / AstraZeneca sponsorship, now generally with Lipton / Albert Einstein College of Medicine. **Get legal confirmation before embedding item wording** |
| **MSQ** (Migraine-Specific QoL) | Proprietary, licensed via COA vendors (Mapi / ePROVIDE) |
| **MPFID** (Migraine Physical Function Impact Diary) | Developed with pharma sponsorship (erenumab programme). Licensed via ePROVIDE / PPD COA services |

**Consequences:**
- **All four require permission for commercial use.** PROMIS is not meaningfully safer than HIT-6.
- **For personal, individual use, PROMIS is free** — the individual-clinical-use carve-out applies. This is a commercialisation gate, not a build gate.
- **The HEAP inquiry to HealthMeasures is the first letter to write** before selling anything.
- **Migra already ships MIDAS and HIT-6 scoring.** Worth understanding how they cleared that.

---

## 3. FDA posture

Both the **General Wellness** guidance and the **Clinical Decision Support (CDS) Software** guidance were revised and finalised **6 January 2026**, issued without public comment.

### The CDS carve-out does not apply to patient-facing software
**[QA] "The Non-Device CDS exclusion applies only to software intended for use by healthcare professionals."** The statutory carve-out at **FD&C §520(o)(1)(E)** is HCP-directed by construction. A patient-first DTC app sits outside it entirely.

**The operative framework is therefore the General Wellness guidance plus the plain device definition and existing digital-health enforcement policy** — not CDS. The likely conclusion (a structured-capture-and-summarisation tool is non-device) survives, but the reasoning must change, because the reasoning is what counsel and investors interrogate. CDS becomes relevant only if the clinician export becomes a clinician-facing product.

### The January 2026 revision moved *toward* permitting risk prediction
**[QA]** The 2022 CDS guidance said software that "identifies a risk probability or risk score" fails Criterion 3. **The 2026 revision removes that language** and adds enforcement discretion for presenting a single recommendation where only one option is clinically reasonable, with a worked example of software that **predicts risk of future cardiovascular events**. General Wellness simultaneously expanded to cover some non-invasive physiologic monitoring outputs intended solely for wellness uses.

So: **"we don't predict" remains a defensible product and trust position. It is no longer regulatory arbitrage** — competitors shipping prediction are not obviously offside.

**Analysts disagree on the transparency direction.** Ropes & Gray and the FDA Law Blog read Criterion 4 as relaxed; others read the disclosure element list as expanded. **Treat as contested; do not restate either reading as settled.**

### Still worth avoiding regardless of framework
1. Asserting a diagnosis or presenting a differential
2. Recommending specific treatment ("take a triptan now," "ask about CGRP")
3. Time-critical decision support (real-time dosing guidance) — excluded from safe harbours regardless of transparency
4. Opaque scoring untraceable to a cited source

### Generative AI — the genuine open question
FDA issued a comprehensive draft guidance for AI-enabled medical devices (2025), and there was documented FDA/CMS activity on **generative-AI-enabled mental health devices in late 2025** — indicating active scrutiny of consumer-facing generative AI in health.

**Covington's analysis notes the 2026 CDS revision does not specifically address generative AI or consumer-facing symptom checkers.** This is a real gap in guidance, not settled law.

**Inference (moderate confidence):** using an LLM to *summarise or restructure* patient-reported data — turning a conversation into a structured ICHD-3-formatted summary — is very likely non-device, functionally a smart form. Risk rises sharply if the LLM *interprets*: suggesting a headache subtype, flagging red-flag secondary-headache features with diagnostic framing, or generating free-text clinical impressions. **Get legal review before shipping any interpretation feature.**

---

## 4. Privacy

### HIPAA
The determinant is **relationship with a covered entity**, not the presence of health data.

- **Pure DTC, downloaded independently, no contract with a provider or plan → not a covered entity or business associate.**
- **Becomes a business associate** when (a) contracted by a clinic or health system to provide monitoring/data services, or (b) data flows into a provider's EHR at that provider's direction.
- **A patient voluntarily exporting and emailing their own PDF does NOT trigger BA status.**

**Strategic bind: HIPAA obligations and the most credible revenue model (RTM billing) arrive together.** Going clinical is a one-way door.

### FTC Health Breach Notification Rule — the operative DTC regime
Final rule effective 2024. Applies to health apps and PHR-related entities outside HIPAA.
- Notify affected individuals within **60 days** of unauthorised acquisition or disclosure of unsecured health data
- **Simultaneous** FTC notice for breaches affecting **≥500 people**
- Fewer than 500: report to the FTC within 60 days of **calendar year end**
- Media notice triggered at **500 residents of a single state**
- Plus FTC Act **§5** on deceptive or unfair privacy practices

### [QA] State consumer health data laws — more binding in practice
**Washington My Health My Data Act** (in effect since March 2024):
- Applies to WA residents' consumer health data
- **Separate opt-in consent for collection AND separate authorisation for sharing**
- Mandates a **distinct Consumer Health Data Privacy Policy**
- **Enforceable by private action** under the WA Consumer Protection Act

Also **Nevada SB 370** and Connecticut's CTDPA health provisions.

**HBNR is a breach-response obligation. MHMDA is a product-design obligation** — it shapes onboarding consent UX, and it is the regime with litigation exposure. It also governs the assistant-connector data flow (see `assistant-integration.md` §3).

### Non-US
- **GDPR** — health data is Article 9 special category; requires an explicit lawful basis (typically explicit consent, Art. 9(2)(a)) beyond the Article 6 basis. Data minimisation and likely a DPIA given volume and sensitivity.
- **EU MDR** — under **MDCG 2019-11**, software is Medical Device Software only if it performs an action on data beyond storage/communication/retrieval serving a medical purpose. A diary with no diagnostic algorithm plausibly falls **outside MDSW scope** (comparable to the cited nutrition-app example). Adding scoring, flagging or AI interpretation requires re-assessment against MDR Rule 11; a threshold flag citing ICHD-3 would likely remain Class I, while predictive or diagnostic-support features could reach IIa/IIb.
- **UK MHRA** standalone software guidance was **not independently verified** — MDCG 2019-11 used as proxy. Confirm directly.

US and EU/UK logic point the same direction: **data display good, prediction and diagnosis bad.** One conservative design principle covers both.

---

## 5. Treatment landscape and payer criteria

### What the data feeds
CGRP monoclonal antibodies (erenumab, fremanezumab, galcanezumab, eptinezumab) and gepants (rimegepant, atogepant, zavegepant) for prevention; **onabotulinumtoxinA** for chronic migraine; neuromodulation devices (Cefaly, gammaCore, Nerivio).

### Prior-authorisation criteria — verified across payer policy documents

**CGRP mAbs / gepants:**
- Episodic migraine: typically **≥4 migraine days/month** (some payers ≥6 or ≥8). Cigna IP0050: *"4 or more migraine headache days per month (prior to initiating a migraine-preventative medication)"*
- Chronic migraine: **≥15 headache days/month with ≥8 meeting migraine criteria**
- **Documented failure of two oral preventive classes** (beta-blockers, anticonvulsants, TCAs, SNRIs, CCBs) at adequate trial — 6–8 weeks at therapeutic dose, some policies ≥3 months
- Reviewers explicitly want *"headache diary data or validated outcome measures (MIDAS, HIT-6)"* submitted. *"Reviewers should not have to calculate frequency from clinical notes."*

**OnabotulinumtoxinA for chronic migraine:**
- **≥15 headache days/month with ≥8 migraine days.** Cigna verbatim: *"greater than or equal to 15 headache days per month with headache lasting 4 hours per day or longer"*
- Documented failure of **≥3 prophylactics from ≥2 classes**
- **Continuation requires demonstrating ≥50% reduction in monthly migraine frequency from baseline** after ≥2 cycles

**[QA] No payer policy reviewed contains any provision for status migrainosus, refractory migraine, or continuous headache.** Criteria are expressed entirely in **days**.

**Therefore payer exports must default to day counts, never attack counts.** A 21-day continuous episode with ≥4 h of headache daily is **21 headache days** and clears the ≥15 threshold outright. Reported as one attack — or three — it fails every threshold in every policy checked. **Attack counts are clinically interesting and reimbursement-toxic.**

---

## 6. Reimbursement

### RTM codes — no code reaches headache
Established codes:

| Code | Function | Requirement |
|---|---|---|
| 98975 | Initial setup / patient education | One-time |
| 98976 | Device supply — **respiratory**, per 30 days | ≥16 days of data |
| 98977 | Device supply — **musculoskeletal**, per 30 days | ≥16 days of data |
| 98978 | Device supply — **cognitive behavioural therapy monitoring** | **[QA] Arguably the closest existing fit — worth a specific look** |
| 98980 | Treatment management, first 20 min | ≥1 interactive communication that month |
| 98981 | Each additional 20 min | Same |

**New effective 1 January 2026** (CMS Physician Fee Schedule final rule):
- **98979** — treatment management, **first 10 minutes** (down from a 20-minute floor). Confirmed, **not** body-system-restricted.
- **98985** — **[QA] MUSCULOSKELETAL-SPECIFIC.** Official descriptor: *"…device(s) supply for data access or data transmissions to support monitoring of **musculoskeletal system**, 2–15 days in a 30-day period."* **98985 is musculoskeletal-specific and does not reach headache, so the 2026 additions did not lower the bar for a neurology use case.**
- Companion codes 98984 / 98986 exist in the same family.

**Structural fit:** RTM — unlike RPM, which requires FDA-regulated devices collecting physiologic data — explicitly accommodates **self-reported, non-physiologic data** counting toward the day threshold. Good fit for a symptom diary in principle.

**But:** there is **no neurology or headache RTM device-supply code**, and the 2026 additions did not create one. **No evidence of any existing migraine or headache RTM billing programme was found** — searches surfaced only generic PT/OT/MSK and respiratory explainer content.

**[QA] Unavailable as described.** RTM also requires clinic-side billing by a treating provider, so it is unavailable to a pure-DTC product regardless. Requires direct conversation with a health-system billing office or reimbursement consultant before being counted on.

### [QA] Germany / DiGA — the only proven pathway
BfArM's **DiGA** directory already lists **two migraine apps** — **M-sense Migräne** (Newsenselab; Germany's 7th DiGA) and **sinCephalea** (Perfood). Both are prescribable and reimbursed by statutory health insurance.

**This is the only jurisdiction where a migraine app is reimbursed at scale.** Slow, evidence-heavy, CE-mark-gated. Probably wrong today — but it should be an **explicitly rejected option with reasons**, not an absence from the analysis.

### Other pathways
**No evidence found** of alternative payer routes (e-PRO administration codes, DTx-specific reimbursement) beyond RTM/RPM. Worth a dedicated follow-up if RTM proves unworkable.

---

## 7. Building clinical credibility

**The Migraine Buddy playbook** (reported, partially unverified): named senior KOL association, peer-reviewed real-world-evidence publications co-authored with academic headache centres, then pharma-sponsored RWE partnerships. Reported publications include Goadsby et al., *European Journal of Neurology* 2021 (multinational headache burden via Migraine Buddy); *Neurology and Therapy* 2018 (burden of migraine in Europe from self-reported digital diary data); and a Novartis-partnered migraine/mental-health comorbidity study. **These were not independently verified.**

**The replicable pattern:** (1) get a recognised headache-field KOL endorsement or advisory role, (2) publish RWE analyses of aggregate de-identified diary data in headache/neurology journals with academic co-authors, (3) pursue pharma-sponsored RWE partnerships once data volume exists. A flywheel, not a one-time certification.

**Other trust signals:**
- **ORCHA / NHS DTAC** — the UK NHS-procurement quality bar (clinical safety, data protection, technical security, interoperability, usability/accessibility). ORCHA scores are a recognisable signal to UK clinicians even outside formal procurement.
- **FHIR / SMART-on-FHIR EHR integration** — standard practice for clinically-oriented apps wanting real EHR presence. No headache-tracker-specific precedent found. A scaling concern, not an early one.

**Cheapest high-value early move: recruit one or two headache specialists as advisors.** Costs little, buys real credibility, and informs which instrument to licence.

**Design the clinician export as a transparent, ICHD-3-labelled, threshold-flagged summary** so it is defensible by construction rather than by post-hoc argument.

---

## Open gaps

1. No verified migraine/headache RTM billing programme — needs billing-office confirmation
2. MIDAS commercial rights-holder unconfirmed
3. No first-person clinician quote on diary-data deficiencies
4. FDA's stance on generative AI in consumer symptom interpretation — explicitly unaddressed in 2026 guidance
5. UK MHRA standalone software guidance not independently verified
6. GDPR Article 9 specifics not pulled beyond the general framing
7. Whether FDA's 2026 CDS Criterion 4 was relaxed or expanded — analysts disagree
8. Migraine Buddy's publication record and KOL association
9. The published Correction to the 2026 Delphi consensus paper (DOI 10.1186/s10194-026-02407-x) has not been retrieved or reviewed

---

## Sources

ICHD-3: [1.1 Migraine without aura](https://ichd-3.org/1-migraine/1-1-migraine-without-aura/) · [Definition of Terms](https://ichd-3.org/definition-of-terms/) · [8.2 MOH](https://ichd-3.org/8-headache-attributed-to-a-substance-or-its-withdrawal/8-2-medication-overuse-headache-moh/) · [8.2.2 triptan overuse](https://ichd-3.org/8-headache-attributed-to-a-substance-or-its-withdrawal/8-2-medication-overuse-headache-moh/8-2-2-triptan-overuse-headache/) · [8.2.3 simple analgesic overuse](https://ichd-3.org/8-headache-attributed-to-a-substance-or-its-withdrawal/8-2-medication-overuse-headache-moh/8-2-3-simple-analgesic-overuse-headache/) · [8.2.6 multiple drug classes](https://ichd-3.org/8-headache-attributed-to-a-substance-or-its-withdrawal/8-2-medication-overuse-headache-moh/8-2-6-medication-overuse-headache-attributed-to-multiple-drug-classes-not-individually-overused/) · [ICHD-4 Work in Progress](https://journals.sagepub.com/doi/10.1177/03331024241233937)

Clinical guidance: [AMF headache journals](https://americanmigrainefoundation.org/resource-library/headache-journals/) · [AHS clinician guidelines](https://americanheadachesociety.org/resources/clinicians/guidelines) · [AHS primary care](https://americanheadachesociety.org/resources/primary-care) · [Defining migraine days from e-diary data (2023)](https://journals.sagepub.com/doi/10.1177/03331024231166625) · [Digital headache calendar validation](https://pmc.ncbi.nlm.nih.gov/articles/PMC10795797/) · [E-headache diary psychometrics 2025](https://journals.sagepub.com/doi/10.1177/03331024251359222) · [MMD as treatment target](https://link.springer.com/article/10.1007/s40122-023-00525-x) · De Loose L, Paemeleire K, Deligianni C, et al. "What's a migraine day? Towards a consensus definition for clinical care and research — a Delphi panel-based study." *J Headache Pain.* 2026;27:135. DOI [10.1186/s10194-026-02383-2](https://link.springer.com/article/10.1186/s10194-026-02383-2) (Correction: DOI 10.1186/s10194-026-02407-x, not yet retrieved)

Instruments: [HealthMeasures Terms of Use (PDF)](https://promishungary.org/wp-content/uploads/2019/07/Terms_of_Use_HM_approved_1-12-17_-_Updated_Copyright_Notices.pdf) · [PROMIS obtain & administer](https://www.healthmeasures.net/explore-measurement-systems/promis/obtain-administer-measures) · [QualityMetric HIT-6](https://www.qualitymetric.com/health-surveys/the-headache-impact-test-hit-6/) · [MIDAS development](https://www.neurology.org/doi/abs/10.1212/wnl.56.suppl_1.s20) · [MPFID / ePROVIDE](https://eprovide.mapi-trust.org/instruments/migraine-physical-function-impact-diary)

FDA: [FDA Law Blog on Jan 2026 guidances](https://www.thefdalawblog.com/2026/01/a-busy-day-in-the-cdrh-neighborhood-updates-to-the-cds-and-general-wellness-guidance-documents/) · [Ropes & Gray analysis](https://www.ropesgray.com/en/insights/alerts/2026/01/fda-adapts-with-the-times-on-digital-health-updated-guidances-on-general-wellness-products) · [Troutman on 2026 General Wellness](https://www.troutman.com/insights/fdas-2026-guidance-on-general-wellness-devices-policy-for-low-risk-devices/) · [Faegre Drinker summary](https://www.faegredrinker.com/en/insights/publications/2026/1/key-updates-in-fdas-2026-general-wellness-and-clinical-decision-support-software-guidance) · [Covington on revised CDS](https://www.cov.com/news-and-insights/insights/2026/01/5-key-takeaways-from-fdas-revised-clinical-decision-support-cds-software-guidance) · [King & Spalding](https://www.kslaw.com/news-and-insights/fda-updates-general-wellness-and-clinical-decision-support-guidance-documents) · [CDS device/non-device scope](https://meddeviceguide.com/blog/fda-clinical-decision-support-cds-guide) · [McDermott on CDS final guidance](https://www.mwe.com/insights/fda-issues-long-awaited-final-clinical-decision-support-software-guidance/) · [2016 General Wellness FR notice](https://www.federalregister.gov/documents/2016/07/29/2016-17902/general-wellness-policy-for-low-risk-devices-guidance-for-industry-and-food-and-drug-administration) · [2022 CDS FR notice](https://www.federalregister.gov/documents/2022/09/28/2022-20993/clinical-decision-support-software-guidance-for-industry-and-food-and-drug-administration-staff) · [FDA AI device draft guidance](https://www.fda.gov/news-events/press-announcements/fda-issues-comprehensive-draft-guidance-developers-artificial-intelligence-enabled-medical-devices) · [Sidley on generative-AI mental health devices](https://www.sidley.com/en/insights/newsupdates/2025/11/us-fda-and-cms-actions-on-generative-ai-enabled-mental-health-devices-yield-insights-across-ai)

Privacy: [FTC HBNR compliance guide](https://www.ftc.gov/business-guidance/resources/complying-ftcs-health-breach-notification-rule-0) · [FTC finalises HBNR changes](https://www.ftc.gov/news-events/news/press-releases/2024/04/ftc-finalizes-changes-health-breach-notification-rule) · [HBNR final rule (FR)](https://www.federalregister.gov/documents/2024/05/30/2024-10855/health-breach-notification-rule) · [DWT on MHMDA & Nevada SB370](https://www.dwt.com/blogs/privacy--security-law-blog/2024/04/my-health-my-data-act-washington-nevada) · [FPF WA/NV comparison chart](https://fpf.org/wp-content/uploads/2023/06/NV_WA-Comparison-Chart-FINAL.pdf) · [HHS on health apps & APIs](https://www.hhs.gov/hipaa/for-professionals/privacy/guidance/access-right-health-apps-apis/index.html) · [When app developers are business associates](https://healthlawcenterplc.com/hipaa-when-are-mobile-app-developers-business-associates/) · [MDCG 2019-11 / MDSW classification](https://namsa.com/resources/blog/eu-mdr-and-ivdr-classifying-medical-device-software-mdsw/)

Payer: [Cigna Botox PA form](https://static.cigna.com/assets/chcp/pdf/resourceLibrary/prescription/Botox.pdf) · [Cigna CGRP policy IP0050](https://static.cigna.com/assets/chcp/pdf/coveragePolicies/pharmacy/ip_0050_coveragepositioncriteria_cgrp_inhibitors_preventative_employer.pdf) · [Cigna Ajovy PA criteria](https://static.cigna.com/assets/chcp/pdf/coveragePolicies/cnf/cnf_332_coveragepositioncriteria_calcitonin_gene-related_peptide_inhibitors_ajovy_pa.pdf) · [Premera 5.01.584](https://www.premera.com/medicalpolicies/5.01.584.pdf) · [UnitedHealthcare CGRP notification](https://www.uhcprovider.com/content/dam/provider/docs/public/prior-auth/drugs-pharmacy/commercial/a-g/PA-Notification-CGRP-Receptor-Antagonists.pdf) · [Centene/Coordinated Care Botox WA.PHAR.232](https://www.coordinatedcarehealth.com/content/dam/centene/Coordinated%20Care/policies/pharmacy-policies/508-WA.PHAR.232-OnabotulinumtoxinA-(Botox)-for-Chronic-Migraine-Final.pdf) · [CGRP PA guide](https://www.artificerhealth.com/blog/cgrp-prior-authorization-guide/) · [MMD as the driving number](https://migrainecgrp.com/learn/monthly-migraine-days)

Reimbursement: [Force Therapeutics on 2026 RTM codes](https://www.forcetherapeutics.com/blog/cms-updates-remote-therapeutic-monitoring-rtm-codes-in-the-2026-physician-fee-schedule) · [Limber Health 2026 CMS final rule](https://www.limberhealth.com/blog/2026-cms-final-rule-rtm-codes) · [Nixon Law Group RPM/RTM 2026](https://www.nixonlawgroup.com/resources/cms-finalizes-2026-remote-monitoring-reimbursement-updates-what-changed-for-rpm-and-rtm) · [M-sense DiGA listing](https://healthcare-startups.de/die-siebte-diga-ist-da-m-sense-migraene-app/)

Credibility: [Goadsby et al. EJN 2021](https://onlinelibrary.wiley.com/doi/10.1111/ene.15037) · [Burden of migraine in Europe, Neurol Ther 2018](https://link.springer.com/article/10.1007/s40120-018-0113-0) · [Novartis/Healint study coverage](https://www.mobihealthnews.com/news/novartis-healint-study-uses-self-tracking-app-examine-migraine-mental-health-connection) · [Lipton on Migraine Buddy](https://www.neurologylive.com/view/advantages-migraine-buddy-app-richard-lipton) · [NHS DTAC / ORCHA](https://www.orchahealth.com/resources/assessment-frameworks/dtac) · [Paediatric recall bias study](https://www.researchgate.net/publication/12160906_The_Occurrence_of_Recall_Bias_in_Pediatric_Headache_A_Comparison_of_Questionnaire_and_Diary_Data)
