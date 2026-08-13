# Migraine Buddy (Android) — Complete Feature Breakdown

*Research compiled 13 August 2026. Based on the Google Play listing, the official Migraine Buddy website and Help Center, the MBplus product pages, and third-party reviews. Where sources disagree or a feature is platform-limited, that is flagged inline.*

---

## 1. At a Glance

| Attribute | Detail |
|---|---|
| **App name (Play)** | "Migraine Buddy: Track Headache" (US) / "Migraine Buddy – Track&Control" (UK — regional title variants exist) |
| **Package ID** | `com.healint.migraineapp` |
| **Developer** | Aptar Digital Health (acquired Healint, Singapore, the original developer, in an acquisition announced 21 February 2024; iOS listing still shows Voluntis, an Aptar subsidiary) |
| **Developer address** | 250 N Route 303, Congers, NY 10920, USA |
| **Support contact** | jenny@migrainebuddy.com · migrainebuddy.com |
| **Category** | Medical |
| **Price** | Free to download, with in-app purchases (MBplus subscription) |
| **Ads** | None reported on the Play listing |
| **Play rating** | ~4.8 / 5 from ~66,400 reviews |
| **Installs** | 1,000,000+ (Play), ~3.1M lifetime downloads per third-party trackers |
| **Last updated** | 7 August 2026 |
| **Version (Android)** | 89.x series (e.g. 89.0.1779211097); iOS on 90.0.x |
| **APK size** | ~173 MB |
| **Minimum OS** | Android 8.0+ |
| **Content rating** | Rated for 3+ (Play); 9+ on App Store |
| **Languages** | ~9 languages (per App Store listing) |
| **Community size claimed** | 3.5 million (store listings) / 4.8 million (website) — the number has been revised upward over time |
| **Provenance claim** | "Developed by migraine patients, designed with neurologists" |

---

## 2. Account, Profile and Onboarding

- **Account required.** Data is tied to a registered account rather than stored purely on-device, which enables cross-device sync but means records live on the company's servers.
- **Registration and login**, with password reset / forgotten-password recovery.
- **Email address change** and **password change** from within the app.
- **Account reset** — wipe your logged data and start fresh without deleting the account entirely.
- **Account deletion** and **data deletion requests** (the Play data-safety card confirms users can request deletion).
- **Profile setup and editing** — demographic and condition details used to benchmark your data against similar users.
- **Single profile per account.** The Help Center explicitly addresses "Can we have several profiles for one account?" — multiple profiles are not the supported model, so a parent tracking a child's attacks needs a separate account.
- **Onboarding questionnaire** that tailors which recording screens and options you see.
- **Measurement system toggle** — metric vs imperial (affects temperature and pressure units).

---

## 3. Attack Recording — The Core Feature

Migraine Buddy uses a **wizard-style questionnaire: one question per screen**, designed so it can be completed while symptomatic. The documented recording flow covers:

1. **Onset and duration** — "Do you know when it started?" Start time, end time, and total duration. Times can be entered precisely or approximated.
2. **Pain intensity** — "How bad was it?" A severity scale for the attack.
3. **Pain location** — "Where did the pain start?" Selected on a **head diagram / pain-map interface**, so unilateral vs bilateral, frontal, temporal, occipital, orbital, and neck locations can be distinguished.
4. **Medication taken** — "Did you take any medication?" Which drug(s), from a customizable list.
5. **Dose and exact time of each dose** — supported as a separate step (exact-time logging is called out as an MBplus-enhanced capability).
6. **Relief methods** — "How about any of these relief methods?" Non-drug interventions: sleep, dark room, ice/heat, caffeine, hydration, massage, etc.
7. **Symptoms** — "Were you experiencing any of these symptoms?" Nausea, vomiting, photophobia, phonophobia, osmophobia, dizziness, neck pain, and many more.
8. **Aura / premonitory warning** — "Did you sense it coming?"
9. **Impact on activities** — "How did it affect your activities?" Functional impairment and productivity loss (the basis of disability reporting for doctors, insurers and employers).
10. **Location context** — "Where were you when the migraine started?" (home, work, outdoors, travelling, etc.)
11. **Suspected triggers** — "Can you guess what could have triggered it?" Dozens of built-in options across food, sleep, stress, hormones, sensory, environmental and activity categories.
12. **Menstrual cycle status** — "Were you having your menstrual cycles?" Shown conditionally.
13. **Free-text notes** — attach arbitrary detail to any attack.
14. **Automatic weather capture** — temperature, humidity and barometric pressure at the time and place of onset are recorded for you, with no manual entry.
15. **Automated post-entry summary** — a recap of the attack you just logged.

