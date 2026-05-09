# COURT_SKELETON.md

This document defines the `Court` class skeleton — the reconciler loop,
state machine driver, and rule engine entry points.

It is 100 % aligned with COURT_DESIGN.md v5, STATE_MACHINE.md v4,
PERSISTENT_CORE_SCHEMA.md v3, and PROMPT_VERIFICATION_WRAPPER.md v1.

---

## 1. PersistentCoreStore Interface

```python
from abc import ABC, abstractmethod
from typing import List, Optional
from court.schema import Node, NodeKind, NodeState, EntityTimeIndexRecord


class PersistentCoreStore(ABC):

    @abstractmethod
    async def get_node(self, node_id: str) -> Optional[Node]: ...

    @abstractmethod
    async def write_node(self, node: Node) -> None: ...

    @abstractmethod
    async def query_entity_time_index(
        self,
        primary_entity: str,
        kind: NodeKind,
        time_window: Optional[str],
        states: List[NodeState],
    ) -> List[EntityTimeIndexRecord]: ...

    @abstractmethod
    async def update_entity_time_index(self, record: EntityTimeIndexRecord) -> None: ...

    @abstractmethod
    async def update_edges_index(
        self,
        source_id: str,
        target_id: str,
        relation: str,
    ) -> None: ...
```

---

## 2. Court Configuration + Exceptions

```python
from dataclasses import dataclass
from typing import Optional


@dataclass
class CourtConfig:
    batch_window_ms: int        = 500
    nomination_queue_depth: int = 20
    llm_model: str              = "claude-sonnet-4-20250514"
    llm_max_tokens: int         = 1024

    # Semantic similarity bands — set after empirical calibration.
    # Until set, the contradiction scan skips band classification
    # and flags all structural candidates for human review.
    # See COURT_DESIGN.md §1 — do not hardcode these values.
    high_band:   Optional[float] = None   # score >= high_band   → Contested
    middle_band: Optional[float] = None   # score >= middle_band → Conflicted


class CourtError(Exception):             pass
class CourtQueueFullError(CourtError):   pass
class NodeNotFoundError(CourtError):     pass
class InvalidTransitionError(CourtError):pass
class CourtInvariantError(CourtError):   pass
```

---

## 3. Court Class

