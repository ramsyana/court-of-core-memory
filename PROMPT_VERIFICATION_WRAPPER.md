# PROMPT_VERIFICATION_WRAPPER.md

This document defines the reusable `PromptVerificationWrapper` — the mandatory harness
through which **every Court-internal LLM subroutine** must pass.

It is 100 % aligned with COURT_DESIGN.md v5, STATE_MACHINE.md v4, and
PERSISTENT_CORE_SCHEMA.md v3.

---

## 1. Design Principles

- **Always log.** Stage 0 is written in a `finally` block — even on LLM error or verification failure.
- **Verifier is injected.** The wrapper never knows domain correctness rules.
- **LLM call is isolated.** All client concerns live in one private method.
- **No silent swallowing.** Errors are logged then re-raised. The caller decides recovery.

---

## 2. Supporting Types

```python
from __future__ import annotations

from dataclasses import dataclass
from typing import Any, Callable, Literal, Optional
from datetime import datetime, timezone

@dataclass
class VerificationResult:
    status: Literal["pass", "override", "pending-human-review"]
    parsed_output: Any = None
    confidence: Optional[float] = None   # contradiction-local score when applicable; not node Confidence
    override_reason: str = ""          # always present; empty when status == "pass"


@dataclass
class SubroutineResult:
    raw_output: str
    verification: VerificationResult
    log_entry_id: str
```

## 3. Stage0Writer Interface

```python
from abc import ABC, abstractmethod
from court.schema import ObservedInternal

class Stage0Writer(ABC):
    @abstractmethod
    async def append(self, entry: ObservedInternal) -> None:
        """Append one immutable Observed-Internal entry."""
        ...
```

## 4. PromptVerificationWrapper

```python
import uuid
import anthropic
from datetime import datetime, timezone
from court.schema import ObservedInternal
from .types import VerificationResult, SubroutineResult, VerificationError   # VerificationError defined below

class PromptVerificationWrapper:
    DEFAULT_MODEL = "claude-sonnet-4-20250514"
    DEFAULT_MAX_TOKENS = 1024

    def __init__(
        self,
        stage0: Stage0Writer,
        model: str = DEFAULT_MODEL,
        max_tokens: int = DEFAULT_MAX_TOKENS,
    ) -> None:
        self._client = anthropic.AsyncAnthropic()
        self._stage0 = stage0
        self._model = model
        self._max_tokens = max_tokens

    async def run(
        self,
        subroutine_name: str,
        prompt: str,
        verifier: Callable[[str], VerificationResult],
        node_id_affected: Optional[str] = None,
    ) -> SubroutineResult:
        """
        Execute one LLM subroutine with mandatory audit logging.
        """
        raw_output: str = ""
        verification = VerificationResult(status="pending-human-review")
        entry_id = str(uuid.uuid4())

        try:
            raw_output = await self._call_llm(prompt)
            verification = verifier(raw_output)

        except anthropic.APIError as exc:
            raw_output = f"ERROR: {exc}"
            verification = VerificationResult(
                status="pending-human-review",
                override_reason=f"LLM API error: {exc}",
            )
            raise

        except VerificationError as exc:
            raw_output = f"ERROR: {exc}"
            verification = VerificationResult(
                status="pending-human-review",
                override_reason=f"Verifier error: {exc}",
            )
            raise

        finally:
            entry = ObservedInternal(
                entry_id=entry_id,
                source="court-subroutine",
                subroutine_name=subroutine_name,
                full_prompt=prompt,
                raw_output=raw_output,
                rule_verification_result=verification.status,
                confidence=verification.confidence,
                resolver=None,
                decision=None,
                node_id_affected=node_id_affected,
                timestamp=datetime.now(timezone.utc),
            )
            await self._stage0.append(entry)

        return SubroutineResult(
            raw_output=raw_output,
            verification=verification,
            log_entry_id=entry_id,
        )

    async def _call_llm(self, prompt: str) -> str:
        response = await self._client.messages.create(
            model=self._model,
            max_tokens=self._max_tokens,
            messages=[{"role": "user", "content": prompt}],
        )
        return response.content[0].text
```

## 5. VerificationError & Human Resolution Helper

```python
class VerificationError(Exception):
    """Raised by verifiers on hard structural failures."""
    pass


# Helper for human resolutions (keeps Stage 0 uniform)
async def write_human_resolution(
    stage0: Stage0Writer,
    node_id: str,
    decision: Literal["certify", "supersede", "defer"],
    resolver: str,
    reason: str = "",
) -> str:
    entry = ObservedInternal(
        entry_id=str(uuid.uuid4()),
        source="human-resolution",
        subroutine_name=None,
        full_prompt=None,
        raw_output=reason,
        rule_verification_result="override",   # human always overrides
        confidence=None,
        resolver=resolver,
        decision=decision,
        node_id_affected=node_id,
        timestamp=datetime.now(timezone.utc),
    )
    await stage0.append(entry)
    return entry.entry_id
```

## 6. Example Verifiers

```python
def node_kind_verifier(raw_output: str) -> VerificationResult:
    first_line = raw_output.strip().splitlines()[0].strip()
    valid_kinds = {k.value for k in NodeKind}
    if first_line in valid_kinds:
        return VerificationResult(
            status="pass",
            parsed_output=NodeKind(first_line),
        )
    return VerificationResult(
        status="override",
        override_reason=f"Unknown kind '{first_line}'",
    )


def semantic_similarity_verifier(raw_output: str) -> VerificationResult:
    first_line = raw_output.strip().splitlines()[0].strip()
    try:
        score = float(first_line)
    except ValueError:
        raise VerificationError(f"Unparseable score: '{first_line}'")

    if not (0.0 <= score <= 1.0):
        raise VerificationError(f"Score out of range: {score}")

    return VerificationResult(
        status="pass",
        parsed_output=score,
        confidence=score,  # stored for audit of the subroutine, not as node Confidence
    )


def evidence_justification_verifier(
    raw_output: str,
    min_length: int = 200,
) -> VerificationResult:
    text = raw_output.strip()
    if len(text) >= min_length:
        return VerificationResult(
            status="pass",
            parsed_output=text,
        )
    return VerificationResult(
        status="override",
        override_reason=f"Justification too short; requires at least {min_length} characters.",
    )
```

## 7. Out of Scope for v1

- Retry logic (caller responsibility)
- Streaming responses
- Multi-turn prompts
- Token usage / cost tracking

**Single source of truth:** Every LLM call inside the Court **must** go through `PromptVerificationWrapper.run()`. Direct client usage is forbidden.

---