### Recording variants and edge cases

- **Record a past (retrospective) attack** — backdate entries you missed.
- **Record an ongoing attack** and update it as it progresses.
- **Duplicate an attack** — clone a previous entry when an attack repeats identically, to avoid re-answering every screen.
- **Chronic migraine mode** — dedicated guidance and a "chronic migraine recording tips" program for people with near-daily attacks, where per-attack logging is impractical. Concretely, this means users can open a **daily, weekly or monthly record**, add timestamped free-text entries in the Notes section as pain intensity or symptoms change through the episode, and press **"Remind me later"** so the record stays open rather than being force-closed, letting them return and keep adding to it. This means Migraine Buddy **can** hold a multi-week continuous episode as a single open record rather than forcing it into discrete daily attacks. The weakness is what happens inside that open record: intra-episode variation (pain spikes, new symptoms, medication response) lives only in **unstructured, timestamped free-text notes** — there is no structured field capturing how severity or symptoms changed over the course of the open record, so that detail is unqueryable and does not export as structured data.
- **Custom attack types** — add your own attack classifications beyond the defaults.
- **Attack-type selection, including prodrome and postdrome as selectable attack types — not phases of an attack.** Per the Help Center: "Select the type of attack you wish to record, such as migraine, tension-type headache, cluster headache, prodrome and postdrome," and users can "add your own attack types." Prodrome and postdrome are recorded as **separate, sibling records** alongside migraine, tension-type and cluster headache — not as linked stages of a single migraine event. There is **no linkage from a prodrome or postdrome record to a specific subsequent or preceding attack, and no phase graph** connecting them into one episode. The Migraine Impact Report does allow a user to retrospectively change an already-logged record's type between prodrome and postdrome after the fact, but this only edits that record's own label — it does not attach the record to a specific later attack.
- **Record another condition** — track non-migraine conditions (e.g. tension headache, cluster, sinus) alongside migraine in the same app.
- **Edit any already-logged attack** after the fact. **Non-overlapping-records constraint:** the Help Center warns "Be careful of creating overlapping records when editing the Start time or End time of your attacks" — records occupy exclusive time intervals, consistent with an attack-centric schema where each record owns a non-overlapping start/end window rather than allowing concurrent or nested records.

### Customization of the recording flow

- **Add or delete medications, symptoms and triggers** from the pick lists, including entirely custom entries.
- **Rearrange the icon order** on recording screens so your most common selections come first.
- **Skip screens** you don't care about, shortening the flow permanently.

---

## 4. Medication and Treatment Management

- **Acute medication logging** with drug name, dose and exact time.
- **"Migraine cocktail" support** — log multi-drug combinations taken together, which the marketing explicitly calls out.
- **Preventive medication tracking** — a separate class from acute meds, with its own setup flow ("How do I add my preventive medications?").
- **Medication reminders** — schedule reminders for preventive treatments directly from the home screen. This is a recent addition, shipped in the August 2026 release.
- **Refill awareness** — reviewers describe the app helping track when medication needs re-taking and refilling.
- **Treatment effectiveness comparison** — compare different medications and relief methods against attack duration, severity and frequency to see what actually works for you.
- **Medication-overuse visibility** — because doses and dates are captured, the Migraine Impact Report surfaces medication-intake patterns relevant to medication-overuse headache assessment.

---

## 5. Automatic and Passive Tracking

- **Automatic sleep detection (Android)** — the app infers sleep and wake using the phone's **motion sensors and microphone**, requiring no wearable. There is a dedicated Help Center article for activating sleep detection on Android.
- **Sleep–attack correlation** — sleep duration and disruption are analysed against migraine onset.
- **Automatic weather logging** at attack onset (temperature, humidity, barometric pressure) using device location.
- **Location-based context** — needs location permission to attach weather and place data.
- **Migraine-free day counter** — the app proactively counts and celebrates your good days rather than only surfacing bad ones; users repeatedly cite this as a morale feature.

