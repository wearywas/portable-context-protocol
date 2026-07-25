# Portable Context Protocol — Specification v1.2 (draft)

**Status:** Draft · **License:** MIT · **Reference implementation:** Memorandai

The key words MUST, SHOULD, MAY are to be interpreted as in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

---

## 1. Overview

PCP defines how a person's curated personal context can be **owned, exported, and queried** in a vendor-neutral way, so any AI client can be authorized to read it. PCP is two things:

- **A data vocabulary** (§3) — typed JSON shapes for the portable pieces of personal context.
- **An MCP query profile** (§4) — a read-only set of MCP operations that serve that vocabulary on demand.

PCP is a *profile on top of [MCP](https://modelcontextprotocol.io)*: it does not define transport, authentication, or message framing — MCP does. (Analogy: OpenID Connect profiles OAuth for identity; PCP profiles MCP for personal context.)

### Goals
- Solve the **context cold-start problem** across AI platforms.
- Make curated personal context **portable** (no vendor lock-in) and **sovereign** (the user's device is the source of truth).
- Let a reading AI **trust-weight** context via provenance.

### Non-goals (v1)
- Not a storage engine, app, or server — those are *implementations*.
- Not a write or sync protocol. The query profile is **read-only by design** (§5). Writing into a user's context is an implementation's internal, trusted concern — not part of the portable read surface.
- Not an attempt to model *all* personal knowledge. v1 is a deliberately small core (§3); everything else is an extension (§6).

## 2. Terminology
- **Context store** — any system holding a user's PCP-shaped context (reference implementation: Memorandai).
- **Client** — an AI application authorized to read the user's context (e.g. Claude Desktop, Cursor).
- **Core type** — one of the three v1 vocabulary types (§3).
- **Extension** — a namespaced non-core field or type (§6).

## 3. Data vocabulary

All values are JSON. Timestamps are ISO 8601 strings. Every core object MAY include an `extensions` object (§6).

### 3.1 `profile`
The distilled "who you are" — the cold-start bundle (Layer 1).

| Field | Type | Req | Notes |
|-------|------|-----|-------|
| `schema_version` | string | MUST | `"1.0"` |
| `generated_at` | string (ISO 8601) | SHOULD | when the profile was distilled |
| `name` | string | MAY | |
| `profession` | string | MAY | |
| `primary_focus` | string | MAY | one-line current focus |
| `summary` | string (Markdown) | MAY | distilled prose overview |
| `domain_expertise` | string[] | MAY | |
| `values` | string[] | MAY | values & priorities |
| `communication_style` | string | MAY | |
| `working_style` | string | MAY | how the person works / collaborates |
| `tools_and_workflows` | string | MAY | |
| `how_interacts_with_ai` | string | MAY | preferences for working with AI |
| `extensions` | object | MAY | see §6 |

### 3.2 `keystone`
A single curated, high-signal, user-affirmed claim about the person. The richest type — its provenance fields are what let a reader trust-weight it.

| Field | Type | Req | Notes |
|-------|------|-----|-------|
| `id` | string | MUST | stable identifier |
| `content` | string | MUST | the claim (1–2 sentences) |
| `category` | enum | SHOULD | `identity` \| `preference` \| `rule` \| `objective` \| `constraint` \| `relationship` \| `glossary` \| `other` |
| `confidence` | number (0–1) | SHOULD | how strongly the claim is held / trusted |
| `importance` | number (0–1) | MAY | retention priority |
| `source` | enum | SHOULD | `user_stated` \| `inferred` \| `imported` \| `other` |
| `status` | enum | MAY | `active` \| `superseded` \| `deleted` — **record lifecycle only** |
| `sensitivity` | enum | MAY | `low` \| `medium` \| `high` |
| `created_at` | string (ISO 8601) | MAY | |
| `generated_by` | string (actor) | MAY | who authored the record — see §3.2.1 |
| `last_confirmed_at` | string (ISO 8601) | MAY | last time the claim was re-affirmed |
| `last_confirmed_by` | string (actor) | MAY | who last affirmed it — see §3.2.1 |
| `stale_after` | string (`YYYY-MM-DD`) | MAY | date on/after which the claim should be treated as stale — see §3.2.2 |
| `supersedes` | string (id) | MAY | id of a keystone this replaces |
| `extensions` | object | MAY | see §6 |

> **`status` is lifecycle, not approval.** It records whether a record is live / superseded / deleted — *not* whether it has been reviewed. A context store serves only **user-affirmed** keystones over the query profile; proposals awaiting a human's review are an implementation's internal concern and MUST NOT appear on the portable read surface. (Conflating the two has repeatedly misled reading agents; keep approval out of band.)

#### 3.2.1 Actors: authorship vs. affirmation

Who *wrote* a claim is not who *confirmed* it, and the difference is load-bearing: a keystone an agent drafted and a person affirmed carries more authority than one an agent both drafted and re-affirmed. `generated_by` and `last_confirmed_by` separate the two.

**Actor strings.** Both fields carry an *actor* — a string naming who performed the act, in one of three forms:

| Form | Shape | Example |
|------|-------|---------|
| Person | `human:<id>` | `human:ada` |
| Agent or model | `<producer>/<version>` | `memorandai/claude-opus-5` |
| Automated process | `process:<id>` | `process:calendar-import` |

**Affirmation tiers (derived, never stored).** A reader derives a tier from `last_confirmed_by`:

- absent → **unattributed** — affirmed per §3.2, but the store does not say by whom
- a non-`human:` actor → **machine-reaffirmed**
- a `human:` actor → **human-affirmed**

Readers MUST NOT discard unattributed keystones: absence of attribution is not evidence of low quality, and stores predating v1.2 have none. Stores MUST NOT serve a precomputed tier — it is a reader's inference, not a fact about the record.

**This does not weaken §3.2.** Reaching the portable surface still requires human affirmation. A machine re-affirmation refreshes `last_confirmed_at` on an *already-published* keystone; it MUST NOT be used to satisfy the publication rule. The tiers describe the most recent affirmation, not the right to be served.

**Consistency with `source`.** `source` classifies the *kind* of origin; `generated_by` names the *actor*. Where both are present they SHOULD agree — a `user_stated` keystone SHOULD carry a `human:` `generated_by`.

#### 3.2.2 Staleness

`last_confirmed_at` records when a claim was last affirmed; it cannot express when a claim should stop being trusted. Workflow facts rot silently — a tool preference stated in good faith a year ago is still served at full confidence today, and misleads the reader. `stale_after` closes that gap.

`stale_after` is an **absolute date** (`YYYY-MM-DD`), not a relative TTL. A record is stale on or after that day. An absolute date keeps the decision a plain comparison against today's date, with no reference to when the record was written or read, and it survives export, copy, and re-import unchanged.

Staleness is **advisory, not lifecycle.** A stale keystone is still true-as-recorded and is still served — labeled, per §4.2. It is not `superseded`, which is a `status` fact asserting that a replacement exists. A store MUST NOT auto-transition a stale keystone to `superseded` or `deleted`; only a person retires a claim.

`stale_after` is defined on `keystone` only. A `timeline_event` records something that happened on a date; it does not decay.

### 3.3 `timeline_event`
A dated event in the person's life.

| Field | Type | Req | Notes |
|-------|------|-----|-------|
| `id` | string | MUST | stable identifier |
| `date` | string | MUST | ISO 8601; **partial dates allowed** (`YYYY`, `YYYY-MM`, `YYYY-MM-DD`) |
| `summary` | string | MUST | what happened |
| `category` | enum | SHOULD | `job` \| `move` \| `milestone` \| `publication` \| `relationship` \| `other` |
| `confidence` | number (0–1) | MAY | |
| `source` | enum | MAY | `user_stated` \| `inferred` \| `imported` \| `other` |
| `extensions` | object | MAY | see §6 |

### 3.4 `search_result`
A single ranked match returned by `pcp_search`. Search reaches across heterogeneous content (keystones, conversations, documents, notes), so every result MUST carry an **origin classification** — the field that lets a reader trust-weight material it did not curate.

| Field | Type | Req | Notes |
|-------|------|-----|-------|
| `id` | string | MUST | stable identifier within the store |
| `type` | string | MUST | store-native record type (e.g. `keystone`, `conversation`) |
| `origin` | enum | MUST | `user_stated` \| `approved` \| `recorded` \| `imported` \| `generated` \| `inferred` (see below) |
| `snippet` | string | MUST | matched content excerpt; stores SHOULD cap length |
| `score` | number (0–1) | SHOULD | relevance score where the engine produces one |
| `timestamp` | string (ISO 8601) | SHOULD | when the underlying record was created/observed |
| `stale` | boolean | MAY | `true` where the underlying record is past its `stale_after` (§3.2.2) |
| `sensitivity` | enum | MAY | `low` \| `medium` \| `high`, where the record is classified |
| `source_system` | string | SHOULD | the store's identifier (e.g. `"memorandai"`) |
| `source_id` | string | SHOULD | stable id for retrieving the full record, where retrievable |
| `extensions` | object | MAY | see §6 |

**The origin taxonomy.** `user_stated` — a direct declaration by the person. `approved` — curated material the person has affirmed (e.g. keystones). `recorded` — a verbatim record of mixed authorship (a conversation transcript, a notebook page): faithful as a record, but neither party's endorsed claim. `imported` — brought in from an external source. `generated` — system-produced derivations (summaries, memos, notes-to-self). `inferred` — a system's guess. A store that cannot classify a record MUST default to `generated` rather than a higher-authority origin — under-claim, never over-claim.

## 4. MCP query profile (Layer 3)

A PCP-compliant context store exposes the following MCP tools. The names are **normative**; a store MAY also expose its own vendor-specific tools alongside them.

| Tool | Params | Returns |
|------|--------|---------|
| `pcp_get_profile` | `format?: "json" \| "markdown" \| "micro_prompt"` | a `profile` (`json` MUST be the strict §3.1 object) |
| `pcp_search` | `query: string`, `limit?: number` (default SHOULD be ≤ 10), plus the §4.2 filter params a store MAY support | `search_result[]` (§3.4) plus a `filters_applied` report (§4.2) |
| `pcp_get_timeline` | `from?`, `to?`, `query?`, `limit?` | `timeline_event[]` |
| `pcp_stats` | — | high-level counts + time range, for orientation |

Layers 1–2 (the distilled bundle and the full export) are this same vocabulary serialized to a file; Layer 3 is the live equivalent.

### 4.1 The `micro_prompt` contract

The compact profile is the recommended routine integration surface — the format a client loads at every cold start — so its behavior is specified:

- **Budget.** SHOULD fit within ~350 tokens (roughly 1,400 characters). Small enough to load unconditionally.
- **Content.** SHOULD lead with identity/role, current environment and workflow, and collaboration preferences — the facts that shape a working session.
- **Freshness (MUST).** The rendering MUST prefer the newest high-authority declarations and MUST NOT include superseded facts. Material drawn from keystones past their `stale_after` (§3.2.2) MUST NOT be presented as current — omit it, since the compact profile has no room to label it. It MUST carry a currency indicator (e.g. a trailing `(profile as of YYYY-MM-DD)` line) so a reader can weigh staleness.
- **Sensitivity.** MUST NOT include high-sensitivity material; the compact profile is the *least*-privileged view.
- **Determinism.** An empty or unavailable profile MUST return a fixed, unambiguous string (reference implementation: `"No profile is available yet in this context store."`) — never an error, never a hallucination-inviting blank.

### 4.2 Server-side retrieval safeguards

Client restraint is a courtesy; provider enforcement is a guarantee. A store SHOULD enforce, and MUST honestly report, retrieval safeguards on `pcp_search`:

- **Low default cap** (SHOULD default ≤ 10, hard max SHOULD ≤ 25).
- **Current-only by default** — superseded/deleted curated records do not surface unless explicitly requested.
- **Sensitivity exclusion by default** — `high`-sensitivity classified records require an explicit `include_sensitive` opt-in.
- **Optional relevance floor** (`min_score`) — score-less results SHOULD be kept and labeled rather than silently dropped.
- **Type filters** (`types`, `exclude_types`).
- **Stale labeling, not stale hiding** — results from records past their `stale_after` (§3.2.2) SHOULD be returned and marked (e.g. a `stale: true` flag on the result), not dropped. Staleness means *verify before relying*, not *false*; silently withholding a stale record leaves the reader with no fact at all rather than a dated one. A store MAY offer an `exclude_stale` opt-in for callers that want the stricter view.

Every response MUST include a `filters_applied` object stating what actually ran — including the limits of what the store *can* filter (e.g. `"sensitivity_filtering": "where_classified_v1"` when only some record types carry sensitivity classification). An unstated safeguard is indistinguishable from an absent one; a store MUST NOT imply filtering it did not perform.

## 5. Trust & governance

- **Read-only boundary (MUST).** The query profile defines no write operations. A compliant store MUST NOT mutate user context as a result of a query-profile call.
- **Provenance (SHOULD).** `keystone` and `timeline_event` SHOULD carry `source` and `confidence`; `pcp_search` SHOULD return them, so a client can weigh trust (a `user_stated` claim outranks an `inferred` one). Where a store knows the actors, `keystone` SHOULD also carry `generated_by` and `last_confirmed_by` (§3.2.1) — a claim a person affirmed outranks one only an agent re-affirmed.
- **Audit (SHOULD).** A store SHOULD record query-profile access — actor, what was requested, when — in a user-viewable log.
- **Client identification (MAY).** Over MCP a store MAY require client-identifying metadata (an actor category and a client id) and stamp it into the audit log.
- **Freshness (SHOULD).** Compact renderings and default query results SHOULD prefer the newest high-authority declarations and exclude superseded material (see §4.1, §4.2). Stale workflow facts served as current have demonstrably misled reading agents. Stores SHOULD set `stale_after` (§3.2.2) on claims they expect to decay — tool preferences, current projects, working arrangements — rather than relying on a reader to guess.
- **Future writes (normative note).** If a later version of PCP standardizes write operations, they will be a **separate capability** requiring explicit user authority — never an extension of the query profile. A client authorized to read is not thereby authorized to write.

## 6. Extensions & versioning

- **Extension points.** Any core object MAY carry an `extensions` object whose keys are **namespaced** (reverse-DNS or a clear vendor prefix, e.g. `"com.example.mood"`). Clients MUST ignore extension keys they don't understand, and MUST preserve them on round-trip. Implementations MUST NOT add non-namespaced top-level fields to core types.
- **New types.** Implementations MAY define additional types; propose them for the core via [CONTRIBUTING.md](../CONTRIBUTING.md).
- **Versioning.** `schema_version` tracks **wire compatibility**, not the spec document. It remains `"1.0"` through spec v1.2, because every field added since v1.0 is optional and every v1.0 payload is still valid. It bumps only on a breaking change: a removed or renamed field, a narrowed type, or a new requirement on an existing field. The core grows only across versions; widely-adopted extensions MAY graduate into the core.

## 7. Conformance

### 7.1 Producers

An implementation **produces** conformant PCP if it:

1. Emits `profile`, `keystone`, `timeline_event`, and `search_result` objects valid against the v1 schemas in [spec/schema/](schema).
2. If it exposes a live query surface, implements the §4 tools with the read-only semantics of §5.
3. Preserves unknown `extensions` on round-trip rather than dropping them.

### 7.2 Consumers

A reading client **consumes** PCP conformantly if it tolerates the whole legal range of the format. Concretely, a consumer MUST NOT reject or discard context because of:

- **Missing optional fields.** Everything but the MUST-columns of §3 is optional. A keystone with only `id` and `content` is valid.
- **Unknown `extensions` keys** (§6), or unknown values in a store-native `search_result.type`.
- **Absent provenance.** No `source`, no `confidence`, no actor fields — treat as lower-confidence, not as invalid. Stores predating v1.2 have no actor fields at all.
- **A dangling `supersedes`.** The referenced keystone may be outside the returned set or withheld by a §4.2 filter.
- **Staleness.** A stale record (§3.2.2) is dated, not false; weigh it down, don't drop it.

The asymmetry is deliberate. Producers are held to schema validity because they define the data; consumers are held to tolerance because the alternative — a client that hard-fails on a field it has not seen — makes the format unextendable in practice. Where a consumer cannot use something, it SHOULD ignore it quietly and proceed with the rest.

---

## Appendix A (informative): consumer authority ordering

From the first cross-agent field evaluation of this protocol (an OpenAI Codex integration reading a Memorandai store, 2026-07): a reading client benefits from an explicit authority order when sources conflict. The evaluated client used:

1. The person's current statements in the live session
2. Current project files and project instructions
3. Recent user-stated PCP results, with provenance
4. Approved keystones and curated profile material
5. Imported or generated summaries
6. Inference

Missing provenance lowers trust; conflicts are surfaced only when they materially affect the active task. This ordering is **informative** — it is good client practice, not a protocol requirement — but the §3.4 origin taxonomy and §3.2 provenance fields exist precisely so a client *can* implement it.

---

## Appendix B (informative): relationship to adjacent formats

### Open Knowledge Format (OKF)

Google Cloud's [Open Knowledge Format](https://github.com/GoogleCloudPlatform/knowledge-catalog/tree/main/okf) (published 2026; v0.2 at time of writing) is a vendor-neutral format for organizational knowledge. OKF and PCP share a thesis — knowledge trapped in proprietary systems should be portable, and a *format* beats a *platform* — but they occupy different scopes:

| | OKF | PCP |
|---|---|---|
| Subject | Organizational knowledge about *assets* (tables, metrics, runbooks) | Personal context about *a person* |
| Serialization | Markdown + YAML frontmatter, as a directory bundle | Typed JSON |
| Access surface | Explicitly out of scope | §4 — a normative read-only MCP query profile |
| Type system | One required field; producer-defined, no registry | A small fixed core vocabulary with schemas |
| Privacy | Not addressed | Sensitivity tiers, retrieval safeguards, audit, read-only boundary |

The scopes are complementary, and PCP treats OKF as a peer rather than a competitor. Where semantics genuinely coincide, PCP deliberately reuses OKF's vocabulary instead of inventing a parallel one: the actor convention of §3.2.1 and the absolute-date staleness rule of §3.2.2 are both adopted from OKF, so a consumer that already speaks one has less to learn about the other.

PCP does not adopt OKF's markdown serialization for its core. OKF is permissive by necessity — its domain has unbounded type variety, so a closed vocabulary would be futile. PCP's domain is bounded, and a small validated vocabulary is what lets a reading client trust-weight anything at all. A future version MAY define an OKF-conformant export profile for Layer 2, where human-readability and git-diffability are worth more than strict validation.

The distinction that matters most is the one OKF does not need to make: **a person is not a data asset.** Consent, sensitivity, and an auditable record of who read what are load-bearing for personal context and absent from a catalog format — which is precisely why PCP specifies a serving contract (§4, §5) where OKF stops at the file.
