# INVARIANTS — what must never change

This file protects architecture; it does not invent it. The implementation *will* evolve —
new models, embedders, maintenance passes, graph algorithms. These invariants are what keep
it from drifting every time one of those changes. When a proposed change violates one of
these, the change is wrong, not the invariant — unless the invariant is retired here first,
in writing, with the reason.

Status: **ratified 2026-07-26** (Claude · GPT · Gemini, three review rounds). Numbered
invariants are locked; unproven ideas are parked at the bottom as hypotheses, explicitly
*not* invariants. Retiring or amending a numbered invariant requires an entry here with the
reason.

Every invariant answers one question: **what bug does this prevent?**

---

## The two roots

The whole architecture reduces to two principles. Every numbered invariant is a consequence
of one of them.

**Root 1 — structural (holds a priori).**
> The substrate is an append-only, replayable, deterministic memory system whose derived
> state may evolve indefinitely while its public cognitive interface never becomes more
> demanding.

**Root 2 — empirical (measured, not derived).**
> Relevance and retention are different quantities. Neither may do the other's job.

Root 2 cannot be deduced from Root 1. We learned it from data: folding a durability weight
into the ranking inverted correct rankings (2026-07-23 — `koneko-vitals-tool` cos 0.6254 lost
to a theory file cos 0.5749 purely on a 0.55-vs-0.90 tag). It is an axiom precisely because it
was measured. Do not promote it to "a consequence of Root 1"; keep it labelled as earned.

---

## The layers

Only one thing is truth. Everything else is a *materialized view* of it — deterministically
derived, safe to delete, rebuildable.

```
Layer 0  Event Log        append-only: save / recall / edit / delete   ← THE ONLY TRUTH
Layer 1  Knowledge        "what exists"  — semantic graph (materialized from the save stream)
Layer 2  Dynamics         "what is active" — Hebbian weights (materialized from the recall stream)
```

- Even "the set of memories that currently exist" is a materialization: apply the log's
  saves, edits, and deletes in order. Raw episodic text is never mutated; `edit` and `delete`
  are *new events* (supersede / tombstone), classic event sourcing.
- **Rebuild order is fixed:** Log → Knowledge → Dynamics. Dynamics depends on Knowledge's
  topology (co-activation reinforces edges the neighborhood surfaced), so Knowledge rebuilds
  first, then the recall stream replays on top of it.

---

## The invariants

### 1. Monotonic Interface
The public API is `save` / `recall` / `edit` / `delete`. It may become *simpler*, never more
cognitively demanding. New capability appears by making an existing verb smarter, never by
adding a verb or a required argument. **Prevents:** API growth, model cognitive overload,
model-specific prompting. *(Root 1.)*

### 2. Interface targets a substrate, not an implementation
The four verbs sit in front of a `Store` abstraction. JSONL today, Lantern (hypergraph) later,
a pure vector index or a hybrid tomorrow — the protocol must never couple to one backend's
internals. **Prevents:** protocol lock-in to a single implementation. *(Root 1.)*

### 3. The append-only event log is the single source of truth
`save` / `recall` / `edit` / `delete` are all logged events, with raw text preserved verbatim.
The log is the only authoritative state. **Prevents:** unrecoverable state, and — critically —
the loss of use-learned structure (Hebbian dynamics live in the *recall* stream, not in the
text; without logging recalls, "fire together, wire together" is unreconstructable). *(Root 1.)*

**Recall-event schema (lightweight).** A recall event stores only what Dynamics replay needs,
never the full query text:
`{ ts, query_hash?, primary_ids[], neighborhood_ids[] }`.
The **primary/neighborhood split is required** — the Hebbian rule is provenance-discounted
(primary-primary 0.1, primary-neighborhood 0.02, neighborhood-neighborhood 0), so a flat
`surfaced_ids[]` would erase the very distinction replay depends on. `query_hash` is optional
(dedup / analytics, not replay). A `selected_ids` field — which surfaced memories the model
*actually used* — would be a stronger reinforcement signal, but we don't capture it today (the
server returns text and never learns what was used); parked until that feedback channel exists.

**Bounding log volume — checkpoint + archive.** Hundreds of recalls/hour will grow Layer 0.
Treat it like a write-ahead log: periodically checkpoint the materialized Knowledge + Dynamics,
keep only post-checkpoint events on the hot replay path, and **archive** older events (never
delete — invariant 10). Full-history rebuild from t=0 stays possible from the archive; routine
rebuilds replay from the last checkpoint. Checkpoints are optimizations; the archived log is
truth.

### 4. Replayability
Any derived substrate state must be reproducible from the event log and deterministic
algorithms. Delete Knowledge and Dynamics, replay the log, get the same state back (or the
intended new state, if an algorithm changed). **Prevents:** migration failure, irreversible
corruption, algorithm lock-in. Replay is the escape hatch for *every* future improvement —
`maintenance_v2`, a better graph algorithm, a better Hebbian rule, even a new embedder (a
re-embed is a full rebuild of Layers 1–2 under the new model; vectors are never mixed across
embedder versions). *(Root 1.)*

### 5. Substrate purity
The substrate may contain only three kinds of thing: (a) logged episodic events, (b) the
output of deterministic computation over them, (c) reproducible synthesis that carries its
provenance. No model-generated metadata, no model opinions, no speculative relationship
becoming permanent. **Prevents:** hallucinated metadata, model opinions leaking into truth,
unsupported knowledge. *(Root 1.)*