---

## 6. Integrations (Android-specific)

- **Health Connect** — the Android integration hub. Migraine Buddy reads/centralizes:
  - Sleep
  - Exercise / activity
  - Menstrual cycle
  - Heart rate variability (HRV)
  - Hydration
- **Wearables via Health Connect** — any tracker that writes to Health Connect (Fitbit, Samsung Health, Garmin, Google Fit successors) can feed the above data streams.
- On iOS the equivalents are **Apple Watch and a direct Fitbit link — with no Apple Health sync.** Android users get the same data classes through Health Connect rather than per-vendor integrations.
- **Oura Ring integration** — announced 27 January 2026.
- **Cloud sync across iOS and Android** — the same account works on both platforms and data follows you between devices.

---

## 7. Weather Features

- **7-day weather forecast** built specifically around migraine-relevant variables: **barometric pressure change, temperature change and humidity**.
- **Daily pressure forecast / pressure-change alerts** warning you ahead of a swing that historically precedes your attacks.
- **Weather analysis in reports** — correlations between your attacks and temperature, pressure and humidity.
- **Pressure-variation tracking relative to attacks** — an MBplus-tier analysis.
- Requires location permission; the Help Center has dedicated articles for activating the forecast and granting location access.

---

## 8. Analytics, Insights and Reports (in-app)

- **Charts, graphs and timeline views** of your migraine history — frequency, duration, intensity over time.
- **Pattern and trigger analysis** — which candidate triggers actually co-occur with your attacks.
- **Trigger-vs-symptom cross reporting** — advanced reporting of triggers against symptoms, pain location and pain intensity (MBplus).
- **Benchmarking against similar users** — your triggers and symptoms compared with the aggregate data of users with comparable profiles. This requires several logged attacks before it becomes meaningful.
- **Historical pattern analysis** over your full log.
- **Pain and symptom progression** — how pain and symptoms changed *through* an attack, not just a single snapshot (MBplus).
- **Migraine Insights & News** — in-app feed of migraine research updates, articles and questionnaires; contributing answers feeds aggregate community data.
- **Educational library** — an extensive set of articles, tips and management guidance, plus the external blog (categories: General, Life With Migraine, Reliefs, Diagnosis, Symptoms, Triggers) and Help Center.

### AI-driven features

- **AI-powered migraine insights.**
- **Symptom and trigger suggestions** — the app proposes likely entries based on your history.
- **Personalized trigger recommendations** — currently **iOS only**.
- **End-of-attack prediction** — predicts when the current attack will end; **MBplus and currently iOS only**.
- **Predictive risk analysis** — advanced risk modelling in the premium tier.

> **Android caveat worth knowing:** two of the marquee AI features (end-of-attack prediction and personalized trigger recommendations) are documented as iOS-only on the MBplus page. Android parity is not guaranteed.

---

## 9. Reports and Exports for Your Medical Team

- **Migraine Impact Report (MIR)** — the flagship clinical export. Presents migraine and medication insights in a compact, clinician-readable format designed to speed up consultation decisions: attack frequency, patterns, triggers, symptoms and medication intake. A sample report is published on the website.
- **Prodrome and postdrome inclusion in the MIR** — explicitly supported and documented.
- **Diary export** — the fuller, raw record of attacks for deeper pattern review.
- **Multiple shareable output formats**, exported from the app and sent ahead of appointments (the company does not publish an explicit format list; PDF is the practical share format users describe).
- **Positioning for third parties** — the reports are pitched not only at neurologists and headache specialists but at **insurers and employers**, for documenting disability and work impact.
- **Clinically-scoped data collection** — the company states it collects only clinically relevant data points, which is why the field set maps onto ICHD-style diagnostic criteria.

---

## 10. Community and Support

