# Story Generation Plan — Storywell

**Role**: Product owner
**Input**: `aidlc-docs/inception/requirements/requirements.md` (approved 2026-11-22)
**Mandatory artifacts**: `stories.md` (INVEST stories with acceptance criteria),
`personas.md` (user archetypes mapped to stories)

---

## Execution Checklist

- [x] Step A: Collect answers to all embedded questions below (Part 1)
- [x] Step B: Analyze answers for ambiguity; raise follow-up questions if needed — 1 gap found
      and resolved (NFR-303 ownership — see
      `story-generation-clarification-questions.md`: assigned to Foundation unit,
      `US-FOUNDATION-xx`, cross-referenced rather than duplicated by the three affected slices)
- [x] Step C: Get explicit approval of this plan — approved 2026-11-22
- [x] Step D: Generate `aidlc-docs/inception/user-stories/personas.md`
  - [x] D1: Internship-seeking student persona
  - [x] D2: New graduate persona
  - [x] D3: International student persona
  - [x] D4: Map each persona to the stories most relevant to their stated need
- [x] Step E: Generate `aidlc-docs/inception/user-stories/stories.md`
  - [x] E1: Foundation/wizard-shell stories (session, navigation, staleness) — 4 stories
  - [x] E2: Resume pipeline slice stories (upload, parse, review/correct) — 2 stories
  - [x] E3: JD/matching slice stories (JD input, requirement review, checklist, follow-up
        questions, confirmed facts) — 5 stories
  - [x] E4: Suggestions/verification slice stories (rewrite suggestions, flag handling,
        reordering, style choice) — 8 stories
  - [x] E5: Cover letter slice stories — 3 stories
  - [x] E6: Export slice stories — 4 stories
  - [x] E7: Verify every story is Independent, Negotiable, Valuable, Estimable, Small, Testable
  - [x] E8: Verify every story carries acceptance criteria and a source citation to the
        requirement ID(s) it covers
- [x] Step F: Self-check traceability — every FR-1xx through FR-9xx group has at least one
      covering story, or an explicit note why it is developer-facing and out of scope for
      stories (e.g., NFR items, FR-115 in-memory file handling) — confirmed, see
      "Story Count Summary" in stories.md
- [ ] Step G: Present completion message and wait for approval

---

## Part 1: Planning Questions

Please fill in each `[Answer]:` tag. Choose the lettered option that fits best, or choose
"Other" and describe your preference.

### Question 1 — Breakdown Approach

`requirements.md` §11 already organizes Construction around a **shared foundation unit** plus
**five vertical feature slices** (resume pipeline; JD/matching; suggestions/verification;
cover letter; export), each owned end-to-end by one developer. Should `stories.md` mirror
that same grouping, or use a different organizing principle?

A) **Feature-slice-based** — group stories under the same five slices plus the foundation
unit, so each developer can find exactly the stories their slice must satisfy.

B) **User-journey-based** — group stories by the 8-step wizard sequence (FR-902) regardless
of which slice implements each step.

C) **Persona-based** — group stories primarily by the three personas (§3), with slice/step
noted as metadata on each story.

D) **Hybrid: feature-slice-based with persona tags** — use the five slices plus foundation
unit as the primary grouping (matching how the team will build), and tag each story with the
persona(s) it most serves.

X) Other (please describe after [Answer]: tag below)

[Answer]: D — Group stories under the foundation unit and the five slices in requirements.md §11 (each story belongs to exactly one group), tag each story with one primary persona (every persona is primary for at least one story), and order stories inside a slice by wizard step. Only user-visible foundation behavior (wizard navigation, out-of-date marking per FR-903, privacy notice) becomes a story, not technical enablers like the session store or LLM client.

### Question 2 — Story Granularity

How finely should requirements be split into stories? For reference, Section 5 has 73
functional requirements across 9 groups (e.g., FR-4xx "Rewrite Suggestions and Verification"
alone has 12 requirements spanning generation, two flag types, and user accept/reject/edit).

A) **One story per FR sub-group heading** (e.g., all of FR-42x verification flagging becomes
one story "Flagged suggestions are shown safely") — fewer, larger stories.

B) **One story per distinct user-visible behavior**, which may combine 2–4 related FRs (e.g.,
FR-430+FR-431 together as "I see why some suggestions were withheld or need editing") —
moderate granularity, roughly one story per bold/starred requirement in the document.

