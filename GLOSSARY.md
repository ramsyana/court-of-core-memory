# Glossary

This document defines the canonical meaning of core architectural terms used across the Court of Core Memory design set.

It is an index and clarification layer, not a new source of rules. If any conflict exists:

- `COURT_DESIGN.md`
- `STATE_MACHINE.md`
- `PERSISTENT_CORE_SCHEMA.md`
- `README.md`

remain authoritative according to their documented precedence.

This glossary exists to prevent terminology drift across:

- `README.md`
- `COURT_DESIGN.md`
- `STATE_MACHINE.md`
- `PERSISTENT_CORE_SCHEMA.md`
- `GAPS.md`
- `LIFECYCLE_WALKTHROUGHS.md`
- `INTEGRATIONS.md`

---

## Core Governance Terms

### Court

The Court is the deterministic governing layer that validates nominations, assigns or confirms certification basis, applies contradiction rules, and performs all legal state transitions. It is the sole authoritative governor of the Persistent Core.

Key boundary:

- Retrieval may rank.
- Reflection may propose.
- Ingestion may nominate.
- Only the Court certifies or changes node state.

Primary references:

- `README.md`
- `COURT_DESIGN.md`
- `STATE_MACHINE.md`

### Stage 0

Stage 0 is the immutable evidence substrate. It stores raw observations, tool outputs, trajectories, and Court-internal logs. It is append-only and is never treated as an automatic promotion queue.

Primary references:

- `README.md`
- `PERSISTENT_CORE_SCHEMA.md`
- `LIFECYCLE_WALKTHROUGHS.md`

### Persistent Core

The Persistent Core is the Court-governed memory structure built on top of Stage 0. It contains nodes that have entered the formal lifecycle and are governed by the state machine.

Primary references:

- `README.md`
- `PERSISTENT_CORE_SCHEMA.md`

### Nomination

Nomination is the explicit act of submitting a proposed node to the Court for review. Nomination is never implicit. Ingestion nominates observed inputs; Reflection nominates synthesis or grounded transformations.

Primary references:

- `README.md`
- `LIFECYCLE_WALKTHROUGHS.md`

### Anchor

An anchor is a required grounding reference attached to a node. In the v1 completeness contract, at least one anchor must exist for Candidate and Certified nodes.

In schema terms:

- `Node.anchors` contains Stage 0 entry IDs or node IDs.

Important clarification:

- MAGMA-style segmentation artifacts, traversal paths, router outputs, and consolidation hints are not anchors by themselves.

Primary references:

- `README.md`
- `PERSISTENT_CORE_SCHEMA.md`
- `COURT_DESIGN.md`
- `GAPS.md`

### Evidence Pointer

An evidence pointer is a reference in `FiveW1HPlusTwo.evidence` that links a candidate to supporting evidence. It is part of the completeness contract together with `what` and at least one anchor.

Primary references:

- `README.md`
- `PERSISTENT_CORE_SCHEMA.md`

---

## Epistemic Terms

### Confidence

Confidence is the node-level epistemic certainty score for a memory. It is anchored exclusively to Stage 0 and accompanies the evidence chain, Evidence Type, and Validation Basis.

Confidence is:

- Court-certified epistemic certainty
- about validity of the evidence basis
- distinct from behavioral frequency or pattern stability

Confidence is not:

- contradiction score
- retrieval score
- router weight
- traversal penalty

Primary references:

- `README.md`
- `COURT_DESIGN.md`
- `GAPS.md`

### conflict-likelihood

`conflict-likelihood` is the contradiction-local score produced by the Court's contradiction subroutine after the structural filter has already identified a possible dispute.

It is:

- a routing signal for contradiction handling
- logged as `Observed-Internal`
- local to the contradiction decision path

It is not:

- generic text similarity
- node Confidence
- a retrieval relevance score

Primary references:

- `COURT_DESIGN.md`
- `PROMPT_TEMPLATES.md`
- `PROMPT_VERIFICATION_WRAPPER.md`
- `GAPS.md`

### Evidence Type

Evidence Type describes where the supporting evidence came from.

The defined values are:

- `Observed-External`
- `Observed-Internal`
- `Inferred`

Evidence Type is orthogonal to Validation Basis.

Primary references:

- `README.md`

### Validation Basis

Validation Basis is the Court-assigned certification basis for a node. It describes the authoritative basis on which the Court accepted or resolved the node.