### 6. Three layers, cleanly separated: Knowledge ≠ Dynamics
"What exists" (semantic graph, a function of saved text + embedder) is a different subsystem
from "what is active" (Hebbian reinforcement, a function of recall history). They are
materialized from different projections of the log and must not be fused. **Prevents:** coupling
storage to activation; losing the ability to swap either algorithm independently. *(Root 1.)*

### 7. Flattened, recursive provenance
Every synthesized memory points at the ground-truth episodic events it derives from — the
*flattened* set, never at another synthesis. If B is built from A + episodes 9–14, B records
1–14, not "A". **Prevents:** semantic drift, provenance-tree drift, unsupported synthesis, and
makes "explain why you believe this" always answerable. *(Root 1; serves the anti-drift goal.)*

### 8. Retrieval vs Health are separate systems — THE MEASURED AXIOM
Retrieval ranks by relevance (cosine) **only**. Retention/health — `importance`, `access_count`,
decay, consolidation, pruning — governs what stays in the corpus and how it evolves, and
**never** touches ranking. **Prevents:** ranking pollution, maintenance silently influencing
relevance, hidden feedback loops. *(Root 2 — measured, not derived. This is the one invariant
you cannot argue your way into; it had to be observed.)*

**The Hebbian boundary (discovery, not ordering).** Layer 2 (Hebbian dynamics) may decide
*what enters the candidate/context set* — expanding the horizon to co-recalled neighbors, and
lifting a low-cosine pair over the edge gate (a "semantic leap"). It must **never** reorder the
primary result by relevance. The primary list is ordered by pure cosine; the associative
neighborhood is presented as *context*, explicitly not as a relevance ranking. This is already
how the code behaves: `server.js` computes the primary `ranked` list from cosine *before* any
field logic, and `field.js`'s tanh bonus acts only on edge formation and neighborhood gating.
Measured (Phase 2b): a 0.40-cosine pair enters the neighborhood after 6 co-activations — it
becomes *reachable*, it does not outrank a direct hit. (Rejected: re-ranking the primary result
with a Hebbian multiplier — that would fold a dynamics signal into relevance order and reopen
exactly the pollution Root 2 forbids.)

### 9. Maintenance is a background, transactional, replayable pass
`save` stays O(embed): store, embed, done. All expensive work — extract, merge, cluster,
consolidate, repair — happens later in a `maintenance()` pass that builds a *new* materialization
and swaps it in atomically. Never half-mutated: either the old Knowledge/Dynamics layer or the
new one, never half of both. **Prevents:** latency on the hot path, a half-mutated substrate,
non-reproducible maintenance. *(Root 1.)*

### 10. Regret test — archive, never auto-destroy
Every irreversible operation asks: *could we reasonably regret this later?* If yes, archive
instead of destroy — storage is cheap, lost information isn't. Automatic operations never
destroy. User-driven `edit`/`delete` is honored (explicit intent) but remains recoverable via
the log's tombstone; `vacuum()` archives rows, it does not drop them. **Prevents:** irreversible
loss of information that later proves valuable. *(Root 1 — append-only makes this automatic.)*

### 11. Represent contradictions, don't overwrite
When new evidence conflicts with held memory, store both, linked as contradicting — the
substrate represents uncertainty rather than erasing history. Express confidence as **evidence
counts** (which events support this), never as invented confidence floats. **Prevents:** erased
history, false precision, premature collapse of genuine ambiguity. *(Root 1 + Root 2-adjacent.)*

### 12. Determinism relative to a fixed embedder
The chat model (GPT / Claude / Llama / …) is **never** an input to substrate state — it only
passes text. The embedder *is* a declared parameter of materialization: identical text under a
fixed embedder yields identical vectors; changing the embedder triggers a full Layer 1–2 rebuild
(invariant 4). **Prevents:** a model-dependent substrate, and silent mixing of incompatible
vector spaces. *(Root 1.)*

---

## The defining acceptance test

One test subsumes most of the above:

> Delete Knowledge and Dynamics entirely. Replay the event log through the current algorithms
> and the declared embedder. If every capability except raw episodic recall rebuilds
> deterministically, the substrate is pure. If anything cannot be rebuilt, something impure has
> leaked in — find it.

A substrate that passes this is self-healing: intelligence lives in the algorithms, not in
fragile state.

---

## Health metric (not an invariant — an instrument)

**Practical entropy.** Ask each maintenance cycle: is the materialized layer getting *easier or
harder to explain* over time? Track duplicate concepts, contradictory syntheses, and mean
provenance-chain length. Rising = maintenance is accumulating entropy and regressing. This is
the quantitative form of "is maintenance helping or hurting?"

---

## Hypotheses — parked, NOT invariants (pending measurement)

These are attractive and unproven. They do not go in the numbered list until data earns them.

- **Semantic typing** (facts / preferences / beliefs / projects / relationships / skills /
  goals as internal types). Collides with invariants 5 and 12 unless the types are assigned by a
  *deterministic maintenance pass*, never by the chat model. And unproven: we measured that pure
  cosine neighborhoods work; type-partitioning could just as easily over-fragment and hurt.
- **Crystallization** (repeated similar episodes causing a stable semantic structure to emerge).
  Admissible only if a crystal forms from deterministic clustering of episodic evidence, never
  from an LLM summary. Direction, not a thing to build now.

---

## Standing discipline

The invariants are cheap to write down — they constrain future builds, they are not builds. The
builds still wait on evidence. As of ratification the associative field is **not yet validated on
the real corpus** (open item in BACKLOG with a four-part acceptance test). Write the philosophy
down to protect it; measure before pouring the foundation it describes.
