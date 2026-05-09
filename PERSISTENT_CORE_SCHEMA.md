# Persistent Core Schema

This document defines the authoritative data model for the **Persistent Core** and **Stage 0**.

Alignment:

- `COURT_DESIGN.md`
- `STATE_MACHINE.md`

## 1. Core Models

```python
from pydantic import BaseModel, Field, field_validator, model_validator
from datetime import datetime
from typing import Literal, List, Optional
from enum import Enum

class NodeState(str, Enum):
    HYPOTHESIS = "Hypothesis"
    CANDIDATE   = "Candidate"
    CERTIFIED   = "Certified"
    CONTESTED   = "Contested"
    CONFLICTED  = "Conflicted"
    SUPERSEDED  = "Superseded"
    ARCHIVED    = "Archived"
    DEPRECATED  = "Deprecated"

class ValidationBasis(str, Enum):
    STAGE_0_ANCHOR     = "Stage0Anchor"
    CERTIFIED_ANCHOR   = "CertifiedAnchor"
    PROPOSED_SYNTHESIS = "ProposedSynthesis"
    HUMAN_OVERRIDE     = "HumanOverride"

class NodeKind(str, Enum):
    EVENT      = "Event"
    STATE_FACT = "State Fact"
    PRINCIPLE  = "Principle"
    GOAL       = "Goal"
    PROCEDURE  = "Procedure"

class FiveW1HPlusTwo(BaseModel):
    """Explicit structure for completeness checks.
    All fields are Optional at the Pydantic level — partial data is valid
    for Hypothesis nodes. Mandatory field enforcement is the exclusive
    responsibility of the Court's rule engine per NodeKind and state
    (see STATE_MACHINE.md). At minimum, 'what' and 'evidence' are required
    for Candidate and Certified nodes; at least one Node anchor is also required
    at the Court rule-engine layer (v1 completeness: what + evidence + anchors).
    Temporal coverage ('when') is strongly preferred but not a baseline schema
    requirement."""
    who:                Optional[str]       = None
    what:               Optional[str]       = None
    when:               Optional[str]       = None
    where:              Optional[str]       = None
    why:                Optional[str]       = None
    how:                Optional[str]       = None
    evidence:           List[str]           = Field(default_factory=list)  # +1
    additional_context: Optional[str]       = None                         # +2

class Edge(BaseModel):
    target_node_id: str
    relation: Literal[
        "superseded-by", "abstracted-into",
        "supports", "contradicts", "contested-by"
    ]
    created_at: datetime = Field(default_factory=datetime.utcnow)

class Node(BaseModel):
    node_id:             str              = Field(..., description="UUID v7 or ULID")
    kind:                NodeKind
    state:               NodeState
    primary_entity:      str
    time_window:         Optional[str]   = None
    content:             str
    five_w1h_plus_2:     FiveW1HPlusTwo
    validation_basis:    ValidationBasis
    anchors:             List[str]       = Field(default_factory=list)  # Stage0 entry_ids or node_ids
    edges:               List[Edge]      = Field(default_factory=list)  # outgoing edges (canonical)
    human_review_required: bool          = False
    superseded_by:       Optional[str]   = None
    abstracted_into:     Optional[str]   = None
    previous_version_id: Optional[str]   = None  # immutable history chaining
    created_at:          datetime        = Field(default_factory=datetime.utcnow)

    # Canonicality note:
    # - `edges` is the canonical representation of relationships.
    # - `superseded_by` and `abstracted_into` are optional convenience mirrors for
    #   the two most common edge types. When present they must match the
    #   corresponding edge in `edges`.

    @field_validator("human_review_required")
    @classmethod
    def validate_human_review_flag(cls, v: bool, info):
        state = info.data.get("state")
        if v and state not in (NodeState.CONTESTED, NodeState.CONFLICTED):
            raise ValueError("human_review_required only allowed on Contested/Conflicted")
        return v

class ObservedInternal(BaseModel):
    """Stage 0 immutable log entry for every Court-internal LLM call or human resolution."""
    entry_id:                str                                              = Field(..., description="UUID v7 or ULID")
    kind:                    Literal["Observed-Internal"]                     = "Observed-Internal"
    source:                  Literal["court-subroutine", "human-resolution"]
    subroutine_name:         Optional[str]                                   = None
    full_prompt:             Optional[str]                                   = None  # required for court-subroutine
    raw_output:              Optional[str]                                   = None  # required for court-subroutine
    rule_verification_result: Literal["pass", "override", "pending-human-review"]
    confidence:              Optional[float]                                 = None
    resolver:                Optional[str]                                   = None
    decision:                Optional[Literal["certify", "supersede", "defer"]] = None
    timestamp:               datetime                                        = Field(default_factory=datetime.utcnow)
    node_id_affected:        Optional[str]                                   = None

    @model_validator(mode="after")
    def enforce_fields_for_court_subroutines(self):
        if self.source == "court-subroutine":
            if self.full_prompt is None:
                raise ValueError("full_prompt is required when source = 'court-subroutine'")
            if self.raw_output is None:
                raise ValueError("raw_output is required when source = 'court-subroutine'")
        return self
```

