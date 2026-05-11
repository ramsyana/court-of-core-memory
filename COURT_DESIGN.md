# COURT_DESIGN

This document extends and refines the Court specification in README.md.  
Where they conflict, COURT_DESIGN.md takes precedence for implementation decisions.

## Court of Core Memory – Hybrid Design

The Court is a **deterministic rule engine with narrow, auditable LLM subroutines**.  
It never asserts epistemic validity that the rules do not mechanically allow.

### 1. Hybrid Split (Rule Engine + LLM)

| Step                  | Rule Engine (deterministic governor)                          | LLM (narrow, auditable subroutine)                              | Guardrails / Failure Mode                                      |
|-----------------------|---------------------------------------------------------------|-----------------------------------------------------------------|----------------------------------------------------------------|
| Completeness check    | Enforces mandatory 5W1H+2 fields + Node Kind invariants     | —                                                               | Hard reject if missing                                         |
| Node Kind assignment  | Validates final kind against kind rules                       | Proposes initial kind + short rationale from raw text          | Court ratifies or overrides                                    |
| Contradiction scan    | Structural conflicts only (same entity + time window + conflicting State Fact; graph distance; edge-type rules; temporal ordering) | Conflict likelihood + nuance detection **only after** structural filter | Tunable conflict-likelihood bands (defaults determined by empirical calibration against a test corpus). Every score + full prompt + raw output logged as Observed-Internal. The conflict-likelihood output is a contradiction-local routing signal only; it is not the node's certified Confidence field and must not overwrite Stage-0-anchored epistemic confidence. |
| Evidence Type & Validation Basis | Applies lookup table + anchors to Stage 0                     | Drafts natural-language justification                           | Court verifies draft matches Stage 0 evidence; rejects if drift |
| Hypothesis synthesis (Reflection) | —                                                             | (handled in Reflection layer)                                   | —                                                              |

### 2. Observed-Internal Logging (mandatory for every LLM sub-call)
Every Court-internal LLM call produces an immutable Stage 0 entry:
- `kind`: `Observed-Internal`
- `source`: `court-subroutine`
- `subroutine_name`: `node-kind-proposal | conflict-likelihood | evidence-justification | …`
- `full_prompt`: verbatim prompt (never hashed)
- `raw_output`
- `rule_verification_result`: `pass | override | pending-human-review`
- `confidence`: (conflict scan only; contradiction-local score, not node Confidence)

### 3. Reflection Rule Layer (runs before nomination)
1. Structural hygiene – ≥1 anchor to Certified node or Stage 0 evidence  
2. Novelty check – **no node with identical Kind + primary entity + overlapping time window already exists as Certified or Candidate**  
3. Abstraction level – must list children if promoting Higher-Order Concept  
4. Epistemic modesty – Validation Basis must be `ProposedSynthesis`
5. MAGMA-style consolidation outputs are never anchors by themselves. A proposal may enter as `Candidate` only when explicitly anchored to Stage 0 evidence or existing Certified nodes; otherwise it must enter as `Hypothesis`.

### 4. Reconciler Loop (library mode)
- Event-driven off Stage 0 appends with configurable debounce (`batch_window_ms`, default 500 ms)  
- Court-side nomination queue depth limit (default 20) prevents flooding  
- `court.run_cycle()` manual escape hatch  
- Optional `start_background_reconciler()`

### 5. Contested State with Human Review
- No new state is introduced.  
- `Contested` nodes carry an optional flag `human_review_required: bool`.  
- Set when contradiction-band policy requires explicit human escalation.  
- Resolved via `court.resolve_contested(node_id, decision)` where `decision` is `Literal["certify", "supersede", "defer"]`.
- `court.resolve_contested(...)` applies to both `Contested` and `Conflicted` nodes despite the historical method name.
- Every human resolution must log `resolver`, `decision`, `reason`, and the affected node IDs to Stage 0.

### 6. Out of Scope for v1
- Concurrent task handling and task-level isolation  
- Multi-agent shared Court service (single-agent library first)  
- Behavioral stability / convergence guarantees  
- Goal activation and motivation semantics  

This document is the single source of truth for Court implementation. All code must comply with the table, logging contract, Reflection rules, reconciler API, and v1 scope boundaries above.

### 7. v1.1 MAGMA Integration (Architecture Only)
- See `INTEGRATIONS.md` for the alignment table and integration rationale.
- Confirmed mappings: MAGMA Entity/Semantic/Temporal/Causal graph patterns map to Court's Entity/Semantic/Temporal/Causal projections as a constrained subset only. Court retains first-class Spatial, Procedural, Implication, and Resources projections, plus bi-temporal validity and Court-governed lifecycle semantics.
- New optional components: a Retrieval-layer Intent-Aware Router may classify query intent and choose traversal weights; Stage-0-adjacent segmentation and indexing may support low-latency evidence recall; asynchronous consolidation may propose candidate links or abstractions. None of these components may certify facts, assign Validation Basis, change node state, or write directly into the Persistent Core.
- Updated out-of-scope list: persisted multi-graph storage design, distributed or shared Court topologies, benchmark claims, direct-write consolidation, new memory states, and any router that bypasses Court retrieval penalties or evidence boundaries remain out of scope for v1.1.
- Conflict-likelihood bands: no change in v1.1. Existing contradiction-scan banding and human-review behavior remain the governing mechanism.
- Reflection rules: no weakening in v1.1. Reflection may consume MAGMA-style retrieval hints or consolidation proposals, but every output still follows the existing nomination rules, structural hygiene, novelty checks, and epistemic modesty constraints before Court review.
- State machine: no change in v1.1. No new states or transition authorities are introduced; Court remains the sole authoritative governor of all state transitions.
- Retrieval-layer scores, router weights, segmentation metadata, and consolidation hints are advisory only. They may influence retrieval order or nomination proposals, but they do not count as evidence anchors and cannot modify certified Confidence or node state.
