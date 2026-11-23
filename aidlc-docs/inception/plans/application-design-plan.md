# Application Design Plan — Storywell

**Role**: Software architect — high-level component identification and service layer design.
**Input**: `requirements.md` (approved), `stories.md` (26 stories, approved),
`technical-environment.md` (mandated stack, directory layout, endpoint sketch, core Pydantic
model, example code patterns).
**Mandatory artifacts**: `components.md`, `component-methods.md`, `services.md`,
`component-dependency.md`, and a consolidated `application-design.md`.

**Note on scope**: `technical-environment.md` already fixes a lot of this — the directory
layout (`app/api`, `app/services`, `app/verification`, `app/llm`, `app/models`, `app/store.py`),
the endpoint sketch, the core Pydantic models, and example code patterns (route handlers call
plain async service functions; services raise domain exceptions the API layer translates to
HTTP errors; `verify_suggestion` runs before any suggestion reaches the UI). This plan does
not re-ask anything already settled there — it focuses on the genuine open design decisions:
how business capabilities map onto concrete modules, and how the six units of work (from
`requirements.md` §11) depend on each other.

---

## Execution Checklist

- [x] Step A: Collect answers to all embedded questions below
- [x] Step B: Analyze answers for ambiguity; raise follow-up questions if needed — none
      needed; all 6 answers are specific and mutually consistent (Q3's "shared things go to
      the Foundation unit" principle is reapplied consistently in Q5 and Q6)
- [x] Step C: Get explicit approval of this plan — not required as a separate gate per
      `application-design.md` (unlike `user-stories.md`, this stage has no Part 1/Part 2
      split); proceeding directly to generation
- [x] Step D: Generate `aidlc-docs/inception/application-design/components.md`
- [x] Step E: Generate `aidlc-docs/inception/application-design/component-methods.md`
- [x] Step F: Generate `aidlc-docs/inception/application-design/services.md`
- [x] Step G: Generate `aidlc-docs/inception/application-design/component-dependency.md`
- [x] Step H: Generate consolidated `aidlc-docs/inception/application-design/application-design.md`
- [x] Step I: Validate design completeness — every FR/NFR group and every story has a home in
      some component; the FR-330 cross-slice dependency has an explicit, named resolution
      (`services/shared.py::create_confirmed_fact()`, Foundation unit) — see
      application-design.md's Traceability Check table
- [ ] Step J: Present completion message and wait for approval

---

## Planning Questions

Please fill in each `[Answer]:` tag. Choose the lettered option that fits best, or choose
"Other" and describe your preference.

### Question 1 — Module Granularity Within Each Feature Slice (Component Identification)

`technical-environment.md` puts all business logic for all five feature slices under one
flat `app/services/` package. Should each feature slice map to exactly one `services/` module,
or should some slices split into more than one module?

A) **One module per feature slice** — `services/resume.py`, `services/matching.py`,
`services/suggestions.py`, `services/cover_letter.py`, `services/export.py`. Everything a
slice does lives in its one module.

B) **One module per distinct capability**, which may put more than one module under a single
slice — e.g. `services/matching.py` (FR-210–224) and `services/follow_up.py` (FR-301–333) both
under JD & Matching; `services/suggestions.py` (FR-401–441) and `services/reordering.py`
(FR-501–507) both under Suggestions & Verification — grouped the same way `stories.md` already
groups its story IDs (`US-MATCH-01..03` vs. `US-MATCH-04..05`; `US-SUGGEST-01..05` vs.
`US-SUGGEST-06..07`).

C) **One module per story group**, i.e. close to a 1:1 mapping with each `US-*` story's scope —
finest granularity.

X) Other (please describe after [Answer]: tag below)

[Answer]: B — One module per distinct capability, grouped the same way stories.md groups its stories: matching.py and follow_up.py under JD & Matching, suggestions.py and reordering.py under Suggestions & Verification, and one module each for Resume Pipeline, Cover Letter, and Export. Keeping reordering.py separate matters because bullet reordering is a Could story; if it is cut, nothing in suggestions.py changes.

### Question 2 — Is `app/verification/` a Shared Package or Owned by One Slice? (Component Identification / Dependencies)

`requirements.md` now has **three** deterministic invariant checks: `verify_suggestion`
(FR-420–423, owned conceptually by Suggestions & Verification), the reorder exact-permutation
check (FR-503–504, also Suggestions & Verification), and the newer `found`-requires-evidence /
`not_found`-requires-no-evidence checks on `RequirementMatch` (FR-222–223, added 2026-11-22 —
conceptually owned by JD & Matching). `technical-environment.md` NFR-603 says deterministic
truthfulness checks live in one `app/verification/` package, not inside `services/`.

A) **`app/verification/` is a shared, cross-slice package** — it contains all three checks as
sibling modules/functions (e.g. `verification/suggestions.py`, `verification/reordering.py`,
`verification/matching.py`), used by both the JD & Matching and Suggestions & Verification
slices. Neither slice "owns" it; it's part of the Foundation unit's contract, like the Pydantic
models.

B) **`app/verification/` is owned by Suggestions & Verification only** — it contains just
`verify_suggestion` and the permutation check. The `RequirementMatch` evidence invariant
(FR-222–223) is instead enforced inline inside `services/matching.py`, not in a shared package.

C) **Two separate packages** — `app/verification/` stays Suggestions & Verification-only (as
in B), and a second small module, e.g. `services/matching_checks.py`, holds the
`RequirementMatch` invariant, owned by JD & Matching.

X) Other (please describe after [Answer]: tag below)

