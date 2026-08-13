# Research — Product Architecture & User Flows

**HeadState (headstate.health) · researched August 2026**

> **Note on the data model in this document.** The event schema and entity-extraction model described here were subsequently superseded by the continuous-timeline model documented in `continuous-timeline-verification.md` and `../headstate-strategy-brief.md`, in which attacks/phases/day-classifications are derived views over a continuous observation stream rather than the atomic unit.
>
> **Still current:** the conversational extraction discipline (tri-state clinical fields, per-field provenance and confidence, follow-up-versus-null rules, the anti-fabrication constraint), the correction-as-core-interaction pattern, the mid-attack flow, retrospective logging, and the differentiation and risk analysis.
>
> **Superseded, with pointers:**
> - The **event-centric schema** — replaced by the immutable-observation plus derived-projection model in `../headstate-strategy-brief.md` and `../CLAUDE.md`.
> - The **information architecture**, which is keyed on `/event/:id` and a list of "events" — needs re-keying to observations, with episodes as derived views.
> - **Principle 1's** claim that UI controls exist "never the primary path" — `../CLAUDE.md` establishes that text input is first-class, not a fallback, because voice is unusable on transit, in shared offices and around family.
> - **Flow D's** treatment of silence as an inferred "no attack reported" — superseded by the npj event-diary practice of capturing an active "no events" confirmation rather than inferring from silence.
> - The **MVP scope list**, which includes basic Insights in v1 and omits derived episodes, day classification and the prior-authorisation packet — the current build order in `../README.md` and the strategy brief places derived episodes third, correlation sixth and the prior-auth packet seventh.
>
> The sections below are kept as the record of this design pass and have not been rewritten; read them against the map above rather than as current on every point.

---

## Product principles

Seven principles, each stated with its own tradeoff or risk.

1. **Talk first, tap to correct.** Primary input is speech-to-structured-data; every screen has a mic as the default affordance; UI controls exist only as fallback or correction, never the primary path. *Risk:* pushes almost all product risk into NLU quality — a mediocre parser is far more damaging than a mediocre form, because there's no form to fall back on gracefully.

2. **Log the absence, not just the attack.** A "no migraine today" day is a first-class record, not a null. Trigger inference is a paired-sample problem (attack days vs non-attack days), so under-logging good days silently destroys the core value prop. *Risk:* requires some daily engagement ask, which is exactly the mechanic every symptom tracker's retention curve dies on.

3. **Capture now, structure later.** The instant of highest information (mid-attack) is the worst instant to ask questions. Real-time capture must never block on completeness. *Risk:* records live in a "provisional" state for a while; needs a reliable low-friction re-engagement mechanism or data stays permanently thin.

4. **One sentence beats thirteen fields.** A user should say one unstructured thing and have most of the record populate. Extraction, not interrogation. *Risk:* ambiguity is inevitable; the system must be disciplined about leaving fields null rather than guessing.

5. **Passive is default, active recall is the exception.** Sleep, weather, cycle/HR should arrive without being asked. Conversation reserved for what a phone cannot infer — how bad it was, where it hurt, what helped. *Risk:* makes the product materially worse for users without integrations connected, and requires real integration engineering before the "no nagging" promise is even true.

6. **Every record is provider-legible by construction.** There is always a structured, clinically-mappable object behind the conversation — not a chat transcript a clinician must read. *Risk:* constrains how loose the conversational layer can be; some clinically-relevant fields (laterality, aura, functional impact) need near-mandatory normalisation even though the interaction feels free-form.

7. **Correction is a core interaction, not a bug-fix screen.** The user sees what the AI understood immediately and can override any field in one tap or one sentence, every time, forever. *Risk:* surfacing "here's what I heard" after every utterance adds a UI beat; it repeatedly exposes the AI's mistakes, which is a trust cost even as it's a trust builder.

---

## Information architecture

Six live surfaces plus onboarding.