The canonical stored values are:

- `Stage0Anchor`
- `CertifiedAnchor`
- `ProposedSynthesis`
- `HumanOverride`

Important clarification:

- These are schema-level canonical values and should be used consistently across implementation-facing docs.
- Richer explanatory language may exist in audit or UI layers, but it must map back to these stored values.

Primary references:

- `README.md`
- `PERSISTENT_CORE_SCHEMA.md`
- `GAPS.md`

### Derivation Distance

Derivation Distance is the graph distance from a node to its nearest observed evidence in Stage 0. It is computed at certification time, stored as the certified baseline, and used to attenuate confidence during retrieval.

Primary references:

- `README.md`

### Priority

Priority is a retrieval-time quantity, not a stored node field. It is computed lazily from current goals and used for retrieval and eviction behavior.

Primary references:

- `README.md`

---

## Lifecycle Entry Terms

### Ingestion

Ingestion is the pipeline that processes raw Stage 0 entries, fills the `5W1H+2` scaffold, assigns provisional classification, tags evidence type, and nominates a `Candidate`.

Primary references:

- `README.md`
- `LIFECYCLE_WALKTHROUGHS.md`

### Reflection

Reflection is the resource-bounded maintenance and synthesis process that analyzes existing material, proposes new synthesis, resolves weak spots, and nominates changes to the Court.

Reflection never writes directly into the Persistent Core.

Primary references:

- `README.md`
- `COURT_DESIGN.md`

### Candidate

`Candidate` is the state for a node that has been nominated for Court review and is awaiting adjudication. It can originate from ingestion or from grounded Reflection transformations.

Retrieval behavior:

- excluded until certified

Primary references:

- `STATE_MACHINE.md`
- `README.md`

### Hypothesis

`Hypothesis` is the state for a novel Reflection proposal that is not yet grounded enough to count as a normal candidate. It is high-uncertainty and excluded by default from retrieval.

Important clarification:

- Novel synthesis enters as `Hypothesis`.
- Grounded transformations may enter as `Candidate`.

Primary references:

- `STATE_MACHINE.md`
- `README.md`
- `COURT_DESIGN.md`

### Certified

`Certified` is the default truth state. A Certified node is Court-approved and is the default retrieval state.

Primary references:

- `STATE_MACHINE.md`
- `README.md`

---

## Dispute And Resolution Terms

### Contradiction Scan

Contradiction scan is the Court step that checks a nominated node against existing Certified material. It first applies a structural filter and only then, if needed, invokes the `conflict-likelihood` subroutine.

Primary references:

- `README.md`
- `COURT_DESIGN.md`
- `STATE_MACHINE.md`

### Direct Negation

Direct Negation is the contradiction type where two claims cannot both be true. Under the contradiction protocol, the weaker side is marked `Contested`; if evidence is equal, both may be marked `Contested` and flagged for Reflection.

Primary references:

- `README.md`
- `LIFECYCLE_WALKTHROUGHS.md`

### Conflict

Conflict is the contradiction type where two claims are inconsistent but still plausible together under uncertainty. Under the contradiction protocol, both nodes are marked `Conflicted`.

Primary references:

- `README.md`
- `LIFECYCLE_WALKTHROUGHS.md`

### Contested

`Contested` is the node state used for the weaker side of a direct negation dispute, or for conservative escalation when contradiction-band policy requires explicit human review.

Retrieval behavior:

- excluded by default

Human review:

- allowed via the documented human override path

Primary references:

- `STATE_MACHINE.md`
- `COURT_DESIGN.md`

### Conflicted

`Conflicted` is the node state for a plausible non-negation inconsistency. It preserves both sides of the dispute rather than forcing a premature collapse.

Retrieval behavior:

- retrieval penalty plus visible flag

Human review:

- allowed via the documented human override path

Primary references:

- `STATE_MACHINE.md`
- `README.md`
- `LIFECYCLE_WALKTHROUGHS.md`

### human_review_required

`human_review_required` is an escalation flag that may be attached only to `Contested` or `Conflicted` nodes. It is not a state. It indicates that contradiction-band policy requires explicit review before final confidence in resolution.

Primary references:

- `STATE_MACHINE.md`
- `PERSISTENT_CORE_SCHEMA.md`
- `COURT_DESIGN.md`

### Human Override

Human Override is the privileged, explicitly logged path for resolving a `Contested` or `Conflicted` node through `court.resolve_contested(...)`.

