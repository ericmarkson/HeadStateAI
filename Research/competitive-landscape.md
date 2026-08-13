# Research — Competitive Landscape

**Migraine and headache tracking apps · researched August 2026**
Status: findings verified where marked; see `qa-verification-report.md` for independent checks.

---

## Summary

The category is large, fragmented, and already being colonised by AI voice-logging entrants. At least six apps launched since mid-2025 with "speak your symptoms, AI extracts the data" as the core pitch. None has meaningful scale, funding or brand recognition. Voice-first migraine logging is the **crowded** end of this market, not the empty one.

The incumbent is corporate-backed, not a lazy legacy app. The clinical-legacy tier has weak momentum and may contain acquisition targets. The genuine gaps are documented in `continuous-timeline-verification.md`, not here.

---

## 1. The incumbent

**Migraine Buddy** — owned by **Aptar Digital Health** since Aptar acquired Healint, announced 21 February 2024. Parent is AptarGroup (NYSE: ATR), a pharma-packaging conglomerate.

| Attribute | Value |
|---|---|
| Package | `com.healint.migraineapp` |
| Play installs | 1,000,000+ (≈3.1M lifetime per third-party trackers) |
| Play rating | 4.8★ from ~66,400 reviews |
| Community claim | 3.5M (store listings) / 4.8M (website) |
| Pricing | Free + MBplus. Monthly $9.99 or $12.99; two-month $24.99; annual $69.99–$89.99. Multiple concurrent SKUs — price depends on offer cohort |
| Ads | None |
| Notable 2026 | **Oura Ring** integration announced 27 January 2026 |
| Support | jenny@migrainebuddy.com · 250 N Route 303, Congers, NY 10920 |

**MBplus gates:** 7-day weather forecast (humidity, pressure), weather correlation analysis, advanced trigger × symptom × location reporting, predictive risk analysis, end-of-attack prediction *(iOS only)*, personalised trigger recommendations *(iOS only)*, symptom progression through an attack, exact-time medication logging, coaching programmes and the Good Days Guide.

**Named coaching programmes:** Weather Trends and Migraine · Get Food Cravings Under Control · Is Your Migraine Related to Hormones? · 10 Tips and 5 Recipes for a Migraine-Friendly Diet · Good Days Program (GDP) · Chronic Migraine recording-tips programme.

**Evidence base — REPORTED, NOT VERIFIED.** Migraine Buddy is said to have peer-reviewed real-world-evidence publications with Novartis and academic co-authors including Goadsby, plus a named KOL association with Richard Lipton (developer of MIDAS). **Neither claim was independently verified.** They are load-bearing for how formidable the incumbent appears — confirm before relying on them.

**Exploitable gap:** Migraine Buddy still has **no Apple Health sync** — the top request on its public feedback board (131xmm.nolt.io/287). Only a native build can exploit this.

Full feature inventory: see `migraine-buddy-features.md`.

---

## 2. Clinical-legacy tier — weak momentum

| App | Position | Health signal |
|---|---|---|
| **N1-Headache** (Curelator, Boston) | Only app producing individual Trigger / Protector / No-Association maps. **Mandatory daily diary**, 90-day minimum before analysis. Partnered with Migraine Trust and National Headache Foundation | App live both stores. **No confirmed company news since ~2019.** "Possibly stagnant" is inference, not fact |
| **Ctrl M Health** (Philadelphia) | Co-developed with **Jefferson Headache Center** (Dr. William Young) | Crunchbase: 1–10 employees, funding undisclosed, quiet since the 2020–21 press wave |
| **Migraine Monitor** (RPM Healthcare LLC) | B2B2C — free to patients, optional clinician dashboard. Won a 2018 Digital Health Award. **Not AMF-owned** despite the association | Site shows © 2026; independent review (getmigra.com) calls it "outdated, last meaningful update ~2022, not recommended." Conflicting signals |
| **iHeadache** | One of the original clinically-designed headache diaries | Still listed, minimally updated |
| **Migraine Insight** | Freemium. **Has an explicit chronic "pain level tracking" mode with open-ended episodes** | Live, sparse footprint. Ownership not discoverable |

**N1-Headache is strategically interesting** — closest existing analogue to a continuous-capture thesis, clinically framed, possibly available. **But see `qa-verification-report.md`:** given the M-sense RCT found trigger analytics did not beat a bare diary, do not pay for its methodology.

