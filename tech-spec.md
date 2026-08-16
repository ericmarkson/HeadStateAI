# HeadState — Technical Specification

**Scope:** build steps 0–4. Repository scaffold, capture and the observation store, recall, derived episodes and day classification, and integrations.
**Out of scope:** phase annotation and retrospective relabelling (step 5), correlation (step 6), the prior-authorisation packet (step 7).

**Binding inputs.** `decision-log.md` governs. `CLAUDE.md` carries the invariants. Research files are the evidence behind decisions.

**Marker convention.** **[V]** verified — backed by an executed test recorded here, or by a citation in the named `Research/` file, which carries the URL. A `[V]` with neither is a defect. **[I]** inference. **[U]** unverified, confirm before building. **[H]** house decision where no published rule exists, recorded in the decision log. Version numbers carry an as-of date rather than a `[V]`; they go stale.

**All SQL in this document — including §4's consent, audit and export-link tables — was executed against PostgreSQL 16.13.** Statements that did not run have been corrected.

---

## 1. Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  Capacitor shell (iOS · Android)                            │
│  ┌────────────────────────────────────────────────────┐     │
│  │  Web app — React + Vite (also deployed to browser  │     │
│  │  as a degraded, connectivity-assumed peer)         │     │
│  └────────────────────────────────────────────────────┘     │
│  SQLite (SQLCipher) — system of record on device            │
│  Native plugins: health · notifications · print · share     │
└───────────────────────────┬─────────────────────────────────┘
                            │ sync (append-only, watermark)
┌───────────────────────────▼─────────────────────────────────┐
│  Node / TypeScript API                                      │
│  ├─ /api/*   ├─ /mcp   ├─ /ingest/transcript   ├─ /s/:token │
│  @headstate/derivation — one package, two call sites        │
└───────────────────────────┬─────────────────────────────────┘
                   ┌────────▼────────┐
                   │  Postgres       │
                   └─────────────────┘
```

The derivation engine is **one package imported by both server and client**, with golden-fixture tests asserting byte-identical output. Two implementations of clinical logic that must agree is the only irreducible complexity here, and it is managed by not having two.

---

## 2. Stack

| Layer | Choice | As of |
|---|---|---|
| App | **React + Vite** (D14) | — |
| Native shell | **Capacitor 8** (`@capacitor/core` 8.5.0) | 31 Jul 2026 |
| | Requires Xcode ≥26.0, iOS target ≥15.0, Android Studio Otter 2025.2.1, minSdk 24, Node ≥22 **[V]** | |
| | iOS defaults to **Swift Package Manager** since Capacitor 8; opt out with `--packagemanager CocoaPods` **[V]** | |
| Server | Node + TypeScript | — |
| Database | **Fly.io managed Postgres 16** (D14) — matches the 16.13 the DDL was executed against | — |
| Device database | `@capacitor-community/sqlite` **8.1.1** | Aug 2026 |
| Identity | Delegated authorisation server — §8.2 | — |

Repository layout unchanged from the prior revision: `apps/{app,api}`, `packages/{derivation,clinical-config,schema}`, committed `ios/` and `android/`, `docs/` for ADRs.

Bundle identifier is **`health.headstate.app`**, locked (D14).

**Never commit `server.url`** — a shipped build would load a dev machine's IP. **[V]**

---

## 3. Data model

### 3.1 Observation layer

**Prerequisites.** The DDL below references five objects it does not define. They must exist first, and FR-0.7's CI gate must create them before running the schema:

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;
CREATE ROLE headstate_app;
CREATE ROLE headstate_ops;
CREATE TABLE users (id uuid PRIMARY KEY /* … */);
CREATE TABLE clinical_config (
  hash text PRIMARY KEY, version text NOT NULL, body jsonb NOT NULL,
  citations jsonb NOT NULL, created_at timestamptz NOT NULL DEFAULT now()
);
CREATE TABLE consent_policy_versions (
  id uuid PRIMARY KEY, scope text NOT NULL, version int NOT NULL,
  effective_at timestamptz NOT NULL, body_markdown text NOT NULL,
  body_sha256 text NOT NULL, UNIQUE (scope, version)
);
```


`observed_at` and `observed_date` are **nullable** — an unresolvable time is stored as null rather than guessed (§6.5). A null-time observation is excluded from day classification and surfaced for follow-up.

```sql
CREATE TABLE observations (
  id            uuid PRIMARY KEY,            -- UUIDv7, client-generated
  seq           bigint GENERATED ALWAYS AS IDENTITY,
  user_id       uuid NOT NULL REFERENCES users(id),
  observed_at   timestamptz,                 -- nullable
  observed_tz   text,
  observed_date date,                        -- local calendar date, frozen at write
  recorded_at   timestamptz NOT NULL,
  recorded_tz   text NOT NULL,
  kind          text NOT NULL,
  body          jsonb NOT NULL,
  drug_class    text GENERATED ALWAYS AS (body->>'drug_class') STORED,
  severity      numeric GENERATED ALWAYS AS
                  (CASE WHEN jsonb_typeof(body->'severity')='number'
                        THEN (body->>'severity')::numeric END) STORED,
  source        text NOT NULL,
  provenance    jsonb,                       -- shape in §3.6
  confidence    real,
  created_at    timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT observed_pair CHECK ((observed_at IS NULL) = (observed_date IS NULL)),
  CONSTRAINT severity_typed CHECK (jsonb_typeof(body->'severity') IN ('number','null')
                                   OR body->'severity' IS NULL)
);
```

The `severity` guard matters: without it a non-numeric JSONB value raises a **database** error on insert rather than a validation error the API can return. Executed and confirmed.

**Supersession is a table, not a column.** The prior revision specified both, which is a contradiction — the immutability trigger makes a `superseded_by` column unsettable by the application role. The column is removed.

```sql
CREATE TABLE supersessions (
  superseded_id uuid PRIMARY KEY REFERENCES observations(id),
  superseded_by uuid NOT NULL REFERENCES observations(id),
  reason text, created_at timestamptz NOT NULL DEFAULT now(),
  CHECK (superseded_id <> superseded_by)
);

CREATE VIEW observations_live AS
  SELECT o.* FROM observations o
  LEFT JOIN supersessions s ON s.superseded_id = o.id
  WHERE s.superseded_id IS NULL;
```

Derivation reads `observations_live`, never `observations` directly.

**Indexes.** Partial indexes on `superseded_by IS NULL` are gone with the column. These serve the anti-join:

```sql
CREATE UNIQUE INDEX ON observations (seq);
CREATE INDEX obs_user_time ON observations (user_id, observed_at DESC);
CREATE INDEX obs_user_date ON observations (user_id, observed_date);
CREATE INDEX obs_med       ON observations (user_id, drug_class, observed_date) WHERE kind='medication';
CREATE INDEX obs_kind      ON observations (user_id, kind, observed_at DESC);
```

**Immutability, with the ops escape hatch that legal erasure requires.**

```sql
CREATE OR REPLACE FUNCTION reject_observation_mutation() RETURNS trigger AS $$
BEGIN
  IF pg_has_role(current_user, 'headstate_ops', 'MEMBER') THEN RETURN COALESCE(NEW, OLD); END IF;
  RAISE EXCEPTION 'observations are append-only (% attempted by %)', TG_OP, current_user;