Allowed decisions:

- `certify`
- `supersede`
- `defer`

It must be logged to Stage 0 with resolver, decision, reason, and affected node context.

Primary references:

- `COURT_DESIGN.md`
- `STATE_MACHINE.md`
- `PERSISTENT_CORE_SCHEMA.md`
- `LIFECYCLE_WALKTHROUGHS.md`

---

## Historical State Terms

### Supersession

Supersession is the contradiction outcome used when a newer version of the same mutable fact replaces an older one. The older fact is retained and linked by `superseded-by`; it is not deleted.

Primary references:

- `README.md`
- `STATE_MACHINE.md`
- `LIFECYCLE_WALKTHROUGHS.md`

### Superseded

`Superseded` is the state of a previously Certified node that has been replaced by a newer Certified version of the same fact.

Retrieval behavior:

- excluded by default

Primary references:

- `STATE_MACHINE.md`
- `README.md`

### Archived

`Archived` is the state used when a node has been abstracted into a higher-order node. It remains in history and is linked by `abstracted-into`.

Retrieval behavior:

- excluded by default

Primary references:

- `STATE_MACHINE.md`
- `README.md`

### Deprecated

`Deprecated` is the retained terminal state for low-value, obsolete, or failed nodes. It preserves audit history rather than deleting the node.

Retrieval behavior:

- excluded by default

Primary references:

- `STATE_MACHINE.md`

---

## Structure Terms

### Node Kind

Node Kind describes how a memory behaves under contradiction and validation. It is not the same thing as a content dimension and not the same thing as lifecycle state.

Defined values:

- `Event`
- `State Fact`
- `Principle`
- `Goal`
- `Procedure`

Primary references:

- `README.md`
- `PERSISTENT_CORE_SCHEMA.md`
- `COURT_DESIGN.md`

### Node State

Node State describes where a node is in the Court lifecycle. It is a formal state-machine concept rather than a semantic content category.

Defined values:

- `Hypothesis`
- `Candidate`
- `Certified`
- `Contested`
- `Conflicted`
- `Superseded`
- `Archived`
- `Deprecated`

Primary references:

- `STATE_MACHINE.md`
- `PERSISTENT_CORE_SCHEMA.md`

### 5W1H+2

`5W1H+2` is the completeness scaffold used by the Court during ingestion and review. It is a checklist, not the full storage ontology.

Mapped elements:

- `Who`
- `What`
- `When`
- `Where`
- `Why`
- `How`
- `What it means`
- `What aided it`

Primary references:

- `README.md`
- `PERSISTENT_CORE_SCHEMA.md`

### Orthogonal Projections

The eight orthogonal projections are the content dimensions of the Persistent Core:

- `Entity`
- `Semantic`
- `Temporal`
- `Spatial`
- `Causal`
- `Procedural`
- `Implication`
- `Resources`

They are separate dimensions and must not be collapsed into a single semantic layer.

Primary references:

- `README.md`
- `INTEGRATIONS.md`

---

## Retrieval Terms

### Retrieval

Retrieval is the policy-guided traversal process that surfaces relevant material from the Persistent Core into working memory. It may use graph distance, dimension weights, derivation distance attenuation, and state penalties.

Retrieval does not:

- certify facts
- assign Validation Basis
- change node state
- overwrite Confidence

Primary references:

- `README.md`
- `COURT_DESIGN.md`

### Intent-Aware Router

The Intent-Aware Router is an optional retrieval-layer component that classifies query intent and chooses traversal weights or policy profiles.

It may influence retrieval order, but it has no legal transition authority and does not create anchors by itself.

Primary references:

- `INTEGRATIONS.md`
- `COURT_DESIGN.md`
- `GAPS.md`

### Retrieval Traceability

Retrieval traceability is the optional recording of why a result was surfaced for a task, including traversal path, scoring contributions, penalties, and goal-alignment basis.

Primary references:

- `README.md`

---

## Usage Notes

- Use `Confidence` only for the Court-certified epistemic score attached to a node.
- Use `conflict-likelihood` only for the contradiction-local subroutine score.
- Use the canonical `Validation Basis` values exactly as stored in schema-facing docs.
- Treat `anchor` as a grounding reference, not as a synonym for any retrieval artifact or graph hint.
- Treat `Candidate` and `Hypothesis` as different epistemic entry modes, not as cosmetic labels.
