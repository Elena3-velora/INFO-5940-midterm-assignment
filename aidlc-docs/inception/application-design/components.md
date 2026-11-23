# Components: Storywell

**Stage**: INCEPTION — Application Design
**Date**: 2026-11-22
**Basis**: `requirements.md`, `stories.md`, `technical-environment.md`, the answered
`application-design-plan.md`

Components are grouped by unit of work, per `requirements.md` §11: the Foundation unit first,
then the five vertical feature slices. Within `technical-environment.md`'s fixed directory
layout, each component is a module or small package.

A note on two applied rules that recur below: per **Q3** of the plan, anything two or more
slices both need is placed in the Foundation unit rather than duplicated or owned by one
slice; this is applied not only to the two examples the plan named explicitly (confirmed-fact
creation, final-resume-content assembly) but consistently to writing style/sample as well,
since both the Suggestions & Verification slice (FR-604) and the Cover Letter slice need the
same style choice. Per **Q2**, `app/verification/` is a shared package, not owned by any one
slice.

---

## Foundation Unit

*Owner: Linghui Zhou (also owns final integration). Built first; every other unit depends on
it.*

### `app/models/` — Pydantic Data Models
**Purpose**: The single source of truth for every structured type used across the backend.
**Responsibilities**: Define `ResumeItem`, `ConfirmedFact`, `RequirementMatch` (with the
2026-11-22 evidence invariant), `Suggestion`, `ReorderSuggestion`, `StyleChoice`, and the
request/response models for every endpoint.
**Interface**: Pure data definitions; no behavior. Imported by every other component.

### `app/store.py` — Session Store and Session State
**Purpose**: Holds all per-session state in memory, with TTL-based expiry (FR-103), and owns
the staleness rule from FR-903/904 (per **Q6**).
**Responsibilities**:
- `SessionStore`: create/get/expire sessions, keyed by `secrets.token_urlsafe(16)` (FR-101).
- `SessionState`: the per-session data container (resume items, JD text, requirement matches,
  confirmed facts, suggestions, reorder suggestions, cover letter draft, style choice, writing
  sample, and `stale` flags for `matches`/`suggestions`/`cover_letter`).
- `SessionState.mark_dependents_stale()`: the single place the FR-903 dependency rule is
  defined — called by the Resume Pipeline and JD & Matching slices' edit endpoints.
- `SessionState.clear_stale(flag)`: called by each slice's own regeneration function.
**Interface**: Used by every API route handler (via FastAPI `Depends`) and by every slice's
service functions.

### `app/llm/client.py` — LLM Client Wrapper
**Purpose**: The sole module that imports `anthropic` (NFR-301), keeping the provider
swappable.
**Responsibilities**: `LLMClient.structured()` — issue one structured-output call, check
`stop_reason`, raise `LLMError` on `refusal`/`max_tokens`. `FakeLLMClient` for tests
(NFR-501).
**Interface**: Injected via `Depends(get_llm_client)` into any service function that needs the
LLM — matching, follow-up questions, suggestions, reordering, and the cover letter.

### `app/verification/` — Shared Deterministic Checks (per Q2)
**Purpose**: Every deterministic truthfulness/safety check in the system, isolated from
`services/` per NFR-603, and property-tested per PBT-03. Part of the Foundation unit's
contract — used by, but not owned by, the slices below.
**Responsibilities**:
- `verification/suggestions.py`: `verify_suggestion()`, `find_unsupported_numbers()`,
  `find_verb_inflation()` (FR-420–423).
- `verification/reordering.py`: the exact-permutation check (FR-503–504).
- `verification/matching.py`: the `RequirementMatch` evidence invariants — `found` implies
  non-empty `evidence_ids`, `not_found` implies empty `evidence_ids` (FR-222–223, added
  2026-11-22).
**Interface**: Pure functions — no LLM calls, no session store access — called internally by
the generation services in JD & Matching and Suggestions & Verification (per **Q4**).

### `app/services/shared.py` — Cross-Slice Shared Logic (per Q3)
**Purpose**: Houses the capabilities more than one slice needs, so no slice imports another
slice's module.
**Responsibilities**:
- `create_confirmed_fact()`: used by JD & Matching (FR-320, answering a follow-up question)
  and by Suggestions & Verification (FR-330, confirming a flagged suggestion is true).
- `build_final_resume_content()`: assembles the "final resume content" defined term
  (`requirements.md` §5) — used by Cover Letter (FR-701) and Export (FR-805).
- Style/writing-sample access helpers (FR-601–604) — used by Suggestions & Verification
  (rewrite generation) and Cover Letter (draft generation), applying the Q3 rule by extension.
**Interface**: Plain functions operating on `SessionState`; imported by the three slices named
above. This is also the named resolution of the FR-330 cross-slice dependency flagged in
`requirements.md` §11 and `stories.md` US-SUGGEST-05 — both the JD & Matching and Suggestions
& Verification API layers call the same shared function rather than one slice importing the
other's `services/` module.

### `app/main.py` + `app/api/__init__.py` — API Contract and App Shell
**Purpose**: FastAPI app instance, router registration, CORS and security-header middleware
(NFR-202, NFR-204).
**Responsibilities**: Wires every slice's `api/` router under `/api`; global error handler
(NFR-305).