| Surface (route) | Job | Deliberately does NOT do |
|---|---|---|
| Home (`/`) | Primary logging surface — conversation feed, mic + text input, current status pill ("In a migraine" / "Symptom-free"), today's auto-populated context card inline | No forms, no multi-step wizards, no nested menus; doesn't try to be a dashboard |
| Timeline (`/timeline`) | Chronological scannable list of every event (attacks + daily context), expandable; the audit trail / "everything I've told it" | No analysis, no charts, no correlation claims — pure record |
| Event detail (`/event/:id`) | The ground-truth structured record for one event — every field, editable, with source/confidence tags | Doesn't reopen the chat; it's form-like precisely because correction should be fast and unambiguous, not conversational |
| Insights (`/insights`) | Pattern review — candidate triggers ranked by evidence strength, plain-language summaries, conversational Q&A over your own history | Doesn't let you log from here, doesn't assert causation, doesn't forecast |
| Report (`/report`) | Generate, preview and share a clinician-facing summary for a date range | No chat, no editorialising beyond the structured data; not a portal or messaging tool |
| Settings (`/settings`) | Integrations (Health Connect/Apple Health, calendar, weather, voice assistant setup), notification cadence, export/delete data, sharing permissions | No feature-discovery/marketing surface; utilitarian only |
| Onboarding (`/onboarding`, first run only) | Conversational baseline capture + permission grants | Never reappears as a "complete your profile" nag later |

---

## Core flows

### A. First-run onboarding (~2 minutes, no 20-question intake)

1. Welcome screen: one-line value prop + privacy line + "Let's set you up in under 2 minutes."
2. Open prompt (voice or text): "Tell me about your migraines — how often, what they're usually like, and what you're currently taking."
3. User gives one free-form paragraph. App extracts frequency band, typical duration/severity, known triggers, current meds/preventives, aura history — shown as editable confirmation cards, not re-asked one at a time.
4. Permission asks framed as labour-saving, not compliance: "Connect your health data so I can stop asking about sleep." → Health Connect/Apple Health, location (weather), notifications, optional cycle tracking, optional Siri Shortcut setup.
5. Single calibration question: "Are you in a migraine right now, or symptom-free today?" — seeds current state.
6. Close: "I'll check in briefly, about once a day. Anytime you're having one, just say 'I have a migraine.'" → lands on Home.

### B. Mid-attack logging (voice, dark, one-handed, cognitively impaired)

1. Activation without navigation: lock-screen widget, Siri Shortcut ("Hey Siri, log migraine"), or one oversized mic target on Home — no unlock-and-hunt.
2. Haptic/audio confirmation only: "I'm here."
3. User: "It's happening again, right behind my left eye, really bad."
4. App writes the record immediately (onset=now, location=left/behind eye, severity≈severe from verbal intensity) and replies with the shortest possible confirmation: "Logged. Feel better." NO follow-up questions fired mid-attack.
5. If the user keeps talking — "took two Excedrin" — it's appended silently with a single haptic tick, zero back-and-forth.
6. Re-engagement is app-initiated, later, once the attack has likely resolved (timer or user says "I'm better"): "Glad that's easing up. Still behind the left eye, or did it move? Anything help?" — this is where richer detail (aura, full symptom list, relief efficacy) gets collected, once cognition has returned.
7. All of it stitches into ONE event record (onset → meds → resolution), not three disconnected logs.

### C. Retrospective logging

1. User, days later, in normal chat: "I had a bad one Tuesday afternoon, forgot to log it."
2. App resolves "Tuesday" against the calendar, confirms nothing already exists for that slot, opens a draft event anchored there.
3. App asks 1–2 targeted questions, using passive data as a hint rather than a blind ask: "Got it, adding that to Tuesday afternoon. Your sleep data shows only 4 hours Monday night — related? And how bad, roughly?"
4. User answers in one breath: "Barely slept, an 8 out of 10, couldn't work the afternoon."
5. Severity, functional impact and a CANDIDATE trigger link (short sleep) are filled — the trigger link shown as a suggestion for confirm/deny, never asserted as fact.
6. Final card: "Tuesday, 1–5pm, severe, missed work. Possible link: short sleep Monday night." — tap any field to adjust.

### D. Non-attack / daily check-in (the hard one)

- Passive by default: sleep and where available HR/HRV from Health Connect/Apple Health; weather from location; cycle day from a connected period app or opt-in in-app tracking. None requires the user to say anything.
- One optional daily nudge, not mandatory: an evening (or usage-time-adaptive) notification with three one-tap answers — Great / Okay / Rough — plus a mic icon for anyone who wants to say more. Skipping is silent and penalty-free.
- Inference over interrogation: if the user does nothing, the app treats the day as an unconfirmed "no attack reported" rather than forcing a report — but tags that day's status as *inferred*, not *confirmed*, in the data model, so correlation analysis can weight it differently from an explicit "I felt fine."
- Contextual, not scheduled, deeper prompts: occasionally, only when passive signals are unusual (bad sleep + a weather front + cycle day 1), the app asks a real question: "Rough sleep last night and a pressure drop today — how are you feeling?" This targets the highest-information moments instead of asking the same thing daily, which causes habituation and dismissal.
- Weekly, not daily, pulse for what passive data can't get: stress, hydration, meals — bundled once a week: "How was your stress and eating this week, generally?"
- Net effect: a continuous background-context record exists even on days the user never opens the app — the actual prerequisite for trigger inference — but every field's confirmation status (confirmed vs assumed) travels with the data, not just the value.

