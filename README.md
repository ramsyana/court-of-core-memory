# Court of Core Memory: A Formal Agent Memory Architecture — Design Specification [DRAFT]

> **Status:** Concept & design specification only. No production implementation exists.
> This repository contains architecture documents, data schemas, and proposed interfaces.
> Code implementation is planned but not yet begun.

**Author:** Ramsyana  
**GitHub:** @ramsyana

Licensed under the MIT License.

---

## Overview

Most agent memory systems treat memory as a flat store of retrievable chunks. Retrieval is similarity search. Connections between memories are implicit at best. There is no concept of truth, no validation, no account of where a belief came from or how confident the system should be in it. The result is an agent that hallucinates from its own memory as readily as from its weights.

This document specifies a different approach: a formally governed, multi-dimensional memory architecture in which every fact must earn its place through validation, every belief carries traceable provenance, and every transition between epistemic states is mediated by a single authoritative process called the Validation Court.

The architecture is built around four core ideas:

- Memory is not a database. It is a cognitive map with structure, relationships, and epistemic status.
- Truth is not asserted. It is validated. Raw experience and certified fact are kept strictly separate.
- Relationships are first-class citizens. Every memory is projected across multiple orthogonal graph layers, each capturing a different dimension of meaning.
- Governance is explicit. A formal state machine defines what a memory node can be, what transitions are legal, and who has authority to make them.

The result is a closed-loop system in which raw evidence becomes validated truth, validated truth is used in active cognition, and the outputs of that cognition feed back into the evidence log for future validation.

---

## 1. Lifecycle: Four Stages

The system has four distinct stages. These must not be conflated.

### Stage 0: Immutable Evidence Log

A permanent, append-only log of all raw observations, trajectories, tool outputs, and intermediate reasoning. This is the single source of truth for all confidence anchors. Nothing in Stage 0 is ever discarded or modified. It grows indefinitely by design.

Stage 0 is not a promotion queue. Not everything logged here becomes a certified memory. It is purely an evidence substrate.

### Stage 1: Validation Court

The promotion and adjudication gate. All candidates for the Persistent Core must pass through the Court. The Court is the sole authority for state transitions, certifications, and rejections. It never acts autonomously on Stage 0 content -- it only processes explicitly nominated candidates.

### Stage 2: Persistent Core

The long-term, orthogonal multi-graph truth store. Only court-certified facts live here. The Persistent Core is self-evolving but only via Court-certified changes.

### Stage 3: Working Memory

A transient, task-scoped scratchpad. Created at task start, destroyed at task end. Working Memory is where cognition happens -- where retrieved facts meet current task inputs and produce actions.

**Internal structure -- three buckets:**

- **Retrieved**: Nodes pulled from the Persistent Core by the retrieval policy. Retains lightweight projected subgraphs (Core Node ID plus relevant edges across all 8 content dimensions). Not flattened to a list -- relational structure is preserved because reasoning depends on knowing how facts connect, not just what they are.
- **Current**: Live task inputs -- the active user message, current goal, and available tool specs. This bucket seeds the retrieval query.
- **Generated**: Tool results and intermediate reasoning steps, flushed immediately to Stage 0 with content-type-specific tagging:
  - Tool results → **Observed-External**
  - Intermediate reasoning steps → **Observed-Internal**

Generated content flushes immediately rather than accumulating until task end, so that observations are not lost if a task fails mid-execution.

**The full closed loop:**

```
Persistent Core
  → Retrieval (seeded by Current bucket)
  → Working Memory
  → Action / Reasoning
  → Stage 0 (immediate append for Generated content)
  → Nomination
  → Validation Court
  → Persistent Core (certified) or Deprecated (rejected)
```

---

## 2. Memory State Lifecycle: Formal State Machine

Every Core Node exists in exactly one state at any time. The Court is the sole state-transition authority. All transitions not listed in the table below are forbidden without explicit Court-mediated re-certification.

### Legal Transitions

