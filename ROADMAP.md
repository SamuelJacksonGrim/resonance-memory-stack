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

- [x] **Embed-on-save.** Embed once at save-time, store the vector in the record. Recall
  embeds only the query (+ any legacy vectorless records, backfilled once), then cosine
  against stored vectors. Pattern proven by `recall.ps1 -Rebuild` / `memory-embeddings.json`.
- [x] **`edit_memory(id, text)`** — replace text, regenerate embedding, update `modified`.
- [x] **`delete_memory(id)`** — soft delete (`deleted=true`) + `vacuum()` (runs at startup
  when deleted rows exist). Stable IDs, recoverable.
- [x] **Server-owned metadata**, never exposed to the model: `id, created, modified, text,
  embedding, importance, access_count, last_access, deleted`.
- [x] **Abstract `Store` interface** (`JsonlStore`) behind the four verbs — verbs call the
  store, not `fs` directly. JSONL is the first implementation; Lantern is the second.
- [x] **Ranking = cosine only.** `importance/access_count` are stored and bumped on recall
  for *retention*, never folded into *retrieval order* — until a labelled-set A/B proves it.

**Acceptance:** MCP round-trip (initialize / tools-list / save / recall / edit / delete /
empty-guard / bad-id) — **PASSED 2026-07-25** (`scratchpad/mcptest2.js`, endpoint up, vector
stored, pure-cosine recall correct). **Field test PASSED 2026-07-25** — `openai/gpt-oss-20b`
drove all four verbs end-to-end across fresh conversations (true cross-session), 5/5 verbs,
recalled-before-edit to fetch the id, edit/delete ids well-formed. The model that failed the
knowledge-graph server drives the four-verb server flawlessly. **Phase 1 complete.**

---

## Phase 2 — Lantern (the associative memory field)

Goal: the associative memory field — recall a *region*, not a record. This is where
"infer the neighborhood" lives.

**Course correction (2026-07-25), after reading Lantern's crate.** Two facts reshaped
this phase: (1) Lantern's `lantern_memory` crate is an edge-weight hypergraph with *zero*
embeddings / vector search, and its daemon won't build on this box (webkit2gtk); (2)
`simple-memory` already has the two listed "prerequisites" (JSONL is persistent; it does
embed-on-save + cosine recall). So a like-for-like port to Rust-Lantern would be *strictly
worse* — reimplementing working Node minus any new capability. Lantern's only real add is
the field itself. Decision: **prototype the field in Node first, prove its value, then port
the proven design to Rust-Lantern.** (Samuel's call, 2026-07-25.)

**Key enabling insight:** association between two *stored memories* is a full-text ↔
full-text comparison — the RICH-TO-RICH regime the finding proved works — so the semantic
association graph is buildable from the vectors we already store, with **no LLM extraction
and no new embed calls.**

### Phase 2a — Node associative field  ✅ built + measured (2026-07-25)
- [x] `field.js`: per-node kNN association edges from stored vectors (rich-to-rich cosine).
- [x] `neighborhood()`: expand from cosine seeds to related memories.
- [x] Wired into `recall_memory` as an opt-in additive layer (`SIMPLE_MEMORY_FIELD=1`);
  primary cosine results are byte-identical to Phase 1 (regression test still passes).
- [x] **Measured** on a labelled clustered set (real nomic embeddings): within-cluster
  cosine 0.596 vs cross 0.462 (gap 0.134); kNN edge precision k=1 **0.933**, k=2 **0.900**,
  k=3 0.756 vs chance **0.200**; example neighborhoods stayed on-topic. Edges are real.

### Phase 2b — reshaping with use  ✅ built + proven (2026-07-25)
Hebbian sidecar (`ledger.js`), separate from the vector store. Design hardened across a
Claude/Gemini review; three corrections baked in:
- [x] **Co-activation reinforcement**, provenance-discounted: primary↔primary `α=0.1`,
  primary↔neighborhood `α=0.02`, neighborhood↔neighborhood `0` — so the graph learns from
  the user's queries, not from its own suggestions (breaks the self-reinforcing loop).
- [x] **Bounded blend** (not unbounded add): neighborhood score = `cosine + 0.3·tanh(weight)`.
  Frequency can lift a weak edge over the 0.55 gate but can never swamp the semantic floor.
- [x] **Canonical undirected edge key** (ids sorted) so `a:b` and `b:a` are one edge.
- [x] **Decay + prune**: every 10th recall, `weight *= 0.95`; drop below 0.05.
- [x] Wired into recall behind `SIMPLE_MEMORY_FIELD=1`; primary path still byte-identical
  (Phase 1 regression still passes). Reinforcement never touches query ranking.
- [x] **Deterministic proof** (`scratchpad/simulate_2b.js`, 10/10): climb == 5·α; neigh↔neigh
  stays 0; unreinforced edge prunes at exactly n=14 epochs; a 0.40-cosine pair enters the
  neighborhood after exactly 6 co-activations (score 0.561); bonus bounded < 0.3. Sidecar
  save/load round-trips.

Deferred: concept/entity enrichment (the harder extraction layer) — only if it beats the
pure-vector field on real data. Not needed for the field to work.

### Phase 2c — port to Rust-Lantern  (only once 2a/2b prove out)
- [ ] Install a Rust toolchain; add embeddings + file-backed SQLite + cosine + the proven
  edge model to `lantern_memory`; expose a Tauri-free HTTP service.
- [ ] A Node `LanternStore` implementing the same surface as `JsonlStore`; swap via env,
  no MCP API change. Then the field lives in the substrate Lantern was designed to be.

**Acceptance:** recall returns a coherent weighted neighborhood, not just a flat list, with
the field off-by-default-safe; the reshaping is measurable; and (2c) Simple Memory runs
unchanged against a Lantern backend.

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
