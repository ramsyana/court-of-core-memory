# PROMPT_TEMPLATES.md

This document defines the v0 prompt templates for Court-internal LLM subroutines.

Alignment:

- COURT_SKELETON.md
- PROMPT_VERIFICATION_WRAPPER.md
- PERSISTENT_CORE_SCHEMA.md

All prompts are executed as **single user messages** (no separate system role). Therefore, each prompt begins with an embedded “System Instructions” section.

## 0. Global Constraints (applies to every subroutine)

- The verifier reads **only the first line** of the raw model output.
- Therefore every prompt must:
  - Force the correct **first-line** output format.
  - Allow optional rationale only on **subsequent lines**.
- Output must not include code blocks.
- Output must not include JSON.
- Output must not include surrounding quotes.

## 1. Output Contracts (Strict)

### 1.1 `node-kind-proposal`

**Verifier:** `node_kind_verifier` (PROMPT_VERIFICATION_WRAPPER.md §6)

- **Line 1:** exactly one of the `NodeKind` enum values:
  - `Event`
  - `State Fact`
  - `Principle`
  - `Goal`
  - `Procedure`
- **Lines 2+:** optional brief rationale (ignored by verifier)

**Failure behavior in Court:**

- If verifier returns `override`, Court keeps the node’s existing `kind` (COURT_SKELETON.md `_assign_node_kind`).

### 1.2 `conflict-likelihood`

**Verifier:** `semantic_similarity_verifier` (PROMPT_VERIFICATION_WRAPPER.md §6)

- **Line 1:** a single float parseable by Python `float()` in `[0.0, 1.0]`.
  - Recommended formatting: `0.00` to `1.00` (two decimals), but any valid float is acceptable.
- **Lines 2+:** optional brief rationale (ignored by verifier)

**Semantic definition (v0):**

- The score represents **conflict likelihood under Court use**: how likely Observation A and B should be treated as semantically conflicting given the Court has already found a structural candidate match.
- It is **not** a generic text similarity score.

**Failure behavior in Court:**

- If the verifier raises `VerificationError` (unparseable or out-of-range), the wrapper logs then re-raises; Court treats non-pass as conservative human review by returning `Contested` (COURT_SKELETON.md `_contradiction_scan`).

## 2. Prompt Templates

### 2.1 `node-kind-proposal` Prompt Template

```text
SYSTEM INSTRUCTIONS (Court Subroutine: node-kind-proposal)
- You are a classification component used inside a deterministic rule engine.
- You must output the answer on the FIRST LINE ONLY.
- The FIRST LINE must be exactly one of these values (case and spacing must match):
  Event
  State Fact
  Principle
  Goal
  Procedure
- Do not output anything else on the first line.
- Do not output JSON.
- Do not output code blocks.
- Do not wrap the answer in quotes.
- If uncertain, choose the single best category.
- You may add a brief rationale starting on line 2.

TASK
Classify the observation into exactly one category.

CATEGORY DEFINITIONS
- Event:
  - A specific occurrence or change at a point in time.
  - Often fits "something happened".
- State Fact:
  - A persistent property/state about an entity (may be time-bounded).
  - Often fits "X is/has/was".
- Principle:
  - A general rule, invariant, or normative statement.
  - Often fits "In general" / "Always" / "Never".
- Goal:
  - A desired outcome or intention.
  - Often fits "wants to" / "aims to" / "should achieve".
- Procedure:
  - A method, steps, or algorithm for doing something.
  - Often fits "To do X, follow steps".

INPUT
Observation content:
"""
{content}
"""

Primary entity: {primary_entity}
Time window: {time_window}
FiveW1H.what: {what}
FiveW1H.when: {when}
FiveW1H.where: {where}

RESPONSE FORMAT
Line 1: one of [Event | State Fact | Principle | Goal | Procedure]
Line 2+: optional rationale
```

**Notes (v0):**

- This template is designed to be robust to the `node_kind_verifier`, which only checks the first line.
- Including a few `FiveW1HPlusTwo` fields helps disambiguate Event vs State Fact without requiring multi-turn prompting.

### 2.2 `conflict-likelihood` Prompt Template

```text
SYSTEM INSTRUCTIONS (Court Subroutine: conflict-likelihood)
- You are a scoring component used inside a deterministic rule engine.
- You must output the score on the FIRST LINE ONLY.
- The FIRST LINE must be a single decimal number parseable as a Python float.
- The score must be between 0.0 and 1.0 inclusive.
- Do not output JSON.
- Do not output code blocks.
- Do not wrap the number in quotes.
- You may add a brief rationale starting on line 2.

TASK
Given two observations, estimate CONFLICT LIKELIHOOD: how likely they express incompatible claims about the same underlying subject, assuming the Court has already flagged them as structural candidates.

SCORING RUBRIC (Conflict Likelihood)
- 1.0:
  - Direct negation or mutually exclusive claims.
  - Example shape: "X is true" vs "X is false" for the same entity/time.
- 0.7–0.9:
  - Strong tension: claims are difficult to reconcile without additional assumptions.
- 0.4–0.6:
  - Mild inconsistency or ambiguity: could be compatible, could be conflict.
- 0.1–0.3:
  - Likely compatible: different aspects or scopes; low chance of conflict.
- 0.0:
  - Clearly compatible or unrelated.

IMPORTANT
- Focus on semantic compatibility/incompatibility, not surface text similarity.
- When uncertain, choose a conservative score near the middle (around 0.5).

INPUT
Observation A:
"""
{content_a}
"""
Primary entity: {primary_entity_a}
Time window: {time_window_a}
FiveW1H.what: {what_a}
FiveW1H.when: {when_a}

Observation B:
"""
{content_b}
"""
Primary entity: {primary_entity_b}
Time window: {time_window_b}
FiveW1H.what: {what_b}
FiveW1H.when: {when_b}

RESPONSE FORMAT
Line 1: a float in [0.0, 1.0] (example: 0.82)
Line 2+: optional rationale
```

## 3. Minimal Input Field Policy

To keep the subroutines narrow and auditable, the prompt should include only:

- Node-level:
  - `content`
  - `primary_entity`
  - `time_window` (string or literal placeholder like `None`)
- Selected `FiveW1HPlusTwo` fields:
  - `what`, `when`, optionally `where`

It should not include:

- Full Stage 0 logs
- Full edges lists
- Any hidden chain-of-thought request

## 4. Prompt Iteration Notes

- If the model occasionally outputs extra leading whitespace, that is acceptable (verifiers strip).
- If the model sometimes outputs prefixes like `Score: 0.73`, that will FAIL the similarity verifier. If seen in practice, add stronger formatting warnings and a short negative example.
- If the model sometimes outputs markdown (e.g., `**Event**`), that will FAIL the node-kind verifier. Add explicit "no markdown" instruction if needed.
