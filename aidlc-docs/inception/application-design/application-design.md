# Application Design: Storywell

**Stage**: INCEPTION — Application Design
**Date**: 2026-11-22
**Basis**: `requirements.md` (approved), `stories.md` (26 stories, approved),
`technical-environment.md`, `aidlc-docs/inception/plans/application-design-plan.md` (6
questions, answered, no ambiguity)

This document consolidates `components.md`, `component-methods.md`, `services.md`, and
`component-dependency.md`. See those files for full detail; this is the summary.

---

## Design Decisions (from the Answered Plan)

| # | Decision | Answer |
|---|---|---|
| 1 | Module granularity within each feature slice | One module per distinct capability (e.g. `matching.py` + `follow_up.py` under JD & Matching), not one monolithic module per slice, and not one module per story |
| 2 | `app/verification/` ownership | Shared, cross-slice package in the Foundation unit's contract — now covers all three deterministic checks: `verify_suggestion`, the reorder permutation check, and the `RequirementMatch` evidence invariant (added 2026-11-22) |
| 3 | Cross-slice coupling | Shared-state-only — slices never call each other's `services/` modules; anything two+ slices need (confirmed facts, final resume content, style context) lives in the Foundation unit's `services/shared.py` |
| 4 | Where verification is invoked | Inside each generation service, before it returns — route handlers never call `app/verification/` directly |
| 5 | Export module granularity | One `services/export.py`: one content-assembly function, two renderers (`render_docx`, `render_pdf`), one `.zip` packager |
| 6 | Staleness tracking location | On `SessionState` itself (Foundation unit) — `mark_dependents_stale()` / `clear_stale()` / `is_export_blocked()` |

**Resolved**: the FR-330 cross-slice dependency `requirements.md` §11 flagged as needing an
explicit owner now has one — `services/shared.py::create_confirmed_fact()`, in the Foundation
unit, called by both the JD & Matching and Suggestions & Verification slices.

---

## Components by Unit (summary — see `components.md` for full detail)

- **Foundation**: `models/`, `store.py` (`SessionState`/`SessionStore`), `llm/client.py`,
  `verification/` (3 modules: suggestions, reordering, matching), `services/shared.py`
  (confirmed-fact creation, final-resume-content assembly, style/writing-sample access),
  `main.py`/API contract, and the frontend wizard shell.
- **Resume Pipeline**: `services/resume.py`, `api/resume.py`, `ResumeUpload`,
  `ParsedResumeReview`.
- **JD & Matching**: `services/matching.py`, `services/follow_up.py`,
  `api/requirements.py`, `api/facts.py`, `JDInput`, `RequirementListReview`,
  `RequirementChecklist`, `FollowUpQuestions`.
- **Suggestions & Verification**: `services/suggestions.py`, `services/reordering.py`,
  `api/suggestions.py`, `api/reordering.py`, `SuggestionsReview`, `ReorderReview`,
  `StyleChoice`.
- **Cover Letter**: `services/cover_letter.py`, `api/cover_letter.py`, `CoverLetterDraft`.
- **Export**: `services/export.py`, `api/export.py`, `ExportStep`.

---

## The Orchestration Pattern (summary — see `services.md`)

```
API route handler
  -> look up session (404 if missing/expired)
  -> call exactly one services/*.py function
  -> translate any raised domain exception to HTTP
  -> return the Pydantic response model
```

Every generation-path service function (matching, suggestions, reordering) calls its
corresponding `app/verification/*` check internally before returning, so **no code path in
the system can return unverified LLM output** — the architectural guarantee behind PR-1.6.

---

## Dependency Summary (summary — see `component-dependency.md`)

No feature slice's `services/*.py` module is ever imported by another feature slice's
`services/*.py` module. All cross-slice needs route through the Foundation unit
(`services/shared.py`, `verification/*`, or `SessionState` fields). This was validated against
every dependency `requirements.md` §11 and `stories.md` flagged, with FR-330 given an explicit
resolution above.

---

## Traceability Check (Step I)

Every FR/NFR group in `requirements.md` §5–§6 and every story in `stories.md` has a home in
exactly one component above:

| Requirement group | Component(s) |
|---|---|
| FR-1xx (session/resume input) | `store.py`, `services/resume.py` |
| FR-2xx (JD/matching) | `services/matching.py`, `verification/matching.py` |
| FR-3xx (follow-ups/facts) | `services/follow_up.py`, `services/shared.py` |
| FR-4xx (suggestions/verification) | `services/suggestions.py`, `verification/suggestions.py` |
| FR-5xx (reordering) | `services/reordering.py`, `verification/reordering.py` |
| FR-6xx (style) | `services/shared.py` (style context), consumed by `suggestions.py`/`cover_letter.py` |
| FR-7xx (cover letter) | `services/cover_letter.py` |
| FR-8xx (export) | `services/export.py` |
| FR-9xx (wizard/staleness) | Frontend wizard shell, `store.py` |
| NFR-1xx/2xx (privacy/security) | `store.py`, `main.py` middleware, `llm/client.py` prompt-wrapping |
| NFR-3xx (reliability/LLM) | `llm/client.py` |
| NFR-5xx (testability) | `llm/client.py` (`FakeLLMClient`), `verification/*` (property-testable, pure) |
| NFR-6xx (maintainability) | Directory layout itself, per `technical-environment.md` §5 |

No requirement group is unhoused. Application Design is complete; detailed business rules for
each component's methods are deferred to Functional Design, per unit, in Construction (per the
Milestone Boundary in `execution-plan.md` — not entered in this session).
