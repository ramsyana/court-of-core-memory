# STATE_MACHINE

This document defines the **formal state machine** for every node in the Persistent Core.  
It extends and refines the Court specification in README.md and the hybrid design in COURT_DESIGN.md.  
Where any conflict exists, this document + COURT_DESIGN.md together take precedence for implementation.

## 1. Node States (complete list)

Every node in the Persistent Core is **always in exactly one state**.  
Terminal states are permanent for auditability; deletion is forbidden.

| State        | Description (verbatim from README + refinements)                                                                 | Retrieval Behaviour                  | Human Review Possible? |
|--------------|------------------------------------------------------------------------------------------------------------------|--------------------------------------|------------------------|
| **Hypothesis**   | Novel proposal from Reflection. High uncertainty. Not yet grounded.                                              | Excluded by default                  | No                     |
| **Candidate**    | Nominated for Court review (from Ingestion or grounded Reflection).                                              | Excluded until certified             | No                     |
| **Certified**    | Court-approved truth. Single source of truth.                                                                    | Default retrieval state              | No                     |
| **Contested**    | Weaker side of a Direct Negation contradiction.                                                                  | Excluded by default (flag available) | Yes (via flag)         |
| **Conflicted**   | Plausible inconsistency with another node (non-negation).                                                        | Retrieval penalty + visible flag     | Yes (via flag)         |
| **Superseded**   | Replaced by a newer certified fact. Retained with `superseded-by` edge.                                          | Excluded by default                  | No                     |
| **Archived**     | Abstracted into a higher-order node. Retained with `abstracted-into` edge.                                       | Excluded by default                  | No                     |
| **Deprecated**   | Low-value, obsolete, or failed maturation. Retained for audit.                                                   | Excluded by default                  | No                     |

**Note**: `human_review_required: bool` is an optional flag that can be attached to **Contested** or **Conflicted** nodes only (set by the semantic similarity band in the contradiction scan).

## 2. Legal Transitions + Guard Conditions + Entry/Exit Actions

All transitions are **enforced exclusively by the Court**. Illegal transitions are rejected at the rule-engine layer.

| From              | To                  | Guard Conditions (Rule Engine)                                                                 | LLM Subroutine Used (if any)                  | Entry Actions                                                                 | Exit Actions (on the old node)                  |
|-------------------|---------------------|------------------------------------------------------------------------------------------------|-----------------------------------------------|-------------------------------------------------------------------------------|-------------------------------------------------|
| Hypothesis        | Candidate           | Passes Reflection Rule Layer (see COURT_DESIGN.md §3)                                          | —                                             | Write Stage 0 Observed-Internal (nomination)                                 | —                                               |
| Hypothesis        | Deprecated          | Fails Reflection Rule Layer or Court completeness check                                        | —                                             | Write Stage 0 Observed-Internal (rejection reason)                            | —                                               |
| Candidate         | Certified           | Completeness ✓ + no structural/semantic conflict + Validation Basis assigned                   | Node-kind proposal, evidence justification    | Update indexes, write full_prompt + raw_output logs                           | —                                               |
| Candidate         | Contested           | Structural conflict OR semantic score in middle band                                           | Semantic similarity (post-structural filter)  | Set `human_review_required` if middle band; write Observed-Internal           | —                                               |
| Candidate         | Conflicted          | Plausible inconsistency (non-negation)                                                         | Semantic similarity                           | Set `human_review_required` if middle band; write Observed-Internal           | —                                               |
| Candidate         | Deprecated          | Fails completeness or epistemic modesty rules                                                  | —                                             | Write rejection reason to Stage 0                                             | —                                               |
| Certified         | Contested           | Direct Negation contradiction detected (new stronger evidence)                                 | Semantic similarity                           | Set weaker node to Contested; write Observed-Internal                         | —                                               |
| Certified         | Conflicted          | Conflict (non-negation) detected                                                               | Semantic similarity                           | Write Observed-Internal                                                       | —                                               |
| Certified         | Superseded          | Newer version of same fact (Supersession rule)                                                 | —                                             | Create `superseded-by` edge; write Observed-Internal                          | —                                               |
| Certified         | Archived            | Abstraction into Higher-Order Concept approved by Court                                        | —                                             | Create `abstracted-into` edge                                                 | —                                               |
| Contested         | Certified           | `resolve_contested(..., "certify")` **(privileged human override)**                            | —                                             | Clear `human_review_required`; write Observed-Internal (resolver + decision + timestamp) | —                                               |
| Contested         | Superseded          | `resolve_contested(..., "supersede")`                                                          | —                                             | Create `superseded-by` edge; write Observed-Internal (resolver + decision + timestamp) | —                                               |
| Contested         | (no change)         | `resolve_contested(..., "defer")`                                                              | —                                             | Write Observed-Internal (resolver + decision + timestamp); flag remains true  | —                                               |
| **Conflicted**    | **Candidate**       | New nomination submitted by Reflection or human with additional evidence anchors               | —                                             | Write Stage 0 Observed-Internal (new nomination); reset `human_review_required` | —                                               |
| Conflicted        | Contested           | Reclassified as Direct Negation                                                                | Semantic similarity                           | Set flag if required; write Observed-Internal                                 | —                                               |
| **Conflicted**    | **Certified**       | `resolve_contested(..., "certify")` **(privileged human override)**                            | —                                             | Clear `human_review_required`; write Observed-Internal (resolver + decision + timestamp) | —                                               |
| Conflicted        | Superseded          | `resolve_contested(..., "supersede")`                                                          | —                                             | Create `superseded-by` edge; write Observed-Internal (resolver + decision + timestamp) | —                                               |
| Conflicted        | (no change)         | `resolve_contested(..., "defer")`                                                              | —                                             | Write Observed-Internal (resolver + decision + timestamp); flag remains true  | —                                               |
| Conflicted        | Deprecated          | Court downgrade                                                                                | —                                             | Write reason to Stage 0                                                       | —                                               |
| Superseded        | Archived            | Later abstraction                                                                              | —                                             | Create `abstracted-into` edge                                                 | —                                               |
| Superseded        | Deprecated          | Explicit deprecation by Reflection/Court                                                       | —                                             | Write reason                                                                  | —                                               |

**Key invariants enforced by the rule engine (never overridden by LLM):**
- Terminal states (Archived, Deprecated) have no outgoing transitions.
- **No node that has reached Certified may return to Candidate or earlier states.**  
  (Conflicted → Candidate is permitted as an explicit re-nomination path; Conflicted → Certified and Contested → Certified are privileged human overrides only.)
- Every transition that uses an LLM subroutine **must** produce a full Observed-Internal Stage 0 entry (per COURT_DESIGN.md §2).
- Contradiction scan always runs structural filter first, then (only if needed) semantic LLM call with tunable bands.

## 3. Public Court API for state resolution (implementation contract)

```python
court.resolve_contested(
    node_id: str,
    decision: Literal["certify", "supersede", "defer"],
    resolver: str = "human"   # or agent ID
) -> None