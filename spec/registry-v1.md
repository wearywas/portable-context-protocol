# PCP Device Registry — Specification (extension, draft)

**Status:** Draft extension to [PCP v1](pcp-v1.md) · **License:** MIT

Core PCP ([pcp-v1.md](pcp-v1.md)) standardizes how *one* store exposes a user's context — the data vocabulary plus the `pcp_*` MCP query profile. The **device registry** is the layer above it: a user-curated directory of the PCP sources on their machine, so any agent can discover and query *across* them as one context layer — with no central broker.

## 1. What it is (and isn't)

- **Scope: one user, one device, their *own* stores.** The registry lists the PCP sources a single user has chosen to expose on their own machine (e.g. Memorandai + Yap + a community sidecar). It is **not** a way to reach *other people's* context — cross-person sharing is a separate, much heavier problem (consent, auth, identity) and is explicitly out of scope.
- **Opt-in per source.** The user adds each source deliberately. Agents MUST NOT auto-scan the machine for context stores — a source appears only because the user listed it. (The same "plug in the wallet" gesture as connecting a single store, once per store.)
- **A manifest, not a daemon.** The registry is *data the user owns* — a file — not a server that routing flows through (see §6).

## 2. The manifest

A JSON file at a well-known location:
- **`~/.pcp/sources.json`** (`%USERPROFILE%\.pcp\sources.json` on Windows) — the user-global registry.
- A project-local `./.pcp/sources.json` MAY extend/override it (same precedence idea as other dev tools).

It conforms to [`spec/schema/registry.schema.json`](schema/registry.schema.json); see [`examples/sources.example.json`](../examples/sources.example.json).

## 3. Source entries

Each entry describes one PCP store and how to reach its query profile. Connection fields mirror MCP client config, so a registry entry converts cleanly to/from an agent's MCP config.

| Field | Type | Req | Notes |
|-------|------|-----|-------|
| `id` | string | MUST | stable local id |
| `name` | string | MUST | display name |
| `transport` | enum | MUST | `stdio` or `http` |
| `kind` | string | SHOULD | freeform descriptor (`knowledge-studio`, `context-store`, `sidecar`, …) |
| `description` | string | MAY | what's in it |
| `command` / `args` / `env` | — | stdio | how to launch a stdio MCP server |
| `url` / `headers` | — | http | endpoint + auth for an http MCP server |
| `added_by` | enum | MAY | `user` \| `app` \| `sidecar` (provenance of the entry) |
| `extensions` | object | MAY | namespaced custom fields |

## 4. How an agent uses it
1. Read the manifest → the user's opted-in sources.
2. For a context question, call `pcp_search` (and/or `pcp_get_profile` / `pcp_get_timeline`) on the relevant sources.
3. Merge results — v1: **group by source** and reason across them; the core spec's provenance fields help weigh them. Global cross-store ranking (whose relevance score wins?) is a later problem; don't block on it.

## 5. The honest part: manual today, good with adoption

This extension is deliberately *just a convention*, so it works at three levels of effort and quality:

- **Manual — works today, no one's permission required.** A user can hand-write `sources.json`, and a small shared "PCP fan-out" skill (or simply an agent whose MCP config already lists the stores) can query across them. Real *now* — but the user wires it up and maintains it.
- **Community sidecars — extends reach, but fragile.** For an app that hasn't adopted PCP yet but stores data locally (e.g. Cursor's SQLite, Claude Code's JSON), a third party can run a sidecar MCP server exposing `pcp_*` over that data and add it to the registry — with no vendor involvement. Powerful, but brittle: it reverse-engineers a format that can change, and won't have real semantic search unless the sidecar builds its own index.
- **First-party — makes it good.** When an app natively (a) exposes the `pcp_*` profile and (b) registers itself on install (writes its own `sources.json` entry), and agents read the registry natively, the whole thing becomes seamless and robust. That's the destination; the two levels above are the bottom-up path to it.

So: the registry needs **no central adoption to *exist*** — a user can do it by hand — but it only becomes *frictionless and trustworthy* as apps and agents support it first-party. Honest status: a convention you can use today, a standard that gets good as others opt in.

## 6. Non-goals
- **Not cross-person sharing** (see §1).
- **Not a required aggregator.** An optional local aggregator MAY fan out on an agent's behalf, but it MUST be swappable and never the sole path — the registry (the data) is the standard; any aggregator is just one consumer of it. A mandatory broker would re-create the centralization PCP exists to dissolve.
- **Not auto-discovery.** Opt-in only.

## 7. Extensions & versioning
`pcp_registry_version` is `"1.0"`. Entries carry a namespaced `extensions` object like core objects. New transports or fields go through the process in [CONTRIBUTING.md](../CONTRIBUTING.md).