### E. Reviewing your own patterns

1. User asks conversationally or opens Insights: "Why do I keep getting migraines on Mondays?"
2. App answers in plain language with the evidence behind it, not a bare chart: "6 of your last 9 attacks started Monday or Tuesday morning. The common thread looks like short sleep Sunday night — 5.2h average vs your usual 7h. Want to see those days side by side?"
3. Candidate triggers ranked by supporting evidence and explicitly framed as "possible pattern, N supporting days, keep tracking" — never causal or certain, given realistically small N per user.
4. User can ask arbitrary follow-ups ("did stress or weather matter more this month?") — a chat-native analytics interface, not a fixed dashboard the user must learn to read.

### F. Generating and sharing a provider report

1. User: "I have a neurology appointment Thursday, make a report" or taps Report.
2. App proposes a sensible default range and scope: "I'll cover the last 3 months — 14 attacks logged. Include medication response and trigger notes too?"
3. Generates a clinician-oriented summary: attack frequency, characteristics (location, quality, duration, associated symptoms, aura) in terms a neurologist recognises, acute-medication use with overuse flags, functional-impact days, candidate patterns clearly labelled patient-reported/inferred rather than diagnostic.
4. Plain structured preview (not chat-shaped) — sections can be toggled off before sharing (e.g. exclude mood notes) for the user's own privacy control.
5. Share via PDF download, share sheet, or an expiring link; a "one-page summary" option alongside the full log.

### G. Correcting the AI when it mis-parses

1. App logs from voice: "Logged: severe migraine, right temple." User actually said left.
2. Every logged utterance produces an immediate visible receipt with a lightweight "not quite?" affordance right next to it — so the error is visible within seconds, not discovered days later in a report.
3. User taps the field (or says "no, it's the left side") — only that field updates; the rest of the record untouched, re-confirmed in one line.
4. If a correction pattern repeats (e.g. consistent left/right confusion), the system quietly lowers confidence on that field type and is more likely to ask explicitly next time — an adaptive loop, not a user-facing ML settings panel.
5. The same correction affordance exists later from Event Detail, so mistakes caught while reviewing a report are just as fixable as ones caught in the moment.

---

## The conversational data model

The crux of the product. See the superseded-model note at the top of this document — this section reflects the earlier, attack-centric event schema.

**Entity extraction targets per utterance**, mapped into one or more events:

- Event type: attack start / attack update / attack resolution / medication-only / context note / daily check-in
- Timestamps: onset, resolution (both nullable, both resolvable from relative language — "last night," "an hour ago")
- Pain descriptors: laterality + region (multi-value — pain moves), quality (throbbing/pressure/stabbing/other), severity (0–10, or mapped from verbal intensity language)
- Associated symptoms: nausea, vomiting, photophobia, phonophobia, dizziness, aura (presence, type, duration) — each as a TRI-STATE, not boolean
- Medications: name, dose, time, perceived relief
- Functional impact: none/mild/moderate/severe (e.g. missed work, cancelled plans)
- Trigger/context mentions: only what the user explicitly asserts, normalised to tags, never inferred into this field automatically
- Free text: the verbatim utterance always retained alongside the structured fields, as audit trail and fallback

**Ambiguity and missing fields.** Fields are nullable with a TRI-STATE distinction — *not asked*, *asked and answered no*, *asked and unknown* — because clinically "no nausea" and "we never discussed nausea" are different facts and collapsing them into a false boolean corrupts the report's clinical value. The system is never allowed to fill a field it wasn't given evidence for.

**When to ask a follow-up vs leave null.** A follow-up is only justified when all three hold — the field is high clinical/insight value, the moment is appropriate (not mid-attack, not a fourth question in one exchange), and it can be asked in one short sentence. Otherwise the field stays null, permanently if need be. Clinician intake forms have blanks too; a null is honest, a guess is not.

