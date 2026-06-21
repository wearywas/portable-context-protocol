# Contributing to PCP

PCP grows through use. The core vocabulary is intentionally small and stable; the goal is for the community to extend it without fragmenting it.

## How to propose a change

- **A new field on a core type, or a whole new type** — open an issue describing the use case and the shape. Core changes need broad agreement and ship only across versions.
- **A vendor- or domain-specific field** — you don't need permission. Add it under the `extensions` namespace (reverse-DNS or a clear vendor prefix, e.g. `"com.example.mood"`). Readers ignore extensions they don't understand and preserve them on round-trip.

## Core vs. extensions

- The **core** (`profile`, `keystone`, `timeline_event` in v1) is the shared, portable baseline every implementation can rely on. It stays minimal.
- **Extensions** are where experimentation happens. A widely-adopted extension MAY graduate into the core in a later version.

This is the deal that keeps PCP both coherent (a small core every reader understands) and alive (anyone can extend without forking).

## Principles

- **Neutral.** The spec names no vendor except as a reference implementation. Keep it that way — PCP belongs to everyone who implements it.
- **Read-only at the boundary.** The query profile is for *reading* context, never silently writing it. Proposals that add write operations to the portable surface are out of scope (spec §5).
- **Provenance matters.** Anything that helps a reading AI trust-weight — `source`, `confidence`, dates — is welcome.

## Process

Issues for discussion, PRs for concrete changes. A schema change MUST update both `spec/pcp-v1.md` and the JSON Schema in `spec/schema/`, and SHOULD add or update an example.

> Note: the schema `$id` URLs in `spec/schema/` are placeholders pending a canonical home — adjust them once the repository's permanent location/domain is set.
