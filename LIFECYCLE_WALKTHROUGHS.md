# Lifecycle Walkthroughs

This document gives end-to-end examples anchored strictly to the current architecture in:

- `README.md`
- `COURT_DESIGN.md`
- `STATE_MACHINE.md`

It does not introduce new rules. It only makes the existing lifecycle easier to read operationally.

## How To Read These

Each walkthrough uses the same structure:

1. Stage 0 event or proposal
2. Nomination path
3. Court checks
4. State transition
5. Outcome and retrieval effect
6. Human path, where the current rules allow one

The examples use simple fictional content to illustrate the architecture. The normative behavior comes from the referenced design docs, not from the example facts themselves.

---

## 1. Clean Certification

### Scenario

A user says: "My preferred output style is concise."

This is treated as a direct external observation about a mutable preference-like state.

### Stage 0

- A Stage 0 entry is appended as `Observed-External`.
- The raw user message is immutable and becomes the evidence substrate.

### Nomination

- Ingestion processes the Stage 0 entry.
- The extractor fills `5W1H+2`, assigns a provisional Node Kind, tags the evidence type, and nominates a node as `Candidate`.
- Entry mode is `Candidate`, not `Hypothesis`, because ingestion nominations are grounded in observed input.

### Court Checks

The Court runs the normal validation sequence:

1. Completeness check:
   - `what` is present.
   - At least one evidence pointer exists.
   - At least one anchor exists.
2. Evidence verification:
   - The confidence anchor points to a valid Stage 0 entry.
   - Evidence Type is consistent with the source.
3. Node Kind confirmation:
   - The Court ratifies or overrides the provisional kind.
4. Contradiction scan:
   - No conflicting Certified node is found for the same entity and time window.
5. Abstraction handling:
   - Not applicable.
6. Validation Basis assignment:
   - The node receives the appropriate Court-assigned basis.

### State Transition

- `Candidate -> Certified`

### Outcome

- The node becomes part of the Persistent Core as certified truth.
- Court updates any relevant indexes.
- The certification chain remains anchored to Stage 0.

### Retrieval Effect

- The node is available in default retrieval because `Certified` is the default retrieval state.
- Retrieval may rank it higher or lower based on current goals, graph distance, and derivation distance attenuation, but retrieval does not modify the node.

### Human Path

- No human path is required in the normal case.

---

## 2. Direct Negation

### Scenario

There is already a Certified node saying: "The preferred output style is concise."

Later, a new observed input says: "I want detailed answers now."

This creates a same-subject contradiction candidate for a mutable state-like fact.

### Stage 0

- The new user input is appended to Stage 0 as `Observed-External`.

### Nomination

- Ingestion nominates a new `Candidate` node from the new Stage 0 entry.

### Court Checks

1. Completeness check passes.
2. Evidence verification passes.
3. Node Kind confirmation treats the content as the appropriate kind for contradiction behavior.
4. Contradiction scan:
   - The structural filter finds an existing Certified node with the same subject and overlapping time relevance.
   - The LLM conflict-likelihood subroutine runs only after the structural match.
   - The subroutine output is logged as `Observed-Internal`.
   - The contradiction-local score informs routing but does not overwrite node Confidence.
   - Court interprets the case as direct negation under the contradiction protocol.

### State Transition

- The weaker side is routed to `Contested`.
- Under the current state machine, a new `Candidate` can transition to `Contested`.
- If a new stronger fact supersedes an older one through the contradiction workflow, the older Certified fact may also be moved into a non-default state as Court rules require.

### Outcome

- The disputed side is no longer part of default retrieval.
- The contradiction remains auditable through logged Stage 0 evidence and Court decision records.

### Retrieval Effect

- `Contested` nodes are excluded by default from retrieval.
- The stronger retained truth remains retrievable under default policy.

### Human Path

- If contradiction-band policy requires escalation, `human_review_required` is set.
- A human may use `court.resolve_contested(..., "certify" | "supersede" | "defer")`.
- The human decision is logged to Stage 0 with resolver, decision, reason, and affected node context.

---

## 3. Conflicted Pair

### Scenario

There is already a Certified node saying: "The user prefers short checklists."

A later observation says: "The user prefers narrative explanations for planning tasks."

These claims may be in tension without being strict logical negations.

### Stage 0

- The new input is appended to Stage 0 as immutable evidence.

### Nomination

- Ingestion produces a new `Candidate`.

### Court Checks

1. Completeness check passes.
2. Evidence verification passes.
3. Node Kind confirmation passes.
4. Contradiction scan:
   - Structural matching finds an existing Certified node close enough to inspect.
   - The conflict-likelihood subroutine runs and is logged.
   - Court interprets the result as plausible inconsistency rather than direct negation.
   - Under the contradiction protocol, this is a `Conflict`, not `Supersession`.

### State Transition

- The new node transitions `Candidate -> Conflicted`.
- The conflicting relationship is preserved as an auditable dispute rather than silently collapsing one side into the other.

### Outcome

- Both facts remain preserved.
- The architecture explicitly avoids deleting either side.
- Reflection may later examine the pair and propose a reconciliation or a refined replacement candidate.

### Retrieval Effect

- `Conflicted` nodes carry a retrieval penalty and visible flag.
- They are not default-clean truth in the same way `Certified` nodes are.

### Human Path

- If policy requires, `human_review_required` can be set.
- A human may:
  - certify one side through the privileged override path,
  - supersede one side,
  - or defer.
- A human or Reflection may also submit a new evidence-backed nomination, which can route `Conflicted -> Candidate`.