**Confidence scoring.** Every extracted field carries a confidence score reflecting whether it came from an explicit lexical match ("8 out of 10," "left side") versus a paraphrase/inference ("really bad" → severe). Low-confidence fields are flagged for review and prioritised for a later follow-up — never silently written as certain.

**Review and correction is structural, not optional.** Every parse produces a visible "receipt" card before being treated as final, collapsed by default, one tap from full edit.

**Avoiding fabrication — the hard constraint the whole model rests on.** Extraction must be strictly grounded in (a) the transcript span or (b) an explicitly source-tagged passive-data feed — never open generation dressed as fact. Practically: use structured extraction (function-calling / schema-constrained output) against the transcript rather than free generation, and run a second-pass check that every populated field traces back to a specific span of what was said or a named passive source before it's committed. The record carries provenance per field — "you said," "estimated from sleep data," "suggested pattern, unconfirmed" — and that provenance travels all the way into the provider report, because a clinician reading "nausea: yes" needs to know if that's self-report or an LLM's inference from adjacent context.

**Minimum viable structured schema for a headache event:**

```
event_id, user_id, event_type
onset_ts, resolution_ts (nullable)
severity (0–10, nullable, + source)
pain_location[] (laterality + region)
pain_quality (enum, nullable)
associated_symptoms{} (nausea, vomiting, photophobia, phonophobia, dizziness — each tri-state)
aura{ present/absent/unknown, type, duration }
functional_impact (none/mild/moderate/severe)
medications[]{ name, dose, time, route, perceived_relief }
triggers_reported[] (user-asserted only, free text + normalised tag)
context_snapshot{ sleep_hours, weather, cycle_day, stress_self_report — each tagged passive/inferred/self-reported }
free_text_note (verbatim)
field_confidence{} / field_source{} (per-field provenance)
created_via (voice/text/passive), last_edited_by (user/system)
```

---

## Differentiation strategy

**Be sharply different on:**

- Input modality at the moment of highest cost — Migraine Buddy's long wizard is most burdensome exactly during an attack; collapsing that to one sentence and deferring the rest is the wedge, and should be the entire first impression. See `competitive-landscape.md`.
- Symmetric treatment of non-attack days.
- Provider-report quality as a native, un-gated output — if the pitch is "provider-grade data," gating the one thing a clinician sees behind a paywall undercuts the whole positioning.
- Honesty about confidence — show evidence strength and provenance instead of asserting causal triggers or forecasts from thin data.

**Do NOT try to beat them on at launch:**

- Integration breadth — Migraine Buddy has years of maturity in Health Connect, wearables, weather; support the two or three that matter and don't chase parity.
- Community/social features — tonally wrong for a chronic-pain log, a distraction from the core loop, not a retention lever worth building.
- Forecasting/coaching ML sophistication — scientifically shaky at small N per user; compete on epistemic honesty instead.
- Scope breadth — don't become a general symptom/mood tracker, that's Bearable's lane; stay migraine-adjacent so the conversational model can specialise.
- Gamification — streaks and badges read wrong applied to pain tracking; the habit hook should be personal insight payoff, not points.

---

## MVP scope vs deferred

**V1:**

