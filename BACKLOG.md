# BACKLOG — per-repo work orders

Grouped so each repo's independent to-do is scannable on its own. When working inside one
repo, you only need that repo's section. Check items off in the PR that ships them.

Status legend: `[ ]` not started · `[~]` in progress · `[x]` done.

---

## simple-memory  · `C:\Users\spamw\tools\simple-memory\server.js`  (Phase 1)

The whole Phase 1 spec lives here. Contained ~200-line file.

- [ ] Embed-on-save: store the vector in each JSONL record at save time.
- [ ] Recall embeds only the query; cosine against stored vectors (drop the per-recall
  re-embed of the whole store).
- [ ] Add `edit_memory(id, text)` — replace text, regen embedding, bump `modified`.
- [ ] Add `delete_memory(id)` — soft delete (`deleted=true`); skip deleted on recall.
- [ ] Add periodic `vacuum()` to rewrite the JSONL without deleted rows.
- [ ] Extend record schema: `id, created, modified, text, embedding, importance,
  access_count, last_access, deleted`. Bump `access_count`/`last_access` on recall.
- [ ] Introduce a `Store` abstraction; `save/recall/edit/delete` call it, not `fs` directly.
- [ ] Keep ranking = pure cosine. Do NOT fold importance/recency into rank yet.
- [ ] Re-run the full MCP round-trip test (`scratchpad\mcptest.js` pattern) incl. edit/delete.
- [ ] Keep the empty-content guard. Keep the keyword-overlap fallback for endpoint outages.

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