```python
import asyncio
import uuid
from datetime import datetime, timezone
from typing import Literal, Optional

from court.schema import (
    Node, NodeState, NodeKind, ValidationBasis,
    EntityTimeIndexRecord,
)
from court.wrapper import (
    PromptVerificationWrapper,
    write_human_resolution,
    node_kind_verifier,
    semantic_similarity_verifier,
)
from court.stage0 import Stage0Writer


class Court:
    """
    Deterministic rule engine with narrow, auditable LLM subroutines.
    Single source of all Persistent Core state transitions.

    Do not instantiate more than one Court per agent in v1.
    Concurrent Court instances against the same store are out of scope
    (see §4 Out of Scope).
    """

    def __init__(
        self,
        store: PersistentCoreStore,
        stage0: Stage0Writer,
        config: CourtConfig = CourtConfig(),
    ) -> None:
        self._store   = store
        self._stage0  = stage0
        self._config  = config
        self._wrapper = PromptVerificationWrapper(
            stage0=stage0,
            model=config.llm_model,
            max_tokens=config.llm_max_tokens,
        )
        self._queue: asyncio.Queue[Node] = asyncio.Queue(
            maxsize=config.nomination_queue_depth
        )
        self._debounce_task:   Optional[asyncio.Task] = None
        self._background_task: Optional[asyncio.Task] = None

    # ── Public API ────────────────────────────────────────────────────────────

    async def nominate(self, node: Node) -> None:
        """
        Submit a node to the nomination queue.
        Triggers a debounced run_cycle after batch_window_ms.
        Raises CourtQueueFullError if the queue is at capacity.
        """
        try:
            self._queue.put_nowait(node)
        except asyncio.QueueFull:
            raise CourtQueueFullError(
                f"Nomination queue full (depth={self._config.nomination_queue_depth}). "
                "Reduce nomination rate or increase CourtConfig.nomination_queue_depth."
            )
        await self._schedule_debounced_cycle()

    async def run_cycle(self) -> None:
        """
        Process all pending nominations.
        Manual escape hatch — safe for tests and explicit control flows.
        """
        while not self._queue.empty():
            node = self._queue.get_nowait()
            try:
                await self._process_nomination(node)
            finally:
                self._queue.task_done()

    def start_background_reconciler(self, interval_seconds: float = 5.0) -> None:
        """
        Opt-in background reconciler. Calls run_cycle on a fixed interval.
        Only one background reconciler per Court instance.
        """
        if self._background_task and not self._background_task.done():
            return
        self._background_task = asyncio.create_task(
            self._background_loop(interval_seconds)
        )

    def stop_background_reconciler(self) -> None:
        if self._background_task:
            self._background_task.cancel()

    async def resolve_contested(
        self,
        node_id: str,
        decision: Literal["certify", "supersede", "defer"],
        resolver: str = "human",
    ) -> None:
        """
        Privileged human or agent override for Contested or Conflicted nodes.
        Bypasses Court completeness checks — use only when evidence is
        verified out-of-band. Always produces a Stage 0 Observed-Internal entry.
        See STATE_MACHINE.md §3 and §5.
        """
        node = await self._store.get_node(node_id)
        if node is None:
            raise NodeNotFoundError(node_id)

        if node.state not in (NodeState.CONTESTED, NodeState.CONFLICTED):
            raise InvalidTransitionError(
                f"resolve_contested only applies to Contested/Conflicted nodes "
                f"(got state={node.state!r} for node_id={node_id!r})"
            )

        await write_human_resolution(
            stage0=self._stage0,
            node_id=node_id,
            decision=decision,
            resolver=resolver,
        )

        if decision == "defer":
            return  # no state change; human_review_required stays true

        if decision == "certify":
            await self._apply_transition(
                node,
                new_state=NodeState.CERTIFIED,
                validation_basis=ValidationBasis.HUMAN_OVERRIDE,
                clear_human_review=True,
            )

        elif decision == "supersede":
            await self._apply_transition(
                node,
                new_state=NodeState.SUPERSEDED,
                clear_human_review=True,
            )

    # ── Private: reconciler ───────────────────────────────────────────────────

    async def _schedule_debounced_cycle(self) -> None:
        if self._debounce_task and not self._debounce_task.done():
            self._debounce_task.cancel()
        self._debounce_task = asyncio.create_task(
            self._debounced_run(self._config.batch_window_ms / 1000)
        )

    async def _debounced_run(self, delay: float) -> None:
        await asyncio.sleep(delay)
        await self.run_cycle()

    async def _background_loop(self, interval: float) -> None:
        while True:
            await asyncio.sleep(interval)
            await self.run_cycle()

    # ── Private: nomination pipeline ──────────────────────────────────────────

    async def _process_nomination(self, node: Node) -> None:
        """
        Full Court pipeline for one nominated node.
        Order is fixed and must not be changed.
        """
        # 1. Completeness check — pure rule engine, no LLM
        if not self._completeness_check(node):
            await self._apply_transition(node, NodeState.DEPRECATED)
            return

        # 2. Node kind assignment — LLM proposes, Court ratifies
        node = await self._assign_node_kind(node)

        # 3. Contradiction scan — structural filter first, LLM only if needed
        conflict = await self._contradiction_scan(node)
        if conflict:
            target_state, conflict_node_id = conflict
            await self._apply_transition(
                node,
                new_state=target_state,
                conflict_node_id=conflict_node_id,
            )
            return

        # 4. Validation basis assignment — pure rule engine, no LLM
        node = self._assign_validation_basis(node)

        # 5. Certify
        await self._apply_transition(node, NodeState.CERTIFIED)

    # ── Private: rule engine steps ────────────────────────────────────────────

    def _completeness_check(self, node: Node) -> bool:
        """
        Deterministic. No LLM.
        Returns False → node transitions to Deprecated.
        """
        w = node.five_w1h_plus_2
        if node.state == NodeState.CANDIDATE:
            if not w.what or not w.evidence:
                return False
        if not node.anchors:
            return False
        # TODO: expand kind-specific rules from README spec
        return True

    async def _assign_node_kind(self, node: Node) -> Node:
        """
        LLM proposes a NodeKind from node.content.
        Court ratifies if valid; keeps existing kind on override.
        """
        prompt = self._build_node_kind_prompt(node)
        result = await self._wrapper.run(
            subroutine_name="node-kind-proposal",
            prompt=prompt,
            verifier=node_kind_verifier,
            node_id_affected=node.node_id,
        )
        if result.verification.status == "pass":
            return node.model_copy(update={"kind": result.verification.parsed_output})
        return node

    async def _contradiction_scan(
        self, node: Node
    ) -> Optional[tuple[NodeState, str]]:
        """
        Step 1 (rule engine): structural candidates via entity-time index.
        Step 2 (LLM): semantic similarity, only if candidates exist.

        Confidence bands are read from CourtConfig and must be set via
        empirical calibration before production use (see COURT_DESIGN.md §1).
        Until bands are configured, all structural candidates are flagged
        for human review as Contested.

        Returns (target_state, conflicting_node_id) or None if clean.
        """
        candidates = await self._store.query_entity_time_index(
            primary_entity=node.primary_entity,
            kind=node.kind,
            time_window=node.time_window,
            states=[NodeState.CERTIFIED, NodeState.CONTESTED, NodeState.CONFLICTED],
        )

        if not candidates:
            return None

        for record in candidates:
            other = await self._store.get_node(record.node_id)
            if other is None or other.node_id == node.node_id:
                continue

            prompt = self._build_semantic_similarity_prompt(node, other)
            result = await self._wrapper.run(
                subroutine_name="semantic-similarity",
                prompt=prompt,
                verifier=semantic_similarity_verifier,
                node_id_affected=node.node_id,
            )

            if result.verification.status != "pass":
                # Hard verifier failure → flag for human review
                return (NodeState.CONTESTED, other.node_id)

            score: float = result.verification.parsed_output

            if self._config.high_band is not None and score >= self._config.high_band:
                return (NodeState.CONTESTED, other.node_id)
            if self._config.middle_band is not None and score >= self._config.middle_band:
                return (NodeState.CONFLICTED, other.node_id)
            if self._config.high_band is None and self._config.middle_band is None:
                # Bands unconfigured — conservative fallback
                return (NodeState.CONTESTED, other.node_id)

        return None

    def _assign_validation_basis(self, node: Node) -> Node:
        """
        Pure rule engine lookup table.
        ProposedSynthesis nodes (from Reflection) are passed through unchanged.
        """
        if node.validation_basis == ValidationBasis.PROPOSED_SYNTHESIS:
            return node
        if any(a.startswith("stage0:") for a in node.anchors):
            return node.model_copy(
                update={"validation_basis": ValidationBasis.STAGE_0_ANCHOR}
            )
        if node.anchors:
            return node.model_copy(
                update={"validation_basis": ValidationBasis.CERTIFIED_ANCHOR}
            )
        # Should never reach here after completeness check passes
        raise CourtInvariantError(
            f"Node {node.node_id} passed completeness check but has no valid anchors."
        )

    async def _apply_transition(
        self,
        node: Node,
        new_state: NodeState,
        validation_basis: Optional[ValidationBasis] = None,
        clear_human_review: bool = False,
        conflict_node_id: Optional[str] = None,
    ) -> Node:
        """
        Create a new immutable node version reflecting the state transition.
        Update the entity-time index.
        No LLM call — callers have already logged via the wrapper.
        """
        updates: dict = {
            "node_id": str(uuid.uuid4()),
            "state": new_state,
            "previous_version_id": node.node_id,
            "created_at": datetime.now(timezone.utc),
        }
        if validation_basis is not None:
            updates["validation_basis"] = validation_basis
        if clear_human_review:
            updates["human_review_required"] = False
        if new_state in (NodeState.CONTESTED, NodeState.CONFLICTED):
            updates["human_review_required"] = True

        if conflict_node_id:
            # TODO: create appropriate edge in edges index
            pass

        new_node = node.model_copy(update=updates)
        await self._store.write_node(new_node)
        await self._store.update_entity_time_index(
            EntityTimeIndexRecord(
                node_id=new_node.node_id,
                primary_entity=new_node.primary_entity,
                kind=new_node.kind,
                state=new_node.state,
                time_window=new_node.time_window,
                created_at=new_node.created_at,
            )
        )
        return new_node

    # ── Prompt builders (stubs) ───────────────────────────────────────────────

    def _build_node_kind_prompt(self, node: Node) -> str:
        # Must instruct the LLM to return exactly one NodeKind value on the first line.
        raise NotImplementedError("Implement prompt template")

    def _build_semantic_similarity_prompt(self, a: Node, b: Node) -> str:
        # Must instruct the LLM to return a float [0.0, 1.0] on the first line.
        raise NotImplementedError("Implement prompt template")
```

---

## 4. Out of Scope for v1

- Concurrent task handling and task-level isolation
- Multi-agent shared Court service
- Behavioral stability / convergence guarantees
- Goal activation and motivation semantics
- Retry logic on LLM errors (caller responsibility)
- Edge creation in `_apply_transition` (marked TODO; implement alongside prompt templates)

---

**This is the single source of truth for Court class structure.**  
All rule engine logic, LLM subroutine calls, and state transitions must be
implemented inside this class against the pipeline defined in `_process_nomination`.  
Direct storage writes outside `_apply_transition` are forbidden.