| From State  | To State(s)                                                    | Trigger                                                              |
|-------------|----------------------------------------------------------------|----------------------------------------------------------------------|
| Hypothesis  | Candidate or Deprecated                                        | Court review (accept to Candidate; reject to Deprecated)             |
| Candidate   | Certified, Contested, Conflicted, or Deprecated                | Court certification or rejection                                     |
| Certified   | Superseded, Contested, Conflicted, Archived, or Deprecated     | Contradiction protocol, abstraction promotion, or obsolescence       |
| Contested   | Certified, Conflicted, or Deprecated                           | New evidence or Reflection resolution                                |
| Conflicted  | Certified, Contested, or Deprecated                            | Reflection resolution                                                |
| Superseded  | Archived or Deprecated                                         | Historical compression or abstraction by Reflection                  |
| Archived    | (terminal)                                                     | --                                                                   |
| Deprecated  | (terminal)                                                     | --                                                                   |

Terminal states retain their nodes permanently in the Persistent Core for audit purposes. Terminal means no further state transitions, not deletion.

### State Definitions

- **Hypothesis**: A novel proposal from Reflection. High uncertainty. Inferred tagging. Entry point for speculative synthesis that is not yet grounded in certified evidence.
- **Candidate**: Nominated and under Court review. Produced by Ingestion (always) or by Reflection for grounded non-novel changes.
- **Certified**: Court-approved truth. Default retrieval state.
- **Contested**: Weaker side of a Direct Negation contradiction. Excluded from retrieval by default. Retrievable with an explicit contested flag.
- **Conflicted**: Plausible inconsistency with another node. Subject to retrieval penalty. Visible conflict flag on retrieval.
- **Superseded**: Replaced by a newer certified fact. Retained permanently with a superseded-by edge to the replacement.
- **Archived**: Abstracted into a higher-level node. Retained permanently with an abstracted-into edge.
- **Deprecated**: Court-designated as low-value or weakly supported. Excluded from default retrieval. Retained permanently for audit.

### Deprecated Trigger

Reflection nominates a Deprecated candidate when one or more of the following criteria are met:

- Zero relevance to any active Goal Core Node over a configurable period (e.g., 30 days)
- Persistent low confidence
- Excessive Derivation Distance from observed evidence
- Failed Hypothesis maturation (a Hypothesis that never accumulated sufficient evidence to be certified)
- Explicit Reflection downgrade recommendation based on graph analysis

Court certifies the transition. Reflection nominates; Court decides.

---

## 3. Structure of the Persistent Core

Every memory in the Persistent Core consists of:

- One **central Core Node** with a unique ID that serves as the Master Index.
- Eight **orthogonal projections** -- one per content dimension -- each in its own dedicated graph.

### Node Kind

Every Core Node carries a Node Kind field that classifies its epistemic behavior. Node Kind is distinct from the content dimensions (which describe what a memory is about) and from the state machine (which describes where a memory is in its lifecycle). Node Kind describes how a memory should be treated under contradiction, supersession, and validation.

| Node Kind   | Nature                                           | Contradiction Behavior                                                                 |
|-------------|--------------------------------------------------|----------------------------------------------------------------------------------------|
| Event       | An immutable occurrence that happened at a point in time | Cannot be superseded -- the event happened. Disputes apply to interpretation, not the occurrence itself. Typically resolves via Conflict or Contested. |
| State Fact  | A mutable world, user, or system state           | Supersession is the normal case when the state changes. Full contradiction protocol applies. |
| Principle   | An abstract or generalized synthesis             | Typically Contested or Conflicted rather than Superseded. Requires multi-source corroboration to certify. |
| Goal        | An intentional target of the agent               | Managed via Goal Index. Supersession on achievement or replacement. Cannot be Conflicted with itself. |
| Procedure   | Executable process knowledge or step sequences   | Superseded when a better procedure is certified. Versioned via superseded-by edges.    |

Node Kind is assigned by the LLM extractor during Ingestion and confirmed by the Court at certification. It is stored as a top-level field on the Core Node, not as a dimension projection.

### Inter-layer Connections

The eight content layers are connected via shared Core Node ID as primary key, with a fast join index enabling multi-layer traversal. Optional lightweight cross-layer link edges are created only when the Court certifies a direct, explicit relationship between two specific Core Nodes (for example, a Court-verified causal link between two nodes that would otherwise only be connected through the shared ID join).