C) **One story per individual FR** — finest granularity, maximum traceability, many small
stories.

X) Other (please describe after [Answer]: tag below)

[Answer]: B — One story per distinct user-visible behavior, combining 2-4 related FRs (e.g., FR-430+FR-431 as one story). If a story would need more than four acceptance criteria, split it. Every FR that affects what the user sees or does must be covered by at least one story, and each story lists the FR IDs it covers. FRs that describe only internal behavior (for example FR-101, FR-102, FR-115) are not stories; they are covered by the unit of work and listed as such in the story map.

### Question 3 — Acceptance Criteria Format

A) **Given/When/Then (Gherkin-style)** — e.g., "Given a suggestion flagged
`unsupported_number`, When the user views it, Then it shows a warning badge and cannot be
accepted without editing."

B) **Plain checklist bullets** — e.g., "- Shows a warning badge naming the flag. - Cannot be
accepted until edited."

C) **Hybrid** — Given/When/Then for stories with branching truthfulness logic (verification
flags, staleness, pending-at-export), plain checklist for simpler stories.

X) Other (please describe after [Answer]: tag below)

[Answer]: A — Use Given/When/Then for every story, with 2-4 acceptance criteria per story (the same format everywhere, no plain checklists). Every story includes at least one failure or edge-case criterion where one exists in requirements.md (for example rejected uploads, flagged suggestions, stale results), and criteria use the exact status labels and flag names from requirements.md (found, needs_confirmation, not_found, unsupported_number, verb_inflation, invalid_source_ids).

### Question 4 — Persona Depth

`requirements.md` §3 already gives each persona one line (primary need). How much more detail
should `personas.md` add?

A) **Lightweight** — keep each persona to primary need plus 2–3 bullet pain points drawn
directly from `vision.md` and the requirements; no invented biographical detail.

B) **Full persona profiles** — name, a short narrative paragraph, goals, pain points, and a
representative quote, in the common product-management persona-card style.

C) **Lightweight, plus explicit mapping table** — same as A, with an added table listing every
story each persona is most invested in (feeds Step D4 directly).

X) Other (please describe after [Answer]: tag below)

[Answer]: X — Full persona profiles as in B (name, short narrative paragraph, goals, pain points, one representative quote), plus the mapping table from C. Rules: the three personas stay the ones in requirements.md §3 (student seeking an internship, new graduate, international student); the profiles are labeled as fictional archetypes, and every goal and pain point must trace to vision.md or requirements.md (for example, fear of exaggerating maps to the truthfulness rules), with no invented features or facts about the product. The table lists every story under its primary persona, and each persona is primary for at least one story.

### Question 5 — Treatment of the Foundation Unit

The foundation unit (Pydantic models, in-memory session store, LLM client wrapper, API
contract, wizard shell — §11) is developer-facing infrastructure with no direct end-user
action of its own, though the wizard shell and staleness behavior (FR-901–FR-904) are
user-visible.

A) **Include as user stories** — e.g., "As a user, I see a progress indicator and can go back
without losing my place" (FR-901), "As a user, if I change my resume after matching ran, I'm
told my results are out of date" (FR-903) — written from the user's point of view even though
the foundation unit team implements them.

B) **Exclude from stories.md** — note in `stories.md` that the foundation unit is
infrastructure-only and reference `requirements.md` FR-901–FR-904 directly; Construction's
Functional Design stage will cover it instead.

C) **Include only the user-visible subset** (wizard navigation, staleness, the pending-export
notice) as stories; leave the purely internal parts (data models, LLM client wrapper, API
contract shape) out of stories.md entirely.

X) Other (please describe after [Answer]: tag below)

[Answer]: C — Include only the user-visible subset as stories (wizard navigation and progress indicator per FR-901/FR-902, out-of-date marking per FR-903, the export-blocked notice per FR-904, and the privacy notice per NFR-104). Leave the purely internal parts (data models, session store, LLM client wrapper, API contract shape) out of stories.md; the story map lists them as covered by the foundation unit of work.

### Question 6 — Error and Edge-Case Scenarios

Several requirements describe what happens when something goes wrong or is incomplete (e.g.,
FR-114 likely-scanned-PDF rejection, NFR-303 LLM refusal/max-tokens handling, FR-904
export-blocked-while-stale).

A) **Fold into the acceptance criteria of the main story** — e.g., the upload story's
acceptance criteria include the scanned-PDF-rejection case.