## 2. Storage Layout (MinIO / S3-compatible)

Root prefix: `persistent-core/`

```
persistent-core/
├── nodes/                       # immutable; state change = new node_id + previous_version_id link
│   └── {node_id}.json
├── stage0/                      # append-only; never mutated
│   └── {year}/{month}/{day}/
│       └── {entry_id}.json
├── indexes/
│   ├── entity-time-index.jsonl  # structural conflict detection
│   ├── edges-index.jsonl        # bidirectional edge traversal (Court guarantees consistency)
│   └── embedding-cache.jsonl    # cosine similarity cache (optional; format is implementation-defined;
│                                #   each record must contain at minimum node_id and embedding vector)
└── metadata/
    └── court-version.txt
```

### Immutability rule (enforced by the Court)

- Nodes are never updated in place.
- Any state change, supersession, or abstraction creates a new node with `previous_version_id` pointing to the prior version.
- Edges are stored on the source node (canonical) and mirrored in `edges-index.jsonl` for O(1) reverse traversal.

### Edge Index Record Format (`edges-index.jsonl`)

Each line is one JSON object. Reverse records swap `source_node_id` ↔ `target_node_id`.

```python
class EdgeIndexRecord(BaseModel):
    source_node_id: str
    target_node_id: str
    relation: Literal[
        "superseded-by", "abstracted-into",
        "supports", "contradicts", "contested-by"
    ]
    direction:  Literal["forward", "reverse"]
    created_at: datetime
```

**Forward example** (A contradicts B):
```json
{"source_node_id": "01j8k9...", "target_node_id": "01j8ka...", "relation": "contradicts", "direction": "forward", "created_at": "2026-05-09T16:03:00Z"}
```

**Reverse example** (same logical edge, IDs swapped):
```json
{"source_node_id": "01j8ka...", "target_node_id": "01j8k9...", "relation": "contradicts", "direction": "reverse", "created_at": "2026-05-09T16:03:00Z"}
```

### Entity-Time Index Record Format (`entity-time-index.jsonl`)

Used by the Court's structural conflict scan ("same entity + time window + conflicting State Fact").

```python
class EntityTimeIndexRecord(BaseModel):
    node_id:        str
    primary_entity: str
    kind:           NodeKind
    state:          NodeState
    time_window:    Optional[str] = None
    created_at:     datetime
```

**Example record:**
```json
{"node_id": "01j8k9...", "primary_entity": "user_preference", "kind": "State Fact", "state": "Certified", "time_window": "2026-05-01T00:00:00Z/2026-05-31T23:59:59Z", "created_at": "2026-05-09T15:45:12Z"}
```

---

**This is the single source of truth for the Persistent Core data model.**
All Court methods, storage writes, and index updates must comply with the schemas above.