### Lightweight Indexes

- **Status Index**: Maintained list of all Core Node IDs currently in Contested, Conflicted, Superseded, Archived, or Deprecated states. Court is the sole writer. Reflection reads this index directly to find nodes requiring attention, without needing to scan the full graph.
- **Goal Index**: Maintained list of all active Goal Core Nodes. Court is the sole writer (adds on goal certification, removes on goal supersession or achievement). Retrieval uses this index to locate current goals for lazy Priority computation.

### A. Content Dimensions: 8 Orthogonal Graph Layers

Each memory node is projected across all eight layers simultaneously. Each layer has its own dedicated graph with its own edge types. The orthogonality rules define what belongs in each layer and prevent contamination between dimensions.

| Dimension   | 5W1H+2 Mapping      | Purpose                                                         | Orthogonality Rule                                         |
|-------------|---------------------|-----------------------------------------------------------------|------------------------------------------------------------|
| Entity      | Who                 | Participants and objects involved                               | Named instances only; concepts go to Semantic              |
| Semantic    | What / topic        | Core meaning, concept clusters, topic organization             | Concepts and topics only; named instances go to Entity     |
| Temporal    | When                | Time of occurrence, bi-temporal validity windows               | Time only; sequence of steps goes to Procedural            |
| Spatial     | Where               | Container or scope of the event (codebase path, file, conversation ID, workspace, process, context-window slice) | Tight definition; no catch-all use |
| Causal      | Why                 | Backward-observed cause or trigger                              | Backward-only; forward consequences go to Implication      |
| Procedural  | How                 | Sequence of actions or steps                                    | Sequence only; external dependencies go to Resources       |
| Implication | What it means       | Forward outcomes, lessons, higher-level insights                | Forward-only; backward causes go to Causal                 |
| Resources   | What aided          | External dependencies invoked by the steps                      | External dependencies only; the steps themselves go to Procedural |

### B. Meta-Properties

Applied to every Core Node and every edge across all layers.

**Confidence**: A score from 0 to 1, plus a pointer to the relevant entry in the Stage 0 Evidence Log, plus the Evidence Type and Validation Basis fields (defined in Section 4). Confidence is anchored exclusively to the immutable Stage 0 log -- there are no circular references.

Note: Confidence represents epistemic certainty -- how sure the system is that the evidence is valid. It does not represent behavioral frequency or pattern stability. Distributional facts such as "the user usually prefers concise answers" are a known limitation of the current confidence model. Behavioral stability as a separate field is deferred to a future version.

**Priority**: Not stored statically. Computed lazily at retrieval time based on the current Goal Core Nodes in the Goal Index. When goals change, Priority values do not need cascading updates -- they are recomputed on demand.

**Abstraction Level**: A vertical hierarchy cutting across all eight content layers: Raw Episode, Synthesis, Principle. This is not a flat layer alongside the eight content dimensions. It is a hierarchical structure within the graph that allows the same fact to exist at different levels of generality.

**Derivation Distance**: The graph distance from a node to its nearest Observed evidence in Stage 0. Computed at certification time and stored as the certified baseline. Attenuates confidence as distance grows -- nodes derived through many inference hops from observed reality are treated as less reliable than nodes directly grounded in observation.

Hybrid update policy: Derivation Distance is frozen at certification for audit stability. Reflection periodically recomputes a live estimate during maintenance passes. If materially different (distance changes by two or more hops, or Reflection judges the difference materially impacts confidence), Reflection nominates a confidence recalibration candidate to the Court.

### Current Goals

Agent goals are full Core Nodes with all eight content-dimension projections and a Node Kind of Goal, identical in structure to any other memory. They are identified by the Goal Index rather than by a separate node type. Goals are updated only through the Court, ensuring consistency with the rest of the Persistent Core.

---

## 4. Evidence Type and Validation Basis

These two fields accompany every Confidence pointer. They are assigned during ingestion (Evidence Type) and at Court certification (Validation Basis).

### Evidence Type: Source Category

Describes where the evidence came from.

