# Temporal data model for HeadState on Postgres — implementation guidance

> ## ⚠️ Largely superseded — read for reasoning, never for schema
>
> `tech-spec.md` is the binding schema. This document is the exploration it was
> built from, and four of its decisions were reversed:
> - `superseded_by` as a **column** on `observations` — reversed; supersession is a
>   separate table, because the immutability trigger makes the column unsettable.
> - `config_version` as a text primary key — reversed; the binding schema uses
>   `config_hash`.
> - A **fixed ±72 h** recompute window — reversed; status migrainosus episodes are
>   unbounded, so recompute iteratively widens to a fixed point.
> - `attenuated` as a `rule` value — reversed; attenuation is a boolean flag
>   orthogonal to both `class` and `rule`.
>
> §8's claim that Postgres rejects `AT TIME ZONE` with a named zone in a generated
> column has been corrected in place — both forms are accepted on PG 16.13.

Scope: this answers the 9 research questions with concrete decisions and DDL. Where a claim comes from a source, it's cited inline. Where I'm reasoning from first principles (no citation available), it's marked **[reasoned]**. Anything I could not verify against a primary source is marked **[unverified]**.

---

## 1. Which pattern fits

**Recommendation: append-only fact log + explicitly versioned, hand-rolled derived tables, computed by application code — not full bitemporal machinery, not a generic event-sourcing framework, not native Postgres materialized views.**

Three candidates, and why two of them are the wrong amount of machinery for this problem:

**Full bitemporal modelling (SQL:2011 system-versioned tables / valid-time+transaction-time on every table).** Postgres has no native SQL:2011 temporal-table support — this has been asked on the mailing list repeatedly and the answer is consistently "not implemented" ([postgresql.org thread, Nov 2024](https://www.postgresql.org/message-id/842239.1732389267%40sss.pgh.pa.us)). The available implementations are third-party: `temporal_tables` (trigger-based, last meaningfully active years ago — [GitHub](https://github.com/arkhipov/temporal_tables), [pgxn](https://pgxn.org/dist/temporal_tables/)), and a pure-SQL bitemporal solution announced in **beta** in September 2025 using views + `INSTEAD OF` triggers over `_current`/`_history` table pairs ([postgresql.org mailing list announcement](https://www.postgresql.org/message-id/CAJMpnG5FxZw7dJd%2BEUu19GoETShT_AUGqx1SG6hsyMoTOH61%3Dw%40mail.gmail.com)). Neither is something you'd depend on for a health record. More importantly, bitemporal machinery earns its cost when rows in the *same table* get corrected and you need to reconstruct "what did the record look like as of transaction time T." You've already ruled that out at the source: observations are never updated in place. You don't need a generic engine to answer "what was true then" — you need to filter out superseded rows and record a cutoff (`recorded_at` / `computed_at`), which is much cheaper than bitemporal range machinery (`period` columns, GiST range overlap enforcement, temporal joins).

Your `observed_at` / `recorded_at` pair **is** the bitemporal pair — valid time and transaction/recording time — but purpose-built and immutable-append-only rather than generic and update-driven. That's enough. Full bitemporal machinery (temporal primary keys, `PERIOD` types, temporal foreign keys — see the general pattern description in [Aiven's bitemporal overview](https://aiven.io/blog/two-dimensional-time-with-bitemporal-data) and [Jamie Lord on bitemporal primary keys](https://lord.technology/2025/01/28/understanding-temporal-primary-keys.html)) is warranted when *corrections retroactively change what a row means at a point in valid time and you need to query "as of" both axes with SQL range operators*. Here corrections are new rows (`superseded_by`), and "as of" queries are answered by filtering `superseded_by IS NULL` as of a chosen `recorded_at` cutoff — a plain predicate, not a temporal join. **[reasoned]**

**Full event sourcing (CQRS-style, generic event store + replay-to-rebuild-state).** This is the closest named pattern to what you're describing, but "event sourcing" as an infrastructure pattern (see [reference implementations](https://github.com/eugene-khyst/postgresql-event-sourcing), [SoftwareMill's writeup](https://softwaremill.com/implementing-event-sourcing-using-a-relational-database/)) typically brings: a generic `events` table storing serialized deltas against an aggregate, snapshotting to avoid full replay, and a projection/rebuild pipeline that's genuinely rebuild-from-scratch on demand. You have one aggregate type (a user's observation stream) and a small, fixed set of projections (episodes, phase annotations, day classifications). You don't need generic event serialization, aggregate versioning, or a message bus. What you *do* want from event sourcing is the core discipline — immutable facts, projections are disposable and recomputable, replay is the source of truth — implemented directly as plain tables and a plain TypeScript function, not a framework. Calling this "event sourcing" in your spec is fine as a mental model; don't reach for an event-sourcing library or generic event store schema (`aggregate_id`, `event_type`, `payload jsonb`, `stream_version`) — it buys you serialization/versioning complexity you don't need when your event shape is already fixed and typed. **[reasoned]**

**Plain append-only log + materialized views.** Native Postgres `MATERIALIZED VIEW` doesn't fit because (a) `REFRESH MATERIALIZED VIEW` is a full rebuild unless you use `CONCURRENTLY`, which still recomputes the whole view; (b) you need *multiple simultaneous versions* of a projection to coexist (different `algorithm_version`, and per requirement 5, both relapse-collapsed and relapse-split readings at once) — a materialized view represents exactly one current snapshot of a query, not a versioned set; (c) episode-boundary detection (gap detection over `observed_at` ordered by user) is exactly the kind of window-function-heavy procedural logic that the one credible incremental-maintenance extension for Postgres, `pg_ivm`, explicitly does **not** support — window functions, HAVING, ORDER BY/LIMIT, and UNION are all unsupported by IVM ([pg_ivm README](https://github.com/ChuckHend/pg_ivm)). So neither plain nor incrementally-maintained materialized views can express the actual derivation logic. Use ordinary tables that your application writes to explicitly, versioned like the observation layer.

**What to actually build:** an immutable `observations` table; derived tables (`episodes`, `phase_annotations`, `day_classifications`) that are also insert-only and version-stamped (never `UPDATE`d — a recompute inserts a new version row); an `overrides` table that's pure input, never touched by recomputation; a `clinical_config` table that versions the thresholds. A TypeScript derivation engine reads observations + overrides + a pinned config version and produces new versioned rows. This is "event sourcing" in spirit, sized correctly for one aggregate and a handful of projections, with the observation-level bitemporal pair already covering what you need without a bitemporal engine.

---

## 2. Concrete DDL

### Primary keys: UUIDv7, not ULID, not bigint-as-primary

Postgres 18 added **native** UUID v4/v7 generation: `uuidv7([shift interval]) → uuid`, alongside `uuidv4()`, confirmed against the current docs ([PostgreSQL 18 UUID Functions](https://www.postgresql.org/docs/current/functions-uuid.html)). UUIDv7 is time-ordered (Unix ms timestamp in the high bits + random tail), which fixes the classic complaint about UUIDv4 primary keys (index bloat/page fragmentation from random insert order) while staying a standard 128-bit `uuid` column type — no new column type, no extra library to encode/decode. ULID is functionally similar (time-ordered, sortable) but has no native Postgres type; you'd store it as `text`/`char(26)`/`bytea` and lose the built-in `uuid` type's compactness, operators, and tooling support. Given Postgres is your fixed target, **UUIDv7 is the correct choice specifically because Postgres now has a first-class type and generator for it** — ULID's only edge (human-readable base32 sorting) doesn't matter here. **[reasoned, given cited native support]**

Caveat you need to check before committing to `uuidv7()` in DDL: your managed provider's actual Postgres version. As of this research, Fly.io's Managed Postgres offering documents **PostgreSQL 16** ([Fly.io MPG docs](https://fly.io/docs/mpg/)), Railway offers **PostgreSQL 17** templates ([Railway Postgres 17](https://railway.com/deploy/postgres-17--postgres-17-1)), and Render added **PostgreSQL 17** support ([Render changelog](https://render.com/changelog/postgresql-17-is-now-available-for-render-postgres-databases)). PG18 (which has the native function) shipped recently ([PostgreSQL 18 Released](https://www.postgresql.org/about/news/postgresql-18-released-3142/)) but I could not verify any of Fly/Railway/Render offering it yet — **[unverified, check at implementation time]**. Two fallbacks if you're stuck on ≤17: the `pg_uuidv7` extension ([pgxn](https://pgxn.org/dist/pg_uuidv7/), [craigpastro/pg_uuidv7](https://github.com/craigpastro/pg_uuidv7)) if your provider allows installing extensions, or generate UUIDv7 in Node before insert (the `uuidv7` / `uuid` npm packages both support it) and pass it as a literal — this also sidesteps the extension-availability question on managed Postgres entirely and is what I'd actually recommend for a single developer shipping against a managed provider: **generate IDs in TypeScript, not in SQL**, so the ID scheme isn't coupled to which Postgres version/extensions the host happens to offer.

Don't bother with a bigint surrogate key + UUID public key split. That pattern exists to keep the clustered/insert-hot index small and monotonic for high-throughput multi-tenant systems. You're single-user going to modest multi-tenant; UUIDv7 is already roughly insert-order-monotonic, so the index-bloat problem the bigint-surrogate pattern solves barely exists here. One key, one column, less schema surface.

### Observation table

```sql
CREATE TABLE observations (
    observation_id   uuid PRIMARY KEY DEFAULT uuidv7(),  -- or app-supplied UUIDv7 literal
    user_id           uuid NOT NULL REFERENCES users(user_id),

    observed_at       timestamptz NOT NULL,   -- instant, UTC-normalized (see §8)
    observed_at_tz    text NOT NULL,          -- IANA zone name captured at log time, e.g. 'America/Chicago'
    observed_local_date date NOT NULL,        -- frozen at write time, see §8 — NOT a generated column

    recorded_at       timestamptz NOT NULL DEFAULT now(),

    kind              text NOT NULL CHECK (kind IN (
                          'symptom','pain','medication','medication_response','sleep',
                          'food','activity','stress','cycle','environment','care_event','note')),

    body              jsonb NOT NULL,

    -- promoted, indexable fields extracted from body for the hot query paths (see §6)
    drug_class        text GENERATED ALWAYS AS (body->>'drug_class') STORED,
    severity          numeric GENERATED ALWAYS AS ((body->>'severity')::numeric) STORED,

    source            text NOT NULL CHECK (source IN
                          ('user_voice','user_text','user_tap','passive_feed','imported')),
    provenance        jsonb,                  -- { verbatim_span } | { feed_name }
    confidence        real,

    superseded_by     uuid REFERENCES observations(observation_id),
    CHECK (superseded_by IS NULL OR superseded_by <> observation_id)
);

CREATE INDEX idx_obs_user_time        ON observations (user_id, observed_at);
CREATE INDEX idx_obs_user_current     ON observations (user_id, observed_at)
    WHERE superseded_by IS NULL;
CREATE INDEX idx_obs_superseded_by    ON observations (superseded_by) WHERE superseded_by IS NOT NULL;
CREATE INDEX idx_obs_med_drug_class   ON observations (user_id, drug_class, observed_at)
    WHERE kind = 'medication' AND drug_class IS NOT NULL;
CREATE INDEX idx_obs_body_gin         ON observations USING gin (body jsonb_path_ops);
```

Notes on the `body`/JSONB decision: keep `body` as JSONB because the shape genuinely varies by `kind` (a `sleep` observation and a `medication` observation share almost no fields), and you don't want a wide sparse column table or a subtype-table-per-kind explosion at this stage. But **promote the fields that drive indexed lookups to real generated columns** — `drug_class` for MOH rolling-window counting, `severity` if you'll range/threshold-filter on it. This matches current guidance: GIN indexes on JSONB are good for containment/key-existence queries (`body @> '{"drug_class":"triptan"}'`) but don't help range scans, sorting, or exact predicate pushdown the way a real indexed column does — the pattern is "GIN for exploratory containment queries, B-tree expression/generated columns for your known hot predicates" ([Crunchy Data, Indexing JSONB in Postgres](https://www.crunchydata.com/blog/indexing-jsonb-in-postgres)). `GENERATED ALWAYS AS (...) STORED` is the mechanism — it materializes the extraction so you get a normal indexable, typed column without denormalizing storage into a second write. Note the generation expression must be `IMMUTABLE` — plain JSONB text extraction (`->>`) qualifies; this restriction is why you can't do the same trick for timezone-dependent expressions (see §8).

Per-kind shape validation: Postgres has no native JSON Schema `CHECK`. Options are a hand-written `CHECK` expression per `kind` (verbose, brittle to extend but zero dependencies), or the `pg_jsonschema` extension (Supabase-maintained, lets you `CHECK (pg_jsonschema.is_valid(schema, body))`) if your host permits installing it — **[unverified: extension availability on Fly/Railway/Render managed Postgres]**. Given a single developer and TypeScript end-to-end, I'd actually validate `body` shape in the Node layer (e.g., a Zod schema keyed by `kind`) before insert, and treat the DB `CHECK` as a coarse backstop (e.g., `body ? 'type'` and `body ? 'value'` at minimum) rather than the primary validation layer — cheaper to iterate on as the schema evolves, and it's the layer that's actually generating the data. **[reasoned]**

### Enforcing immutability at the database level

Layer three things: **role privileges** (primary control), **triggers** (defense in depth / clear error messages), and, if you want DB-level tamper-evidence beyond what your app role can bypass, nothing stronger is idiomatic in stock Postgres — there's no built-in "append-only table" flag. This is a well-trodden but unglamorous area; there's no single canonical extension, the practice is role + trigger ([PostgreSQL mailing list discussion of exactly this need](https://www.postgresql.org/message-id/200802212231.58019.xzilla@users.sourceforge.net)).

```sql
-- application role gets INSERT/SELECT only
REVOKE UPDATE, DELETE ON observations FROM app_user;
GRANT INSERT, SELECT ON observations TO app_user;

-- belt-and-suspenders: even a role with UPDATE/DELETE (migrations, a bug, a future
-- teammate) gets a clear failure instead of silent mutation
CREATE OR REPLACE FUNCTION forbid_mutate_observation() RETURNS trigger AS $$
BEGIN
    IF current_setting('app.allow_hard_mutation', true) IS DISTINCT FROM 'on' THEN
        RAISE EXCEPTION 'observations are append-only; corrections must insert a new row and set superseded_by';
    END IF;
    RETURN NEW; -- or OLD for DELETE
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_forbid_update BEFORE UPDATE ON observations
    FOR EACH ROW EXECUTE FUNCTION forbid_mutate_observation();
CREATE TRIGGER trg_forbid_delete BEFORE DELETE ON observations
    FOR EACH ROW EXECUTE FUNCTION forbid_mutate_observation();
```

The `app.allow_hard_mutation` escape hatch exists because you will eventually need one legitimate exception: GDPR/CCPA erasure requests, or removing genuinely wrong data entered by mistake (not a clinical correction, an actual mis-tap) that you don't want haunting the record forever. Make that path require an explicit `SET LOCAL app.allow_hard_mutation = 'on'` inside an ops-only transaction, never something the app role can set for itself — i.e., grant `SET` on that setting only to a separate maintenance role. **[reasoned — this is a common real-world gap in "immutable log" designs that's worth deciding now rather than discovering later.]**

### `superseded_by` chains

Keep it a nullable self-FK as above. Query "current" observations via a view so callers never have to remember the filter:

```sql
CREATE VIEW current_observations AS
    SELECT * FROM observations WHERE superseded_by IS NULL;
```

Two things worth deciding explicitly:

- **Chain depth.** Nothing stops a correction from itself being corrected (multi-hop chains). Your derivation engine should always resolve to the *terminal* row in a chain, not just filter one hop. A recursive CTE handles arbitrary depth if you ever need to walk history rather than just find current:
```sql
WITH RECURSIVE chain AS (
    SELECT observation_id, superseded_by, 1 AS depth
    FROM observations WHERE observation_id = $1
    UNION ALL
    SELECT o.observation_id, o.superseded_by, c.depth + 1
    FROM observations o JOIN chain c ON o.observation_id = c.superseded_by
)
SELECT * FROM chain ORDER BY depth DESC LIMIT 1;
```
  In practice, for the hot path (derivation engine reading "current" observations for a user), the `current_observations` view / partial index is what you use — the recursive walk is only for audit/debugging views ("show me the correction history of this entry").
- **A correction's `observed_at` may legitimately differ from the original** (user realizes the migraine started an hour earlier than first logged). The derivation engine must re-key by the *current* row's `observed_at`, not the superseded one — this is exactly why episode/day-classification recompute needs to be triggered off both the new observation *and* the window around the observation it superseded (see §3).

---

## 3. Recomputation strategy

**Trigger conditions**, in order of frequency: (1) a new observation insert, (2) an override insert, (3) a `clinical_config` version change. All three should enqueue work rather than recompute synchronously inline with the insert — keep the write path (what the user experiences when logging a symptom) fast and dumb; derivation is a background concern.

**Why not automatic incremental view maintenance (`pg_ivm` or similar):** `pg_ivm` is the credible current option (compatible PG13–18 per its own README, actively maintained — [ChuckHend/pg_ivm](https://github.com/ChuckHend/pg_ivm), background at [pganalyze's coverage](https://pganalyze.com/blog/5mins-postgres-15-beta1-incremental-materialized-views-pg-ivm)). It supports joins, `DISTINCT`, and standard aggregates incrementally, but explicitly **not** window functions, `HAVING`, `ORDER BY`/`LIMIT`, or `UNION` ([pg_ivm README](https://github.com/ChuckHend/pg_ivm)). Episode-boundary detection is inherently a `LAG`/`LEAD`-over-`observed_at`-partitioned-by-user gap-scan — a window function — so the actual derivation query can't be expressed as something `pg_ivm` (or plain `MATERIALIZED VIEW REFRESH`) could maintain incrementally even if you wanted it to. And separately, IVM-maintained views represent one current answer, not a versioned set with overrides layered on top and multiple algorithm versions coexisting. So: don't use it for the episode/day-classification derivation itself. (It could still be useful for a genuinely simple rollup — e.g., a `SELECT count(*)` per user per day for a dashboard tile — but that's not the hard part of this problem.)

**Bounding the recompute window (avoiding "recompute five years on every entry").** Your boundary rules give you a natural bound: the longest rule window is 48h (ICD episode boundary). A new or corrected observation at time `T` can only change episode membership/boundaries within `[T - 48h, T + 48h]`, and can only change day classifications for the calendar date(s) touched by episodes overlapping that window (a slightly wider margin — say 72h — covers the medication-response 12h/30min windows plus the overnight Delphi rule safely). So recompute is naturally local:

1. On new observation insert (or override insert), find all `episodes` whose `episode_id` appears in the join table for observations in `[observed_at - 72h, observed_at + 72h]` for that user (see bridge table below), plus any *new* episode boundary that might form in that window.
2. Recompute those episodes and the day classifications for the calendar dates they touch.
3. Insert new versioned rows (see §4); do not touch the old ones.

**Config version change** is different in kind: it doesn't invalidate existing rows (see §7 — old classifications under an old config stay valid readings under that config, by design/requirement 5). A config bump means: new observations from now on classify under the new config, and if the user explicitly asks "recompute my history under the new criteria," that's an explicit, user-initiated, potentially-expensive batch job — not something that fires automatically. **[reasoned]**

**Implementation shape**, plain and boring — this is the right amount of infrastructure for one developer:

```sql
CREATE TABLE recompute_queue (
    queue_id      bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    user_id       uuid NOT NULL,
    reason        text NOT NULL CHECK (reason IN ('new_observation','override','config_bump','manual')),
    window_start  timestamptz NOT NULL,
    window_end    timestamptz NOT NULL,
    enqueued_at   timestamptz NOT NULL DEFAULT now(),
    claimed_at    timestamptz,
    completed_at  timestamptz
);
```

Enqueue a row in the same transaction as the observation/override insert (`INSERT ... ; INSERT INTO recompute_queue ...` in one transaction, so you never lose a recompute obligation). A worker (a Node process, or `pg_cron` firing a `NOTIFY`/poll every few seconds) claims and processes rows. At single-user scale this can literally run synchronously after the HTTP request returns (fire-and-forget within the same process) — the queue table matters more once you have multiple tenants/concurrent writers and want to serialize recomputation per user without blocking writes. Build the queue table now (cheap), and start with in-process synchronous processing.

---

## 4. Versioning derived objects

Treat `episodes`, `phase_annotations`, and `day_classifications` with the **same append-only discipline as observations**, not as a table you `UPDATE` in place, and not as ephemeral output you throw away between recomputes. Reasoning: their row volume is tiny relative to observations (dozens of episodes/year, 365 day-classifications/year, even with several versions per day over years that's low thousands of rows for a single user over a decade) — so "unbounded" retention is not actually a storage problem here, and retention is exactly what requirement 3 (regenerate a March report in June) needs without relying on being able to re-run the derivation engine byte-identically forever.

```sql
CREATE TABLE day_classifications (
    day_classification_id uuid PRIMARY KEY DEFAULT uuidv7(),
    user_id            uuid NOT NULL,
    date               date NOT NULL,

    class              text NOT NULL CHECK (class IN ('none','headache','migraine','aura_only')),
    rule               text NOT NULL CHECK (rule IN
                            ('delphi_untreated_4h','delphi_treated_30min','delphi_aura','attenuated')),
    attenuation_flag   boolean NOT NULL DEFAULT false,
    active_pharmacology jsonb NOT NULL DEFAULT '[]',

    algorithm_version  text NOT NULL,
    config_version     text NOT NULL REFERENCES clinical_config(config_version),
    computed_at        timestamptz NOT NULL DEFAULT now(),
    computed_from       uuid[] NOT NULL,  -- see bridge-table note below re: querying this

    confirmed_by       text NOT NULL CHECK (confirmed_by IN ('system','user')),
    review_status      text NOT NULL CHECK (review_status IN ('unreviewed','timed_out','user_reviewed')),

    superseded_by      uuid REFERENCES day_classifications(day_classification_id)
);

CREATE INDEX idx_dayclass_user_date ON day_classifications (user_id, date);
CREATE INDEX idx_dayclass_current   ON day_classifications (user_id, date)
    WHERE superseded_by IS NULL;
```

`superseded_by` here means "a later recompute produced a newer reading under the *same* rule/variant" — i.e., it's how you distinguish "this is stale, a better answer exists" from "this is a deliberately-coexisting alternate reading" (which is what `rule`/variant-as-identity, not supersession, is for — see §5). A `(user_id, date, rule)` uniqueness-among-current constraint (partial unique index `WHERE superseded_by IS NULL`) keeps exactly one current row per date per rule variant:

```sql
CREATE UNIQUE INDEX uq_dayclass_current_per_rule
    ON day_classifications (user_id, date, rule) WHERE superseded_by IS NULL;
```

**Should old versions be retained, snapshotted only on export, or recomputed on demand?** Retain, given the row-count math above — don't build a pruning mechanism you don't need yet. But **additionally** snapshot at export time into a dedicated, fully immutable table for anything payer-facing, because "the current row still exists and points at the right config/algorithm version" is not the same guarantee as "this is exactly, unambiguously, what was shown to the payer in March" — you want the export itself to be a frozen artifact independent of future schema evolution in `day_classifications`:

```sql
CREATE TABLE report_exports (
    export_id          uuid PRIMARY KEY DEFAULT uuidv7(),
    user_id            uuid NOT NULL,
    generated_at       timestamptz NOT NULL DEFAULT now(),
    period_start       date NOT NULL,
    period_end         date NOT NULL,
    headache_days      integer NOT NULL,
    migraine_days      integer NOT NULL,
    day_classification_ids uuid[] NOT NULL,  -- exact frozen row ids used
    algorithm_version  text NOT NULL,
    config_version     text NOT NULL,
    payload            jsonb NOT NULL  -- full rendered report content, frozen
);
```
`report_exports` rows are themselves append-only (same trigger pattern as `observations`). This is the artifact that answers "regenerate the March report in June" — you don't actually regenerate it from `day_classifications`, you re-serve the frozen `report_exports` row, and *separately* you can prove it's still reproducible by re-running the engine against the same `computed_from` observation set + `config_version` and diffing. **[reasoned]**

---

## 5. The dual-reading requirement (no interval-exclusivity)

Do **not** put a GiST `EXCLUDE` constraint on episode time ranges — that constraint exists specifically to *prevent* what you need (two overlapping candidate episodes coexisting). `EXCLUDE USING gist (user_id WITH =, span WITH &&)` is the standard non-overlap pattern ([Cybertec, Exclusion constraints in PostgreSQL](https://www.cybertec-postgresql.com/en/exclusion-constraints-in-postgresql-and-a-tricky-problem/), [PostgreSQL Range Types docs](https://www.postgresql.org/docs/current/rangetypes.html)) and it's the wrong tool here by design, not by oversight.

Instead, make the **boundary rule / reading variant part of the row's identity**, so relapse-collapsed and relapse-split are two independently-addressable rows that both exist simultaneously, not competing versions of one slot:

```sql
CREATE TABLE episodes (
    episode_id         uuid PRIMARY KEY DEFAULT uuidv7(),
    user_id            uuid NOT NULL,

    boundary_rule      text NOT NULL CHECK (boundary_rule IN
                            ('icd_48h','status_migrainosus_12h','med_response_inference')),
    -- relapse_collapsed / relapse_split as an explicit second axis, since a single
    -- boundary_rule can still be read two ways within the <48h recurrence window:
    reading_variant    text NOT NULL DEFAULT 'default' CHECK (reading_variant IN
                            ('default','relapse_collapsed','relapse_split')),

    span               tstzrange NOT NULL,   -- for range-overlap *querying*, not exclusion
    open               boolean NOT NULL,

    algorithm_version  text NOT NULL,
    config_version     text NOT NULL REFERENCES clinical_config(config_version),
    computed_at        timestamptz NOT NULL DEFAULT now(),

    confirmed_by       text NOT NULL CHECK (confirmed_by IN ('system','user')),
    review_status      text NOT NULL CHECK (review_status IN ('unreviewed','timed_out','user_reviewed')),

    superseded_by      uuid REFERENCES episodes(episode_id)
);

-- GiST index WITHOUT exclusion — for fast "which episodes overlap this window" queries only
CREATE EXTENSION IF NOT EXISTS btree_gist;
CREATE INDEX idx_episode_span ON episodes USING gist (user_id, span);

CREATE INDEX idx_episode_open ON episodes (user_id) WHERE open = true;
```

Two overlapping rows with `reading_variant = 'relapse_collapsed'` vs `'relapse_split'` are both legitimate current rows — the partial-unique-per-current-variant pattern from §4 (`UNIQUE (user_id, ..., reading_variant) WHERE superseded_by IS NULL`) prevents accidental duplicates *within* a variant while allowing the two variants to coexist. The API layer decides which variant to surface as the default (per your spec: both, labelled with their governing rule), and the "current open episode" query filters to `reading_variant = 'default'` (or whichever variant your product treats as canonical for that single-episode UI) plus `open = true`.

Bridge table instead of `computed_from`/`member_observations` as raw UUID arrays — needed both for §3's windowed-recompute lookups and for straightforward indexing:

```sql
CREATE TABLE episode_observations (
    episode_id       uuid NOT NULL REFERENCES episodes(episode_id),
    observation_id   uuid NOT NULL REFERENCES observations(observation_id),
    role             text NOT NULL CHECK (role IN ('member','boundary_evidence')),
    PRIMARY KEY (episode_id, observation_id)
);
CREATE INDEX idx_episode_obs_reverse ON episode_observations (observation_id);
```
The reverse index (`observation_id → episodes`) is exactly what §3's "find episodes touching this new/corrected observation" query needs, and it's much cheaper to index/query than a GIN index over a `uuid[]` column would be for that direction. Keep the array on `day_classifications.computed_from` (§4) if you don't need reverse lookups from it as urgently, or mirror the same bridge-table pattern (`day_classification_observations`) if you do — I'd do the bridge table for both, given how central "which derived rows does this observation feed" is to the recompute strategy.

---

## 6. Indexing and query patterns

| Hot query | Index | Notes |
|---|---|---|
| Observations in a date range, one user | `idx_obs_user_time (user_id, observed_at)` btree | Standard; add `WHERE superseded_by IS NULL` partial variant if "current only" is the overwhelmingly common case, which it will be for the UI |
| Current open episode | `idx_episode_open (user_id) WHERE open = true` partial | Tiny index, always current, self-maintaining |
| Day classifications for a month with rule | `idx_dayclass_user_date (user_id, date)` + `rule` in the row | Range scan on `date`, filter `rule` in the query; add `rule` to the index as a third column if you always filter by a fixed rule for a given UI view |
| Medication observations by drug class, rolling 30-day window | `idx_obs_med_drug_class (user_id, drug_class, observed_at) WHERE kind='medication'` | Partial index keeps it small; the generated `drug_class` column (§2) is what makes this indexable at all — without promoting it out of JSONB you'd need a GIN + containment query, which is worse for a range+equality combo like this |

**What fights the append-only write pattern, and why it mostly doesn't matter here:** GIN indexes are the one index type with real write-amplification cost — every insert has to update the inverted index structure, which is heavier than a btree insert. On a high-throughput multi-tenant table this is a real tradeoff (batch GIN updates, or use `fastupdate` pending-list behavior deliberately). At single-user personal-logging insert rates (a handful of observations per day), this cost is immaterial — flag it as something to revisit if/when this becomes genuinely multi-tenant with concurrent high-frequency writers, not something to design around now. **[reasoned]** Btree indexes on an insert-only table are close to the best case for Postgres — no `UPDATE`-driven index bloat, no HOT-chain complications, straightforward autovacuum behavior. So: index liberally on `observations` for the query patterns you know you need; don't reach for BRIN yet. BRIN is the right call once the table is large (many millions of rows) and heavily range-scanned, trading precision for tiny index size on physically-correlated data — general guidance on this tradeoff in [Percona's BRIN writeup](https://www.percona.com/blog/brin-index-for-postgresql-dont-forget-the-benefits/) and [the Postgres wiki's month-based partitioning notes](https://wiki.postgresql.org/wiki/Month_based_partitioning). Not relevant at your current or near-term scale; a plain btree on `(user_id, observed_at)` is strictly better until the table is orders of magnitude larger than one person's lifetime of symptom logs will ever be. Revisit if/when multi-tenant volume changes the picture.

---

## 7. Migration and threshold-config versioning

Both, deliberately, not either/or: **a source-controlled TypeScript/JSON file is the single source of truth and the thing your tests assert against; a DB table is a referential-integrity anchor that derived rows FK against.** A file alone can't be joined against by SQL and can't guarantee a derived row's config reference stays resolvable; a table alone loses git blame, PR review, and inline citation comments next to each constant.

```sql
CREATE TABLE clinical_config (
    config_version   text PRIMARY KEY,        -- e.g. 'ichd3-2024.1' or a content hash
    ichd_edition     text NOT NULL,            -- 'ICHD-3' | 'ICHD-4-draft-2026'
    source_citation  text NOT NULL,            -- URL/clause, e.g. ichd-3.org §8.2.2
    thresholds       jsonb NOT NULL,           -- the actual numeric constants
    content_hash     text NOT NULL,            -- sha256 of the canonical source file, for drift detection
    created_at       timestamptz NOT NULL DEFAULT now()
);
```
Same immutability discipline as observations: `config_version` rows are never edited once any `day_classification`/`episode` references them (enforce with the same revoke-UPDATE/DELETE + trigger pattern). A migration loads new rows from the checked-in config file at deploy time; the deploy step computes `content_hash` and either inserts a new `config_version` row (if the hash changed) or is a no-op (idempotent). This directly satisfies requirement 5 ("a day classification computed under one threshold set must remain distinguishable from one computed under another") — the FK on `day_classifications.config_version` makes it a hard database guarantee, not a convention. What makes a past computation auditable: `(observation set via computed_from/bridge table) + algorithm_version + config_version` is a complete, reproducible input triple for any derived row; store all three, always, on every derived row (already reflected in the DDL above).

For the specific ICHD-3 medication-overuse thresholds you'll encode into `thresholds`: the ICHD-3 MOH criteria (§8.2) require regular overuse for **>3 months**, with headache on **≥15 days/month**, and the per-drug-class day counts (triptans, ergots, opioids, combination analgesics generally ~10 days/month; simple analgesics ~15 days/month) are specified in ICHD-3 subsections **8.2.1–8.2.8** which I could not fetch the exact per-subtype numbers from during this research — **[unverified, verify each numeric constant directly against the corresponding subtype page before encoding, e.g. §8.2.2 for triptans](https://ichd-3.org/8-headache-attributed-to-a-substance-or-its-withdrawal/8-2-medication-overuse-headache-moh/)**. ICHD-3 itself states these thresholds are "based on expert opinion rather than on formal evidence" — worth keeping that framing in your citation comments, since it's directly relevant to why the thresholds need to be versioned/swappable rather than hardcoded.

---

## 8. Time zones and DST

Store `observed_at` and `recorded_at` as `timestamptz`, always — never bare `timestamp`. `timestamptz` stores an unambiguous UTC instant and Postgres handles the display-side conversion; bare `timestamp` plus a separate zone string is more error-prone for interval arithmetic and comparisons because nothing forces the arithmetic to account for the zone consistently ([PostgreSQL Date/Time Types docs](https://www.postgresql.org/docs/current/datatype-datetime.html); general treatment in [boringSQL's timezone piece](https://boringsql.com/posts/know-the-time-in-postgresql/)).

But `timestamptz` alone can't answer "what calendar date did this belong to" — that depends on the zone the user was actually in, which you need for calendar-date-based day classification. So carry **both**: `observed_at timestamptz` (the unambiguous instant, for all interval/range arithmetic — the 48h/12h/30min boundary rules) and `observed_at_tz text` (the IANA zone name captured from the device at logging time — `'America/Chicago'`, not a bare UTC offset, so DST transitions resolve correctly for that specific date).

The important, easy-to-miss decision: **compute `observed_local_date` in application code at write time and store it as a plain column — do not rely on a `GENERATED ALWAYS AS ((observed_at AT TIME ZONE observed_at_tz)::date) STORED` column to do the freezing for you.** **Tested against PostgreSQL 16.13:** both `(ts AT TIME ZONE 'America/New_York')::date` (a literal zone) and `(ts AT TIME ZONE z)::date` (`z` a `text` column reference) are **accepted** in a `GENERATED ALWAYS AS ... STORED` column — `timezone(text, timestamptz)` is marked `IMMUTABLE` in Postgres, so both forms parse and create successfully. A generated column is not rejected here.

The real reason to freeze the date in application code instead is **tzdata changes**, not a parser rejection: a generated column of this shape is recomputed from `observed_at` and the zone name whenever the row is read back through certain paths, so a future update to the server's tzdata database (jurisdictions do periodically revise historical DST rules) could silently reclassify a historical observation's calendar date out from under a report that has already been handed to a clinician. Freezing the date at write time makes the classification stable regardless of any later tzdata revision. Resolve the zone conversion in Node (Luxon or `date-fns-tz` both handle IANA-zone-aware conversion correctly across DST boundaries) at insert time, and persist the result as an ordinary `date NOT NULL` column, indexed normally.

For the Delphi overnight rule spanning midnight: do the actual gap/window comparisons on `timestamptz` instants (subtracting/comparing UTC instants is always correct, DST-proof, by construction), and only convert to a local calendar date at the point where you need to *bucket* an episode into a day-classification row, never earlier. The bug pattern to avoid is doing arithmetic like "add 24 hours to a local wall-clock timestamp" to walk across a day boundary — on a DST transition day that's either 23 or 25 real hours, and reasoning in local wall-clock arithmetic across it produces off-by-one-hour errors that then can flip which side of a threshold (e.g., the 30-minute treated-response window) an event falls on. Always: compare instants, bucket to date last. **[reasoned, standard practice, no single citation but consistent across all the timestamptz-vs-timestamp sources reviewed]**

A report is stable across the user's own travel/DST changes specifically because `observed_local_date` was frozen using the zone the user was actually in *at the time*, not recomputed later against whatever zone context happens to be active when the report runs.

---

## 9. Testability

Structure the derivation engine as a **pure function**: `deriveEpisodesAndClassifications(observations: Observation[], overrides: Override[], config: ClinicalConfig) → { episodes, phaseAnnotations, dayClassifications }`, with no DB reads inside it. All DB access (fetching the windowed observation set, the applicable overrides, the pinned config) happens in a thin caller that assembles inputs and hands them to the pure function, then persists the pure function's output as new versioned rows. This is what makes deterministic replay possible: tests construct a fixed `Observation[]` array by hand (or load a fixture), pin a specific `ClinicalConfig` object, call the function, and assert exact output — no database, no clock, no I/O in the loop, so the same input always produces byte-identical output.

For asserting clinical constants against cited sources: keep the numeric thresholds in the same source file whose hash populates `clinical_config.content_hash` (§7), with an inline citation comment on every constant (e.g. `// ICHD-3 §8.2.2: triptan overuse ≥10 days/month for >3 months — https://ichd-3.org/...`). Write a lint/test step that walks the config object and fails if any leaf numeric value lacks an adjacent citation comment — cheap to write (a small script over the AST or even a regex over the source file), and it turns "someone changes a threshold without checking the source" into a build failure rather than a silent drift. Name clinical test cases after the citation they exercise (`"§8.2.2 triptan overuse: 10 injection-days across 3 months → MOH flag"`) so a failing test is immediately traceable to the clinical rule it's protecting, not just a diff in output.

For DB-level behavior specifically (the immutability triggers, the partial-unique-per-variant constraint, generated-column extraction, `EXCLUDE`-constraint absence), test against a real disposable Postgres instance rather than a JS-based Postgres emulator — JSONB generated columns, GiST indexes, and trigger behavior are exactly the kind of thing an emulator gets subtly wrong. Testcontainers (spinning up a real Postgres in Docker per test run) is the current standard approach for this and works cleanly from a TypeScript/Node test suite. **[reasoned — general current practice, not tied to a single source]**

---

## Summary of concrete decisions

- Pattern: append-only observation log + explicitly versioned derived tables written by application code. Not full bitemporal machinery, not a generic event-sourcing framework, not native/incrementally-maintained materialized views.
- PKs: UUIDv7 everywhere, generated in TypeScript (not relying on `uuidv7()` existing on your managed Postgres version yet — verify PG18 availability on Fly/Railway/Render before depending on the native function).
- `body`: JSONB with promoted `GENERATED ALWAYS AS (...) STORED` columns for `drug_class` and any other hot-filter field; validate shape in the Node layer, not via a Postgres JSON-schema extension.
- Immutability: `REVOKE UPDATE, DELETE` from the app role + a trigger backstop with an explicit, ops-only escape hatch for legal erasure.
- `superseded_by`: self-FK, resolved via a `current_observations` view for the hot path, recursive CTE only for audit trails.
- Recomputation: windowed (±72h around a touched observation) not full-history, queued in a `recompute_queue` table written in the same transaction as the triggering insert, processed synchronously for now.
- Derived-row versioning: keep every version (row counts are small enough that this isn't a storage problem), plus a fully frozen `report_exports` snapshot table for anything payer-facing.
- Dual readings: `reading_variant`/`boundary_rule` as part of row identity, no `EXCLUDE` constraint, GiST index for overlap *querying* only.
- Config: file is source of truth + citation comments + lint check; `clinical_config` table is the FK anchor derived rows reference, keyed by content hash.
- Time: `timestamptz` for instants, IANA zone name alongside it, local calendar date computed in application code and stored as a plain (not generated) column — frozen at write time.
