# HeadState

HeadState is a migraine and symptom record built around a continuous observation timeline. Individual timestamped observations are the atomic unit; attacks, phases, episodes and day-classifications are derived, recomputable views over that stream. It is being built by Eric — a migraine patient — for his own use first, with commercialisation second. Domain: headstate.health.

## Why it exists

I had a continuous headache lasting roughly three weeks, with three emergency department presentations. On the second presentation, an abortive agent plus a corticosteroid produced a period of remission, after which the headache returned at close to its original intensity once the steroid cleared.

Two things about that episode were impossible to record usefully in the tracking app I was using. First, the episode fragmented: because the app is organised around discrete attacks, the medication-induced gaps split one continuous episode into what the record showed as several separate attacks, so every subsequent count described something that did not happen. Second, symptoms occurring on days without head pain had nowhere structured to live — I could type them as free-text notes but not later analyse them, count them, or relate them to what followed.

The question that motivated the project: *is the nausea I have right now the beginning of the next one?* That is not answerable prospectively. It is only answerable in retrospect, and only if the earlier observation was recorded in a form that can be re-examined.

## The three jobs

**Capture** — frictionless logging. One utterance becomes structured observations, with a visible receipt and one-tap correction. No wizard. Mid-attack: write the record, confirm briefly, ask nothing.

**Recall** — an on-demand one-page summary for an office visit or emergency presentation. Where the current episode stands and how long it has run; every medication and dose in the preceding 72 hours with times; baseline pattern and typical presentation; current preventives; recent acute-medication day count; what has been tried this episode and what happened. Must work offline, be reachable in one tap, be printable, and be shareable with someone who is not the patient — an ER physician will not browse an app.

**Documents** — structured output supporting prior authorisation. Headache days, migraine days and acute-medication days, each traceable to the published rule that produced it, with medication-overuse thresholds flagged against ICHD-3 and pharmacologically attenuated days marked rather than dropped.

## What is defensible — and what is not

Ranked by durability:

1. **The correction corpus.** Every time a user overrides an episode boundary, confirms a retrospective phase label, or rejects a proposal, that is ground truth on migraine structure that exists nowhere else. The algorithm is the mechanism; the accumulated corrections are the asset. It cannot be copied and it compounds.
2. **The reporting platform.** The algorithm produces numbers; the report is what gets a prior authorisation approved, what an ER physician reads, and what a neurologist trusts. Reports have a purchaser; numbers have a spec sheet.
3. **Correct by default.** Competitors do have relevant capabilities buried in them — Migraine Buddy's chronic mode can hold an open monthly record, Migraine Insight supports open-ended episodes — but a feature requiring a help-centre article is functionally not a feature, especially for someone photophobic and cognitively impaired. The claim is not "we can do what they can't"; it is "the clinically correct model is the default."
4. **Medication response as a phenotyping signal.** Treatment response used as evidence about structure rather than only as an outcome: a triptan working implies the headache phase had begun; a steroid producing multi-day remission followed by recurrence implies one continuous episode. No implementation of this was found. Caveat: triptan response as diagnostically informative has an older clinical literature, so the *idea* is not new — only, apparently, its use as a computational input to boundary resolution.
5. **Retrospective phase linkage.** No app **for which a data model could be documented** links an earlier observation to a phase of a later attack — 8 of the 21 apps surveyed are unverifiable on this capability, so treat the negative as a documented gap, not a confirmed absence across the whole market. The evidence for why it matters is strong: across 229 attacks in 34 patients, 69% had premonitory symptoms but only 27.5% were recognised before pain onset, with 85.1% positive predictive value and a mean lead time of 10 hours 38 minutes when patients did predict. Roughly 72% of premonitory data is only recoverable retrospectively.
6. **Attenuation-aware day classification.** No app or paper **for which a data model could be documented** models the possibility that a symptom-free window is drug-attenuated rather than genuinely clear — 5 of the 21 apps surveyed are unverifiable on this point.
7. **The standards gap.** ICHD-3 deliberately declines to resolve whether recurrence within 48 hours is a relapse or a new attack, and the 2026 Delphi consensus is silent on multi-day continuity, status migrainosus, midnight-spanning attacks and same-day recurrence. Storing raw observations and recomputing both readings fills a genuine gap rather than inventing a house rule.

**Honesty note:** the continuous-timeline architecture itself is *not* novel — it is the prevailing model in academic headache research (the Leiden e-headache diary already classifies days algorithmically from a daily observation stream; npj Digital Medicine published event-driven diary best practices using migraine as the worked example; the Delphi rules were written for automated e-diary labelling). What does not exist is an implementation a patient can actually use. That is an execution position, not an architectural one. None of the seven items above is a technology moat; all are copyable in a quarter. The durable assets are the corpus and the credibility.

