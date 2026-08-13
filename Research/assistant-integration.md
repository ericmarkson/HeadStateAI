# Research — Assistant Integration & Platform Feasibility

**Researched August 2026.** Covers making HeadState reachable *from* AI assistants (Claude, ChatGPT, Gemini, Siri) rather than building voice into the app, plus PWA vs native constraints.
Claims independently verified against `qa-verification-report.md` are marked **[QA]**.

---

## Verdict

Because the assistant does the listening and the language understanding, HeadState only needs to expose an **API and a tool surface**. That dissolves most device-API blockers — **with one important exception**: the documented route into ChatGPT Health runs through Apple HealthKit, which is native-only. On that specific path, a PWA creates the blocker rather than avoiding it.

**One backend, one MCP server, many front doors.**

| Surface | Buildable as PWA? | Realistic ceiling |
|---|---|---|
| Web logging in the PWA | Yes | Universal fallback and system of record. Must be first-class — voice is unusable on transit, in shared offices, around family |
| Remote MCP server (Claude, ChatGPT, Gemini Spark) | Yes — days of work | Full multi-turn conversation **and** history queries. Strongest surface |
| Claude custom connector by URL | Yes — no review, no gatekeeper | Immediate power-user beta |
| iOS Shortcuts + Siri | Yes | Good single-turn hands-free capture |
| Android voice → webhook | Only via Tasker + AutoVoice (paid, two extra apps) | Poor |
| "Hey Siri, HeadState, log a migraine" by app name | **No** | Requires native App Intents |
| Android AppFunctions | **No** | Android 16 platform feature, Google's on-device MCP counterpart, broadening in Android 17. Native only **[QA]** |
| Reading Apple Health / Health Connect | **No** | Native only, no exceptions |

---

## 1. Model Context Protocol — the primary integration surface

MCP is the de facto standard for "assistant calls your API." As of the **2026-07-28 spec revision**: remote servers over **streamable HTTP** (SSE deprecated/legacy), authorisation via **OAuth 2.1 + PKCE + Resource Indicators (RFC 8707)**. Standard, well-understood, not proprietary per assistant.

### Claude
- **Any user can add any remote MCP server by URL** via Settings → Connectors → "Add custom connector." **No review, no approval.**
- **[QA] Caveats:** Free tier is capped at **one** custom connector; Team/Enterprise additions are **Owner-only**. The realistic soft-launch audience is paid individual users.
- **Connectors Directory** for discoverability: requires OAuth 2.0 with a fixed callback URL, privacy policy, setup docs, support contact, a populated test account, correctly annotated tools (`readOnlyHint` / `destructiveHint`), and passing functional + policy review. **Submission tooling is gated to Team/Enterprise workspace admins.**
- Anthropic cites "over 300 third-party connectors… used by millions of people every day." No per-connector adoption or organic-discovery data disclosed.
- Simpler alternative: **request-header authentication** (bearer token), in beta. Fine for a personal connector, weak for multi-tenant consumer use — no per-user consent screen or revocation UX.
- **Unconfirmed, single source:** Anthropic's HIPAA-compliant Enterprise tier reportedly blocks all custom remote-MCP connectors with no admin unblock UI (GitHub issue anthropics/claude-ai-mcp#339). Consumer Claude unaffected.

### ChatGPT
- Third-party app submissions to the **App Directory** (Apps SDK, MCP-based) reopened **17 December 2025**. Requires verified individual/organisation identity, accessible support contact, strict data minimisation, compliant privacy policy.
- **[QA] The general Apps SDK guidelines classify PHI as Restricted Data:** *"Do not collect, solicit, or process… Protected health information."* This is a prohibition, not a soft discouragement.
- **ChatGPT Health** launched **7 January 2026**, US-wide **23 July 2026**, 18+, web and iOS. A dedicated isolated space for connecting personal health data, with additional security review specific to Health inclusion, minimum-necessary data rules and instant revocability.
- Named launch connectors: **Apple Health, Function, MyFitnessPal, Weight Watchers, AllTrails, Instacart, Peloton, b.well, Epic, Oracle Health, One Medical.**
- **[QA] No documented third-party submission path into Health exists.** Those launch connectors are bespoke brand partnerships, not evidence of an open pathway for third-party developers.
- **[QA] The documented route for a small developer runs through Apple Health.** OpenAI has stated that if an app integrates with Apple Health, ChatGPT can use that data with permission. HealthKit requires native. Note **Migra already has bidirectional Apple Health sync** and is therefore already inside that pipe.