---

## 3. The 2026 AI wave — the actual competitive set

| App | Platform | Pricing | Differentiator |
|---|---|---|---|
| **Migraine Trail** | iOS only | Free (3 voice entries/mo, 60-day history) / $4.99mo, $49.99yr | **Voice-first, tap-and-speak with AI field extraction.** Explicitly designed for dark-room use — marketing copy: "whisper your symptoms into your phone on the bedside table," "never opened your eyes," soft audio confirmation instead of visual. 14-day barometric forecast using GFS/ECMWF models. PDF doctor reports. **Closest direct competitor.** No company/funding info surfaced |
| **Claru** (Reelpop) | iOS + Android | Free / Pro (undisclosed) | AI voice logging, 7-day risk forecast, "Migraine Mode" distraction-free UI. **Its "#1 ranking" from "Halstead Research" is an openpr.com press release syndicated verbatim across unrelated news-wire sites — paid PR, not an independent benchmark** |
| **Migra** (getmigra.com) | iOS only | Free / $29.99yr or $4.99mo | **On-device Apple Foundation Models (iOS 26+)** — no data leaves device. Barometric pressure via iPhone altimeter. Apple Watch app. **MIDAS and HIT-6 scoring** (worth understanding how they cleared licensing). Bidirectional Apple Health sync — therefore already inside the ChatGPT Health pipe |
| **MigrAid** | iOS | Freemium | One-tap logging, **16-zone pain-location mapping**, built-in MIDAS calculator, on-device storage, no account required. Positions explicitly against Migraine Buddy's "overwhelming" form |
| **MigraPulse** | iOS | — | "Calm, voice-first migraine tracking." Sixth entrant in the same category |
| **MigraSense** | iOS | Freemium | Another AI-branded entrant, TikTok-marketed |
| **Pressure Pal** | iOS / iPadOS / macOS / Vision Pro | Free + premium | Barometric-pressure-first niche. Claims to be "the only major migraine app that makes pressure tracking a first-class feature" |
| **heyvie** (Germany) | iOS + Android | Freemium | CBT / behavioural-coaching angle rather than pure logging |
| Migrio, MygRa, MigraFocus, Ease, Relief, "Headache Migraine Tracker Pro" | iOS mostly | Low/no info | Undifferentiated me-too listings. Evidence of an app-store gold rush of single-developer AI-branded trackers. Several appear asset-flip-style |

**Pattern:** overwhelmingly iOS-only, all native, **none delivered as a PWA**, **none assistant-reachable**.

---

## 4. Device-linked and prescription apps

| App | Model |
|---|---|
| **Nerivio companion** (Theranica) | Prescription REN neuromodulation device, ~$599 list, insurance/UpScript coverage. **Theranica raised $45M**, Israel-based |
| **CEFALY "CeCe"** | Free, works standalone without owning the CEFALY eTNS device. Belgium; company claims 2B+ migraines treated since 2008 |

---

## 5. Europe — the omission that matters

**M-sense Migräne** (Newsenselab, Berlin) — VC-backed (**High-Tech Gründerfonds, IBB Ventures**), CE-marked, and **Germany's 7th DiGA**: prescribable and reimbursed by statutory health insurance.

**sinCephalea** (Perfood) — a second migraine DiGA.

**Germany is the only jurisdiction where a migraine app is reimbursed at scale.** See `clinical-regulatory.md` §Reimbursement.

M-sense also produced the null RCT that constrains the trigger-analytics thesis — see `qa-verification-report.md`.

---

## 6. Adjacent trackers migraine patients actually use

