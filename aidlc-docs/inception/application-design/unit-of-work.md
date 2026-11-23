# Units of Work: Storywell

**Stage**: INCEPTION — Units Generation
**Date**: 2026-11-22
**Basis**: `requirements.md` §11, `stories.md` (26 stories), `application-design/` (approved),
`unit-of-work-plan.md` (2 questions, answered, no blocking ambiguity)

**Definition** (per `units-generation.md`): Storywell is a **monolith** — one backend process,
one frontend dev server (`technical-environment.md` §4). Each unit below is a **logical
module grouping** for development purposes, not an independently deployable service.

---

## Unit 1: Foundation

**Owner**: Linghui Zhou. **Built first.** Also owns final integration.

**Responsibilities**: `app/models/`, `app/store.py` (`SessionState`/`SessionStore`,
including the FR-903/904 staleness methods), `app/llm/client.py` (`LLMClient`,
`FakeLLMClient`), `app/verification/` (all 3 shared check modules — `suggestions.py`,
`reordering.py`, `matching.py`), `app/services/shared.py` (confirmed-fact creation,
final-resume-content assembly, style/writing-sample context), `app/main.py`/API contract,
and the frontend wizard shell.

**Stories covered** (4, all Must — see `component-dependency.md`'s coupling rule and
`application-design.md` for why these specific stories sit here rather than in a feature
slice): US-FOUNDATION-01, US-FOUNDATION-02, US-FOUNDATION-03, US-FOUNDATION-04.

**Build sequencing** (per Question 2 of the plan — Interface-Frozen Early Start):
1. Linghui commits the **frozen interfaces** first — Pydantic models, `SessionState`/
   `SessionStore` method shapes, `LLMClient` interface + `FakeLLMClient`, and the
   `services/shared.py` / `verification/*` function signatures from `component-methods.md` —
   within the first few days of Construction.
2. **Governance rule**: once committed, any change to a frozen interface must be agreed by
   all three developers.
3. Feature-slice developers start building and testing against the frozen interfaces
   immediately, using `FakeLLMClient`.
4. Linghui finishes the real Foundation implementations in parallel with the slices' work.
5. Slices rebase onto the real implementations as they land, ahead of final integration.

**Definition of done**: all 4 stories' acceptance criteria pass against the real (not fake)
implementations, and every frozen interface signature is unchanged from what slice
developers built against (or any change was agreed by all three, per the governance rule).

---

## Unit 2: JD & Matching

**Owner**: Xinyi Wang.

**Responsibilities**: `app/services/matching.py`, `app/services/follow_up.py`,
`app/api/requirements.py`, `app/api/facts.py` (shared endpoint with Unit 3 per the FR-330
resolution), `JDInput`, `RequirementListReview`, `RequirementChecklist`,
`FollowUpQuestions`.

**Stories covered** (5, all Must): US-MATCH-01, US-MATCH-02, US-MATCH-03, US-MATCH-04,
US-MATCH-05.

**Definition of done**: all 5 stories' acceptance criteria pass, including the
`found`-requires-evidence / `not_found`-requires-no-evidence invariant (FR-222–223, verified
via `verification/matching.py`).

---

## Unit 3: Suggestions & Verification

**Owner**: Xinyi Wang (same owner as Unit 2 — her combined workload, agreed by the team,
covers both the matching and truthfulness-enforcement core of the product).

**Responsibilities**: `app/services/suggestions.py`, `app/services/reordering.py`,
`app/api/suggestions.py`, `app/api/reordering.py`, `SuggestionsReview`, `ReorderReview`,
`StyleChoice`.

**Stories covered** (8): US-SUGGEST-01 through US-SUGGEST-08 — 4 Must (01–04), 2 Should
(05, 08), 2 Could (06, 07).

**Definition of done — tiered by MoSCoW, per the team's agreed mitigation**:
- **Required**: the 4 Must stories (01–04) and the 2 Should stories (05, 08) pass.
- **Stretch**: the 2 Could stories (06, 07 — bullet reordering) are the **first to cut** if
  the 3-week window is tight. The unit is still considered done without them; `ReorderReview`
  and the reordering endpoints simply ship as a follow-up if time allows.