- **Observed-External**: Direct external input -- user messages, tool outputs, environment responses, external system data.
- **Observed-Internal**: Agent's own intermediate reasoning or action traces.
- **Inferred**: Content generated by the LLM extractor during ingestion, or by Reflection during synthesis. Not directly observed.

Observed-External and Observed-Internal are both grounded but not equally authoritative. External observation is grounded in the world. Internal observation is grounded in the agent's own cognition trace. Inferred content carries lower base weight in Court evaluation.

### Validation Basis: Certification Reason

Court-assigned at certification. Describes why the Court accepted the candidate.

- **Direct Evidence**: A single strong observed fact with high confidence.
- **Multi-source Corroboration**: Multiple independent evidence sources converge on the same conclusion.
- **Statistical Inference**: Probabilistic reasoning over observed patterns.
- **Reflection Synthesis**: A Reflection-generated abstraction or hypothesis that passed Court scrutiny.
- **Human Assertion**: Explicit user-provided statement treated as authoritative input.
- **Temporal Supersession**: Certified because it replaces a previously certified fact with newer evidence.

Evidence Type and Validation Basis are orthogonal. A node can be Observed-External with a Validation Basis of Multi-source Corroboration, or Inferred with a Validation Basis of Reflection Synthesis.

---

## 5. The 5W1H+2 Scaffold

The 5W1H+2 scaffold is a completeness checklist used by the Validation Court during ingestion review. It is not a storage structure. It maps naturally to the eight content dimensions.

| Scaffold Element  | Maps to Dimension |
|-------------------|-------------------|
| Who               | Entity            |
| What              | Semantic          |
| When              | Temporal          |
| Where             | Spatial           |
| Why               | Causal            |
| How               | Procedural        |
| What it means     | Implication       |
| What aided it     | Resources         |

**Mandatory fields (v1 Court completeness)**: `Semantic` + `Evidence anchor`.
In the v1 data model these correspond to:
- `FiveW1HPlusTwo.what` (Semantic)
- At least one anchor (`Node.anchors`) and at least one evidence pointer (`FiveW1HPlusTwo.evidence`)

`Temporal` (`FiveW1HPlusTwo.when`) is strongly preferred and is expected to be required by kind-specific rules in a later revision. The Court may still reject candidates with missing Temporal coverage depending on configured policy, but the baseline v1 completeness contract is `what + evidence + anchors`.

**All other fields**: Partial coverage is permitted. Missing dimensions are flagged as "unknown" with a confidence of 0. This ensures partial memories can be promoted without fabricating content to fill gaps, while making their incompleteness explicit and auditable.

---

## 6. Court Operations

### Contradiction Resolution Protocol

When the Court's contradiction scan detects a conflict between a new candidate F_new and an existing certified fact F_old, it classifies and resolves as follows. Node Kind governs which resolution types are applicable: Events cannot be Superseded; Principles default to Contested or Conflicted rather than Superseded; State Facts and Procedures follow the full protocol.

| Type               | Description                            | Resolution                                                                                                                                                           |
|--------------------|----------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 1. Supersession    | Newer version of the same fact         | F_old receives a bi-temporal end date and a "superseded-by" edge to F_new. F_new is certified normally. Not applicable to Event nodes.                              |
| 2. Direct Negation | Both cannot be true                    | The node with stronger evidence is retained as Certified. The weaker is marked Contested and excluded from default retrieval. If evidence is equal, both are marked Contested and flagged for Reflection. |
| 3. Conflict        | Inconsistent but both plausible        | Both nodes are marked Conflicted. A bidirectional conflict edge is created between them. A retrieval penalty is applied to both. Reflection resolves during idle time. |

Every state transition is logged with timestamp, evidence references, and the Court's decision type. No nodes are ever deleted.

### Abstraction Promotion Protocol

When Reflection proposes promoting a Raw Episode to Synthesis, or a Synthesis to Principle, and the Court certifies:

1. The original node is kept permanently and marked **Archived** (or transitions Superseded → Archived if it was already superseded).
2. The original receives an "abstracted-into" edge pointing to the new higher-level node.
3. The new node receives "derived-from" edges back to all contributing source nodes.
4. Derivation Distance is computed on the new node and frozen at certification.

This preserves the full abstraction lineage for auditability. Higher-level nodes can always be traced back to their observed evidence anchors.