| App | Conversational AI? | Notes |
|---|---|---|
| **Bearable** | No chat; correlation engine only | Multi-condition. Founded 2020 by James Saady, bootstrapped. $6.99/mo or $34.99/yr, 7-day trial. 4.8★/4.7★. Frequently cited as the do-everything alternative. **Complaints:** dense/overwhelming setup, CSV-only export (no PDF), no live support, no migraine-specific attack workflow, no pressure detection. **Not freely timestamped** — buckets into four fixed periods (pre, am, mid, pm) |
| **Guava Health** | No, despite pedigree | Founders built the personal-assistant team at **Graphiq, acquired by Amazon Alexa in 2017**. Visit prep, "Guava Tags," provider dashboard, integrates with 100,000+ US providers (Epic, Cerner) |
| **Visible** | No | Pacing/activity for ME/CFS and Long COVID. Two studies published 2025 (Frontiers in Digital Health). Wearable-centric |
| **Flaredown** | No | Autoimmune/chronic-illness tracker. Commercial entity appears wound down; codebase now under **Ruby for Good** on GitHub. Likely dormant — not confirmed shut down |
| **Symple** | No | Older general symptom diary, low visibility |
| **Daylio** | No | Generic mood/habit tracker used as an improvised diary. No migraine fields |
| **Apple Health / Journal** | Journal has AI-suggested prompts | Passive aggregation layer. **HealthKit has a first-class `headache` category type**, and Shortcuts has a "Log Health Sample" action — so "Hey Siri, log a headache" already works with no third-party app |

---

## 7. Clinical / enterprise players

| Player | Model |
|---|---|
| **MigraineManager.care** (Belgium) | Patient app + neurologist web dashboard. Backed by **imec iStart and UCB** — real pharma-adjacent clinical platform |
| **Migrevention** (Estonia) | Self-described "first fully digital headache clinic." Diary + async chat with a headache nurse + education |
| **Neura Health** (US) | Virtual headache/neurology clinic, membership + insurance-accepted. Tracker syncs to care team, proactive "care coach" outreach. Manual logging, no conversational AI. **Raised $11.4M Series A September 2025**, backed by the AHA's Go Red for Women fund. Best-funded US player adjacent to this lane |
| **Evidation ("MigraineSmart")** | Research/RWE platform, not consumer. PRO surveys + wearable + optional claims/EHR. Monetises via pharma partnerships |

**Evidation's market data — the most useful single fact in this research:** on their platform, **only 43% of migraine sufferers used prescription medication and 41% had not seen a provider in the past year.** The under-treated majority is the addressable market.

