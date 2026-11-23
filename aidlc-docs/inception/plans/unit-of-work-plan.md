# Unit of Work Plan — Storywell

**Role**: Software architect — decomposing the system into units of work for Construction.
**Input**: `requirements.md` §11 (already proposes foundation unit + 5 vertical feature
slices), `stories.md` (26 stories, approved), `application-design/` (components, methods,
services, dependencies — approved), `execution-plan.md` (Milestone Boundary: this workflow
stops after this stage is approved; the 3-week sizing constraint for Construction).

**Note on scope**: Application Design already settled the *runtime* dependency graph (no
feature slice calls another slice's `services/` module; everything shared routes through the
Foundation unit) and resolved the one flagged cross-slice dependency (FR-330). This plan does
not re-decide that. It focuses on what's genuinely still open: how the 5 feature slices map to
the 3 developers named in `requirements.md` §11 (A-5 explicitly defers this), and the
*development-time* sequencing question of when slice work can start relative to the Foundation
unit, given the ~3-week build window.

**Category coverage** (per `units-generation.md` Step 3 — every category addressed, some by
question, some by justified resolution from prior artifacts):
- **Story Grouping** — resolved by `requirements.md` §11 and `stories.md`'s own grouping (the
  5 slices + Foundation); Question 1 below only asks whether to further split one slice.
- **Dependencies** — the communication pattern is resolved (`component-dependency.md`); the
  *timing* of when slice work may start is open — Question 2.
- **Team Alignment** — genuinely open, explicitly deferred by `requirements.md` A-5 —
  Question 1.
- **Technical Considerations** — **justified N/A**: single local monolith
  (`technical-environment.md` §4), no cloud deployment, no per-unit scaling or deployment
  differences; every unit runs in the same one backend process and one frontend dev server.
- **Business Domain** — **already resolved**: the 5 feature slices already are Storywell's
  bounded contexts (resume intake, JD/matching, suggestion/verification, cover letter,
  export), confirmed by `requirements.md` §11 and unchanged by Application Design.
- **Code Organization (greenfield)** — **already resolved**, not re-asked: flat module files
  under the shared `app/services/`, `app/api/`, `app/verification/` packages exactly as named
  in `components.md` — not per-unit subdirectories. `technical-environment.md` §5's directory
  layout and `application-design/components.md` already name every file; `unit-of-work.md`
  documents this directly rather than reopening it.

---

## Execution Checklist

- [x] Step A: Collect answers to the 2 embedded questions below
- [x] Step B: Analyze answers for ambiguity; raise follow-up questions if needed — none
      needed; both answers are specific, with explicit decision/governance rules, and
      mutually consistent (see audit.md validation entry)
- [ ] Step C: Request approval of this plan; wait for explicit confirmation
- [x] Step D: Generate `aidlc-docs/inception/application-design/unit-of-work.md`
- [x] Step E: Generate `aidlc-docs/inception/application-design/unit-of-work-dependency.md`
- [x] Step F: Generate `aidlc-docs/inception/application-design/unit-of-work-story-map.md`
- [x] Step G: Validate — every one of the 26 stories assigned to exactly one unit; unit
      boundaries and dependencies consistent with `component-dependency.md` — confirmed via
      the Completeness Check table in unit-of-work-story-map.md (26/26, no duplicates)
- [ ] Step H: Present completion message; note the Milestone Boundary (this workflow stops
      here — Construction begins later, in the final project)

---

## Planning Questions

### Question 1 — Slice-to-Developer Assignment (Team Alignment / Story Grouping)

The 5 feature slices are very unequal in size: Resume Pipeline (2 stories, all Must),
JD & Matching (5, all Must), Suggestions & Verification (8: 4 Must + 2 Should + 2 Could),
Cover Letter (3, all Must), Export (4, all Must). `requirements.md` A-5 defers ownership of
these 5 slices among Xinyi Wang, Linghui Zhou, and Yunzhu Qiu (Linghui also owns the
Foundation unit and final integration) to this stage.