### Gemini
- Distinguish developer tooling from consumer surfaces. Google Cloud / Gemini Enterprise has official MCP support for Google services — **developer/enterprise tooling, not the consumer app**.
- **Gemini Spark** gained third-party app support including a custom-MCP entry point **30 June 2026**, but is **gated to Google AI Ultra subscribers**. Least mature and least verified surface. Low priority.
- **[QA] The real Android surface is AppFunctions** — an Android 16 platform feature plus Jetpack library, positioned as the on-device counterpart to MCP, broadening in Android 17. **Requires a native Android app.**
- Historical: Conversational Actions (the old Assistant SDK) were **deprecated June 2023**. App Actions / `shortcuts.xml` / Built-in Intents remain documented but are native-only — a PWA has no `AndroidManifest.xml`, no Activities, and cannot register a `capability` element.
- The Google Assistant → Gemini migration on Android is in progress through 2026 (slipped from 2025). How App Actions carries forward under Gemini is **not clearly documented**. Re-check before implementation.

### Others
Microsoft Copilot and Perplexity are known to be adding MCP/plugin support but were **not verified** and have no health-specific precedent. Lower priority.

### Effort matrix
| Path | Effort | Requires |
|---|---|---|
| Claude custom connector, manual add | Days | Remote MCP server, OAuth 2.1 |
| Claude Connectors Directory | Medium | + privacy policy, support contact, test account, review, likely a Team/Enterprise workspace |
| ChatGPT App Directory (general) | Medium | Identity verification, Apps SDK build, OpenAI review |
| ChatGPT Health inclusion | Medium–High | + Health-specific security review; likely needs demonstrated legitimacy. **No documented path** |
| Gemini Spark custom MCP | Low technically | Same server; reach is subscriber-gated and immature |

---

## 2. iOS Shortcuts + Siri — better than expected

**Key finding:** Shortcuts' **"Ask for Input" action automatically switches from a typed prompt to spoken dictation when the shortcut is invoked via Siri.** No special code.

**Limitation:** Shortcuts cannot natively decompose one spoken sentence into multiple parameters. "Add apples to Grocery" won't split into two parameters; each "Ask for Input" captures one value.

**Why that doesn't matter here:** the Shortcut captures the **entire utterance as one raw string** and POSTs it with a stored auth token. The backend does LLM extraction — the same code path the MCP tool uses.

```
"Hey Siri, Log Migraine"
  → Siri: "What's going on?"
  → user speaks freely
  → Shortcut POSTs { transcript, token } to /api/log
  → backend extracts → spoken confirmation or quiet notification
```

**Ceiling:** genuinely good single-turn hands-free capture. **Not** multi-turn dialogue — chaining Ask-for-Input steps degrades the UX fast. Requires the user to install a shortcut you author, distributed via iCloud link during onboarding. **Never discoverable on its own.**

**Android has no clean equivalent.** Assistant Routines support custom trigger phrases but only chain built-in actions — not "POST dictated text to a URL." **Tasker + AutoVoice** achieves the same pattern but requires installing and configuring two additional third-party apps, one paid. Materially worse funnel than iOS.

---

## 3. Auth and identity

| Pattern | Fit |
|---|---|
| **OAuth 2.1 + PKCE** | MCP's native/recommended pattern. Assistant handles the dance; user consents once; scoped bearer token stored. Required anyway for Claude Directory and ChatGPT Apps SDK. **Recommended** |
| Fixed API key / bearer header | Claude supports it. Fine for a personal connector; no per-user consent or revocation UX |
| Magic-link / device-pairing token | Simplest for the Shortcuts path — generate a scoped long-lived token embedded in a personalised Shortcut install link. Needs rotation and revocation UI |

**Build one OAuth 2.1 + PKCE authorisation server and reuse those tokens for the Shortcuts POST path.** Do not maintain two auth systems.

### The unresolved risk
**Whether Anthropic or OpenAI log or retain health data passed through MCP tool calls is not documented publicly.** Both emphasise data minimisation and user-controlled revocation; neither publishes retention terms for tool-call payloads (abuse monitoring vs. training). **This requires direct confirmation from each platform's privacy team before shipping to anyone but yourself.**

**[QA] This is a gating condition, not a checkbox.** OpenAI classes PHI as Restricted Data; Anthropic's docs warn custom connectors reach "arbitrary services that have not been verified." Routing health data through a third-party assistant while telling users it is private is the FTC Act §5 deceptive-privacy fact pattern — and under Washington's My Health My Data Act it is a **sharing** event requiring separate written authorisation, with a private right of action. **The connector needs its own consent flow and disclosure.**

---

## 4. In-app voice on the web — if ever needed