**Market size:** digital therapeutics for migraine projected at **$2.09B in 2026 → $4.82B by 2030 (23.2% CAGR)**. From a paid Research and Markets report — **directional only**. Its participant list names pharma and device sponsors (Pfizer, Dr. Reddy's, Click Therapeutics, Salvia BioElectronics, Tonix, Neurolief, AndHealth, Nordic Brain Tech, Lin Health, Schmerzklinik Kiel, Juva Health, Newsenselab, Ebby Health, Care Tuner, Peachy Day) alongside actual tracker competitors — treat cautiously.

---

## 8. Gap analysis — evidence-based

1. **Mid-attack logging is the #1 usability failure.** Migraine Buddy reviewers describe the screen as "text and buttons of different colors and fonts — all screaming for attention" when photophobic. A quoted user: *"when in pain I don't feel like typing through multiple screens."* Validated pain point — and the stated rationale for six competitors' voice features. **Not unique to target.**
2. **Hormonal/menstrual integration is weak market-wide**, despite hormonal triggers affecting a majority of female patients. Only Migraine Buddy touches it, via the third-party Oura integration.
3. **No remediation tracking.** Nothing systematically follows up on whether a specific medication or intervention actually worked over time and surfaces that back to user or provider.
4. **Weather/pressure inconsistent.** First-class only in niche apps (Pressure Pal, Migra, Migraine Trail). Afterthought in the two largest.
5. **Interictal capture is thin** — but **not absent**; see `continuous-timeline-verification.md`, which refutes the stronger version of this claim.
6. **Subscription and notification friction.** Persistent un-clearable "insight" notifications; inflexible monthly-or-annual-only trial options.
7. **Nothing helps articulate disability to employers.** No app produces anything beyond clinical exports.
8. **Crowded, noisy floor.** The volume of near-identical low-information "AI migraine tracker" listings means low barriers to entry and real discoverability and confusion risk.
9. **No PWA in the category.** Genuine white space, but cuts both ways — competitors market native-only capabilities (Watch complications, altimeter pressure sensing, Vision Pro, HealthKit/Health Connect) as differentiators.

---

## Flagged uncertainties

- N1-Headache / Curelator current company health and funding post-2019
- Migraine Monitor's actual update cadence (site vs. third-party reviews conflict)
- Ctrl M Health's current app-store status and user scale
- Migraine Insight's ownership and company identity
- Flaredown's corporate status — evidence points to dormant, not confirmed
- Exact pricing for N1-Headache, MigrAid Pro, Claru Pro (unpublished)
- Migraine Buddy's Novartis publications and Lipton association — **unverified**
- **No user-forum evidence retrieved** — the research proxy blocked Reddit-scoped queries. Negative user sentiment is unverified rather than absent

---

## Sources

Migraine Buddy: [site](https://migrainebuddy.com/) · [Play](https://play.google.com/store/apps/details?id=com.healint.migraineapp) · [Aptar acquires Healint](https://aptar.com/news-events/aptar-digital-health-acquires-healint-reinforcing-digital-health-portfolio-and-driving-future-growth/) · [Aptar × Oura](https://aptar.com/en-us/news-events/aptar-digital-health-and-ura-collaborate-to-advance-migraine-insights-by-connecting-migraine-buddy-with-oura-ring) · [Apple Health feature request](https://131xmm.nolt.io/287) · [MBplus](https://migrainebuddy.com/mbplus/) · [Actionable Programs](https://migrainebuddy.com/actionable-programs/) · [JustUseApp reviews](https://justuseapp.com/en/app/975074413/migraine-buddy-track-headache/reviews)

Competitors: [N1-Headache](https://n1-headache.com/) · [N1 FAQ](https://n1-headache.com/patients/faq/) · [Ctrl M Crunchbase](https://www.crunchbase.com/organization/ctrl-m-health) · [Jefferson/Ctrl M launch](https://www.jeffersonhealth.org/about-us/news/2020/09/jefferson-headache-center-ctrl-m-health-launch-digital-platform) · [Migraine Insight](https://apps.apple.com/us/app/migraine-insight-tracker/id1364291643) · [Migraine Monitor](https://migrainemonitor.com/) · [iHeadache](https://iheadache.com/) · [Migraine Trail](https://migrainetrail.com/) · [Migraine Trail voice logging](https://migrainetrail.com/features/voice-logging) · [Claru](https://www.claruapp.com/) · [Migra](https://getmigra.com/) · [Migra 2026 comparison](https://www.getmigra.com/blog/best-migraine-tracker-app-2026/) · [MigrAid](https://migraid.app/) · [MigraPulse](https://apps.apple.com/us/app/migrapulse/id6758339023) · [MigraSense](https://www.getmigrasense.com/) · [Pressure Pal](https://pressurepal.app/blog/best-migraine-tracker-app/) · [heyvie](https://en.heyvie.io/) · [Halstead "rankings" — PR](https://www.halsteadresearch.com/insights/best-migraine-trackers-2026)

Devices & clinical: [Nerivio pricing](https://help.nerivio.com/en/articles/9783095-nerivio-pricing-and-insurance-coverage) · [Theranica $45M](https://www.mobihealthnews.com/news/theranica-raises-45m-migraine-digital-therapeutic) · [CeCe](https://www.cefaly.com/cece) · [MigraineManager.care](https://migrainemanager.care/) · [Migrevention](https://migrevention.com/) · [Neura Health Series A](https://www.globenewswire.com/news-release/2025/09/29/3157649/0/en/Neura-Health-Secures-11-4-Million-Series-A-to-Fuel-National-Expansion-of-Comprehensive-Tech-Enabled-Neurology-Care-Model.html) · [Evidation MigraineSmart](https://www.fiercebiotech.com/medtech/evidation-launches-digital-migraine-research-platform-and-symptom-tracker) · [M-sense DiGA](https://healthcare-startups.de/die-siebte-diga-ist-da-m-sense-migraene-app/) · [Newsenselab funding](https://www.ibbventures.de/en/news/newsenselab-financing) · [M-sense diary](https://m-sense.de/kopfschmerztagebuch/)

Adjacent: [Bearable](https://bearable.app/) · [Bearable review](https://www.choosingtherapy.com/bearable-app-review/) · [Bearable time periods](https://bearable.app/support/common-questions/track-multiple-times-per-day-in-the-health-check-in/) · [Guava](https://guavahealth.com/about) · [Visible](https://en.wikipedia.org/wiki/Visible_(mobile_app)) · [Flaredown GitHub](https://github.com/rubyforgood/Flaredown) · [HealthKit headache type](https://developer.apple.com/documentation/healthkit/hkcategorytypeidentifier/headache)

Market: [Research and Markets report](https://www.researchandmarkets.com/reports/6191037/digital-therapeutics-migraine-market-report)