- **In-app community** of 3.5–4.8 million users (figure varies by source) for peer support and experience sharing.
- **Chat groups** — joinable in-app groups, with a Help Center article covering how to join one.
- **Shared-trigger insights** — see which triggers are common among users like you.
- **Questionnaires and polls** feeding aggregate insights back to the community.
- **Research contribution** — logged data is used in aggregate for migraine research and awareness (Migraine Buddy data is **reported, but not independently verified,** to have appeared in peer-reviewed headache literature).
- **Help Center** organised into: Account & Data, Using Migraine Buddy, Subscription & Billing, Troubleshooting, Record an Attack, Security.
- **Direct support contact** from inside the app.

---

## 11. Notifications and Reminders

- **Medication reminders** for preventive treatments, schedulable from the home screen (new as of August 2026).
- **Tracking prompts / timely notifications** to log attacks and follow-ups.
- **Weather and pressure-change alerts.**
- **Granular notification control** — a documented flow for turning Migraine Buddy notifications off.

---

## 12. MBplus — The Premium Subscription

MBplus is the paid tier. Everything below is gated behind it:

**Trigger and forecast**
- 7-day detailed weather forecast with humidity and pressure
- Advanced cycle, pressure-variation and sleep tracking relative to attacks
- Personalized trigger recommendations *(iOS only)*

**Analysis and reporting**
- Weather analysis (temperature, pressure, humidity correlations)
- Advanced reporting of triggers against symptoms, pain location and intensity
- Historical data pattern analysis
- Predictive risk analysis

**Symptom insights**
- End-of-attack time prediction *(iOS only)*
- Pain and symptom progression through an attack
- Exact-time logging for medications and relief methods

**Coaching and education**
- Interactive, stepwise coaching programs from migraine experts
- Good Days Guide (food and lifestyle trigger identification)
- Expert resource library

### Actionable / coaching programs

Programs are "action plans designed by world-leading migraine experts" spanning diet, emotional and relationship management, triggers and relief methods. Named programs include:

- Weather Trends and Migraine
- Get Food Cravings Under Control
- Is Your Migraine Related to Hormones?
- 10 Tips and 5 Recipes for a Migraine-Friendly Diet
- Good Days Program (GDP) — food and lifestyle coaching, run partly through a separate coaching site
- Chronic Migraine recording-tips program

### Pricing (US, as listed)

| Plan | Headline rate | Billed |
|---|---|---|
| Monthly | $9.99 or $12.99 | monthly (tier varies by offer) |
| Two-month | $12.49/mo | $24.99 |
| Annual | $7.49/mo | $89.99 (promo code `save20` seen for $20 off) |
| Annual (discounted tiers) | — | $69.99 and other 30%/50%-off variants |

Multiple parallel price points, free-trial and no-free-trial SKUs exist, so **the price you're shown depends on the offer cohort you land in**. Both the two-month and yearly plans carry identical feature access. Store-billed subscriptions are managed through Google Play; there is also a separate web storefront (migrainebuddyplus.com) selling an annual subscription.

---

## 13. Free vs Premium — Practical Split

| Capability | Free | MBplus |
|---|---|---|
| Attack recording (full wizard) | ✅ | ✅ |
| Custom triggers/symptoms/medications | ✅ | ✅ |
| Pain-map location entry | ✅ | ✅ |
| Automatic sleep detection | ✅ | ✅ (plus sleep-vs-attack analysis) |
| Automatic weather capture at onset | ✅ | ✅ |
| Health Connect integration | ✅ | ✅ |
| Basic charts and history | ✅ | ✅ |
| Community and chat groups | ✅ | ✅ |
| Diary export / MIR export | ✅ (core export is advertised as a free-tier feature) | ✅ |
| 7-day pressure/humidity forecast | ❌ | ✅ |
| Weather correlation analysis | ❌ | ✅ |
| Advanced trigger × symptom × location reporting | ❌ | ✅ |
| Predictive risk analysis | ❌ | ✅ |
| End-of-attack prediction | ❌ | ✅ *(iOS)* |
| Symptom progression through attack | ❌ | ✅ |
| Exact-time medication/relief logging | ❌ | ✅ |
| Coaching programs / Good Days Guide | ❌ | ✅ |

*One competitor's comparison claims free-tier limits on history length and analytics; the official listing does not state a history cap, and the Play listing reports no ads. Treat that claim as unverified.*

---

## 14. Data, Privacy and Security