[Answer]: A — app/verification/ is a shared, cross-slice package that is part of the Foundation unit's contract, with all three checks as sibling modules. Every function in it is pure and deterministic: it does not call the LLM or read the session store, so it can be property-tested directly (PBT-03).

### Question 3 — Cross-Slice Coupling Rule (Component Dependencies)

With three developers building five slices largely in parallel after the foundation unit
lands, how should slices depend on each other, beyond the one flagged cross-slice dependency
(FR-330, the confirmed-fact-from-suggestions-screen endpoint)?

A) **Shared-state-only coupling** — slices never import or call each other's `services/`
functions directly. All coordination happens through reading and writing fields on the shared
session object (from the Foundation unit's `store.py`). FR-330 is the sole named exception,
and it is resolved by giving it one explicit, agreed-upon home (see Question 2's sibling
question below) that any other slice needing it may import from.

B) **Direct calls are allowed** between slices when one slice's output is a hard prerequisite
for another (e.g. Suggestions & Verification calling a function in JD & Matching's module to
re-run matching after FR-212), as long as the dependency direction is documented in
`component-dependency.md`.

C) **Allowed only one direction**: later wizard-step slices (Suggestions & Verification, Cover
Letter, Export) may call earlier wizard-step slices' service functions (Resume Pipeline, JD &
Matching), but never the reverse — matching the wizard order in FR-902, so there are no
circular dependencies between parallel-built slices.

X) Other (please describe after [Answer]: tag below)

[Answer]: A (with additions) — Slices never import each other's services/ modules; they coordinate only through the shared session object and the Foundation unit. Anything two slices both need goes into the Foundation unit instead of one slice. That covers two shared pieces: (1) creating a ConfirmedFact, used by both follow-up questions and the suggestions-screen confirmation (FR-320, FR-330), and (2) building the "final resume content" defined in requirements.md §5, used by both the cover letter (FR-701) and export (FR-805). Both live in Foundation modules that any slice may import.

### Question 4 — Where Does Verification Get Invoked From? (Service Layer Design)

`requirements.md` PR-1.6 requires that no LLM output reaches the UI without passing through
verification first. `technical-environment.md` NFR-602 requires route handlers to contain no
business logic.

A) **Each generation service function calls verification internally** before returning —
e.g. `services/suggestions.py`'s `generate_suggestions()` calls `verify_suggestion()` on every
candidate internally and returns only the final, flag-annotated `Suggestion` objects. The API
route handler never touches `app/verification/` directly. This is the pattern for every
generation path: suggestions, reorders, and (per Question 2) matching.

B) **The API route handler orchestrates both calls** — it calls the generation service to get
raw candidates, then separately calls the verification function, then assembles the response.
Business logic stays in two places (`services/` and the handler's orchestration), not one.

X) Other (please describe after [Answer]: tag below)

[Answer]: A — Each generation service calls verification internally and returns only verified, flag-annotated results; route handlers never call app/verification/ directly. This applies to suggestions, reorders, and requirement matching, and also to re-verification after a fact is confirmed (FR-331). Keeping it inside the service means no code path can return an unverified suggestion, which is how we guarantee PR-1.6.

### Question 5 — Export Module Granularity (Component Methods)

`requirements.md` FR-801–808 requires exporting to both DOCX (`python-docx`) and PDF
(`fpdf2`), individually or together in a `.zip`.

A) **One `services/export.py` module** with one function that assembles the format-agnostic
export content (final resume content + cover letter, per the defined term in `requirements.md`
§5) once, then two renderer functions (`render_docx()`, `render_pdf()`) that both consume that
same assembled content — avoiding the two formats drifting apart.

B) **Two separate modules**, `services/export_docx.py` and `services/export_pdf.py`, each
independently assembling and rendering its own format.

X) Other (please describe after [Answer]: tag below)

[Answer]: A (with additions) — One services/export.py with render_docx() and render_pdf() consuming the same assembled content, plus .zip packaging when both files are selected (FR-802). Per our Question 3 answer, the function that builds the "final resume content" lives in the Foundation unit, not in export.py, because the cover letter (FR-701) uses exactly the same content; export.py calls it instead of re-implementing it.

### Question 6 — Where Does Staleness Tracking Live? (Design Patterns)

`requirements.md` FR-903–904 (and `stories.md` US-FOUNDATION-02) requires that going back and
changing the resume or requirement list marks dependent results (matching, suggestions, cover
letter) out of date, and blocks export until they're regenerated.

A) **On the session object itself**, in the Foundation unit's `store.py` — e.g. a set of
`stale` flags (`"matches"`, `"suggestions"`, `"cover_letter"`) that any mutating service call
sets, and that the export service checks. Every slice's service functions are responsible for
clearing their own flag when they regenerate, and the Foundation unit alone is responsible for
setting flags when resume/requirement-list-editing endpoints are called.

B) **As a separate utility module** (e.g. `services/staleness.py`) that every relevant service
function calls explicitly on both the marking and the clearing side, kept out of the session
object itself to avoid giving `store.py` business-logic responsibilities.

X) Other (please describe after [Answer]: tag below)

[Answer]: A (with additions) — Staleness flags live on the session object owned by the Foundation unit. The dependency rule (which results become stale when the resume or the requirement list changes, per FR-903) is defined once there, as a session method such as mark_dependents_stale(). The resume and matching services call it when the user edits their inputs, each slice clears its own flag after regenerating, and the export service checks the flags to block export (FR-904). This keeps the rule in one place while still letting the slices that own the edit endpoints trigger it.