---

## Unit 4: Cover Letter

**Owner**: Linghui Zhou (bundled with Foundation and Resume Pipeline, per the team's agreed
allocation — all three sit naturally with the Foundation/integration owner, since Cover
Letter is a thin orchestrator over `services/shared.py`, per `services.md`).

**Responsibilities**: `app/services/cover_letter.py`, `app/api/cover_letter.py`,
`CoverLetterDraft`.

**Stories covered** (3, all Must): US-COVER-01, US-COVER-02, US-COVER-03.

**Definition of done**: all 3 stories' acceptance criteria pass, including the PR-2 natural-
writing criterion (checked manually via the NFR-505 eval set, not an automated check).

---

## Unit 5: Resume Pipeline

**Owner**: Linghui Zhou.

**Responsibilities**: `app/services/resume.py`, `app/api/resume.py`, `ResumeUpload`,
`ParsedResumeReview`.

**Stories covered** (2, all Must): US-RESUME-01, US-RESUME-02.

**Definition of done**: both stories' acceptance criteria pass.

---

## Unit 6: Export

**Owner**: Yunzhu Qiu.

**Responsibilities**: `app/services/export.py`, `app/api/export.py`, `ExportStep`.

**Stories covered** (4, all Must): US-EXPORT-01, US-EXPORT-02, US-EXPORT-03, US-EXPORT-04.

**Definition of done**: all 4 stories' acceptance criteria pass; no other unit depends on
this unit.

---

## Workload Summary (team-agreed, per `unit-of-work-plan.md` Question 1 and the approval
response)

| Owner | Units | Must stories | Should | Could | Total |
|---|---|---|---|---|---|
| Linghui Zhou | Foundation, Resume Pipeline, Cover Letter (+ final integration) | 4+2+3=9 | 0 | 0 | 9 |
| Xinyi Wang | JD & Matching, Suggestions & Verification | 5+4=9 | 2 | 2 | 13 |
| Yunzhu Qiu | Export | 4 | 0 | 0 | 4 |

The asymmetry (Xinyi Wang: 13, Linghui Zhou: 9 + Foundation + integration, Yunzhu Qiu: 4) is
**intentional and team-agreed**: per the team's explicit confirmation, the role labels in
`README.md`/`aidlc-state.md`'s Team section describe responsibilities for the mid-semester
AI-DLC documents, not Construction coding capacity — all three developers write code in
Construction, and Xinyi Wang specifically agreed to own the two heaviest, most
truthfulness-critical units. The agreed mitigation if the 3-week window is tight: cut
US-SUGGEST-06/07 (bullet reordering, Could-priority) first — see Unit 3's definition of done.

---

## Code Organization Strategy (Greenfield)

Already fixed by `technical-environment.md` §5 and `application-design/components.md` — not
reopened here. **Flat module files within shared packages**, not per-unit subdirectories:

```
backend/
  app/
    main.py
    api/            # resume.py, requirements.py, facts.py, suggestions.py, reordering.py,
                     # cover_letter.py, export.py — one file per unit's endpoints
    services/        # resume.py, matching.py, follow_up.py, suggestions.py, reordering.py,
                     # cover_letter.py, export.py, shared.py — one file per unit's logic
    verification/    # suggestions.py, reordering.py, matching.py — Foundation-owned, shared
    llm/             # client.py (Foundation-owned)
    models/          # Foundation-owned
    store.py         # Foundation-owned
  tests/
    unit/, api/, fixtures/, eval/   # mirrors the services/ and verification/ structure
frontend/
  src/
    api/, components/, pages/       # one page/component group per unit, per components.md
```

Each developer's unit maps to a **specific, named set of files** above — no two developers
edit the same file, which keeps the 3 developers' parallel work from colliding even though
everything lives in shared top-level packages rather than per-unit subdirectories.