---

## 7. Operations

### Ingestion

The LLM extractor processes every raw Stage 0 entry. It fills the 5W1H+2 fields, assigns Node Kind, applies Evidence Type tagging (Observed-External, Observed-Internal, or Inferred based on content type), and nominates the result directly as a **Candidate** to the Validation Court.

Ingestion always produces Candidate-state nodes, never Hypothesis-state nodes. The Hypothesis state is reserved for Reflection's novel synthesis.

**Timing**: Configurable. Ingestion can run immediately on each Stage 0 append, in periodic batches, or triggered at task completion. The choice affects latency between observation and certified memory, not the correctness of the architecture.

### Nomination Trigger

Nomination is always explicit. Stage 0 is not an automatic promotion queue.

- **Ingestion** nominates every raw Stage 0 entry as a **Candidate**.
- **Reflection** nominates in two modes:
  - Novel synthesis (speculative, not grounded in a single existing certified node) → **Hypothesis** state.
  - Grounded transformations of existing certified facts (rewires, abstraction promotions, contradiction resolutions) → **Candidate** state.

The entry state encodes epistemic provenance. A Candidate from Ingestion is grounded in observed reality. A Hypothesis from Reflection is speculative synthesis. Both route through the Court, but the Court evaluates them with different epistemic weight.

### Validation Court

The Court runs the following steps on every nominated candidate:

1. **Completeness check**: Verifies that mandatory 5W1H+2 fields (Semantic and Evidence anchor) are present. Rejects or returns for enrichment if not.
2. **Evidence verification**: Confirms the confidence anchor points to a valid Stage 0 entry. Checks Evidence Type consistency.
3. **Node Kind confirmation**: Confirms or corrects the Node Kind assigned by the extractor. Adjusts applicable contradiction rules accordingly.
4. **Contradiction scan**: Searches the Persistent Core for conflicts with existing Certified nodes. Applies the 3-type contradiction protocol, respecting Node Kind constraints, if a conflict is found.
5. **Abstraction handling**: If this is an abstraction promotion candidate, applies the Abstraction Promotion Protocol.
6. **State transition**: Certifies the candidate (→ Certified), routes it to a conflict state (→ Contested or Conflicted), or rejects it (→ Deprecated).
7. **Validation Basis assignment**: Assigns the appropriate Validation Basis field at certification.
8. **Index maintenance**: Updates Status Index and Goal Index as required. Court is sole writer for both.

### Retrieval

The Current bucket (active task inputs and goal) seeds the initial retrieval query. The retrieval policy then performs policy-guided multi-layer traversal across the eight content dimensions using an AEL-style Thompson-sampling bandit by default, configurable to full agentic reasoning about the current goal.

The Goal Index is used to locate current Goal Core Nodes. Priority is computed lazily at retrieval time based on those goals -- it is not read from stored node properties.

Hybrid scoring combines graph distance, dimension weights, and on-the-fly relevance computation. Derivation Distance attenuation is applied: nodes further from observed evidence are ranked lower. State-based penalties are applied: Contested and Conflicted nodes are penalized or excluded by default.

Results are returned to Working Memory's Retrieved bucket as lightweight projected subgraphs, preserving the relational structure the agent needs for reasoning.

**Retrieval traceability**: Retrieval optionally emits a trace record for each result, capturing the dimension traversal path, scoring contributions per dimension, penalties applied, goal alignment basis, and Derivation Distance at time of retrieval. This record is not stored in the Persistent Core but is available for debugging, auditing agent behavior, and analyzing retrieval failure modes. The goal is full cognitive auditability: not just why a fact was certified, but why it was surfaced for a specific task.

### Reflection / Evolution

Reflection runs during idle time under configurable compute budgets and scheduling policies. It may defer, batch, or prioritize maintenance tasks based on system constraints and current goal pressure. It is not an omniscient daemon -- it is a resource-bounded maintenance process.

Reflection reads the Status Index to find Contested and Conflicted nodes requiring resolution. It analyzes the Persistent Core for weak dimension coverage, low-confidence nodes, abstraction opportunities, and novel synthesis possibilities.

