# Component Methods: Storywell

**Stage**: INCEPTION — Application Design
**Date**: 2026-11-22

Method signatures and high-level purpose only. Detailed business rules (exact prompt content,
schemas, round-ordering algorithm, etc.) are deferred to Functional Design, per unit, in
Construction.

---

## Foundation Unit

### `app/store.py`

```python
class SessionState:
    def mark_dependents_stale(self, *, changed: Literal["resume", "requirements"]) -> None:
        """Set the stale flags for whichever downstream results depend on what changed (FR-903)."""

    def clear_stale(self, flag: Literal["matches", "suggestions", "cover_letter"]) -> None:
        """Called by a slice's own regeneration function once it has produced a fresh result."""

    def is_export_blocked(self) -> bool:
        """True if any stale flag is set (FR-904)."""

class SessionStore:
    def create(self) -> SessionState: ...
    def get(self, sid: str) -> SessionState:
        """Raises SessionNotFound -> API layer returns 404 (NFR-203)."""
    def expire_inactive(self) -> None: ...
```

### `app/llm/client.py`

```python
class LLMClient:
    async def structured[T: BaseModel](
        self, *, system: str, user: str, schema: type[T], effort: Literal["medium", "high"]
    ) -> T:
        """Raises LLMError on refusal/max_tokens/no parsed output (NFR-303)."""

class FakeLLMClient(LLMClient):
    """Returns pre-registered fixed Pydantic objects; used by all unit/API tests (NFR-501)."""
```

### `app/verification/suggestions.py`

```python
def find_unsupported_numbers(proposed: str, sources: list[str]) -> list[str]: ...
def find_verb_inflation(proposed: str, sources: list[str]) -> list[str]: ...
def verify_suggestion(
    proposed: str, source_ids: list[str], known: dict[str, str]
) -> list[str]:
    """Returns flags; empty list means the suggestion passed (FR-420-423)."""
```

### `app/verification/reordering.py`

```python
def verify_reorder(proposed_order: list[str], existing_ids: list[str]) -> bool:
    """True iff proposed_order is an exact permutation of existing_ids (FR-503-504)."""
```

### `app/verification/matching.py`

```python
def verify_match_evidence(status: str, evidence_ids: list[str]) -> bool:
    """True iff the found<->non-empty and not_found<->empty invariants both hold (FR-222-223)."""
```

### `app/services/shared.py`

```python
async def create_confirmed_fact(
    session: SessionState, *, question: str, answer: str
) -> ConfirmedFact:
    """Used by follow-up answers (FR-320) and suggestion-screen confirmation (FR-330)."""

def build_final_resume_content(session: SessionState) -> list[ResumeItem]:
    """Assembles the 'final resume content' defined term (requirements.md §5):
    accepted/edited suggestion text, original text for rejected/pending/unsuggested lines,
    in accepted bullet order. Used by Cover Letter (FR-701) and Export (FR-805)."""

def get_style_context(session: SessionState) -> tuple[StyleChoice, str | None]:
    """Returns (style, writing_sample) for prompt construction (FR-601-604); used by
    Suggestions & Verification and Cover Letter."""
```

---

## Resume Pipeline Slice

### `app/services/resume.py`

```python
def validate_upload(filename: str, content: bytes) -> None:
    """Raises UploadRejected on bad extension/signature/size/page-count/likely-scanned-PDF
    (FR-111-114)."""

def parse_resume(content: bytes | str, source: Literal["pdf", "docx", "text"]) -> list[ResumeItem]:
    """FR-120."""

def apply_corrections(
    session: SessionState, edits: list[ResumeItemEdit]
) -> list[ResumeItem]:
    """Edit/merge/split/delete (FR-130). Calls session.mark_dependents_stale(changed="resume")
    if matching has already run."""
```

---

## JD & Matching Slice

### `app/services/matching.py`