## Repository structure

```
<project root>/
├── README.md
├── CLAUDE.md
├── decision-log.md
├── headstate-strategy-brief.md
├── HeadState-Clinical-Review.docx
└── Research/
    ├── continuous-timeline-verification.md
    ├── clinical-regulatory.md
    ├── qa-verification-report.md
    ├── assistant-integration.md
    ├── competitive-landscape.md
    ├── product-architecture.md
    ├── migraine-buddy-features.md
    └── open-question-recommendations.md
```

| File | Purpose |
|---|---|
| `CLAUDE.md` | Context and hard invariants for Claude Code sessions working in this repo; read before writing code |
| `decision-log.md` | The binding record of settled decisions — reasoning and consequences; wins over research documents where they disagree |
| `headstate-strategy-brief.md` | Current strategy: defensible position, data model, three jobs, build order, regulatory posture, risks |
| `HeadState-Clinical-Review.docx` | A seven-page document prepared for a neurologist, designed to be argued with; ends in ten questions |
| `Research/continuous-timeline-verification.md` | **Start here.** Independent verification of what is and is not novel; per-app capability matrix; ICHD-3 counting rules; premonitory and corticosteroid evidence; prior-auth counting |
| `Research/clinical-regulatory.md` | ICHD-3 fields, MOH thresholds, instrument licensing, FDA posture, privacy regimes, payer criteria, reimbursement pathways |
| `Research/qa-verification-report.md` | Independent adversarial QA of the strategy: 31 claims checked, 6 corrections, with URLs |
| `Research/assistant-integration.md` | MCP servers, Claude/ChatGPT/Gemini surfaces, Siri Shortcuts, auth patterns, PWA vs native constraints |
| `Research/competitive-landscape.md` | The incumbent, the 2026 AI wave, adjacent and clinical players, evidence-based gap analysis |
| `Research/product-architecture.md` | Product principles, information architecture, core user flows, conversational extraction discipline |
| `Research/migraine-buddy-features.md` | Complete feature inventory of the incumbent |
| `Research/open-question-recommendations.md` | Researched recommendations behind the granularity, recall-format, prodrome and automation-level decisions |

Source code will also live in this root going forward.

**Precedence when sources disagree:** `CLAUDE.md` governs implementation decisions; `Research/` documents govern evidence and citations. Where the two disagree, report the discrepancy rather than silently reconciling it. `decision-log.md` sits above both — it is the binding record of settled decisions, and research documents are the evidence behind a decision, not an alternative to it.

## Build order

Sequenced for the developer as the primary user, not for a market launch.
Capacitor is repository scaffolding, not a numbered step. The technical
specification and functional requirements are scoped to steps 1–4; steps 5–7
are sequenced but deliberately left unspecified until steps 1–4 reveal more.
See `decision-log.md` for the reasoning.

0. **Capacitor scaffold, repo setup, CI** — TypeScript end to end, Postgres, managed platform.
1. **Capture and the observation store** — immutable log, schema with the third review state, event-anchored capture with a next-morning backstop. Everything depends on this.
2. **Recall** — the one-page current-state brief. Printable, shareable, offline. Exempt from the confirmation gate. Pays back before there is enough data for anything else, which also answers the cold-start problem.
3. **Derived episodes and day classification** — ICHD-3 constants, Delphi rules, dual readings, user override. Backfill always counts. Where a multi-week episode becomes representable.
4. **Integrations** — HealthKit and Health Connect for sleep and weather first (highest signal, lowest effort), then cycle and HRV.
5. **Phase annotation and retrospective relabelling**, including medication-response inference. Operates on history, so it can wait. Also the point at which the deferred prodrome surface can be revisited with data.
6. **Correlation.** Needs the most data and is the least proven — a research bet, not a claim.
7. **Prior-authorisation packet.** Needs 30–90 days of clean data and needs to be right — last, despite being the most valuable, because it is only as good as everything above it. Gated by the mandatory confirmation gate.

## Key decisions already made

- The repository is scaffolded as a Capacitor project from commit one — no PWA-only period; the web app remains the core codebase and Capacitor is a wrapper, not a rewrite
- TypeScript end to end (React (D14) with Vite, Node for the API and MCP server) on Postgres, deployed on a managed platform (Fly.io (D14)) with managed Postgres
- One remote MCP server with OAuth 2.1 + PKCE, reused for the iOS Shortcuts path
- Patient-first direct-to-consumer, not provider-channel, at launch
- Payer exports report day counts, never attack counts
- No attack-risk prediction, and no trigger prediction as a claim
- The algorithm proposes and the human confirms, with the output distinguishing the two; derived objects also carry a `review_status` (unreviewed / timed out / user-reviewed) orthogonal to who confirmed them
- Backfilled observations always count toward day counts regardless of latency, flagged with low confidence rather than excluded
- Capture is event-anchored (open/close, dose-triggered follow-ups) as the primary mechanism, with a next-morning daily checkpoint as a backstop only
- The recall summary is exempt from the mandatory confirmation gate; the prior-authorisation packet is not, including for the primary user
- The prodrome pattern surface — alert or passive display — is deferred entirely, revisited once retrospective phase labelling has run
- The technical specification and functional requirements are scoped to build steps 1–4 only
- No community or social features
- No gamification
- Scope stays migraine-adjacent rather than becoming a general symptom tracker