---

## 4. Supersession

### Scenario

There is already a Certified node saying: "Current project branch is `feature-a`."

Later, a newer observed fact says: "Current project branch is `main`."

This is not best treated as two timeless truths. It is a changing state fact.

### Stage 0

- The newer observation is appended to Stage 0.

### Nomination

- Ingestion nominates a new `Candidate`.

### Court Checks

1. Completeness check passes.
2. Evidence verification passes.
3. Node Kind confirmation identifies the fact as a kind that follows full contradiction rules.
4. Contradiction scan finds the earlier Certified fact.
5. Court classifies the case as supersession:
   - the new fact is the newer version of the same underlying mutable fact
   - the old fact should remain in history, not remain the active default truth

### State Transition

- The older node transitions `Certified -> Superseded`.
- The newer node transitions `Candidate -> Certified`.
- A `superseded-by` edge is created from the older fact to the newer fact.

### Outcome

- No fact is deleted.
- Temporal validity and history remain explicit.
- The latest certified version becomes the active truth for default retrieval.

### Retrieval Effect

- `Superseded` nodes are excluded by default.
- The new Certified version is the default retrievable fact.
- Audit and lineage remain available through the edge structure and previous versions.

### Human Path

- No human path is inherently required.
- If the case becomes disputed rather than cleanly superseded, contradiction handling or human review may be used instead.

---

## 5. Reflection Synthesis

### Scenario

Reflection observes several Certified raw episodes about the user preferring short, structured outputs across similar tasks.

It proposes a higher-level synthesis: "The user generally prefers concise structured responses during planning tasks."

### Stage 0

- Reflection work itself is not raw external truth.
- Any Court-internal LLM subcalls are logged to Stage 0 as `Observed-Internal`.
- The synthesis proposal remains distinct from the underlying observed evidence.

### Nomination

- Because this is novel synthesis rather than direct ingestion, Reflection nominates a `Hypothesis`.
- This is required by the current rule set for novel synthesis not grounded as a direct transformation of a single certified fact.

### Court Checks

1. Reflection Rule Layer applies before nomination:
   - structural hygiene
   - novelty check
   - abstraction requirements if promoting
   - epistemic modesty
2. Court processes the nominated proposal.
3. Completeness and evidence checks still apply.
4. Contradiction scan checks whether the synthesis conflicts with existing Certified material.
5. If this is an abstraction promotion candidate and Court certifies it, the Abstraction Promotion Protocol applies.

### State Transition

There are two valid end states under the current architecture:

- `Hypothesis -> Deprecated` if the proposal fails Reflection rules or Court checks
- `Hypothesis -> Candidate` if it passes the Reflection Rule Layer and proceeds as a Court candidate for full review

If the abstraction is ultimately certified:

- the higher-level node becomes `Certified`
- source node lineage is preserved through `derived-from`
- source nodes may be `Archived` if the promotion path requires it

### Outcome

- Reflection does not write directly into the Persistent Core.
- Court decides whether the synthesis remains speculative, becomes a candidate, or is rejected.
- The abstraction lineage remains auditable back to observed evidence anchors.

### Retrieval Effect

- A certified synthesis can participate in retrieval as a higher-level node.
- Derivation Distance matters here because synthesis-derived nodes are further from observed evidence than raw observations.

### Human Path

- No special human path is required by default.
- Human involvement may still occur through the normal disputed-state routes if the synthesis becomes contested.

---

## 6. Human Override

### Scenario

A node is already in `Contested` or `Conflicted`, and automated resolution is intentionally deferred pending explicit human judgment.

### Stage 0

- The disputed state already exists because earlier Court activity logged the contradiction or conflict path.
- The human resolution itself is also a Stage 0 event:
  - source: `human-resolution`
  - resolver recorded
  - decision recorded
  - reason recorded

### Nomination

- This is not a new ingestion nomination.
- It is a privileged human override path explicitly allowed by the current design.

### Court Checks

- The Court verifies that the node is in `Contested` or `Conflicted`.
- Illegal use on other states is rejected.
- The human decision bypasses normal completeness re-checks because it is an explicit override path documented in the state machine and skeleton contract.

### State Transition

Allowed outcomes are:

- `Contested -> Certified`
- `Contested -> Superseded`
- `Contested -> (no change)` via `defer`
- `Conflicted -> Certified`
- `Conflicted -> Superseded`
- `Conflicted -> (no change)` via `defer`

### Outcome

- The decision becomes part of the immutable audit chain.
- If the decision is `certify`, the node returns to a default-truth state.
- If the decision is `supersede`, the node becomes historical rather than active.
- If the decision is `defer`, no state change occurs and the review requirement remains.

### Retrieval Effect

- `Certified` after override returns to default retrieval.
- `Superseded` remains excluded by default.
- Deferred disputed nodes continue to carry their non-default retrieval behavior.

### Human Path

- This walkthrough is the human path.
- The architecture treats it as explicit, privileged, and auditable rather than as an informal exception.

---

## Cross-Walk Summary

Across all six walkthroughs, the architecture keeps the same invariants:

- Stage 0 is the immutable evidence substrate.
- Ingestion nominates `Candidate`; novel Reflection synthesis starts as `Hypothesis`.
- Court is the sole authority for state transitions.
- Contradiction scoring is local to routing and does not overwrite node Confidence.
- Retrieval may rank and penalize, but it does not certify or mutate.
- Human review is explicit, bounded, and logged.
- No facts are deleted; history remains auditable.
