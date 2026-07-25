# ROADMAP — Resonance memory stack

North star: **one store, one retrieval contract, one decay/consolidation policy.** Each
repo owns one horizontal layer instead of re-implementing the whole vertical. The public
interface stays four verbs forever (`save`/`recall`/`edit`/`delete`); sophistication grows
in the substrate, never in the model's cognitive load.

Status legend: `[ ]` not started · `[~]` in progress · `[x]` done.

---

## Phase 1 — Simple Memory (the tiny, un-misusable MCP server)

Goal: a memory server GPT-OSS-20B-class models cannot misuse, that scales past a handful
of memories. Contained to `C:\Users\spamw\tools\simple-memory\server.js` (~200 lines).

- [ ] **Embed-on-save.** Embed once at save-time, store the vector in the record. Recall
  embeds only the query, then cosine against stored vectors. (Today it re-embeds every
  memory on every recall — O(n) embed calls per query.) Pattern already proven by
  `recall.ps1 -Rebuild` and its `memory-embeddings.json` cache.
- [ ] **`edit_memory(id, text)`** — replace text, regenerate embedding, update `modified`.
- [ ] **`delete_memory(id)`** — soft delete (`deleted=true`) + periodic `vacuum()`. Stable
  IDs, recoverable.
- [ ] **Server-owned metadata**, never exposed to the model: `id, created, modified, text,
  embedding, importance, access_count, last_access, deleted`.
- [ ] **Abstract `Store` interface** behind the four verbs (verbs call `Store`, not
  `fs.appendFileSync` directly). JSONL is the first implementation; Lantern is the second.
- [ ] **Ranking = cosine only.** Keep `importance/access_count/durability` as stored
  substrate metadata governing *retention*, never *retrieval order* — until a labelled-set
  A/B proves blending helps. (Measured: adding a durability weight to cosine inverted
  correct rankings — see the ranking constraint below.)

**Acceptance:** MCP round-trip (initialize / tools-list / save / recall / edit / delete /
empty-guard) passes; recall issues exactly one embedding call regardless of store size;
ranking is pure cosine; a small local model uses it end-to-end without touching internals.

---

## Phase 2 — Lantern (the associative memory field)

Goal: the hypergraph substrate behind the *same* MCP API. This is where "infer the
neighborhood" lives — the hard step Phase 1 deliberately does not attempt.

**Prerequisites before Lantern can back Simple Memory:**
- [ ] **File-backed SQLite.** Today `Hypergraph::ignite()` opens `:memory:` → forgets on
  restart. Plumb the file path (currently stubbed).
- [ ] **Semantic / vector query path.** Today `/query` is `query_by_source_type` (exact
  type match). A store backing Simple Memory needs vector recall.

**Then the field:**
- [ ] Entity extraction (hybrid: store raw immediately, enrich edges in a background
  maintenance pass — never block `save_memory()` on extraction).
- [ ] Relationship inference → hypergraph edges.
- [ ] Activation propagation / neighborhood expansion on recall.
- [ ] Swap Simple Memory's `Store` to a Lantern-backed implementation. No MCP API change.

**Acceptance:** Simple Memory runs unchanged against a Lantern backend; recall returns a
weighted neighborhood, not just a flat list; extraction runs off the hot path.

---

## Cross-cutting (spans multiple repos)

- [ ] **One ranker, fed rich-to-rich.** Whatever becomes "the ranker" compares full query
  → full document. Short-probe → long-text is the failing shape (measured).
- [ ] **Consolidation policy (Pascal).** N-shards → reflection. Guardrail: **never
  summarize a summary without re-reading its sources** (iterative drift is a live bug in
  Pascal). Synthesized concepts must trace back to their supporting memories.
- [ ] **Contradiction gate (pruning-poc → memory).** Evaluate pruning-poc's `dissonance`
  metric as the gate deciding what a new memory is allowed to consolidate against held
  memory. Belongs to the mind/PLE side; the tie to memory is consolidation admission only.
- [ ] **Per-repo `CLAUDE.md` pointers** back to this hub (see BACKLOG).

---

## The ranking constraint (measured, do not re-litigate)

- Bare keyword → title does **not** separate (flat 0.30–0.45 band).
- Tier-exemplar phrase → content: worst-true 0.467 vs best-false 0.477 → **not separable**.
- Full query sentence → full document (`recall.ps1`): **works**, spread ≈0.13.
- Therefore: feed any ranker rich-to-rich. And weight = lifespan/retention, **not** rank.
