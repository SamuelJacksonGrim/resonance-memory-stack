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
- [x] Field-verified: `openai/gpt-oss-20b` drove all four verbs across fresh conversations
  (5/5, recalled-before-edit, well-formed ids) — 2026-07-25. **Phase 1 done.**

Phase 2a — associative field (Node prototype, in `field.js`):
- [x] kNN association graph from stored vectors (rich-to-rich); `neighborhood()` expansion.
- [x] Wired into recall as opt-in `SIMPLE_MEMORY_FIELD=1`; Phase 1 output unchanged when off.
- [x] Measured meaningful: kNN edge precision k=2 = 0.900 vs chance 0.200 (2026-07-25).

Phase 2b — Hebbian reshaping (`ledger.js`), built + proven 2026-07-25:
- [x] Co-activation reinforcement, provenance-discounted (0.1 / 0.02 / 0).
- [x] Bounded tanh blend (`cosine + 0.3·tanh(weight)`); canonical sorted edge key.
- [x] Decay (`*0.95` every 10 recalls) + prune below 0.05.
- [x] Deterministic sim 10/10 (`scratchpad/simulate_2b.js`); sidecar save/load round-trips.
- [ ] Validate on the REAL memory corpus (not just the labelled set), then flip the field
  on by default (drop the `SIMPLE_MEMORY_FIELD` gate). **Acceptance test** — run
  `inspect_sidecar.js` after a few real field-on sessions; all four must hold:
  1. **Breathing** — edge count rises within a session, then falls as recalls tick the
     decay (`*0.95`/10 recalls). Red flag: count climbs day over day, never falls → the
     0.95 decay or 0.05 floor is too weak for real conversation drift (hairball).
     *Note: decay is recall-driven, not wall-clock — it only shrinks as you keep recalling.*
  2. **Semantic leaps** — inspect edges with HIGH learned weight but LOW base cosine
     (< 0.55). The two joined memories should be an intuitive link, not filler that
     co-fired. Red flag: threshold-crossers are generic → `0.3·tanh` cap too generous.
  3. **Provenance dominance** — top-weight edges are core, repeatedly-typed projects.
     Red flag: high weight between obscure pairs you didn't type recently → the 0.02
     neighborhood discount is too weak; the graph is validating its own noise.
  4. **Hot-path** — no perceptible stutter from the 10th-recall `writeFileSync`. Low risk
     (decay keeps the file tiny); fix = debounce/async if it ever bites.
  Tool: `inspect_sidecar.js` (dependency-free) joins edge ids → memory text and recomputes
  base cosine so leaps/provenance are actually judgeable. Local working copy lives in
  `C:\Users\spamw\tools\simple-memory\`; archival copy in `scripts/` here.

Zero-terminal UX (`panel.js`, `config.json`, launchers) — built + tested 2026-07-25:
- [x] Field controlled by shared `config.json`, read per recall (live toggle, no restart).
- [x] Local browser control panel with a real toggle switch; double-click launchers.
- [x] Bundled single executable via Node SEA (`build-exe.js` → `memory.exe`, 88 MB, no Node
  needed). Both modes tested: `--mcp` server + double-click panel. Unsigned (SmartScreen).
- [x] Auto-installer (`install.js`): detects LM Studio + Claude Desktop, wires
  `memory.exe --mcp`, preserves other servers, `.bak` backup. Panel Connect/Disconnect
  buttons + CLI `--install`/`--uninstall`. Tested against temp configs.
- [ ] Build the Mac binary on a Mac; package as a (public?) repo with exe release assets.
- [ ] Write the plain-language README ("what this does" / "how it works") for non-coders.
- [ ] Client-neutral default data dir (not `~/.lmstudio`).

Proactive save/recall nudge — the "model forgets to press the button" fix (done 2026-07-26):
- [x] Rewrote all four tool `description`s to encode WHEN, not just what: save fires
  proactively on durable facts (with anti-spam guards — no small talk/secrets, prefer
  edit over duplicate); recall fires at conversation start + on back-references; edit/
  delete carry "recall first to get the id". Fed to the model every turn, zero user setup.
- [x] `system-prompt.md` — optional copy-paste block for the client's system prompt, for
  weaker models that still under-call tools. Stronger lever, opt-in.
- [x] Rebuilt `memory.exe`; confirmed the exe serves the new descriptions in `--mcp` mode.
- [ ] Surface the `system-prompt.md` snippet in the panel with a one-click Copy button
  (pairs with the plain-language README pass).

## Lantern  · `github.com/SamuelJacksonGrim/Lantern`  (Phase 2c — only after 2a/2b prove out)

Course correction (2026-07-25): Lantern's crate has **no embeddings/vector search** and its
daemon won't build here (webkit2gtk); simple-memory already has persistence + vector recall.
So Lantern becomes the field's substrate *after* the field design is proven in Node — not a
like-for-like store swap. Prereqs when we get there:
- [ ] Install Rust toolchain (none on this box).
- [ ] File-backed SQLite: plumb the stubbed path; stop opening `:memory:`.
- [ ] Add embeddings + cosine vector recall + the proven edge model to `lantern_memory`.
- [ ] Tauri-free HTTP service; a Node `LanternStore` (same surface as `JsonlStore`), env-swap.

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