END; $$ LANGUAGE plpgsql;

CREATE TRIGGER observations_immutable BEFORE UPDATE OR DELETE ON observations
  FOR EACH ROW EXECUTE FUNCTION reject_observation_mutation();

GRANT SELECT, INSERT                 ON observations TO headstate_app;
GRANT SELECT, INSERT, UPDATE, DELETE ON observations TO headstate_ops;
```

Executed: the app role is denied at the grant, with the trigger as backstop; the ops role can mutate, which §11.6's crypto-shred-then-tombstone path requires. Without this the erasure path is unimplementable.

### 3.2 Derived layer

**A `derivations` table is the parent of everything derived.** This solves three problems the prior revision had: it makes every count scoped by construction, it lets competing readings coexist without a fixed three-value enum in a primary key, and it puts `algorithm_version` in the identity so a report is reproducible.

```sql
CREATE TABLE derivations (
  id                uuid PRIMARY KEY,
  user_id           uuid NOT NULL REFERENCES users(id),
  variant           text NOT NULL,   -- see value set below
  algorithm_version text NOT NULL,
  config_hash       text NOT NULL REFERENCES clinical_config(hash),
  computed_at       timestamptz NOT NULL,
  is_current        boolean NOT NULL DEFAULT true,
  UNIQUE (user_id, variant, algorithm_version, config_hash)
);
CREATE INDEX ON derivations (user_id, is_current) WHERE is_current;
```

**`variant` value set.** `primary` is the reading with no unresolved ambiguity — it always exists. Additional variants are emitted **only** where a rule is genuinely ambiguous: `relapse_collapsed` and `relapse_split` for sub-48-hour recurrence (§5.3), and `status_migrainosus` where that classification competes with a distinct-attack reading. The set is extensible: a variant is a label for a governing boundary decision, not a fixed enum. If two ambiguities touch one date, the variant key is the ordered composition of both decisions.

```sql
CREATE TABLE episodes (
  id            uuid PRIMARY KEY,
  derivation_id uuid NOT NULL REFERENCES derivations(id) ON DELETE CASCADE,
  user_id       uuid NOT NULL REFERENCES users(id),
  natural_key   text NOT NULL,        -- stable across recompute; see below
  started_at    timestamptz NOT NULL,
  ended_at      timestamptz,
  open          boolean NOT NULL,
  boundary_rule text NOT NULL,
  computed_from uuid[] NOT NULL,
  confirmed_by  text NOT NULL DEFAULT 'system',
  review_status text NOT NULL DEFAULT 'unreviewed',
  UNIQUE (derivation_id, natural_key)
);
CREATE INDEX ON episodes USING gist (user_id, tstzrange(started_at, ended_at));
CREATE INDEX ON episodes (derivation_id, open);
```

**No exclusion constraint.** Competing readings must overlap. The GiST index exists for overlap *querying* only.

**`natural_key` is the id of the earliest member observation, resolved through any supersession chain to its original id.** An override keyed to a start timestamp would be orphaned when a recompute moves that start. Keying to the first observation survives mutation because observations are immutable — but it must also survive *supersession*, so the resolution walks `supersessions` back to the original id rather than taking the current live row's id. Without that walk, correcting the first observation of an episode silently orphans every override on it.

Recompute replaces by `(derivation_id, natural_key)`. Because `day_classifications.attributed_episode` references `episodes(id)`, that reference is declared `ON DELETE SET NULL` — without it the replace fails with a foreign-key violation, verified.

```sql
CREATE TABLE day_classifications (
  derivation_id      uuid NOT NULL REFERENCES derivations(id) ON DELETE CASCADE,
  user_id            uuid NOT NULL REFERENCES users(id),
  local_date         date NOT NULL,
  class              text NOT NULL CHECK (class IN ('none','headache','migraine','aura_only')),
  rule               text,                -- nullable: an ordinary 'none' day has no governing rule
  attenuation_flag   boolean NOT NULL DEFAULT false,
  active_pharmacology text[],
  attributed_episode uuid REFERENCES episodes(id) ON DELETE SET NULL,
  computed_from      uuid[] NOT NULL,
  confirmed_by       text NOT NULL DEFAULT 'system',
  review_status      text NOT NULL DEFAULT 'unreviewed',
  PRIMARY KEY (derivation_id, local_date)
);
```

### 3.3 Day counts — always scoped, always labelled

The prior revision's expressions were both invalid SQL and unscoped. Executed against two derivations holding one calendar date, the unscoped form returned **2 headache days for one day**; scoped it returns 1. Under §5.3 every sub-48-hour recurrence emits extra variants, so an unscoped count inflates exactly for the patient this product exists for.

```sql
-- ALWAYS parameterised by a single derivation.
SELECT
  count(*) FILTER (WHERE class IN ('headache','migraine'))  AS headache_days,
  count(*) FILTER (WHERE class IN ('migraine','aura_only')) AS migraine_days
