# BACKLOG — per-repo work orders

Grouped so each repo's independent to-do is scannable on its own. When working inside one
repo, you only need that repo's section. Check items off in the PR that ships them.

Status legend: `[ ]` not started · `[~]` in progress · `[x]` done.

---

## simple-memory  · `C:\Users\spamw\tools\simple-memory\server.js`  (Phase 1)

The whole Phase 1 spec lives here. Contained ~230-line file. **Shipped 2026-07-25** (v2;
local tool, not a GitHub repo — "shipped" = `server.js` updated, `server.js.bak` kept).

- [x] Embed-on-save: store the vector in each JSONL record at save time.
- [x] Recall embeds only the query (+ legacy vectorless records, backfilled once); cosine
  against stored vectors. Dropped the per-recall re-embed of the whole store.
- [x] Add `edit_memory(id, text)` — replace text, regen embedding, bump `modified`.
- [x] Add `delete_memory(id)` — soft delete (`deleted=true`); recall uses `active()` only.
- [x] `vacuum()` rewrites the JSONL without deleted rows (runs at startup when needed).
- [x] Extend record schema: `id, created, modified, text, embedding, importance,
  access_count, last_access, deleted`. Bump `access_count`/`last_access` on recall.
- [x] `JsonlStore` abstraction; `save/recall/edit/delete` call it, not `fs` directly.
- [x] Ranking = pure cosine. importance/recency NOT folded into rank.
- [x] Full MCP round-trip test (`scratchpad/mcptest2.js`) incl. edit/delete/bad-id — PASSED.
- [x] Empty-content guard kept; keyword-overlap fallback for endpoint outages kept.
- [ ] Field-verify the actual local model drives all four verbs in an LM Studio session.

## Lantern  · `github.com/SamuelJacksonGrim/Lantern`  (Phase 2 + own bugs)

Prerequisites to become a Simple Memory backend:
- [ ] File-backed SQLite: plumb the stubbed path; stop opening `:memory:`.
- [ ] Semantic/vector query path (current `/query` is exact `source_type` match only).

Field work (after prereqs):
- [ ] Background entity extraction (async maintenance pass, never blocks `save`).
- [ ] Relationship inference → edges.
- [ ] Activation propagation on recall.
- [ ] Provide a `Store` implementation Simple Memory can point at (same MCP API).

Pre-existing bugs (flagged in-repo, fix deliberately):
- [ ] **Bug 3** — `remember()` takes a `read()` lock for SQL writes; should be `write()`.
- [ ] **Bug 4** — `tauri.conf.json` missing the `build` section → `tauri build`/`dev` fail.
- [ ] `query_pattern()` builds SQL via string interpolation; parameterize if ever exposed
  to untrusted input.

## scraper-framework  · `github.com/SamuelJacksonGrim/scraper-framework`  (the ranker)

- [ ] Decide whether its scoring becomes *the* shared ranker, or stays domain-specific.
- [ ] If shared: expose a rich-to-rich scoring entry point (full query → full document).
- [ ] Build a **labelled set** from real scrape output (pre-declare PASS and FAIL
  signatures) before any semantic-scoring upgrade — the unmerged `claude/semantic-scoring`
  branch failed only on matching quality, which needs real data to tune.

## resonance-pruning-poc  · `github.com/SamuelJacksonGrim/resonance-pruning-poc`  (consistency, not memory)

- [ ] Evaluate `dissonance()` as a **consolidation admission gate**: does a new memory
  contradict held memory beyond a threshold? Prototype against Simple Memory's store.
- [ ] Keep it on the mind/PLE side — it is not a memory layer; the only tie is the gate.

## RFE-Core2  · `github.com/SamuelJacksonGrim/RFE-Core2`  (the consumer)

- [ ] Confirm `sovereign_manifold` hydration path against whichever store wins (JSONL →
  Lantern). It already targets Lantern `:3002`; keep `witness_state.json` as durable
  warm-start substrate.
- [ ] Consolidation (Pascal) guardrail enforced: reflections must trace to sources; never
  summarize summaries without re-reading.

## .claude memory (Windows host)  · `C:\Users\spamw\.claude\...\memory\`

- [ ] This is my own cross-session memory — separate corpus from the local model's
  `simple-memory`. Keep tiers (milestone/standing/episodic); weight governs retention,
  not rank; `recall.ps1 -Rebuild` after edits.

---

## Cross-repo chore — the pointers

Add a one-line pointer in each repo's `CLAUDE.md` back to this hub so the cross-repo plan
is discoverable from inside any single repo:

- [ ] Lantern `CLAUDE.md`
- [ ] scraper-framework `CLAUDE.md`  *(or README)*
- [ ] resonance-pruning-poc `README.md`
- [ ] RFE-Core2 `global-claude/CLAUDE.md`
- [ ] simple-memory — add a short `README.md` with the pointer