Reflection proposes changes in two modes:

- **Novel synthesis** → nominates a **Hypothesis**-state node, routed through Court.
- **Grounded transformations** (rewires, abstraction promotions, contradiction resolutions) → nominates a **Candidate**-state node, routed through Court.

Reflection may also periodically recompute Derivation Distance estimates and nominate confidence recalibration candidates if materially different from the certified baseline.

Reflection proposes. Only the Court certifies.

---

## 8. Working Memory Eviction Policy

Working Memory is bounded by the context window. When the limit is reached, nodes are evicted from the Retrieved bucket in order of lowest priority (computed on the fly from current goals) combined with lowest confidence. The eviction order ensures the most epistemically valuable and goal-relevant nodes are retained longest.

The Generated bucket is never subject to eviction policy -- its contents flush immediately to Stage 0 and do not accumulate in Working Memory.

---

## 9. Locked Principles

These principles govern the entire architecture and resolve all known structural tensions.

- **Partial promotion is allowed.** A memory candidate does not need all eight dimensions populated. Only Semantic and Evidence anchor are mandatory. Gaps are flagged explicitly.
- **Reflection and abstraction outputs route through Court.** Nothing enters the Persistent Core directly from Reflection. Every proposal is a Court candidate.
- **Evidence tagging is required.** Every piece of evidence entering Stage 0 carries an Evidence Type tag. Every certified node carries a Validation Basis assigned by Court.
- **Relevance and Priority are retrieval-time computations.** Neither is stored as a static property. Both are computed on demand from current goals.
- **Confidence is anchored exclusively to Stage 0.** There are no circular references. Every confidence pointer traces to an immutable log entry.
- **Derivation Distance attenuates confidence.** Nodes derived through many inference hops from observed evidence are treated as less reliable. The attenuation is applied at retrieval time.
- **Node Kind governs contradiction behavior.** Events cannot be superseded. Principles require multi-source corroboration. State Facts and Procedures follow the full contradiction protocol.
- **Staleness is handled by bi-temporal validity.** No additional staleness mechanism is needed. The bi-temporal model on the Temporal dimension handles fact expiry and auto-invalidation.
- **All state transitions are Court-mediated.** The illegal transitions principle holds: any transition not listed in the state machine table is forbidden without explicit Court re-certification.
- **No facts are ever deleted.** Terminal states (Archived, Deprecated, Superseded) retain their nodes permanently for audit chain integrity. The evidence anchor for any certified fact must always be traceable.
- **Goal activation semantics are deferred.** The current architecture treats goals as full Core Nodes with no special lifecycle beyond standard state transitions. Active, suspended, satisfied, and competing goal semantics are a future extension point.
- **Behavioral stability is deferred.** Confidence models epistemic certainty, not pattern frequency. Distributional facts are a known limitation addressed in a future version.

---

## 10. Design Rationale: Key Decisions

**Why separate Stage 0 from the Persistent Core?**
Raw observation and validated truth are epistemically different. Conflating them is the root cause of memory hallucination in most agent architectures. Stage 0 is the ground truth. The Persistent Core is the interpreted structure built on top of it. Keeping them separate means the Persistent Core can be rebuilt, corrected, or re-validated from the immutable log.

**Why a formal state machine?**
Without formal state semantics, memory status is descriptive rather than normative. The state machine makes transitions auditable, retrieval deterministic, and governance enforceable. It is the difference between a note system and an epistemic control structure.

**Why eight orthogonal content dimensions?**
Because different aspects of a memory answer different questions and require different graph traversal strategies. Flattening them into a single embedding loses the ability to query "find memories that are causally related to X and temporally recent and involve entity Y." The orthogonality rules prevent dimension contamination and keep the representations clean.

**Why Node Kind as a separate field from dimensions?**
The eight content dimensions describe what a memory is about. Node Kind describes how it behaves under contradiction and validation. An Event and a State Fact can both have strong Temporal and Entity projections but must be treated completely differently when new conflicting evidence arrives. Conflating ontological type with content structure would require the contradiction protocol to infer behavior from dimension weights, which is fragile.