FROM day_classifications
WHERE derivation_id = $1 AND local_date BETWEEN $2 AND $3;
```

An `aura_only` day counts as a migraine day and **not** as a headache day — Delphi rule 3 plus ICHD-3's headache-day definition ("any day affected by headache for any part of the day").

**Two different headache-day counts exist and must never be conflated.**

| Count | Definition | Used for |
|---|---|---|
| **Delphi headache/migraine days** | The classification above | Internal analysis, the recall summary, trend display |
| **Payer headache days** | The payer's own criterion. Cigna's Botox rule is *"≥15 headache days per month with headache lasting **4 hours per day or longer**"* | Prior-authorisation documentation (step 7) |

Delphi explicitly disclaims reimbursement contexts. **No API, tool or export may emit an unlabelled `headache_days`** — every count carries the rule that produced it.

### 3.4 Overrides

```sql
CREATE TABLE overrides (
  id uuid PRIMARY KEY,
  user_id uuid NOT NULL REFERENCES users(id),
  overrides_type text NOT NULL,      -- episode_boundary | phase | day_classification
  overrides_key jsonb NOT NULL,      -- { local_date } or { episode_natural_key }
  overriding_value jsonb NOT NULL,
  set_at timestamptz NOT NULL DEFAULT now(),
  revoked_at timestamptz
);
CREATE INDEX ON overrides (user_id, overrides_type) WHERE revoked_at IS NULL;
```

An episode override keys to `{ episode_natural_key }`, which is an immutable observation id — so it survives a recompute that moves the episode's start time.

### 3.5 Clinical configuration

File is the source of truth; `clinical_config` is the FK anchor, keyed by content hash. A CI lint asserts every constant carries a citation.

| Constant | Value | Source |
|---|---|---|
| `STATUS_MIGRAINOSUS_MIN_DURATION` | >72 h unremitting, debilitating, typical of prior attacks | ICHD-3 1.4.1 |
| `STATUS_MIGRAINOSUS_MAX_REMISSION` | 12 h — **applies only within a status migrainosus assessment**, not as a general episode rule | ICHD-3 1.4.1 |
| `DISTINCT_ATTACK_SEPARATION` | 48 h headache-free | ICHD-3 Definition of Terms |
| `RECURRENCE_AMBIGUOUS_WINDOW` | <48 h | ICHD-3 Definition of Terms |
| `DELPHI_TREATED_DAY_MIN` | 30 min, regardless of drug class or characteristics | Delphi 2026, 94% |
| `DELPHI_UNTREATED_DAY` | ≥4 h **AND** (≥2 of 4 characteristics **OR** (≥1 characteristic **AND** ≥1 associated symptom)) | Delphi 2026, 83% |
| `DELPHI_AURA_DAY` | aura counts regardless of headache | Delphi 2026, 83% |
| `DURATION_QUALIFICATION` | Evaluated **once per attack**, against total duration measured **across midnight**; an attack continuing into sleep is reckoned until awakening | ICHD-3 1.1 for the measurement; **[H]** D11 for the per-attack scope |
| `DAY_OCCUPANCY` | **Every local calendar date the attack overlaps takes the attack's qualification class.** One exception, Delphi rule 4: where the patient slept through and woke headache-free, that attack's occupancy of the waking date is suppressed | ICHD-3 headache-day definition; exception Delphi 2026 rule 4; **[H]** D11 for the attack-to-date mapping |
| `DAY_CLASS_PRECEDENCE` | Where two attacks occupy one date: `migraine` > `headache` > `aura_only` > `none` | **[H]** D11 |
| `DRUG_CLASS` | Taxonomy: `triptan \| ergot \| opioid \| combination_analgesic \| non_opioid_analgesic \| corticosteroid` — required on every medication observation; referenced by every MOH constant | ICHD-3 8.2 |
| `ACUTE_MEDICATION_DAY` | a calendar day on which any acute agent was taken, counted per drug class | ICHD-3 8.2 (derived) |
| `MOH_EVALUATION_WINDOW` | **calendar month** | **[H]** D12 — ICHD-3 says "per month" |
| `MOH_GENERAL_CRITERION` | headache ≥15 days/month with a pre-existing headache disorder, >3 months | ICHD-3 8.2 |
| `MOH_PER_CLASS` | triptan / ergot / opioid / combination_analgesic → 10 days/month; non_opioid_analgesic → 15 | ICHD-3 8.2 |
| `MOH_COMBINED_CLASS` | **distinct calendar days** on which ergotamine, triptans, non-opioid analgesics **or** opioids were taken ≥10/month, no single class overused. Excludes combination analgesics (ICHD-3 handles those at 8.2.5) | ICHD-3 8.2.6 |
| `PAYER_CGRP_EPISODIC` | ≥4 migraine days/month + two failed preventive classes | Payer policy |
| `PAYER_CHRONIC_BOTOX` | ≥15 headache days/month **with headache ≥4 h/day**, ≥8 migraine days, ≥3 failed prophylactics from ≥2 classes | Cigna |
| `PAYER_BOTOX_CONTINUATION` | ≥50% reduction in monthly migraine frequency from baseline | Cigna |
| `REVIEW_TIMEOUT` | 14 days | **[H]** product decision |
| `BACKFILL_WINDOW` | 6 hours — binary, not graduated. `recorded_at − observed_at > BACKFILL_WINDOW` flags the observation backfilled at write time, with the reduced-confidence treatment §3.6/D5 already specifies | **[H]** D19 — product decision |

There is **no `DELPHI_OVERNIGHT` constant and no `delphi_overnight` rule value.** The overnight behaviour is step 3a of the occupancy rule — an explicit, attack-scoped exception — not a constant and not a classification rule. See §5.2.

**Sourcing caveat.** ICHD-3 passages were retrieved electronically, consistent across two sources, **not** from the printed classification. The 2026 Delphi paper has an unretrieved published Correction (DOI 10.1186/s10194-026-02407-x). Verify both before clinical or regulatory use. **[U]**

### 3.6 Provenance and confidence

Invariant 5 requires per-field provenance. The observation-level `confidence` column is insufficient — a single observation can carry several extracted fields with different match types.

```typescript
// observations.provenance
{
  transcript_id?: string,
  feed_name?: string,
  fields?: {
    [fieldPath: string]: {
      quote: string, start: number, end: number,
      match_type: 'lexical' | 'paraphrase',
      confidence: number
    }
  }
}
```

The observation-level `confidence` is the minimum across `fields`. Both travel into the export.

### 3.7 Body shapes

```typescript
// symptom  — tri-state per invariant 4
{ type: 'nausea'|'vomiting'|'photophobia'|'phonophobia'|'dizziness'|'aura',
  ask_status: 'not_asked'|'asked', value: 'yes'|'no'|'unknown'|null,
  aura_type?: string, aura_duration_min?: number }

// pain
{ laterality: ('left'|'right'|'bilateral')[], region: string[],
  quality?: 'throbbing'|'pressure'|'stabbing'|'other', severity?: number,
  aggravated_by_activity?: boolean, avoids_activity?: boolean,   // ICHD-3 criterion C
  started_at?: string, ended_at?: string }                        // attack extent

// medication
{ name: string, dose?: string, route?: string,
  drug_class: 'triptan'|'ergot'|'opioid'|'combination_analgesic'
            |'non_opioid_analgesic'|'corticosteroid',
  is_acute: boolean,                       // false for a preventive
  course_start?: string, course_end?: string,   // for a taper — drives §5.5
  perceived_relief?: 'none'|'partial'|'complete' }

// medication_response
{ medication_observation_id: string, assessed_at: string,
  effect: 'resolved'|'improved'|'unchanged'|'worsened', residual_severity?: number }

