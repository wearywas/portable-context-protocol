# Changelog

## v1.2 (draft) — 2026-07-25

Shaped by Google Cloud's **Open Knowledge Format** (OKF v0.2), published for
organizational knowledge. OKF and PCP share a thesis but split the scope, so
this revision adopts the two OKF mechanisms that address real gaps in PCP —
rather than adopting the format itself. Wire shapes remain backward
compatible (`schema_version` stays `"1.0"`); every field added is optional.

### Added
- **§3.2.1 actors** — `generated_by` and `last_confirmed_by` on `keystone`,
  separating *who wrote a claim* from *who last stood behind it*. Actor
  strings follow OKF's convention (`human:<id>` / `<producer>/<version>` /
  `process:<id>`), so a consumer that classifies OKF actors needs no new
  logic. Readers derive three affirmation tiers (unattributed /
  machine-reaffirmed / human-affirmed) and MUST NOT discard unattributed
  keystones. Publication still requires human affirmation — a machine
  re-affirmation refreshes an already-published keystone and MUST NOT
  satisfy the §3.2 publication rule.
- **§3.2.2 staleness** — `stale_after` on `keystone`: an **absolute date**
  (`YYYY-MM-DD`), not a relative TTL, so the decision is a plain comparison
  against today that survives export and re-import. Advisory, not lifecycle:
  a stale keystone is still true-as-recorded and still served, labeled. A
  store MUST NOT auto-transition stale records to `superseded`/`deleted`.
- **§3.4 `search_result.stale`** — optional boolean marking results whose
  underlying record is past its `stale_after`.
- **§4.2 stale labeling, not stale hiding** — stale results SHOULD be
  returned and marked, with an optional `exclude_stale` opt-in. Staleness
  means *verify before relying*, not *false*.
- **§7.2 consumer conformance** — explicit tolerance obligations: a consumer
  MUST NOT reject context for missing optional fields, unknown extension
  keys or types, absent provenance, dangling `supersedes`, or staleness.
  Producers are held to schema validity; consumers to tolerance.
- **Appendix B (informative)** — relationship to adjacent formats, with a
  scope comparison against OKF and the rationale for *not* adopting its
  markdown serialization for the PCP core (OKF is permissive because its
  type variety is unbounded; PCP's domain is bounded, and a small validated
  vocabulary is what makes trust-weighting possible). Flags a possible
  OKF-conformant Layer 2 export profile as future work.

### Changed
- **§4.1 `micro_prompt`** — freshness MUST now also exclude material drawn
  from keystones past their `stale_after` (the compact profile has no room
  to label it).
- **§5** — provenance bullet now covers the actor fields; freshness bullet
  directs stores to set `stale_after` on claims they expect to decay rather
  than leaving the reader to guess.
- **§6 versioning** — clarified that `schema_version` tracks *wire
  compatibility*, not the spec document, and bumps only on a breaking change.
- **§7** — split into §7.1 producers / §7.2 consumers.

## v1.1 (draft) — 2026-07-17

Shaped by the first cross-agent field evaluation: an OpenAI Codex integration
built a read-only cold-start pattern against the Memorandai reference store,
tested fresh-agent behavior before/after, and reported contract gaps. All
wire shapes remain backward compatible (`schema_version` stays `"1.0"`);
this revision adds one vocabulary type and normative behavior.

### Added
- **§3.4 `search_result`** — normalized search-match type with a six-way
  `origin` authority classification (`user_stated | approved | recorded |
  imported | generated | inferred`; unclassifiable → `generated`, never
  higher). New `spec/schema/search-result.schema.json` + example.
- **§4.1 `micro_prompt` contract** — token budget, required content,
  freshness (MUST prefer newest high-authority declarations, MUST exclude
  superseded facts, MUST carry a currency indicator), sensitivity floor,
  deterministic empty-profile behavior.
- **§4.2 server-side retrieval safeguards** — low default caps,
  current-only and sensitivity-exclusion defaults, optional relevance
  floor, and a mandatory honest `filters_applied` report ("an unstated
  safeguard is indistinguishable from an absent one").
- **§5 freshness bullet** and a **future-writes normative note** (writes,
  if ever standardized, are a separate capability — read authority never
  implies write authority).
- **Appendix A (informative)** — consumer authority ordering, from the
  Codex evaluation.

### Changed
- **§4** — `pcp_get_profile` `json` format now explicitly MUST return the
  strict §3.1 profile object (provider detail belongs in `extensions`);
  `pcp_search` documented with the §4.2 filter params and default-limit
  guidance.
- **§7 conformance** — now includes `search_result` validity.

## v1.0 (draft) — 2026-06-19

Initial public draft: three core types (`profile`, `keystone`,
`timeline_event`), the read-only MCP query profile, trust & governance
(read-only MUST, provenance SHOULD, audit SHOULD), extensions & versioning.
