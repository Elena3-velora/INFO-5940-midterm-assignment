# Services: Storywell

**Stage**: INCEPTION — Application Design
**Date**: 2026-11-22

This document describes the **service layer pattern** itself — how requests flow through the
system — rather than re-listing every method (see `component-methods.md` for signatures).

---

## The Orchestration Pattern (applies to every slice)

Per `technical-environment.md` NFR-602 ("route handlers contain no business logic") and the
plan's **Q4** answer, every endpoint follows the same shape:

```
API route handler (app/api/*.py)
  1. look up the session (store.get(sid)) -> 404 if not found/expired (NFR-203)
  2. call exactly one services/*.py function, passing in whatever the session/request holds
  3. translate a raised domain exception (LLMError, UploadRejected, SessionNotFound, ...)
     into the appropriate HTTP error
  4. return the Pydantic response model
```

The called service function is itself responsible for:
- Calling the LLM (via the injected `LLMClient`), if it needs to generate something.
- Calling the relevant `app/verification/*` check **internally**, before returning anything
  the LLM proposed (Q4) — so there is no code path, anywhere in the system, that returns an
  unverified `Suggestion`, `ReorderSuggestion`, or `found`-status `RequirementMatch`.
- Updating `SessionState` (storing the result, clearing/setting staleness flags).

This keeps the API layer uniform across all five feature slices and the foundation unit, which
matters for three developers building slices in parallel without needing to agree on
per-slice conventions.

---

## Service Definitions by Unit

### Foundation Unit Services
- **`services/shared.py`** — the cross-slice shared logic named in Q3: confirmed-fact
  creation (used by two different slices' endpoints), final-resume-content assembly (used by
  two different slices), and style/writing-sample access (used by two different slices). This
  module has no endpoint of its own; it exists purely to be called by other slices' services,
  which is why it lives in the Foundation unit rather than being owned by whichever slice
  happens to need it most.
- **`verification/*`** — not "services" in the orchestration sense above (they're pure
  functions, no session or LLM access) but are the shared safety layer every generation
  service calls into.

### Resume Pipeline Service
- **`services/resume.py`** orchestrates: validate → parse → (later) apply corrections. Two
  endpoints, two entry points into this one module.

### JD & Matching Services
- **`services/matching.py`** orchestrates extraction and matching, including triggering
  re-matching when FR-212 requires it.
- **`services/follow_up.py`** orchestrates question generation and answer recording, calling
  into `services/shared.py` for the actual fact creation. This is a deliberate split: question
  *generation/ordering* is JD & Matching's own concern, but *fact creation* is shared with
  Suggestions & Verification, so only that narrow piece delegates to the Foundation unit.

### Suggestions & Verification Services
- **`services/suggestions.py`** and **`services/reordering.py`** are separate (Q1), so that
  Suggestions (Must priority) and Reordering (Could priority) can be built, tested, and — if
  time runs short — cut independently without touching each other's code.
- `confirm_flagged_suggestion_fact()` is the one function that reaches into `services/shared.py`
  for fact creation and then re-runs `verification/suggestions.py` on a single suggestion — the
  concrete implementation of the FR-330 cross-slice dependency's resolution (see
  `component-dependency.md`).

### Cover Letter Service
- **`services/cover_letter.py`** is a thin orchestrator by design: almost everything it needs
  (final resume content, style context) comes from `services/shared.py`. Its own logic is
  narrow — the company/role-title "never infer" rule (FR-703) and the generic-opening fallback
  (FR-704).

### Export Service
- **`services/export.py`** follows Q5: one content-assembly function fans out to two
  format-specific renderers, so DOCX and PDF cannot silently drift apart from each other. The
  assembly step itself delegates to `services/shared.py`, the same function the cover letter
  service uses — so "final resume content" has exactly one implementation in the whole system.

---

## Error Handling Convention (applies to every service)

Each service module defines or reuses a small set of domain exceptions, translated to HTTP by
the calling route handler (matching the existing pattern in `technical-environment.md` §9):

| Exception | Raised by | HTTP status |
|---|---|---|
| `SessionNotFound` | `store.get()` | 404 (NFR-203) |
| `UploadRejected` | `services/resume.py` validation | 400 |
| `LLMError` | `llm/client.py` on refusal/max_tokens | 502, surfaced as "could not generate, please retry" (NFR-303) |
| `ExportBlocked` | `services/export.py`, when stale flags are set | 409 (FR-904) |

This list may grow slightly during Functional Design but the pattern — one exception type per
failure mode, translated at the API boundary, never a partial/ambiguous response — is fixed
here.