```python
def validate_jd_text(text: str) -> None:
    """Raises if over 15,000 characters (FR-113)."""

async def extract_requirements(llm: LLMClient, jd_text: str) -> list[str]:
    """FR-210."""

def apply_requirement_edits(
    session: SessionState, edits: list[RequirementEdit]
) -> list[str]:
    """FR-211. Calls session.mark_dependents_stale(changed="requirements") if matching has
    already run (FR-212)."""

async def match_requirements(
    llm: LLMClient, items: list[ResumeItem], facts: list[ConfirmedFact], jd_text: str
) -> list[RequirementMatch]:
    """FR-220-224. Calls verification/matching.py's verify_match_evidence() on every result
    internally before returning (Q4) — a found status with no evidence never reaches the
    caller."""
```

### `app/services/follow_up.py`

```python
async def generate_follow_up_questions(
    llm: LLMClient, matches: list[RequirementMatch]
) -> list[FollowUpQuestion]:
    """FR-301. Caller orders into rounds of 3 by JD emphasis (FR-310)."""

async def record_answer(
    session: SessionState, question: FollowUpQuestion, answer: str | None
) -> ConfirmedFact | None:
    """FR-320-321. answer=None means "no" (FR-321): returns None, creates no fact. Otherwise
    delegates to services/shared.py's create_confirmed_fact()."""
```

---

## Suggestions & Verification Slice

### `app/services/suggestions.py`

```python
async def generate_suggestions(
    llm: LLMClient, items: list[ResumeItem], matches: list[RequirementMatch],
    style: StyleChoice, writing_sample: str | None
) -> list[Suggestion]:
    """FR-401, FR-410-411. Calls verification/suggestions.py's verify_suggestion() on every
    candidate internally (Q4); drops invalid_source_ids candidates (FR-430), returning only
    the withheld count plus the remaining, flag-annotated suggestions."""

def decide_suggestion(
    session: SessionState, suggestion_id: str, decision: Literal["accept", "reject", "edit"],
    edited_text: str | None
) -> Suggestion:
    """FR-440-441, FR-412."""

async def confirm_flagged_suggestion_fact(
    llm: LLMClient, session: SessionState, suggestion_id: str, supporting_detail: str
) -> Suggestion:
    """FR-330-333. Calls services/shared.py's create_confirmed_fact(), adds the new fact's id
    to the suggestion's source_ids, then re-runs verify_suggestion() on only that suggestion
    (FR-331-332, FR-432)."""
```

### `app/services/reordering.py`

```python
async def generate_reorder_suggestions(
    llm: LLMClient, entry_id: str, items: list[ResumeItem]
) -> ReorderSuggestion | None:
    """FR-501-502. Calls verification/reordering.py's verify_reorder() internally (Q4); returns
    None (dropped) if the check fails (FR-503-504)."""

def decide_reorder(
    session: SessionState, reorder_id: str, decision: Literal["accept", "reject"]
) -> ReorderSuggestion:
    """FR-501."""
```

---

## Cover Letter Slice

### `app/services/cover_letter.py`

```python
async def draft_cover_letter(
    llm: LLMClient, session: SessionState, company: str | None, role_title: str | None
) -> str:
    """FR-701-706. Calls services/shared.py's build_final_resume_content() and
    get_style_context(); never infers company/role_title if not supplied (FR-703); generic
    opening fallback if absent (FR-704)."""
```

---

## Export Slice

### `app/services/export.py`

```python
def assemble_export_content(session: SessionState) -> ExportContent:
    """FR-805, FR-808. Calls services/shared.py's build_final_resume_content(); includes the
    cover letter draft if selected; reports the pending-suggestion count (FR-808)."""

def render_docx(content: ExportContent) -> bytes:
    """FR-803, FR-806 — python-docx, fixed single-column ATS template."""

def render_pdf(content: ExportContent) -> bytes:
    """FR-803, FR-806 — fpdf2, same template as render_docx()."""

def package_zip(files: dict[str, bytes]) -> bytes:
    """FR-802 — when both resume and cover letter are selected."""
```
