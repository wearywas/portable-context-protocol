# Changelog

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