- **No third-party data sharing** declared on the Play data-safety card.
- **Data encrypted in transit.**
- **Collects location, personal info and five further data categories** (per the data-safety declaration).
- **Users can request data deletion.**
- **Server-side storage with account-based sync** — this is what enables multi-device use and restore, but it does mean health data leaves the device.
- **Backup and restore** — documented flows for backing up and restoring your log.
- **Known sync caveat** — the Help Center has an article for "Why do I see different data on two devices if the account is the same?", so sync conflicts do occur in practice.
- **Aggregate research use** of de-identified data.
- The app carries an explicit disclaimer that it is a **self-management tool and not a substitute for professional medical advice**.

---

## 15. Known Limitations and Friction Points

- **The multi-step wizard is thorough but long** — several reviewers note it can feel overwhelming *during* an active attack, which is exactly when you're meant to use it. Screen-skipping and attack duplication are the intended mitigations.
- **Feature parity gap**: end-of-attack prediction and personalized trigger recommendations are iOS-only.
- **No desktop or web app** — mobile only.
- **Account mandatory**; no offline-only / on-device-only mode.
- **Single profile per account** — no family or caregiver multi-profile support.
- **Best value at higher attack frequency** — reviewers suggest it's over-engineered for someone with one migraine a month or fewer.
- **Confusing price landscape** — many concurrent SKUs and discount tiers mean two users can pay very different amounts for the same features.
- **Large install footprint** (~173 MB).
- **Location permission is effectively required** for the weather features, which are among the app's biggest differentiators.
- **No Apple Health sync.** On iOS, integrations are limited to Apple Watch and a direct Fitbit link; there is no Apple Health integration at all. This is the **top request** on the company's own public feedback board (https://131xmm.nolt.io/287).
- **Non-overlapping-records constraint.** Records occupy exclusive start/end intervals — the Help Center explicitly warns against creating overlapping records when editing an attack's start or end time — so the schema cannot represent concurrent or nested records, only sequential, non-overlapping ones.

---

## Sources

- [Migraine Buddy: Track Headache – Google Play (US)](https://play.google.com/store/apps/details?id=com.healint.migraineapp&hl=en_US)
- [Migraine Buddy – Track&Control – Google Play (UK)](https://play.google.com/store/apps/details?id=com.healint.migraineapp&hl=en_GB)
- [Migraine Buddy official site](https://migrainebuddy.com/)
- [MBplus premium features and pricing](https://migrainebuddy.com/mbplus/)
- [Actionable Programs](https://migrainebuddy.com/actionable-programs/)
- [For Your Medical Team (MIR reports)](https://migrainebuddy.com/for-your-medical-team/)
- [How to Record a Migraine](https://migrainebuddy.com/how-to-record-a-migraine/)
- [What Is Migraine Buddy](https://migrainebuddy.com/what-is-migraine-buddy-free-and-easy-migraine-tracker/)
- [Help Center — Using Migraine Buddy](https://migrainebuddy.com/help-center/category/using-migraine-buddy/)
- [Help Center — Record an Attack](https://migrainebuddy.com/help-center/category/record-an-attack/)
- [Help Center — Account & Data](https://migrainebuddy.com/help-center/category/account-data/)
- [Help Center index](https://migrainebuddy.com/help-center/)
- [Migraine Buddy: Track Headache – Apple App Store (feature/version/pricing detail)](https://apps.apple.com/us/app/migraine-buddy-track-headache/id975074413)
- [Aptar Digital Health product page](https://aptardigitalhealth.com/products/migrainebuddy-leading-app-for-migraine-and-headache/)
- [AppBrain listing (version, size, permissions)](https://www.appbrain.com/app/migraine-buddy-track-headache/com.healint.migraineapp)
- [Soft112 listing (verbatim description)](https://migraine-buddy.soft112.com/)
- [HealthMatch — Migraine Buddy: What Is It?](https://healthmatch.io/migraine/migraine-buddy)
- [MigrAid vs Migraine Buddy (competitor comparison — treat claims with caution)](https://migraid.app/blog/migraid-vs-migraine-buddy)
- [Medical News Today — best migraine apps](https://www.medicalnewstoday.com/articles/319508)
- [MBplus Good Days Program](https://www.coaching.migrainebuddy.com/gdp-good-days-program)