**Why lazy Priority computation?**
Storing Priority statically requires cascading updates whenever goals change. Goals change frequently during active agent operation. Lazy computation is more scalable and eliminates the need for a separate priority propagation mechanism.

**Why freeze Derivation Distance at certification?**
The certified Derivation Distance reflects the epistemic state of the graph at the moment of certification. This is part of the audit record. If later evidence changes the picture, Reflection can nominate a recalibration -- but the original certification record is preserved. Auditability requires that the basis for certification never be retroactively rewritten.

**Why is Reflection not trusted to write directly to the Persistent Core?**
Because Reflection generates Inferred content, which carries lower epistemic standing than Observed content. Reflection proposes; Court certifies. This is the central governance invariant of the architecture. Bypassing it for any operation, however well-intentioned, undermines the epistemic integrity of the entire Persistent Core.

**Why is the Court a single authority rather than distributed?**
A single authoritative Court is the correct design for epistemic consistency. Distributed or partitioned courts risk certifying contradictory facts in different partitions without awareness of each other. This is a known throughput limitation -- under high ingestion load, the Court becomes a bottleneck. Partitioned or hierarchical court designs are the expected direction for a future scalability iteration, but the current design correctly prioritizes correctness over throughput.

---

## 11. Known Limitations and Future Work

**Behavioral stability**: The current confidence model captures epistemic certainty but not behavioral frequency. A fact like "the user usually prefers concise answers" is a distributional pattern, not a binary truth claim. Confidence alone cannot distinguish between a strong single observation and a weak recurring pattern. A dedicated Stability or Consistency field is the expected solution, deferred to a future version.

**Goal activation semantics**: Goals are currently Core Nodes with standard lifecycle states. Active, suspended, satisfied, competing, and failed goal states are not yet modeled. This is sufficient for single-goal agent operation but will become a limitation in multi-goal or hierarchical goal scenarios.

**Court throughput**: The single-authority Court design prioritizes correctness. Under high ingestion load it will become a throughput bottleneck. Partitioned courts, domain-specialized courts, or hierarchical certification are the expected direction for a future scalability version.

**Concurrent task handling**: The architecture specifies a single Working Memory instance per task. Multiple simultaneous tasks would require multiple Working Memory instances. Isolation and shared-retrieval semantics for concurrent tasks are not yet addressed.

**Probabilistic and fuzzy reasoning**: The architecture currently treats truth as propositional. Probabilistic, fuzzy, or confidence-interval-based reasoning is not yet modeled at the graph layer. Confidence scores approximate this but do not replace it.

---

## Summary

The Court of Core Memory architecture is a closed-loop, formally governed memory system for autonomous agents. Its central properties are:

- **Auditability**: Every certified fact traces to immutable observed evidence.
- **Epistemic grounding**: Confidence, Derivation Distance, Evidence Type, and Validation Basis together form a complete epistemic metadata model.
- **Behavioral classification**: Node Kind ensures that Events, State Facts, Principles, Goals, and Procedures are governed by appropriate contradiction and validation logic.
- **Multi-dimensional retrieval**: Eight orthogonal graph layers enable policy-guided traversal across causal, temporal, semantic, procedural, and other dimensions simultaneously.
- **Retrieval traceability**: Optional trace output for each retrieval result enables full cognitive auditability of agent behavior.
- **Contradiction safety**: A three-type contradiction protocol with formal state transitions and Node Kind constraints prevents competing truths from accumulating without resolution.
- **Abstraction lineage**: Promotions through the abstraction hierarchy are fully traceable via derived-from and abstracted-into edges.
- **Governance consistency**: One authority (the Court) mediates all state transitions. Reflection proposes. Court decides.

The architecture is ready for implementation. The next meaningful insights will come from concrete lifecycle traces, contradiction resolution walkthroughs, retrieval simulations, and implementation stress tests -- not further top-level design.

**Scope note (design-only repo):** The eight orthogonal graph layers are the intended conceptual model for the Persistent Core. The v1 storage schema in this repository specifies Court-governed `Node` records plus append-only Stage 0 logs and minimal indexes; a concrete persisted representation of the eight projection graphs is deferred to a later implementation-focused revision.

## License

MIT License

Copyright (c) 2026 Ramsyana

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
