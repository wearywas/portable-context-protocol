# Portable Context Protocol — Specification v1 (draft)

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
| `last_confirmed_at` | string (ISO 8601) | MAY | last time the claim was re-affirmed |
| `supersedes` | string (id) | MAY | id of a keystone this replaces |
| `extensions` | object | MAY | see §6 |

> **`status` is lifecycle, not approval.** It records whether a record is live / superseded / deleted — *not* whether it has been reviewed. A context store serves only **user-affirmed** keystones over the query profile; proposals awaiting a human's review are an implementation's internal concern and MUST NOT appear on the portable read surface. (Conflating the two has repeatedly misled reading agents; keep approval out of band.)

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

## 4. MCP query profile (Layer 3)

A PCP-compliant context store exposes the following MCP tools. The names are **normative**; a store MAY also expose its own vendor-specific tools alongside them.

| Tool | Params | Returns |
|------|--------|---------|
| `pcp_get_profile` | `format?: "json" \| "markdown" \| "micro_prompt"` | a `profile` |
| `pcp_search` | `query: string`, `limit?: number` | ranked matches across the user's context (keystones, events, …), **each with its provenance fields** |
| `pcp_get_timeline` | `from?`, `to?`, `query?`, `limit?` | `timeline_event[]` |
| `pcp_stats` | — | high-level counts + time range, for orientation |

Layers 1–2 (the distilled bundle and the full export) are this same vocabulary serialized to a file; Layer 3 is the live equivalent.

## 5. Trust & governance

- **Read-only boundary (MUST).** The query profile defines no write operations. A compliant store MUST NOT mutate user context as a result of a query-profile call.
- **Provenance (SHOULD).** `keystone` and `timeline_event` SHOULD carry `source` and `confidence`; `pcp_search` SHOULD return them, so a client can weigh trust (a `user_stated` claim outranks an `inferred` one).
- **Audit (SHOULD).** A store SHOULD record query-profile access — actor, what was requested, when — in a user-viewable log.
- **Client identification (MAY).** Over MCP a store MAY require client-identifying metadata (an actor category and a client id) and stamp it into the audit log.

## 6. Extensions & versioning

- **Extension points.** Any core object MAY carry an `extensions` object whose keys are **namespaced** (reverse-DNS or a clear vendor prefix, e.g. `"com.example.mood"`). Clients MUST ignore extension keys they don't understand, and MUST preserve them on round-trip. Implementations MUST NOT add non-namespaced top-level fields to core types.
- **New types.** Implementations MAY define additional types; propose them for the core via [CONTRIBUTING.md](../CONTRIBUTING.md).
- **Versioning.** `schema_version` is `"1.0"` for this spec. The core grows only across versions; widely-adopted extensions MAY graduate into the core.

## 7. Conformance

An implementation is **PCP v1 conformant** if it:

1. Emits `profile`, `keystone`, and `timeline_event` objects valid against the v1 schemas in [spec/schema/](schema).
2. If it exposes a live query surface, implements the §4 tools with the read-only semantics of §5.
3. Preserves unknown `extensions` on round-trip rather than dropping them.
