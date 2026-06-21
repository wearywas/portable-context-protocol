# Portable Context Protocol (PCP)

**An open standard for portable, user-owned personal context that any AI can read.**

> Status: **v1 (draft)** · License: [MIT](LICENSE) · Reference implementation: [Memorandai](https://memorandai.com)

---

## The problem

Two problems, one root:

- **The context cold-start problem.** Every new AI session — every new model, every new app — starts from zero. It doesn't know who you are, what you're working on, or how you like to work. You re-explain yourself, endlessly.
- **Vendor lock-in.** The platforms that *do* remember you (ChatGPT Memories, Claude Projects, …) trap that memory inside one vendor. Your curated context isn't yours to carry.

PCP inverts this: **you hold a portable bundle of your own context, in a neutral format, and authorize any AI to read it.** Your device is the source of truth; AI clients are interchangeable interfaces — the way a hardware wallet holds your keys and any wallet app can connect to it.

## What PCP is

PCP is **not** an app or a server. It's two things any implementation can conform to:

1. **A neutral data vocabulary** — typed shapes (`profile`, `keystone`, `timeline_event`) for the portable pieces of personal context. Plain JSON; vendor-free.
2. **An MCP query profile** — a small, **read-only** set of operations a personal-context store exposes so any MCP-capable AI can query that vocabulary on demand.

The relationship is the same as **OpenID Connect to OAuth**: [MCP](https://modelcontextprotocol.io) is the general protocol; PCP is the profile that standardizes *personal context* on top of it.

## Three layers

PCP context travels at three levels of fidelity, all sharing one vocabulary:

| Layer | What | Use |
|-------|------|-----|
| **1 — Portable Context Profile** | A distilled bundle (JSON / Markdown / ~250-token micro-prompt) | Paste-in cold-start: give a fresh AI a cursory sense of who you are |
| **2 — Full export** | The complete archive of your context | Migration between tools / installs |
| **3 — Live query** | An AI queries your running store over MCP | On-demand, always-current context |

## The core vocabulary (v1)

Three types. Each carries an `extensions` object so vendors and the community can add fields **without forking the core**.

- **`profile`** — the distilled "who you are": identity, focus, working style, values, expertise.
- **`keystone`** — a single curated, high-signal claim about you, carrying provenance (`source`, `confidence`, `last_confirmed_at`) so a reading AI can weigh how much to trust it.
- **`timeline_event`** — a dated life event (jobs, moves, milestones); partial dates allowed.

Full definitions: **[spec/pcp-v1.md](spec/pcp-v1.md)** · JSON Schemas: **[spec/schema/](spec/schema)** · Worked examples (a fully **fictional** persona — no real data): **[examples/](examples)**.

## The trust model

A reading AI is a *guest*, not an editor:

- **Read-only at the boundary.** The query profile exposes no writes — an external AI can read your context but never silently mutate it.
- **Provenance travels with the data.** Every `keystone` carries `source` / `confidence` / `last_confirmed_at`, so a reader can trust-weight: a user-stated fact is not an AI-inferred guess.
- **Auditable.** Implementations log who queried what.

## Status & contributing

v1 is a draft, and the vocabulary deliberately starts **small and stable** — the point is for the community to extend it. Propose fields and types via issues and PRs: the core stays minimal, extensions live in the `extensions` namespace, and widely-adopted extensions graduate into later versions. See **[CONTRIBUTING.md](CONTRIBUTING.md)**.

The reference implementation, **[Memorandai](https://memorandai.com)** (a local-first knowledge studio), implements all three layers.

## License

[MIT](LICENSE) © 2026 Gary Smith.