### Frontend: Wizard Shell (`frontend/src/App.tsx` + routing)
**Purpose**: The linear, 9-step wizard shell (FR-901–902) with progress indicator and back
navigation.
**Responsibilities**: Step routing; reading staleness flags from the session (via the API) to
show the out-of-date state (FR-903) and block export (FR-904); the privacy notice (NFR-104)
on the resume upload screen; the LLM-refusal retry message (NFR-303), shown uniformly
regardless of which step's generation call failed.

---

## Resume Pipeline Slice

*Ownership of this slice (beyond the Foundation unit) is deferred to Units Generation.*

### `app/services/resume.py`
**Purpose**: Resume intake and correction (FR-110–131).
**Responsibilities**: Validate uploads (extension/signature/size/page-count/scanned-PDF
checks, FR-111–114); parse into `ResumeItem`s (FR-120); apply user corrections — edit, merge,
split, delete (FR-130); call `SessionState.mark_dependents_stale()` when corrections happen
after matching has already run.

### `app/api/resume.py`
**Purpose**: Routes for `POST /api/sessions/{sid}/resume` and parsed-item correction.

### Frontend: `ResumeUpload`, `ParsedResumeReview`
**Purpose**: The two screens covering US-RESUME-01 and US-RESUME-02.

---

## JD & Matching Slice

### `app/services/matching.py`
**Purpose**: JD requirement extraction and evidence matching (FR-201–224).
**Responsibilities**: Validate JD text length (FR-113); extract requirements via the LLM
(FR-210); apply user edits to the requirement list and re-run matching when it changes
(FR-211–212); produce `RequirementMatch` records, calling `verification/matching.py`
internally before returning (per **Q4**) so a `found` status can never reach the UI without
evidence; call `mark_dependents_stale()` when the requirement list changes post-matching.

### `app/services/follow_up.py`
**Purpose**: Follow-up question generation and answer recording (FR-301–321).
**Responsibilities**: Generate questions for under-evidenced requirements; order them into
rounds of 3 by JD emphasis (FR-310); record a "no" answer as creating no usable fact
(FR-321); for a "yes" answer, delegate fact creation to `services/shared.py`'s
`create_confirmed_fact()`.

### `app/api/requirements.py`, `app/api/facts.py`
**Purpose**: Routes for `POST /api/sessions/{sid}/requirements/match` and
`POST /api/sessions/{sid}/facts` (shared with Suggestions & Verification per FR-330).

### Frontend: `JDInput`, `RequirementListReview`, `RequirementChecklist`, `FollowUpQuestions`

---

## Suggestions & Verification Slice

### `app/services/suggestions.py`
**Purpose**: Rewrite suggestion generation, verification-gated, and user decisions
(FR-401–441, FR-330–333).
**Responsibilities**: Generate suggestions for JD-evidence and weakly-worded lines (FR-410),
grouped separately (FR-411); call `verification/suggestions.py` internally before returning
(per **Q4**); drop `invalid_source_ids` suggestions and count them (FR-430); accept/reject/edit
(FR-440–441); `confirm_flagged_suggestion_fact()` — calls `services/shared.py`'s
`create_confirmed_fact()`, then re-verifies only that one suggestion (FR-331–332); call
`mark_dependents_stale()`/`clear_stale("suggestions")` as appropriate.

### `app/services/reordering.py`
**Purpose**: Within-entry bullet reorder suggestions (FR-501–507).
**Responsibilities**: Generate proposed orders; call `verification/reordering.py`'s
permutation check internally before returning (per **Q4** and **Q2**); accept/reject.

### `app/api/suggestions.py`, `app/api/reordering.py`

### Frontend: `SuggestionsReview`, `ReorderReview`, `StyleChoice`

---

## Cover Letter Slice

### `app/services/cover_letter.py`
**Purpose**: Cover letter drafting (FR-701–706).
**Responsibilities**: Call `services/shared.py`'s `build_final_resume_content()` and the
style/writing-sample helpers; draft via the LLM; never infer company/role (FR-703); apply the
generic-opening fallback (FR-704).

### `app/api/cover_letter.py`

### Frontend: `CoverLetterDraft`

---

## Export Slice

### `app/services/export.py`
**Purpose**: DOCX/PDF export (FR-801–808), per **Q5**.
**Responsibilities**: `assemble_export_content()` — calls `services/shared.py`'s
`build_final_resume_content()` plus the cover letter draft, once; `render_docx()` and
`render_pdf()` both consume that same assembled content; `package_zip()` when both are
selected (FR-802); check `SessionState` staleness flags to block export (FR-904) and the
pending-suggestions count for the non-blocking notice (FR-808).

### `app/api/export.py`

### Frontend: `ExportStep`

---

## Component Count Summary

| Unit | Backend components | Frontend components |
|---|---|---|
| Foundation | `models/`, `store.py`, `llm/client.py`, `verification/` (3 modules), `services/shared.py`, `main.py`/`api/__init__.py` | Wizard shell |
| Resume Pipeline | `services/resume.py`, `api/resume.py` | `ResumeUpload`, `ParsedResumeReview` |
| JD & Matching | `services/matching.py`, `services/follow_up.py`, `api/requirements.py`, `api/facts.py` | `JDInput`, `RequirementListReview`, `RequirementChecklist`, `FollowUpQuestions` |
| Suggestions & Verification | `services/suggestions.py`, `services/reordering.py`, `api/suggestions.py`, `api/reordering.py` | `SuggestionsReview`, `ReorderReview`, `StyleChoice` |
| Cover Letter | `services/cover_letter.py`, `api/cover_letter.py` | `CoverLetterDraft` |
| Export | `services/export.py`, `api/export.py` | `ExportStep` |