- Voice + text conversational logging for attack start/update/resolution, meds, severity, location, key associated symptoms
- Retrospective logging with relative-date resolution
- Daily lightweight check-in (3-tap + optional voice) plus passive sleep import as the two non-attack data sources
- Full structured event schema with per-field provenance/confidence and the Event Detail correction UI
- Basic Insights (ranked candidate-trigger list with supporting evidence, no forecasting)
- Provider report export (PDF), core clinical fields, un-gated
- Installable offline-capable PWA with a local queue that syncs when connectivity returns (attacks don't wait for wifi)
- The onboarding flow
- Baseline privacy (encryption, full export/delete, no ad-tracking or third-party data sale)

**Deferred:**

- Native Siri/Gemini assistant integration beyond a basic Siri Shortcut wrapper — see `assistant-integration.md`
- Weather forecasting and predictive alerts
- Wearable/HRV-based physiological trigger detection beyond basic sleep import
- Hormonal/cycle correlation modelling beyond simple self-report or import
- Community/social features (possibly never)
- Multi-condition or general symptom tracking
- Clinician-side portal, messaging, or FHIR/EHR integration
- Coaching/behaviour-change programmes
- Multi-language voice beyond the initial language
- Medication interaction/dosing-safety database

---

## Key risks and open questions

**Retention is the dominant risk and it's structural, not cosmetic.** Benchmarks: roughly 70–80% of users never return after day one; fitness-app median day-30 retention around 3–4%; median 70% discontinuation within 100 days even among engaged categories, with steepest drop-off in the first two weeks. The daily check-in, however light, sits on the exact mechanism that kills these apps. Passive-first design mitigates but doesn't solve it — sleep import alone can't capture stress, hydration or meals, so there's real tension between "never nag" and "need daily context to do trigger inference at all." Open question: how many actively-answered days per week is realistic before churn sets in, and does letting silence default to "assumed fine" bias the dataset toward under-reporting mild days?

**Cold start for trigger inference.** Correlation work needs weeks to months of paired attack/non-attack data before it says anything useful — so new users hit "not enough data yet" for exactly the period they're most likely to churn. Week-one value has to come from somewhere other than pattern detection (a calmer logging experience itself; an immediate summary ready for an upcoming appointment) or the product has nothing to show early adopters.

**Conversational input may produce messier data than forms, not cleaner.** Ambiguous time references ("last night" said at 1am), compound utterances describing multiple events at once, non-clinical vocabulary, ASR errors and ordinary speech disfluency all have to be resolved by the extraction layer. There's also a situational-availability problem forms don't have: voice is often unusable in the exact moments quick logging matters most — public transit, a shared office, family in the room — so text must be a fully first-class fallback or the "voice-first" promise breaks exactly when needed. And there's a risk the conversational layer recreates the wizard it's trying to escape, just in prose: too many clarifying questions per turn is the same burden with worse UI.

**Fabrication risk carries real downstream weight.** A misparsed field the user only ever sees themselves is low-stakes; a misparsed field that ends up on a report handed to a physician is a different category of harm. Argues for a conservative extraction posture and clear framing that the output is a self-report log, not a diagnostic instrument. See `clinical-regulatory.md`.

**Monetisation is in tension with the differentiation strategy.** The stated wedge is better correlation/report output as core value; gating that behind a paywall (as Migraine Buddy does with MBplus) undercuts the pitch. Open question: what's actually paywalled — voice usage volume, integration count, history length — without touching the two outputs the product exists to deliver.

**Voice-assistant platform dependency.** The Siri/Gemini aspiration depends on third-party APIs and app-intent capabilities outside HeadState's control and still evolving — cannot be a v1 dependency without a solid in-app fallback. See `assistant-integration.md`.

**Data sensitivity.** Migraine logs combined with cycle data and medication history are high-sensitivity; where ASR/LLM extraction actually runs (on-device vs cloud) and how that's communicated matters more here than in a typical consumer app, particularly given heightened post-2022 wariness around cycle-tracking data specifically. See `clinical-regulatory.md`.

---

## Related documents

- `competitive-landscape.md` — market and incumbent analysis
- `assistant-integration.md` — reaching HeadState from Claude/ChatGPT/Gemini/Siri, PWA vs native constraints
- `clinical-regulatory.md` — clinical and regulatory considerations
- `continuous-timeline-verification.md` — the current data model, superseding the event schema in this document
- `qa-verification-report.md` — independent verification of findings across this research set
- `../headstate-strategy-brief.md` — parent strategy brief

---

## Sources

- [Why Most Health App Users Churn Within 90 Days — Sahha](https://sahha.ai/blog/health-app-churn-retention/)
- [Health & Fitness App Benchmarks (2026) — Business of Apps](https://www.businessofapps.com/data/health-fitness-app-benchmarks/)
- [Migraine Buddy App Review and More — Migraine Road](https://www.migraineroad.com/migraine-buddy-app-review-and-more/)
- [Migraine Buddy App Review — CEFALY](https://www.cefaly.com/blog/migraine-buddy-review)
- [Top 5 Migraine Tracking Apps — Axon Optics](https://axonoptics.com/blogs/post/top-5-migraine-tracking-apps)
- [Bearable Symptom Tracker App](https://bearable.app/)
- [From Conversation to Standardized Terminology: An LLM-RAG Approach for Automated Health Problem Identification (PMC)](https://pmc.ncbi.nlm.nih.gov/articles/PMC12849705/)
- [LLM enabled classification of patient self-reported symptoms and needs (PMC)](https://pmc.ncbi.nlm.nih.gov/articles/PMC12215686/)