- **Web Speech API** is reliable in Chrome/Android. On iOS Safari it is flaky — open WebKit bugs around `interimResults`, and critically: *"This feature works in Safari on iOS but not (yet) for installed web apps"* — i.e. it can fail in standalone (Add-to-Home-Screen) mode, exactly the mode a PWA health app runs in. **Do not rely on it in production.**
- **Streaming STT costs (approximate, Aug 2026):** Deepgram ~$0.0077/min · AssemblyAI ~$0.006/min async · Whisper API ~$0.006/min batch · **Gemini Live API** ~$0.005/min audio in, ~$0.018/min out — and it does the LLM parsing in the same call, which is architecturally attractive. At 15–30 s per entry, cost per log is fractions of a cent. **Cost is a non-issue; latency and reliability are the constraints.**
- **Background capture is impossible in a PWA.** Service workers cannot access the microphone; `getUserMedia` requires a foregrounded document. No equivalent of iOS `AVAudioSession` background modes or Android foreground services. Background Sync can queue a *previously recorded* upload but cannot initiate recording. **"Phone locked on the nightstand, whisper, app listens" needs native.**

---

## 5. PWA viability for a health app

| Capability | Status |
|---|---|
| iOS push | Works for Home-Screen-installed PWAs since iOS 16.4. No background data-only pushes. Reach reported ~10–15× smaller than native due to the manual install funnel |
| Storage | Safari uses LRU eviction under pressure. Persistent Storage API can protect data **but requires notification permission first** — a chained dependency. **Server-side sync is mandatory, not optional** |
| **Apple HealthKit** | **Definitively no.** Native-only framework, no web API |
| **Android Health Connect** | **Also no.** Jetpack/Android SDK consumed via Kotlin/Java inside an installed app. No browser-exposed JS API |
| iOS 26 | Made Home-Screen sites default to opening as web apps — display mode only, not intent exposure |

**What a PWA gives up for this use case:** OS-assistant invocation, HealthKit/Health Connect, reliable background audio, push parity, guaranteed local storage, App Store/TestFlight distribution and its trust signal, Watch complications, altimeter pressure sensing.

**Pragmatic hybrid:** PWA as the core codebase, **Capacitor** wrapper when native capability is needed. Capacitor gives a real native bundle ID (needed for App Intents / AppFunctions), plugin access to HealthKit and Health Connect, background audio session control, native push — while keeping the web codebase for UI. **PWABuilder / Trusted Web Activity** is Android-only and does not solve iOS at all. A React Native rewrite is only warranted if voice/health-OS integration becomes the entire product.

**For personal use, no App Store review is needed to run a Capacitor build on your own device.** Go native early and privately.

**Interim hack:** a Shortcuts automation dumping Apple Health data to the API on a schedule. Unpolished, no wrapper, unblocks correlation work today.

---

## 6. Is voice-first symptom capture novel?

No. **Migraine Trail** already ships it as a native iOS app — marketing copy explicitly targets photophobic dark-room use, hardware-button or wake-word activation, soft audio confirmation, "never opened your eyes." Consistent with the finding that true hands-free background capture requires native.

Research context: *SymptomAI: Toward a Conversational AI Agent for Everyday Symptom Assessment* (arXiv 2605.04012) and *Conversational AI for Automated Patient Questionnaire Completion* (arXiv 2602.19507) — a recognised, funded direction, not fringe, and not first-mover territory.

**Health-tracking MCP precedent exists**, mostly hobbyist: multiple open-source Fitbit MCP servers, a Google Health API (Fitbit + Pixel Watch) MCP server, and the Open Wearables project. At platform level, Apple Health, MyFitnessPal, Function, Weight Watchers, AllTrails and Peloton are live inside ChatGPT Health — **the "assistant reads/writes my health app" pattern is commercially validated by major brands.**

**Verified separately:** no migraine or headache app currently ships an MCP server or assistant connector. See `continuous-timeline-verification.md`.

---

## 7. Strategic risk

**Dependency.** Claude Connectors, ChatGPT Apps/Health and Gemini Spark are all 2025–2026 surfaces with shifting terms, review criteria and even product names. Google killed Conversational Actions in 2023. ChatGPT Health could build native symptom tracking and disintermediate a third-party tracker entirely. **Treat as infrastructure that may shift within a year.**

**Discovery is not a channel.** ChatGPT's App Directory "does not yet support long-tail keyword queries effectively" and favours recognised brands (Spotify, Adobe, Booking.com). No public discovery data exists for Claude's directory. **Assume near-zero organic discovery.** Build connectors as a retention and press asset for users acquired elsewhere — *"the first migraine tracker you can talk to Claude about"* — not as growth.

---

## 8. Design constraints specific to migraine

Migraine patients mid-attack may be photophobic, phonophobic, nauseated, cognitively impaired, occasionally aphasic. Guidance converges on:

- **No bright screens, no flashing or auto-playing motion.** Avoid saturated red and high contrast. Respect dark mode; let the screen stay off during capture. Honour `prefers-reduced-motion`.
- **Radically reduce cognitive load.** Simple language, no idioms, minimal decision points.
- **Minimise turns.** Aim for one utterance → structured log. If clarification is needed, batch it or defer it to when the user is well. Never five sequential slot-filling questions at a nauseated, light-sensitive user.
- **Non-visual confirmation.** Soft chime or haptic, not a screen flash.
- **User control over audio**, and a single lock-screen affordance rather than app navigation.