// sleep       { start: string, end: string, duration_min: number,
//               stages?: {deep,light,rem,awake}, feed: string }
// environment { pressure_hpa?: number, temp_c?: number, humidity_pct?: number,
//               location_granularity: 'coarse', feed: string }
// care_event  { type: 'ed_visit'|'office_visit'|'urgent_care'|'admission',
//               facility?: string, started_at: string, ended_at?: string,
//               episode_natural_key?: string }   // links to the episode §2 recall reports on
```

`corticosteroid` is in the taxonomy. Without it the capture model's next-day taper follow-up cannot be written and the attenuation model has nothing to derive from.

### 3.8 Time and timezone

**`timestamptz` plus an explicit IANA zone, with the local date computed in application code and stored as a plain `date`.**

The prior revision justified this by claiming `AT TIME ZONE` with a named zone is rejected in a generated column. **That is false.** Executed on PG 16.13: both `(ts AT TIME ZONE 'America/New_York')::date` and `(ts AT TIME ZONE z)::date` with `z` a column reference are **accepted** — `timezone(text, timestamptz)` is marked IMMUTABLE.

The real justification stands on its own: **tzdata changes.** A generated column is recomputed on read in some paths and would let a future timezone-database update silently reclassify a historical day out from under a report already handed to a clinician. Freezing the date at write makes the classification stable. *This correction also applies to `Research/tech-postgres-temporal-model.md` §8, where the error originated.*

---

## 4. Consent, audit and export links

### 4.1 Consent

Three tables. `consent_policy_versions` holds the versioned text; `consent_events` is the append-only legal record; `consent_current` is the derived projection and the **only** table read at request time.

```sql
CREATE TABLE consent_events (          -- append-only, the legal record
  id                uuid PRIMARY KEY,
  user_id           uuid NOT NULL REFERENCES users(id),
  scope             text NOT NULL,
  policy_version_id uuid NOT NULL REFERENCES consent_policy_versions(id),
  action            text NOT NULL CHECK (action IN ('grant','withdraw')),
  method            text NOT NULL,
  target_type       text,      -- mcp_client | export_link | health_platform
  target_id         text,      -- 'claude' | 'chatgpt' | <link id>
  ip_address        inet,
  user_agent        text,
  device_id         text,
  created_at        timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX ON consent_events (user_id, scope, created_at DESC);
GRANT SELECT, INSERT ON consent_events TO headstate_app;
```

`consent_current`'s primary key must use `NOT NULL DEFAULT ''` — a `COALESCE` expression in a `PRIMARY KEY` clause does not parse:

```sql
CREATE TABLE consent_current (
  user_id uuid NOT NULL REFERENCES users(id),
  scope text NOT NULL,
  target_type text NOT NULL DEFAULT '',
  target_id   text NOT NULL DEFAULT '',
  is_granted boolean NOT NULL,
  granted_at timestamptz, withdrawn_at timestamptz,
  policy_version_id uuid REFERENCES consent_policy_versions(id),
  PRIMARY KEY (user_id, scope, target_type, target_id)
);
```

Scopes: `collect_symptom_data` · `collect_location` · `share_assistant_connector` · `share_export_link` · `share_health_platform` · `research_aggregate` · `sale` (reserved, no UI without counsel).

`consent_current` is rebuilt **in the same transaction** as the `consent_events` insert. A `requireConsent` guard is called **inside every handler** that moves data outward, and fails closed.

### 4.2 Audit log

`request_summary` records the **shape** of a change, never clinical content — "updated 1 medication record, field: dose", not the dose. Two reasons: it stops the audit log becoming a second copy of consumer health data needing its own governance, and it keeps clinical content out of any log pipeline with looser access controls. Because it holds no clinical content, it survives a user deletion as an orphaned pointer rather than as retained health data.

```sql
CREATE TABLE audit_log (
  id              uuid PRIMARY KEY,
  occurred_at     timestamptz NOT NULL DEFAULT now(),
  correlation_id  uuid NOT NULL,
  user_id         uuid NOT NULL,
  actor_type      text NOT NULL CHECK (actor_type IN ('user','mcp_client','system','ios_shortcut')),
  actor_id        text NOT NULL,      -- 'web' | 'claude' | 'chatgpt' | 'gemini' | 'shortcut:<name>'
  auth_method     text NOT NULL CHECK (auth_method IN ('session','oauth_mcp','pat')),
  scopes_used     text[] NOT NULL,
  action          text NOT NULL,
  target_table    text,
  target_ids      uuid[],
  request_summary jsonb,              -- SHAPE, never clinical content
  outcome         text NOT NULL CHECK (outcome IN ('success','denied_consent','error')),
  ip_address      inet,
  client_version  text
);
CREATE INDEX ON audit_log (user_id, occurred_at DESC);
CREATE INDEX ON audit_log (correlation_id);
CREATE INDEX ON audit_log (actor_id, occurred_at DESC);   -- §11 breach detection

GRANT SELECT, INSERT ON audit_log TO headstate_app;       -- append-only by grant
```

### 4.3 Export links

Token: 256 bits from `crypto.randomBytes(32)`, base64url, opaque — not a JWT. **Only the hash is stored.** Default expiry 24 h, user-selectable to 72 h. `max_views` defaults to 3, server-enforced, never unlimited. Optional 4–6 digit PIN delivered out-of-band from the link, checked server-side with attempt limiting.

```sql
CREATE TABLE export_links (
  id         uuid PRIMARY KEY,
  user_id    uuid NOT NULL REFERENCES users(id),
  token_hash text NOT NULL UNIQUE,     -- sha256; plaintext never stored
  pin_hash   text,
  scope      jsonb NOT NULL,
  created_at timestamptz NOT NULL DEFAULT now(),
  expires_at timestamptz NOT NULL,
  max_views  int NOT NULL DEFAULT 3 CHECK (max_views > 0),
  view_count int NOT NULL DEFAULT 0,
  revoked_at timestamptz
);
CREATE INDEX ON export_links (user_id, created_at DESC);

CREATE TABLE export_link_access_log (   -- append-only
  id          uuid PRIMARY KEY,
  link_id     uuid NOT NULL REFERENCES export_links(id),
  accessed_at timestamptz NOT NULL DEFAULT now(),
  ip_address  inet,
  user_agent  text,
  outcome     text NOT NULL CHECK (outcome IN
                ('served','expired','revoked','max_views_exceeded','bad_pin','not_found'))
);
CREATE INDEX ON export_link_access_log (link_id, accessed_at DESC);

GRANT SELECT, INSERT, UPDATE ON export_links TO headstate_app;   -- UPDATE for view_count, revoked_at
GRANT SELECT, INSERT ON export_link_access_log TO headstate_app;
```

**Every access attempt writes a row regardless of outcome.** The FTC HBNR 60-day clock runs from **discovery**, and unauthorised disclosure counts as a breach — without this table you cannot discover that a link leaked, so the clock never starts, which is itself the compliance gap.

The revoke UI must state plainly what revocation does **not** undo: content already rendered, downloaded or screenshotted is not retractable. That is a fact to disclose, not a limitation to fix.

---

## 5. The derivation engine

`packages/derivation` — pure functions, no I/O, imported by server and client.

### 5.1 Signature

```typescript
export function derive(input: {
  observations: Observation[];   // from observations_live, superseded already excluded
  overrides: Override[];
  config: ClinicalConfig;
  timezone: string;
  window: { start: Date; end: Date } | 'full';
}): {
  derivations: Array<{
    variant: string;
    episodes: Episode[];
    dayClassifications: DayClassification[];
    mohAssessments: MohAssessment[];   // §5.6 — scoped to this derivation
  }>;
  configHash: string;
  algorithmVersion: string;
}
```

### 5.2 Day classification

**Attacks are qualified once, then occupy every day they touch.** Qualification and occupancy are separate operations. Collapsing them produces a counting error in one direction or the other, and both directions have been made and corrected here.

**Step 1 — assemble attacks.** Group contiguous pain observations into attacks with a start and an end. **Duration is measured per attack, across midnight where applicable.** Per ICHD-3 1.1, an attack continuing into sleep is reckoned until awakening.

**Step 2 — qualify each attack.** Evaluated once, against the attack's total duration. First match wins:
1. Acute medication taken during the attack **and** duration ≥30 min → **migraine-qualifying**, rule `delphi_treated_30min`.
2. Duration ≥4 h **and** (≥2 of 4 characteristics **or** (≥1 characteristic **and** ≥1 associated symptom)) → **migraine-qualifying**, rule `delphi_untreated_4h`.
3. Otherwise → **headache-qualifying**.

**Step 3 — occupancy.** **Every local calendar date the attack overlaps takes the attack's qualification class.** A migraine-qualifying attack spanning four dates produces four `migraine` days; a headache-qualifying attack spanning four dates produces four `headache` days.

This is ICHD-3's own definition of a headache day — *"any day affected by headache for any part or the whole of the day"* — applied literally. It is also the only reading compatible with the thresholds this product exists to document: ≥15 headache days/month gates both Botox and `MOH_GENERAL_CRITERION`, and a continuous-headache patient must be able to reach it.

**Step 3a — the one exception, Delphi rule 4.** Where the patient fell asleep with the attack ongoing and **woke headache-free**, **that attack's occupancy of the waking date is suppressed.** The suppression is scoped to the attack, not to the date: if a separate attack also occupies that date, the date still counts. This is the sole named exception, at 100% Delphi consensus, and it is why the daily checkpoint asks the overnight question unconditionally — the exception is unobservable without it.

**Step 3b — merge, because `day_classifications` is one row per date.** Under `relapse_split`, and on any date carrying a resolved attack plus a new one, two attacks routinely occupy the same date. The primary key is `(derivation_id, local_date)`, so occupancy must merge rather than collide:
- **`class`** takes the highest-ranking occupancy: `migraine` > `headache` > `aura_only` > `none` (`DAY_CLASS_PRECEDENCE`).
- **`rule`** is taken from the attack that produced the winning class.
- **`attributed_episode`** is the episode of the **earliest** occupying attack.

Without this a literal implementation either raises a primary-key violation or silently last-write-wins, leaving a date carrying one headache-qualifying and one migraine-qualifying attack undefined.

**Step 4 — aura.** A date with ICHD-3-qualifying aura and no attack occupying it → `aura_only`, rule `delphi_aura`. Aura on a date already carrying a migraine-qualifying attack adds nothing; the date is already `migraine`.

**Step 5 — everything else** → `none`, `rule` null.

**Worked cases, all of which are §12 fixtures.**

| Case | Result |
|---|---|
| Attack Mon 20:00 → wakes headache-free Tue 07:00 | Qualifies under whichever rule applies — treated at ≥30 min, or untreated with the ≥4 h compound predicate. Occupies Mon and Tue; step 3a excludes Tue. → **1 migraine day (Monday)**, Tuesday `none`, **0** headache days from Tuesday |
| Untreated 22:00 Mon → 03:00 Tue, 2 characteristics, asleep through the end and wakes free | 5 h total plus 2 characteristics satisfies the untreated predicate. Occupies Mon and Tue; step 3a excludes Tue. → **1 migraine day**, not 0 |
| Same attack, patient awake at 03:00 | Step 3a does not apply — Tuesday had 3 h of headache. → **2 migraine days** |
| Status migrainosus Mon 09:00 → Fri 09:00, awake throughout | Occupies five dates. → **5 migraine days**, not 1 |
| Continuous three-week episode | → **21 headache days** |

**Why both halves are load-bearing.** Measuring duration per calendar date instead of per attack under-counts nocturnal onset: an untreated 22:00→03:00 attack gives 2 h on Monday and 3 h on Tuesday, reaches 4 h on neither, and yields **zero** migraine days where the answer is one. Attributing an attack only to its onset date under-counts every multi-day attack: a four-day status migrainosus episode yields **one** headache day where the answer is four, which would make the ≥15-day Botox and MOH thresholds structurally unreachable for exactly the patient this product exists for. Qualification per attack fixes the first; occupancy across dates fixes the second.

### 5.3 Episode boundaries

- **≥48 h headache-free** between attacks → distinct episodes.
- **<48 h recurrence** → ICHD-3 declines to resolve it. Emit **both** derivations: `relapse_collapsed` and `relapse_split`. Never choose silently.
- **Status migrainosus** — assessed only where an episode is unremitting **>72 h**, debilitating, and typical of prior attacks. **Within that assessment only**, remissions ≤12 h do not break the run. The 12-hour allowance is not a general episode rule and does not apply outside a status migrainosus assessment.
  Where the status migrainosus reading competes with a distinct-attack reading, it is emitted as an **additional** `status_migrainosus` derivation, not as a replacement.
- **If MOH criteria are met**, ICHD-3 directs coding MOH rather than status migrainosus. **Surface the distinction; do not decide it.**

### 5.4 Recomputation

**Iterative widening to a fixed point**, not a fixed window. A ±72 h window is wrong because status migrainosus episodes are unbounded — the prior revision's justification that "episode boundary rules top out at 48 h" was false.

```
seed  := [observed_at − 72h, observed_at + 72h]
repeat:
  load observations_live over `seed`
  compute episodes
  if any episode extends beyond `seed`: widen `seed` to cover it; continue
  else: converged
```

Termination: each iteration strictly widens or stops, and the observation set is finite. Cap iterations and fall back to full history on the cap.

**Full-history recompute** is required, not optional, for: a `config_hash` change, an `algorithm_version` change, and any MOH re-evaluation (which is monthly-windowed and not local to one observation).

**`computed_from` semantics.** A recompute must load **every observation belonging to each episode it emits**, even where those fall outside the seed window, so `computed_from` is complete for that episode. A `none` day's `computed_from` is the empty array — the absence of observations on that date **is** the finding, and the date itself is the range. No separate window record exists or is needed.

```sql
CREATE TABLE recompute_queue (
  id uuid PRIMARY KEY,
  user_id uuid NOT NULL,
  window_start timestamptz, window_end timestamptz,
  full_history boolean NOT NULL DEFAULT false,
  reason text NOT NULL,
  status text NOT NULL DEFAULT 'pending',
  attempts int NOT NULL DEFAULT 0,
  last_error text,
  leased_until timestamptz,
  enqueued_at timestamptz NOT NULL DEFAULT now(),
  processed_at timestamptz
);
CREATE INDEX ON recompute_queue (status, enqueued_at) WHERE status='pending';
```

Enqueued in the same transaction as the triggering write. **Override writes enqueue exactly as observation writes do.**

**Overrides and `confirmed_by` (D18).** Where a recompute finds a live `overrides` row keyed to an episode's `natural_key`, the `episodes` row it writes carries `confirmed_by: 'user'`, not the `system` default FR-3.26 specifies for a newly computed row — even though `computed_from` and other fields on that same row were, in fact, computed. A human set the boundary; the field should say so. This does not extend to `day_classifications` rows merely *attributed* to that episode (§5.2 step 3b) unless those rows are themselves the subject of a `day_classification` override.

### 5.5 Attenuation

Per **D13**: a day is flagged attenuated when it falls inside a medication window the **user recorded** — a `medication` observation carrying `course_start` and `course_end`. **No half-life model is inferred.** Inventing per-agent pharmacokinetic windows would be the invented-clinical-rule anti-pattern.

Three properties, stated plainly:

1. The flag applies to **any** day inside a recorded window, not only days classifying `none`. A day with twenty minutes of headache under dexamethasone is also confounded, and that is the more common case.
2. **An attenuated `none` day does not count** toward headache days or migraine days. The flag is annotation, not protection. That is defensible because it is not silent — but the documents must say so rather than implying otherwise.
3. Every export reporting a date range **enumerates the attenuated days within that range**, flagged, whatever their class. This is what makes the flag non-silent, and it is why the requirement is written against the range rather than against the day appearing in a count it cannot appear in.

**Framing.** Model as *attenuated symptom expression under active pharmacology* — a confounder on severity and observability — **not** a suppressed-then-restored attack. Pooled corticosteroid data shows steroids *reduce* recurrence (24 h median ARR 30%, NNT 3.3; 72 h ARR 11%, NNT 9.1) and post-steroid recurrences are typically milder.

### 5.6 Medication overuse

The prior revision had five `MUST` requirements, a test row and a recall-summary dependency, and **no algorithm**. Specified here.

```typescript
type MohAssessment = {
  derivationId: string;               // MOH is scoped like every other count
  variant: string;
  month: string;                      // 'YYYY-MM', calendar month per D12
  generalCriterionMet: boolean;       // ≥15 headache days AND attested disorder AND >3mo history
  headacheDays: number;
  perClass: Array<{ drugClass: string; days: number; threshold: number; exceeded: boolean }>;
  combinedClass: { distinctDays: number; threshold: 10; exceeded: boolean };
  flag: boolean;                      // generalCriterionMet AND (any perClass.exceeded OR combinedClass.exceeded)
  sustainedMonths: number;
  configHash: string;
};
```

Rules:
- **Window: calendar month** (D12). A rolling 30-day count may be displayed as a labelled non-clinical early warning; it is never the ICHD-3 determination.
- **Per-class days**: distinct calendar dates with an `is_acute` medication of that class.
- **Combined-class distinct days**: distinct calendar dates on which **any** of ergotamine, triptans, non-opioid analgesics or opioids were taken. **Not** the sum of per-class counts. Six triptan days and six NSAID days overlapping on three dates is **nine** distinct days and does **not** fire.
- **Combination analgesics are excluded** from the 8.2.6 combined rule — ICHD-3 handles them at 8.2.5.
- **`flag` requires the general criterion.** Drug-day counts alone never fire.
- **"Pre-existing headache disorder"** is a **user-attested settings flag**, never inferred. The app has no diagnosis field and inferring one would be a diagnostic assertion the guardrails prohibit.
- **Sustain**: the >3-month requirement is evaluated over consecutive qualifying months.
- **Scoped to a derivation, like every other count.** Variants can legitimately differ in day class — a collapsed relapse measures one longer attack and can qualify where two split attacks do not — so `generalCriterionMet`'s ≥15-headache-day gate must name the reading it was computed from. An MOH assessment is persisted alongside the derivation it belongs to and carries `derivation_id`, `algorithm_version`, `config_hash`, `confirmed_by` and `review_status` like any other derived object.

Any MOH flag shown to the user is presented as *the criterion a named guideline uses*, not as settled fact — ICHD-3 states its own day counts are "based on expert opinion rather than on formal evidence."

### 5.7 Review tiers

Tier 1 auto-applies cheaply-reversible inferences marked `confirmed_by: system`, `review_status: unreviewed`. **Candidate phase labels are retrospective only, triggered by a later attack opening — never at capture time**, and are a step-5 feature.

Tier 2 — medication-response boundary inferences, 48-hour-rule applications, status migrainosus classifications — is displayed immediately and queued. On `REVIEW_TIMEOUT` (14 days) `review_status` becomes `timed_out`; `confirmed_by` is untouched.

Tier 3 never auto-applies: anything already exported, anything contradicting a live override.

**The document gate** force-surfaces unconfirmed inferences before the prior-auth packet (step 7). Non-configurable, **not relaxed for the primary user**. **The recall summary is exempt** — unconfirmed items are labelled on the page instead.

Instrument correction rate, backlog size and **timeout rate**. Timing out is informationally identical to never reviewing.

---

## 6. Capture pipeline

`segment → extract (with spans) → normalise → verify → commit`

### 6.1 Structured output

Anthropic and OpenAI perform genuine constrained decoding **[V]**; Gemini's guarantee is explicitly weaker and its docs say to validate values in the application **[V]**.

**Anthropic documents three exceptions to schema conformance** that matter here **[V]**: a refusal, `max_tokens` truncation, and **enum-value casing drift**. The last one hits `drug_class` and `ask_status` directly — normalise enum casing server-side before validation rather than trusting the returned string.

**Shape guarantees are not truth guarantees.** A 2026 medical information-extraction benchmark on scanned genetic-test reports reported **65.3 F1 on clean text, 37.3 on fax-degraded, 55.6 cross-scenario average** **[V]**. Generalising from that task to this one is inference **[I]**, but the direction is clear enough to justify verification as the actual safety mechanism.

### 6.2 Evidence spans

Every extracted field carries a quoted span. Verification: deterministic exact-then-tight-fuzzy match against the transcript, **hard reject on failure**; then a structurally independent entailment check with a different prompt and ideally a different model. Fields failing either are dropped, not repaired. The verbatim transcript is retained regardless.

### 6.3 Confidence

Compute the lexical-versus-paraphrase distinction **deterministically from the span** against a canonical vocabulary. Do not ask the model.

Anthropic exposes no logprobs. OpenAI logprobs are reported empty under structured outputs on current models **[U — single community report, verify]**. Verbalised self-confidence is documented as systematically miscalibrated **[V]**.

### 6.4 Time resolution

The model extracts the time **expression**; **chrono-node** resolves it against a reference timestamp and IANA zone. Duckling is effectively legacy **[V]**. An unresolvable expression yields `observed_at = null` and a follow-up flag — never a guessed timestamp.

### 6.5 Tri-state and idempotency

`ask_status` required; `value` must be null when `not_asked`. **The server rejects any write where `value` is non-null and `ask_status` is `not_asked`** — schema descriptions are hints a model can ignore; this check is not.

Extraction is keyed by a hash of `(user_id, transcript, reference_timestamp)`; a repeat within the window returns the original result.

### 6.6 Provider

As of August 2026: Claude Sonnet 5 $2/$10 per 1M tokens (documented as standard through 31 Aug 2026 — reconfirm), Haiku 4.5 $1/$5, Gemini Flash-Lite $0.10–0.30 / $0.40–2.50. Pin the exact model id in config. At 15–30 seconds of speech per entry, cost is fractions of a cent — **latency and reliability are the constraints**.

**Anthropic offers a BAA for first-party API access**, signed by the account Primary Owner and enabled by sales; the Batch API and Files API are excluded **[V]**. OpenAI does not train on API data by default and offers a BAA under Zero Data Retention **[V]**.

---

## 7. API surface

Unchanged from the prior revision: `/api/observations`, `/api/sync/observations`, `/api/derived`, `/api/overrides`, `/api/review-queue`, `/api/recall`, `/api/export-links`, `/s/:token`, `/ingest/transcript`, `/mcp`, `/.well-known/oauth-protected-resource`.

`/api/derived` and `/api/recall` take a `variant` parameter and default to `primary`.

---

## 8. MCP server

### 8.1 Transport and SDK

**The 2026-07-28 revision is a large rewrite.** It removes protocol-level sessions and the `initialize` handshake, replaces them with `server/discover`, removes the GET stream endpoint and **SSE resumability including `Last-Event-ID`**, introduces Multi Round-Trip Requests, adds a required `resultType`, deprecates Roots, Sampling and Logging, and **deprecates Dynamic Client Registration in favour of Client ID Metadata Documents**. **[V]**

**Two SDK lines exist.** `@modelcontextprotocol/sdk` 1.30.0 (2026-07-27) implements the **superseded** revision. A **six-package split** — `@modelcontextprotocol/server`, `/client` and others, all at 2.0.0, published 2026-07-27 — implements 2026-07-28. **[V]**

**Recommendation: build against 1.30.0 today, and re-check at build time.** Real clients almost certainly still speak 1.x, and the v2 packages have no track record. Note honestly that this means implementing the superseded protocol revision — a deliberate compatibility choice, not a claim that 1.x is current. *Correction to the prior revision: the MCP organisation's current tutorial uses SDK 1.x over **stdio**, not a session-based Streamable HTTP example.* **[V]**

**Mitigation:** write every handler statelessly. **Skip `subscriptions/listen` entirely for v1** — no tool needs server push, and skipping it removes essentially all proxy-buffering and idle-timeout concerns.

Validate the `Origin` header, reject with 403 on mismatch. Send `X-Accel-Buffering: no` on any SSE response.

### 8.2 Authorisation

**Resource Server only.** The spec puts the Authorization Server role out of scope and assumes delegation **[V]**.

**WorkOS AuthKit** — documents OAuth 2.1, PKCE, DCR, **Client ID Metadata Documents as preferred** **[V]**, RFC 8707 and RFC 9728, with first-class MCP integrations. Alternatives: Auth0 (mature, quote-priced), Stytch (comparable), **Keycloak (no native RFC 8707 — needs a custom protocol mapper)** **[V, as of Aug 2026]**.

Implemented here: RFC 9728 protected-resource metadata with `WWW-Authenticate` on 401; RFC 8707 audience validation; `requireBearerAuth` over a supplied `verifyAccessToken`.

### 8.3 Tools

Six tools with explicit annotations. Defaults are `destructiveHint: true` and `openWorldHint: true`, so an unannotated read tool misrepresents itself.

| Tool | readOnly | destructive | idempotent | openWorld |
|---|---|---|---|---|
| `log_observation` | false | false | false | false |
| `log_medication` | false | false | false | false |
| `get_recent_observations` | true | — | true | false |
| `get_episode_status` | true | — | true | false |
| `get_day_counts` | true | — | true | false |
| `generate_recall_summary` | true | — | true | false |

Write tools return the full canonical record plus a one-line recap. Read tools return **the governing rule with every number**, including which `variant` and `config_hash` produced it, and the count of unreviewed inferences.

Errors use stable codes — `VALIDATION_ERROR`, `AMBIGUOUS_TIME`, `MISSING_REQUIRED_FIELD`, `CONSENT_REQUIRED`, `EPISODE_ALREADY_CLOSED`, `DUPLICATE_SUSPECTED` — with an actionable `suggestion`, and leak no stack traces.

Every handler calls `requireConsent` before touching clinical data. An `episode_id` returned to an assistant is bound server-side to the authenticated user; a handle is never authentication. All tool output is data, never instructions.

**Claude's Connectors Directory submission requires a Team or Enterprise organisation** — a solo builder cannot submit **[V]**. The manual add-by-URL path has no such requirement.

### 8.4 The iOS Shortcut path

A **personal access token**, write-only, scoped to `/ingest/transcript` only — not to MCP tool names, since the Shortcut does not speak JSON-RPC. Hash-only storage, immediate invalidation on regeneration, validated through the same `verifyAccessToken` path.

---

## 9. Sync

Hand-rolled. Pull by server `seq` watermark; push with `INSERT … ON CONFLICT (id) DO NOTHING`, where the client-generated UUIDv7 **is** the idempotency key. Per-row server-side validation so one malformed row never stalls a batch. Exponential backoff with jitter. The observations table is the queue; no separate outbox.

No sync engine. **Replicache was archived by its owner in June 2026** **[V]**. PowerSync's Capacitor SDK is beta **[V]**. CRDTs are the wrong model for immutable single-writer data.

**Derivation runs in both places, server authoritative.** One shared package, golden-fixture parity tests, server overwrites local on reconnect, and the server rejects or flags client-computed objects with an unrecognised `algorithm_version`.

**Override writes trigger server-side recompute exactly as observation writes do** — otherwise a second device can hold an override while displaying derived state that ignores it.

---

## 10. Native surface

### 10.1 Health data

**`@capgo/capacitor-health` 8.10.2** (published 10 Aug 2026). The only actively maintained cross-platform option. Covers sleep sessions with states, HRV, heart rate, steps, distance, calories, exercise time.

**Android floor is API 28+, not 26.** The Health Connect app itself requires API 28, so the plugin's stated Android 8.0/8.1 support is not reachable for Health Connect reads **[V]**.

**Sleep *stages* on both platforms is not independently evidenced** — HealthKit's sleep-analysis categories and Health Connect's stage model differ **[U]**. Verify against real device data before depending on stage granularity.

Maintenance risk is real: effectively a single maintainer, low adoption. Pin the version, be prepared to fork.

**Menstrual cycle remains a gap.** No maintained plugin exposes HealthKit `menstrualFlow` or Health Connect reproductive-health records **[V]**. Deferred; recorded as a custom-plugin task.

**HealthKit read denial is invisible by design** — `authorizationStatus(for:)` is reliable only for write types. The app cannot show a confident "access denied" prompt for reads; zero results is ambiguous with no data **[V]**.

**Health Connect** moved into the platform at **Android 14** **[V]**; below that it is a separate app requiring an install branch. Reads are capped to a **30-day rolling window** without extended-history permission **[V]**.

### 10.2 Local notifications

`@capacitor/local-notifications`. Two pieces of glue are required, not optional:

1. **Android exact-alarm revocation detection.** Android 12+ requires `SCHEDULE_EXACT_ALARM`; on revocation the app restarts and pending exact-alarm notifications are **deleted** **[V]**. Check on foreground, re-request, and surface that reminders were lost.
2. **Android boot reconciliation.** AlarmManager alarms do not survive reboot and the plugin registers no `BOOT_COMPLETED` receiver **[V, as of Aug 2026]**. Re-derive and reschedule on foreground.

Both are Android-only; iOS persists scheduled notifications across restart. iOS pending-request cap is **commonly cited as 64** **[U — developer-forum consensus, not in Apple's primary API reference]**. Doze throttles `allowWhileIdle` to once per 9 minutes **[V]**, well outside this cadence.

### 10.3 Background, storage, printing

No cross-platform push-on-new-health-data exists from a Capacitor plugin. **Design for foreground polling** plus `@capacitor/background-runner` as best-effort. Do not promise near-real-time background sync.

`@capacitor-community/sqlite` 8.1.1 is the device system of record, SQLCipher enabled, key in **iOS Keychain with `kSecAttrAccessibleWhenUnlockedThisDeviceOnly`** or Android Keystore.

**Key-loss behaviour is [U] and must be verified empirically** (§13). The design assumption is that a restore onto a new device brings the encrypted file but **not** the key; the app must detect "database present, key absent" as distinct from corruption and offer a server restore rather than regenerating a key and orphaning the file.

Migrations use `addUpgradeStatement`; the plugin backs up before migrating and restores on failure **[V]**. Migrations must be additive and must never assume `pending` rows are absent.

**Printing:** `window.print()` is not reliable in a WebView **[V]**. Use a native print plugin — Capawesome's Printer is at v8.0.14 (May 2026) and maintained **[V]** — plus local PDF generation through `@capacitor/share`. The PDF-and-share path is arguably primary; an emergency department may have no printer.

### 10.4 Build

iOS free provisioning: App IDs and profiles expire after **7 days**, 3 devices **[V]**. **Whether HealthKit works under free provisioning is [U] and cost-relevant** — verify first (§13).

Android: `./gradlew assembleDebug`, `adb install`, no account required. CI: iOS needs a macOS runner; Android builds anywhere.

**Fix the bundle identifier now.** Keep the write path decoupled from UI code so a future native App Intent can write to the same SQLite database without booting the WebView.

---

## 11. Privacy and security

Not HIPAA-covered as pure DTC. Operative regimes are the **FTC Health Breach Notification Rule** and **Washington's My Health My Data Act**, the latter binding on product design: separate opt-in for collection and separate authorisation for sharing, a distinct Consumer Health Data Privacy Policy, and a **private right of action** **[V, as of Aug 2026]**.

Consent UX: equally weighted buttons, nothing pre-checked, collection and sharing never on one control, declining is a complete working path, assistant consent is per platform.

**Policy placement.** WA AG guidance requires a **separate and distinct link** and that the document contain only Act-required content. It does **not** resolve whether that must be a standalone URL rather than a clearly separated section **[V, with that limit stated]**. A standalone route is the safer reading.

**Do not column-encrypt `drug_class` or any timestamp** — an encrypted column is opaque to the planner and would break the MOH window query and every range scan **[V]**. At this scale, application-level column encryption does not earn its complexity.

**Deletion.** MHMDA requires deletion including backups within 45 days plus one extension, backups permitted six months **[V]**. Resolution is **crypto-shredding**: per-user data encryption key, destroyed on deletion, rendering content unrecoverable including in restored backups; then tombstone. **Build the schema shape now, defer the destroy path.** The ops role (§3.1) is what makes the tombstone step executable.

**Third-party retention.** **Anthropic documents that MCP connectors are not zero-data-retention-eligible** and are retained under standard policy **[V]** — this is documented, not unknown. **OpenAI's position on data passed through third-party MCP tool calls is the part that remains undocumented** **[U]**. Disclose the difference accurately rather than treating both as unknown.

**Error tracking is the sleeper risk.** Disable `sendDefaultPii`, null request bodies in `beforeSend`, log parameterised placeholders only, scrub clinical field names before the event leaves the process. In scope for the privacy policy **only if** scrubbing is verified.

**Breach detection.** The HBNR clock runs from discovery **[V]**, so instrumentation is what makes it startable: export-link access anomalies, audit-log anomalies, 401/403 spikes, database connection logs, dependency and secret scanning, and a **calendar-scheduled monthly review** — for a solo operator that review is the actual discovery mechanism.

---

## 12. Testing

Beyond the prior revision's table, these fixtures are mandatory because each encodes a defect this specification previously contained:

| Fixture | Asserts |
|---|---|
| **Overnight, non-generative** | Attack Mon 20:00 → Tue 07:00 → exactly **1** migraine day, attributed to Monday; Tuesday is `none` and contributes **0** headache days |
| **Midnight-spanning untreated** | 22:00 Mon → 03:00 Tue, 2 characteristics, asleep through the end → **1** migraine day, not 0 |
| **Midnight-spanning, awake** | Same attack, awake at 03:00 → **2** migraine days |
| **Two attacks, one date** | A headache-qualifying and a migraine-qualifying attack both occupying one date → one row, `class = migraine`, `attributed_episode` = the earlier attack's episode |
| **Step 3a is attack-scoped** | A waking date suppressed for attack A but occupied by a separate attack B → the date still counts |
| **Multi-day occupancy** | Qualifying attack Mon 09:00 → Fri 09:00, awake throughout → **5** migraine days, not 1 |
| **Continuous three-week episode** | → **21** headache days, not 1 |
| **Treated day under threshold** | 20 min of headache with a dose → **not** a migraine day |
| **Untreated, one characteristic** | 4 h, one characteristic, no associated symptom → **not** a migraine day |
| **Variant scoping** | A month containing a sub-48 h relapse returns the same `headache_days` scoped to `primary` as it would with one derivation — the count does not multiply by variant |
| **MOH, disjoint** | 6 triptan + 6 NSAID days **disjoint**, general criterion met → combined-class **fires** (12 distinct days) |
| **MOH, overlapping** | Same counts with 3 shared dates → 9 distinct days → **does not fire** |
| **MOH, no general criterion** | Thresholds met, general criterion absent → **no flag** |
| **Attenuated range** | An export over a range enumerates attenuated days within it, flagged, whatever their class |
| **Status migrainosus remission** | An 11 h remission inside a >72 h run does not break it; the same remission outside a status migrainosus assessment does not extend an ordinary episode |
| **Recompute convergence** | An observation inside a 5-day episode widens the window to the full episode and produces a complete `computed_from` |
| **Ops erasure** | The app role cannot mutate; the ops role can |

Cross-runtime derivation parity is a first-class deliverable and a build-failing gate.

**The fabrication-rate gate** must pin model id, temperature and seed, or run as a scheduled non-blocking job. A flaky blocking CI gate on a stochastic system gets disabled, and the safety property goes with it.

---

## 13. Verify before building

1. **Does HealthKit work under free iOS provisioning?** **[U]** Cost-relevant — HealthKit is priority 1.
2. **ICHD-3 against the printed classification, and the 2026 Delphi Correction.** **[U]**
3. **SQLCipher key behaviour across device backup and restore**, both platforms — empirical spike. **[U]**
4. **Sleep-stage granularity** actually returned by `@capgo/capacitor-health` on each platform. **[U]**
5. **OpenAI logprobs under structured outputs.** **[U]**
6. **MCP SDK line** — recheck 1.x versus 2.x client support at build time.
7. **Whether the primary user is a WA, NV or CT resident**, which determines whether MHMDA binds from day one.

Items closed since the prior revision: the print plugin is maintained (v8.0.14, May 2026); WorkOS documents CIMD as preferred.

---

## 14. Not covered

Phase annotation and retrospective relabelling, medication-response inference as a shipped feature, correlation, and the prior-authorisation packet. The schema accommodates all four; the behaviour is unspecified.

The prodrome pattern surface is deferred entirely per D6 and is absent from every layer.