B) **Separate "edge case" stories** alongside the main happy-path story for the same feature,
so they can be prioritized or tested independently.

C) **Case-by-case** — fold minor validation edge cases into the main story's acceptance
criteria (A), but give cross-cutting failure modes that affect multiple screens (LLM refusal
per NFR-303, export-blocked-while-stale per FR-904) their own dedicated story since they are
shared behavior rather than one feature's edge case.

X) Other (please describe after [Answer]: tag below)

[Answer]: C — Fold edge cases that affect only one screen (for example the scanned-PDF rejection FR-114, upload size limits FR-112, a skipped question FR-312) into the acceptance criteria of that screen's main story. Give a dedicated story to any failure mode that can occur on two or more wizard screens or blocks a later step: the LLM refusal or max-tokens error (NFR-303, "could not generate, please retry", nothing partial shown) and the export-blocked-while-stale notice (FR-904, the same story as in Question 5, not a duplicate).

### Question 7 — Story ID Scheme

A) **Reuse FR IDs directly** as the story identifier where there's a 1:1 mapping, and assign a
new `US-xxx` ID only where a story spans multiple FRs.

B) **New independent `US-xxx` numbering** for all stories, with an FR cross-reference column
in `stories.md` for traceability (no ID reuse).

C) **Slice-prefixed numbering** — e.g., `US-RESUME-01`, `US-MATCH-01`, `US-SUGGEST-01`,
`US-COVER-01`, `US-EXPORT-01`, `US-FOUNDATION-01` — so IDs signal ownership per §11 at a
glance.

X) Other (please describe after [Answer]: tag below)

[Answer]: C — Slice-prefixed numbering: US-FOUNDATION-01, US-RESUME-01, US-MATCH-01, US-SUGGEST-01, US-COVER-01, US-EXPORT-01, with a two-digit number that counts up inside each slice and never changes or gets reused once assigned. Each story also lists the FR/NFR IDs it covers in a cross-reference line, so traceability does not depend on the story ID.

### Question 8 — Prioritization / MoSCoW Labeling

Should `stories.md` carry priority labels (e.g., Must/Should/Could), or is the MVP scope
boundary already defined clearly enough in `requirements.md` §10 that every story in scope is
implicitly "Must"?

A) **No priority labels** — everything in `requirements.md` §10's "In scope (MVP)" is a Must;
`stories.md` doesn't need to restate this.

B) **Add MoSCoW labels per story** — even within the MVP scope, flag which stories are more
negotiable (e.g., the `keep_my_voice` writing-sample style vs. core verification) to help
Workflow Planning sequence the vertical slices.

X) Other (please describe after [Answer]: tag below)

[Answer]: B — Label every story with MoSCoW. Must = needed for one complete flow from resume upload to export with the truthfulness and privacy rules intact (FR-1xx, FR-2xx, FR-3xx except FR-330, FR-4xx, FR-7xx, FR-8xx, FR-9xx, and the privacy and security NFRs). Should = valuable, and the flow still works without it: writing style choice including the keep_my_voice writing sample (FR-6xx) and creating a confirmed fact from the suggestions screen (FR-330). Could = within-entry bullet reordering (FR-5xx), the first thing to cut if time runs short. No story is labeled Won't; out-of-scope items stay in requirements.md §10.

---

## Breakdown Approach Reference (for Question 1)

| Approach | Benefit | Trade-off |
|---|---|---|
| Feature-slice-based | Matches how the team will actually build (§11); each developer's stories are together | Personas may feel scattered across slices |
| User-journey-based | Matches the wizard a tester or reviewer would walk through | Doesn't match the three-developer ownership model, so a developer must hunt across the file for "their" stories |
| Persona-based | Strong stakeholder/UX framing | Weakest match to how work will actually be assigned or built in Construction |
| Hybrid (slice + persona tags) | Gets both: build-time usability and persona traceability | Slightly more metadata per story |

A hybrid approach is possible for any question above — if none of the lettered options fit
exactly, use "Other" and describe the combination you want; specify the exact rule for when
each part of the hybrid applies (per `common/question-format-guide.md` guidance on avoiding
vague "mix of" answers).

---

## Mandatory Story Artifacts (Step 4, confirmed)

- [ ] `aidlc-docs/inception/user-stories/stories.md` — INVEST stories with acceptance criteria
- [ ] `aidlc-docs/inception/user-stories/personas.md` — persona archetypes mapped to stories
