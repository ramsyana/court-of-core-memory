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
| Contradiction scan    | Structural conflicts only (same entity + time window + conflicting State Fact; graph distance; edge-type rules; temporal ordering) | Semantic similarity + nuance detection **only after** structural filter | Tunable confidence bands (defaults determined by empirical calibration against a test corpus). Every score + full prompt + raw output logged as Observed-Internal. |
| Evidence Type & Validation Basis | Applies lookup table + anchors to Stage 0                     | Drafts natural-language justification                           | Court verifies draft matches Stage 0 evidence; rejects if drift |
| Hypothesis synthesis (Reflection) | —                                                             | (handled in Reflection layer)                                   | —                                                              |

### 2. Observed-Internal Logging (mandatory for every LLM sub-call)
Every Court-internal LLM call produces an immutable Stage 0 entry:
- `kind`: `Observed-Internal`
- `source`: `court-subroutine`
- `subroutine_name`: `node-kind-proposal | semantic-similarity | evidence-justification | …`
- `full_prompt`: verbatim prompt (never hashed)
- `raw_output`
- `rule_verification_result`: `pass | override | pending-human-review`
- `confidence`: (semantic scan only)

### 3. Reflection Rule Layer (runs before nomination)
1. Structural hygiene – ≥1 anchor to Certified node or Stage 0 evidence  
2. Novelty check – **no node with identical Kind + primary entity + overlapping time window already exists as Certified or Candidate**  
3. Abstraction level – must list children if promoting Higher-Order Concept  
4. Epistemic modesty – Validation Basis must be `Proposed-Synthesis`

### 4. Reconciler Loop (library mode)
- Event-driven off Stage 0 appends with configurable debounce (`batch_window_ms`, default 500 ms)  
- Court-side nomination queue depth limit (default 20) prevents flooding  
- `court.run_cycle()` manual escape hatch  
- Optional `start_background_reconciler()`

### 5. Contested State with Human Review
- No new state is introduced.  
- `Contested` nodes carry an optional flag `human_review_required: bool`.  
- Set when semantic similarity lands in the middle tunable band.  
- Resolved via `court.resolve_contested(node_id, decision)` where `decision` is `Literal["certify", "supersede", "defer"]`.

### 6. Out of Scope for v1
- Concurrent task handling and task-level isolation  
- Multi-agent shared Court service (single-agent library first)  
- Behavioral stability / convergence guarantees  
- Goal activation and motivation semantics  

This document is the single source of truth for Court implementation. All code must comply with the table, logging contract, Reflection rules, reconciler API, and v1 scope boundaries above.