A) **Keep all 5 slices as-is; Linghui sweeps the three smallest after Foundation.** Xinyi Wang
owns JD & Matching (5 stories); Yunzhu Qiu owns Suggestions & Verification (8 stories, the
largest and most truthfulness-critical); Linghui Zhou owns Resume Pipeline (2) + Cover Letter
(3) + Export (4) = 9 stories, picked up as Foundation work winds down — these three are the
smallest individually and sit at the start and end of the wizard, which pairs naturally with
her final-integration role.

B) **Split Suggestions & Verification into two separately-owned units**, reusing the module
boundary Application Design already drew (`services/suggestions.py` vs.
`services/reordering.py`, Q1 of `application-design-plan.md`): a "Suggestions" unit
(US-SUGGEST-01–05, 5 stories, Must/Should) and a "Reordering & Style" unit (US-SUGGEST-06–08,
3 stories, Could/Should). This gives 6 feature units total, assignable more evenly across the
3 developers (e.g. ~7 stories each) — at the cost of one more unit boundary inside what was a
single cohesive slice.

X) Other (please describe your own assignment of the 5 slices — or 6, if splitting
Suggestions & Verification — across Xinyi Wang, Linghui Zhou, and Yunzhu Qiu, after
[Answer]: tag below)

[Answer]: X — Keep the 5 slices as they are (no split). Linghui Zhou owns the Foundation unit, Resume Pipeline (2 stories), Cover Letter (3 stories), and final integration; the cover letter builds directly on the Foundation's final-resume-content assembly and style context, so it stays with the Foundation owner. Xinyi Wang owns JD & Matching (5) and Suggestions & Verification (8), the core matching and truthfulness logic. Yunzhu Qiu owns Export (4 stories), a self-contained slice at the end of the wizard that no other slice depends on. Linghui's and Xinyi's workloads are roughly balanced once the Foundation unit and integration are counted. If time runs short, the two Could stories for bullet reordering (US-SUGGEST-06 and US-SUGGEST-07) are cut first.

### Question 2 — When Can Slice Work Start Relative to the Foundation Unit? (Dependencies)

`requirements.md` §11 says the Foundation unit is "built first," and `execution-plan.md`'s
3-week sizing constraint assumes slice work happens in parallel after it. Every feature
slice's `services/*.py` module depends on Foundation's `models/`, `store.py`, `llm/client.py`,
`verification/*`, and `services/shared.py` (per `component-dependency.md`) — so slice
developers need *something* from Foundation to start building against.

A) **Full gate**: Foundation is fully implemented, tested, and merged to `main` before any
slice branch starts. Simplest and lowest integration risk, but the 3-week window for slice
work doesn't start until Foundation is entirely done.

B) **Interface-frozen early start**: Foundation's *signatures* — the Pydantic models, the
`SessionState`/`SessionStore` method shapes, the `LLMClient` interface (plus `FakeLLMClient`),
and the `services/shared.py` / `verification/*` function signatures from
`component-methods.md` — are agreed and committed first (a short initial pass, not the full 3
weeks), and slice developers start building and testing against those signatures immediately,
using `FakeLLMClient` and the real (or a stub) `SessionStore`. Linghui Zhou finishes the real
Foundation implementations in parallel; slices rebase onto the real implementations as they
land, before final integration.

X) Other (please describe after [Answer]: tag below)

[Answer]: B (with additions) — Linghui Zhou commits the frozen interfaces first, within the first few days of Construction: the Pydantic models, the SessionState/SessionStore method shapes, the LLMClient interface plus FakeLLMClient, and the services/shared.py and verification/* signatures from component-methods.md. Once committed, any change to a frozen interface must be agreed by all three of us. Slice developers build and test against these interfaces with FakeLLMClient right away, while Linghui finishes the real Foundation implementations in parallel; slices rebase onto them before final integration.