---

## Sources

MCP & platforms: [MCP authorization spec](https://modelcontextprotocol.io/specification/2025-11-25/basic/authorization) · [MCP 2026-07-28 release](https://blog.modelcontextprotocol.io/posts/2026-07-28/) · [Claude remote MCP connectors](https://claude.com/docs/connectors/custom/remote-mcp) · [Claude custom connectors support](https://support.claude.com/en/articles/11175166-get-started-with-custom-connectors-using-remote-mcp) · [Claude directory submission](https://sunpeak.ai/blogs/claude-connector-directory-submission/) · [Claude connector observability](https://claude.com/blog/observability-for-developers-building-connectors) · [HIPAA Enterprise connector block (unconfirmed)](https://github.com/anthropics/claude-ai-mcp/issues/339) · [OpenAI app submissions](https://openai.com/index/developers-can-now-submit-apps-to-chatgpt/) · [Apps SDK submission guidelines](https://developers.openai.com/apps-sdk/app-submission-guidelines) · [Introducing ChatGPT Health](https://openai.com/index/introducing-chatgpt-health/) · [Fierce on ChatGPT Health rollout](https://www.fiercehealthcare.com/ai-and-machine-learning/openai-makes-health-chatgpt-widely-available-moving-deeper-consumer-health) · [Anthropic healthcare analysis](https://healthapiguy.substack.com/p/another-one-anthropics-healthcare) · [Google Cloud MCP support](https://cloud.google.com/blog/products/ai-machine-learning/announcing-official-mcp-support-for-google-services) · [Gemini Spark 3rd-party MCP](https://9to5google.com/2026/06/30/gemini-spark-apps-more/) · [Android AppFunctions](https://9to5google.com/2026/02/25/android-appfunctions-gemini/) · [Android built-in intents](https://developer.android.com/develop/devices/assistant/intents) · [Conversational Actions sunset](https://developers.google.com/assistant/ca-sunset/best-practices) · [Assistant → Gemini 2026](https://9to5google.com/2025/12/19/google-assistant-gemini-2026/) · [ChatGPT directory discoverability](https://phiture.com/asostack/chat-gpt-app-directory/)

Apple/Siri: [App Intents explainer](https://blakecrosley.com/blog/app-intents-are-apples-new-api-to-your-app) · [WWDC26 Siri App Schemas](https://developer.apple.com/videos/play/wwdc2026/240/) · [App Intents & Shortcuts iOS 2026](https://sharpskill.dev/en/blog/ios/app-intents-siri-shortcuts-advanced-ios-automation) · [Shortcuts voice input](https://macmost.com/creating-shortcuts-that-accept-voice-input.html) · [HealthKit headache identifier](https://developer.apple.com/documentation/healthkit/hkcategorytypeidentifier/headache) · [Assistant Routines](https://support.google.com/assistant/answer/7672035) · [Tasker + AutoVoice](https://sisyphus-industries.com/community/other-development/tasker-autovoice-google-assistant-control-routines/)

PWA & voice: [PWAs on iOS 2026](https://www.mobiloud.com/blog/progressive-web-apps-ios) · [Web Speech in PWAs](https://whatpwacando.today/speech-recognition/) · [Safari 17.1 speech issues](https://discussions.apple.com/thread/255492924) · [STT API comparison 2026](https://deepgram.com/learn/best-speech-to-text-apis-2026) · [Gemini API pricing](https://ai.google.dev/gemini-api/docs/pricing) · [HealthKit data access limits](https://www.themomentum.ai/blog/what-you-can-and-cant-do-with-apple-healthkit-data) · [Health Connect](https://developer.android.com/health-and-fitness/health-connect) · [TWA vs Capacitor](https://saastostore.com/blog/twa-vs-capacitor)

Precedent & accessibility: [Migraine Trail voice logging](https://migrainetrail.com/features/voice-logging) · [Migraine Trail blog](https://migrainetrail.com/blog/voice-logging-track-migraines) · [SymptomAI](https://arxiv.org/abs/2605.04012) · [Conversational AI questionnaires](https://arxiv.org/html/2602.19507) · [Fitbit MCP server](https://github.com/hiro-777-sky/fitbit_mcp) · [Google Health MCP](https://github.com/davidmosiah/google-health-mcp) · [Open Wearables](https://www.themomentum.ai/blog/introducing-open-wearables-the-open-source-api-for-wearable-health-intelligence) · [Migraine content design accessibility](https://dkconsultingcolorado.com/2024/01/30/neurodivergence-and-content-design-the-migraine-edition/)