See `decision-log.md` for the reasoning behind each of these.

## Personal use now vs. commercialisation gates

| Area | While building for one person | Before selling |
|---|---|---|
| FDA device status | Not in the picture — no distribution, no claims | Real, and the operative framework is General Wellness, not the Clinical Decision Support carve-out, which covers only software for healthcare professionals |
| PROMIS and other validated instruments | Free for individual clinical use | Commercial use requires written permission, and electronic administration needs paid HEAP permission — this is the first letter to write |
| HIPAA | Not a covered entity | Triggered by contracting with clinics or pushing into an EHR, which is a one-way door |
| FTC Health Breach Notification Rule | Not applicable | Applies to DTC health apps |
| Washington My Health My Data Act | Not applicable | A product-design obligation with a private right of action, so design consent for it early |
| Assistant data retention | Your own risk to accept | Undocumented whether Anthropic or OpenAI retain health data through MCP calls, so confirm before exposing to others |
| Trademark | Not applicable | HEADSTATE appears unregistered but Headspace Health is a registered family in overlapping classes with active enforcement — get a clearance opinion before brand spend |

## Open questions

Granularity, the recall-format question, prodrome alerts, and the automation-level question are settled — see `decision-log.md`. What remains open, and deliberately so:

- **The MCP query-scope tension.** A remote MCP server exposing query tools makes any paywall on derived output architecturally unenforceable, and gating at the query-scope layer would reintroduce the history-depth gating that was already ruled out. Does not bind for a personal build.
- **What, if anything, becomes a paid product.** Two candidates — maintained payer-policy matching, and a clinician-cosignature workflow — point toward a clinic or business-to-business direction rather than a consumer SKU.
- **Whether the clinical path is a real ambition.** Contracting with clinics or pushing into an EHR triggers business-associate status — a one-way door.
- **Instrument licensing.** The daily checkpoint's question wording is a licensing surface; any use of PROMIS, HIT-6, MIDAS, MSQ or MPFID item wording needs legal clearance before distribution or sale.

## Status and provenance

This project is at the strategy-and-design stage with no source code yet. The research base comprises: a competitive landscape sweep, an assistant-integration feasibility study, a clinical and regulatory review, a product-architecture design pass, an independent adversarial QA pass that checked 31 load-bearing claims and required 6 corrections, and a focused verification pass that refuted three claims about the novelty of the continuous-timeline model.

Findings throughout are marked verified, inferred, or unverifiable, and several items are explicitly unverifiable rather than absent. The ICHD-3 quotations were retrieved electronically and should be checked against the printed classification (*Cephalalgia* 2018;38(1):1–211) before clinical or regulatory use. The 2026 Delphi paper has a published Correction that has not yet been reviewed.

### What expires

The following facts have shelf lives and should be re-checked before being relied on, rather than assumed current:

- The MCP specification revision (2026-07-28) — assistant-integration guidance is pinned to this revision.
- The CMS RTM/CPT codes effective 1 January 2026.
- ICHD-4, in active development, with MOH's survival as a diagnosis explicitly under debate.
- The ChatGPT Health rollout, and whether any documented third-party submission path into it emerges — none exists as of the last check.
- The 2026 Delphi paper's published Correction, which has not yet been retrieved or reviewed.

The research files themselves are dated by month only, so exact currency within a month cannot be assumed from the date line alone.

### Next actions

The project's working to-do list currently lives inside `Research/qa-verification-report.md`, in a section that describes itself as an audit trail rather than a to-do list. Surfaced here:

1. Send a HEAP inquiry to HealthMeasures before any commercial use of PROMIS.
2. Verify the ICHD-3 quotations against the printed classification (*Cephalalgia* 2018;38(1):1–211) and retrieve the Delphi paper's Correction.
3. Obtain a trademark clearance opinion before brand spend, given the Headspace Health registered family in overlapping classes.
4. Design the Washington My Health My Data Act consent model into v1 rather than retrofitting it later.
5. Confirm with Anthropic and OpenAI whether health data passed through MCP tool calls is retained, before exposing the connector to anyone else.
6. Write the technical specification, scoped to build steps 1–4.
7. Write the functional requirements as testable acceptance criteria, scoped to the same steps, once the technical specification exists.
