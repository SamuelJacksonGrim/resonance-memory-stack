# resonance-memory-stack

**The portable cross-repo work-order for the Resonance memory stack.**

This repo owns no code. It is a coordination artifact: one place that says *what needs
to happen in each repo*, so the plan does not evaporate across sessions, models, or the
gap between the Claude Code CLI and the claude.ai app.

## Why this exists

Memory has been implemented as thin vertical slices across several repos. Each one
re-solved store + rank + decay to get its own job done, so they overlap on the hard parts
and none is complete alone. The fix is **horizontal**: one store, one retrieval contract,
one decay/consolidation policy — each repo owning exactly one layer. That plan spans
5+ repos, which is exactly the kind of thing that gets lost. Hence this hub.

## How to use it

- **From the claude.ai app:** open this repo directly. It's readable regardless of which
  other repo is in context.
- **From the CLI:** clone it alongside whatever you're working in.
- **Each participating repo** carries a one-line pointer in its `CLAUDE.md` back to this
  repo. Do **not** vendor this in as a git submodule — decoupled reference only; submodule
  friction (detached HEAD, `--recursive`, drift) buys nothing for markdown.

Keep `ROADMAP.md` (phased north-star) and `BACKLOG.md` (per-repo checkboxes) updated as
work lands. When an item ships, check it off here in the same PR that ships it.

## The layered map — who owns what

| Layer | Repo(s) |
|---|---|
| Write / capture | Lantern daemon + HTTP `:3002`, `simple-memory` MCP `save_memory` |
| Store | Lantern SQLite hypergraph, `simple-memory.jsonl`, `.claude/memory/*.md`, RFE `witness_state.json` |
| Rank / retrieve | scraper-framework scoring ("hippocampus"), `recall.ps1` cosine, Lantern edge-weight |
| Decay / consolidate | `.claude` tiers, Lantern `weight += 0.3`, `pascal_synthesizer` |
| Consistency / self-correction *(mind, NOT memory)* | resonance-pruning-poc |
| Consumer / mind | RFE-Core2 `sovereign_manifold` |
| **Out of domain** | nexus-block (battery BMS — shares the "resonance" brand, nothing else) |

Two corrections verified by reading the code, not the names:
- **resonance-pruning-poc** is a *reasoning-chain stabilizer* (dissonance = deviation from
  conserved invariants), not memory decay. Its dissonance metric is the same shape as a
  contradiction score — a candidate gate for what memory is allowed to *consolidate*.
- **nexus-block** is zinc-ion battery firmware. Not part of this stack. Do not wire it in.

## The core design principle

**The graph is the substrate, not the interface.** The model never manipulates memory
internals (no entities/observations/relations/edges). It knows only cognitive verbs:
`save`, `recall`, `edit`, `delete`. Everything else — embeddings, importance, decay,
graph enrichment — happens below cognition, server-side. This is why a two-verb server
succeeded where a knowledge-graph API failed a small local model.

## Phase split

- **Phase 1 — Simple Memory.** A tiny MCP server a small model cannot misuse. Flat store,
  embed-on-save, pure-cosine recall, four verbs. No graph, no extraction, no inference.
- **Phase 2 — Lantern.** The hypergraph substrate: entity extraction, relationship
  inference, activation propagation — the associative "memory field." Same MCP API,
  different backend. Simple Memory is designed so Lantern can replace its store without
  changing the interface.

See `ROADMAP.md` for the sequenced plan and `BACKLOG.md` for the actionable items.
