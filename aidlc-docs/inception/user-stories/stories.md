# User Stories: Storywell

**Stage**: INCEPTION — User Stories
**Date**: 2026-11-22
**Basis**: `requirements.md` (approved 2026-11-22), per the approved
`story-generation-plan.md`

## How to Read This Document

- **Grouping**: one group per `requirements.md` §11 unit of work — the Foundation unit, then
  the five vertical feature slices (Resume Pipeline, JD & Matching, Suggestions &
  Verification, Cover Letter, Export) — in that build order. Stories within a group are
  ordered by wizard step (FR-902).
- **IDs**: slice-prefixed (`US-FOUNDATION-`, `US-RESUME-`, `US-MATCH-`, `US-SUGGEST-`,
  `US-COVER-`, `US-EXPORT-`), numbered sequentially within the slice; numbers are never reused.
- **Persona**: the one primary persona the story most directly serves (see `personas.md`).
- **Priority**: MoSCoW. Must = needed for one complete, truthful flow from upload to export.
  Should = valuable, but the flow still works without it. Could = first to cut if time is
  short. No story is labeled Won't — out-of-scope items stay in `requirements.md` §10.
- **Covers**: the requirement ID(s) this story is built from, so traceability does not depend
  on the story ID.
- **Cross-ref**: a note pointing to another story that already covers shared behavior, instead
  of duplicating it (used for the LLM-refusal and staleness/export-block stories).
- **Acceptance criteria**: Given/When/Then, 2–4 per story, using the exact status and flag
  names from `requirements.md` (`found`, `needs_confirmation`, `not_found`,
  `unsupported_number`, `verb_inflation`, `invalid_source_ids`).

Internal-only requirements with no direct user action of their own (FR-101, FR-102, FR-103,
FR-115, FR-120's raw parsing step, the data models, the in-memory session store, the LLM
client wrapper, and the API contract shape) are not stories here — they are implemented as
part of the Foundation unit of work and remain specified only in `requirements.md`.

---

## Foundation Unit

### US-FOUNDATION-01 — Navigate the wizard with a sense of progress
**As a** user tailoring my resume, **I want** a single step visible at a time with a progress
indicator and the ability to go back, **so that** I always know where I am and can revisit an
earlier step without losing my place.

- **Persona**: New Graduate
- **Priority**: Must
- **Covers**: FR-901, FR-902

**Acceptance Criteria**
1. Given I am anywhere in the flow, When the screen loads, Then I see a visible progress
   indicator showing which of the 9 wizard steps I'm on.
2. Given I am on any step, When I choose to go back, Then I return to the previous step with
   my prior inputs preserved.
3. Given the full flow, When I move forward normally, Then the steps occur in this order:
   resume upload → parsed resume review → JD input → requirement list review → requirement
   checklist → follow-up questions → suggestions → cover letter → export.

---

### US-FOUNDATION-02 — Know when my results are out of date
**As a** user, **I want** to be told when going back and changing an earlier answer makes my
later results stale, and to be stopped from exporting stale results, **so that** I never submit
a resume or cover letter built from an answer I've since changed.

- **Persona**: New Graduate
- **Priority**: Must
- **Covers**: FR-903, FR-904

**Acceptance Criteria**
1. Given I have already generated matches, suggestions, or a cover letter draft, When I go
   back and change my resume or the requirement list, Then those dependent results are marked
   out of date.
2. Given a dependent result is marked out of date, When I reach the export step, Then export
   is blocked until I regenerate that result.
3. Given I regenerate the out-of-date result, When regeneration completes, Then the block is
   lifted and export becomes available again.

---

### US-FOUNDATION-03 — Know my data stays private
**As a** user, **I want** to be told how my resume data is handled and know nothing is saved
after my session, **so that** I feel safe uploading personal information.

- **Persona**: Student Seeking an Internship
- **Priority**: Must
- **Covers**: NFR-104, NFR-101, NFR-105

**Acceptance Criteria**
1. Given I am using the app, When I view the resume upload screen, Then the UI states that my
   resume text is sent to the Anthropic API for processing — before I submit it.
2. Given I finish or abandon my session, When the session ends, Then no resume content, JD
   text, or confirmed facts are retained anywhere.
3. Given I return to the app later, When I look for a saved history or account, Then none
   exists — there are no user accounts and no saved history.

---

### US-FOUNDATION-04 — Get a clear message if generation fails
**As a** user, **I want** a plain, consistent message when the assistant can't generate a
result, **so that** I know to retry instead of seeing a broken or partial result.

- **Persona**: Student Seeking an Internship
- **Priority**: Must *(see note below)*
- **Covers**: NFR-303, NFR-301
- **Cross-referenced by**: US-MATCH-01 (requirement extraction/matching), US-SUGGEST-01
  (suggestion generation), US-COVER-01 (cover letter draft) — resolved via the
  `story-generation-clarification-questions.md` answer: one dedicated Foundation-unit story,
  not duplicated per slice, because every LLM call goes through the single client wrapper
  (NFR-301) and the failure message is identical regardless of which step triggered it.

**Acceptance Criteria**
1. Given the LLM refuses to respond or hits the max-token limit, When that call returns, Then
   I see "could not generate, please retry" and nothing partial is shown.
2. Given this failure happens during requirement extraction, matching, suggestion generation,
   or cover letter drafting, When it occurs, Then the same message and behavior apply
   regardless of which step triggered the call.

> **Note (priority, flagged for confirmation)**: `story-generation-plan.md` Q8 enumerated
> Must-priority FR groups and "the privacy and security NFRs" but did not explicitly name
> NFR-3xx (Reliability and LLM Handling). This story is labeled **Must** because a working
> upload-to-export flow cannot tolerate an unhandled LLM failure — consistent with Q8's own
> definition of Must ("needed for one complete flow... with the truthfulness and privacy
> rules intact"). Flagged here for explicit confirmation during story review, the same way
> `requirements.md` assumption A-1 was flagged during Requirements Analysis.

---

## Resume Pipeline

### US-RESUME-01 — Upload or paste my resume
**As a** user, **I want** to provide my resume as a PDF, DOCX, or pasted text, with clear
rejection messages when my input can't be used, **so that** I can get started without
confusion.

- **Persona**: Student Seeking an Internship
- **Priority**: Must
- **Covers**: FR-110, FR-111, FR-112, FR-113, FR-114

**Acceptance Criteria**
1. Given a valid PDF, DOCX, or pasted plain text resume, When I submit it, Then it is accepted
   for parsing.
2. Given a file with the wrong extension, a mismatched file signature, or a file over the size
   or page limit, When I submit it, Then it is rejected with a clear reason before parsing.
3. Given pasted resume text over 20,000 characters, When I submit it, Then it is rejected.
4. Given a PDF whose extracted text is under 200 characters, When I submit it, Then it is
   rejected as a likely scanned PDF and I'm told to paste my resume text instead.

---

### US-RESUME-02 — Review and correct my parsed resume
**As a** user, **I want** to see how my resume was broken into items and fix any mistakes
before anything else happens, **so that** every later step works from a resume that's actually
correct.

- **Persona**: International Student
- **Priority**: Must
- **Covers**: FR-120, FR-130, FR-131

**Acceptance Criteria**
1. Given my resume has been parsed, When I view the review screen, Then I see each item with a
   stable ID, a section label (education/experience/project/skills/other), and its text.
2. Given a parsing mistake, When I edit text, change a section label, merge items, split an
   item, or delete an item, Then my correction is applied.
3. Given I confirm my corrections, When I move to the JD step, Then the corrected item list
   — not the original parse — is used for matching, source attribution, verification, and
   export.

---

## JD & Matching

### US-MATCH-01 — Paste a job description and extract its requirements
**As a** user, **I want** to paste a job description and have its requirements pulled out
automatically, **so that** I don't have to manually list what the job is asking for.

- **Persona**: New Graduate
- **Priority**: Must
- **Covers**: FR-201, FR-210, FR-113
- **Cross-ref**: see US-FOUNDATION-04 for what happens if this LLM call fails.

**Acceptance Criteria**
1. Given a job description, When I paste it as text, Then it is accepted (scraping job sites
   is out of scope).
2. Given the pasted JD, When extraction runs, Then I receive a list of discrete requirements as
   a validated, structured list.
3. Given pasted JD text over 15,000 characters, When I submit it, Then it is rejected.

---

### US-MATCH-02 — Review and edit the extracted requirements
**As a** user, **I want** to add, fix, or remove requirements the system extracted from the
JD, **so that** matching runs against a requirement list I trust.

- **Persona**: New Graduate
- **Priority**: Must
- **Covers**: FR-211, FR-212

**Acceptance Criteria**
1. Given the extracted requirement list, When I add, edit, or delete a requirement, Then my
   change is applied to the list used for matching.
2. Given matching has already run, When I change the requirement list afterward, Then matching
   runs again on the updated list.

---

### US-MATCH-03 — See every JD requirement checked against my resume
**As a** user, **I want** every requirement from the JD labeled against my resume with its
supporting evidence, **so that** I can see exactly where I match and where the real gaps are.

- **Persona**: Student Seeking an Internship
- **Priority**: Must
- **Covers**: FR-220, FR-221, FR-222, FR-223, FR-224
- **Note**: FR-222/FR-223 updated by the change request of 2026-11-22 after User Stories
  review — see `requirements.md` §15.

**Acceptance Criteria**
1. Given the requirement list, When matching completes, Then every requirement appears exactly
   once with one of three statuses — `found`, `needs_confirmation`, or `not_found` — and none
   are silently omitted.
2. Given a requirement with only partial evidence in my resume, When it is matched, Then it is
   labeled `needs_confirmation` (not `found`) and triggers a follow-up question about the
   depth of that experience.
3. Given a requirement's status, When I view the checklist, Then `found` always shows **at
   least one** supporting evidence ID labeled "evidence" (a `found` match can never have zero
   evidence); `needs_confirmation` may show partial evidence labeled "partial evidence"; and
   `not_found` always shows no evidence. Both invariants — `found` implies non-empty
   `evidence_ids`, and `not_found` implies empty `evidence_ids` — are property-tested per
   FR-223.

---

### US-MATCH-04 — Answer follow-up questions in manageable rounds
**As a** user, **I want** to be asked about JD requirements my resume doesn't clearly show,
in small rounds ordered by importance, **so that** I'm not overwhelmed and the most important
gaps get addressed first.

- **Persona**: Student Seeking an Internship
- **Priority**: Must
- **Covers**: FR-301, FR-310, FR-311, FR-312

**Acceptance Criteria**
1. Given requirements lacking sufficient evidence, When the system determines which need a
   question, Then it generates a follow-up question for each.
2. Given multiple follow-up questions, When they're presented, Then they come in rounds of 3,
   ordered by how strongly the JD emphasizes each requirement.
3. Given I finish a round, When I'm prompted, Then I can choose "answer more" or "continue to
   suggestions."
4. Given a requirement's question is skipped or never reached because I moved on, When I
   proceed, Then that requirement stays unconfirmed and is not used in the resume or cover
   letter.

---

### US-MATCH-05 — Confirm a fact by answering a follow-up question
**As a** user, **I want** my answer to a follow-up question to become a fact the system can
use later, in my own words, **so that** my real experience can be used without putting words
in my mouth.

- **Persona**: Student Seeking an Internship
- **Priority**: Must
- **Covers**: FR-320, FR-321

**Acceptance Criteria**
1. Given I answer a follow-up question, When I submit my answer, Then it is stored as a
   confirmed fact containing the question asked and my answer verbatim, in my own words.
2. Given I answer "no" to a question, When I submit that answer, Then it is recorded but
   creates no usable source material.

---

## Suggestions & Verification

### US-SUGGEST-01 — See rewrite suggestions grouped by why they exist
**As a** user, **I want** to see suggested rewrites next to my original lines, with their
source shown, and grouped by whether they're JD-driven or just weakly worded, **so that** I
can focus on what matters most to the job I'm applying for.

- **Persona**: International Student
- **Priority**: Must
- **Covers**: FR-401, FR-410, FR-411, PR-2.1, PR-2.2, PR-2.3, NFR-505
- **Cross-ref**: see US-FOUNDATION-04 for what happens if this LLM call fails.

**Acceptance Criteria**
1. Given a resume line with a suggestion, When I view it, Then the suggestion is shown next to
   the original line with its source displayed.
2. Given my resume and the JD, When suggestions are generated, Then every line that is evidence
   for a JD requirement gets a suggestion, and lines judged weakly worded also get a
   suggestion.
3. Given both kinds of suggestions exist, When I view the suggestions screen, Then JD-driven
   and weakly-worded suggestions appear as two separate, visually distinct groups.
4. Given a generated suggestion, When it is produced, Then it avoids empty phrases and
   repeated JD-keyword stuffing and preserves my real seniority level and tone (PR-2.1–PR-2.3);
   this is checked manually during the truthfulness evaluation set (NFR-505).

---

### US-SUGGEST-02 — Trust that suggestions are checked before I see them
**As a** user, **I want** every suggestion checked by code before it reaches me, **so that** I
never see an invented number or an exaggerated verb presented as a clean suggestion.

- **Persona**: International Student
- **Priority**: Must
- **Covers**: FR-420, FR-421, FR-422, FR-423

**Acceptance Criteria**
1. Given a proposed suggestion, When it is generated, Then it passes through `verify_suggestion`
   before ever reaching the UI.
2. Given a proposed suggestion contains a number not present in any of its sources, When it is
   verified, Then it is flagged `unsupported_number`.
3. Given a proposed suggestion uses a verb implying more seniority than any source verb, When
   it is verified, Then it is flagged `verb_inflation`.
4. Given a proposed suggestion has empty `source_ids`, or a `source_id` not present in my
   session, When it is verified, Then it is flagged `invalid_source_ids`.

---

### US-SUGGEST-03 — See only traceable suggestions, with flagged ones clearly marked
**As a** user, **I want** suggestions that can't be traced to my resume to be hidden, and
suggestions with a specific problem to be clearly marked and require a fix, **so that** I'm
never shown something untraceable, and I know exactly what to fix before accepting something
flagged.

- **Persona**: International Student
- **Priority**: Must
- **Covers**: FR-430, FR-431
- **Note**: what happens once a flag is cleared by re-verification is covered in
  US-SUGGEST-05 (FR-432), not here — moved per the 2026-11-22 story review, so that cutting
  the Should-priority US-SUGGEST-05 can never leave this Must story depending on it.

**Acceptance Criteria**
1. Given a suggestion flagged `invalid_source_ids`, When the suggestion list is shown, Then it
   is dropped from the list, and I instead see a count of withheld suggestions (e.g. "2
   suggestions were withheld because they could not be traced to your resume") without
   exposing the untraceable text.
2. Given a suggestion flagged `unsupported_number` or `verb_inflation`, When I view it, Then it
   shows a warning badge naming the flag, and I cannot accept it as-is — I must edit it first.

---

### US-SUGGEST-04 — Accept, reject, or edit each suggestion
**As a** user, **I want** to decide line by line whether to accept, reject, or edit each
suggestion, **so that** the final resume is one I actually agree with.

- **Persona**: New Graduate
- **Priority**: Must
- **Covers**: FR-440, FR-441, FR-412

**Acceptance Criteria**
1. Given a suggestion, When I accept, reject, or edit it, Then its status is tracked as
   `pending`, `accepted`, `rejected`, or `edited` accordingly.
2. Given I edit a suggestion's text myself, When I save my edit, Then that text is treated as
   confirmed by my own authorship.
3. Given a line whose suggestion I rejected, left pending, or that never received a suggestion,
   When the final resume content is assembled, Then that line's original text is used — it is
   never deleted.

---

### US-SUGGEST-05 — Confirm a flagged suggestion is actually true
**As a** user, **I want** to tell the system a flagged claim is true and back it up in my own
words, **so that** I don't lose a suggestion just because the system couldn't verify it on its
own.

- **Persona**: Student Seeking an Internship
- **Priority**: Should
- **Covers**: FR-330, FR-331, FR-332, FR-333, FR-432
- **Note (cross-slice ownership)**: per `requirements.md` §11, the
  `POST /api/sessions/{sid}/facts` endpoint this story uses belongs to the JD/Matching unit of
  work even though the user takes this action from the Suggestions screen; Units Generation
  will assign the explicit cross-slice owner.
- **Note (moved criterion)**: criterion 4 moved here from US-SUGGEST-03 per the 2026-11-22
  story review, so a Must-priority story never depends on a Should-priority one.

**Acceptance Criteria**
1. Given a flagged suggestion I know is true, When I use the "this is true, let me confirm it"
   control and type the supporting detail in my own words, Then a new confirmed fact is
   created via `POST /api/sessions/{sid}/facts`.
2. Given that new fact, When the originating suggestion is re-verified, Then only that
   suggestion is re-verified — no other suggestion is regenerated, so my existing accept/reject
   decisions are preserved — and the new fact's ID is added to that suggestion's `source_ids`
   before `verify_suggestion` runs.
3. Given the new fact exists, When I later reach the cover letter step, Then it is available as
   ordinary confirmed source material.
4. Given a flagged suggestion is cleared of all flags by this re-verification, When
   re-verification completes, Then it becomes an ordinary suggestion, acceptable without
   further editing.

---

### US-SUGGEST-06 — Reorder bullets to highlight relevant experience
**As a** user, **I want** to reorder the bullets within one resume entry, **so that** I can put
my most relevant experience first for this particular job, without inventing anything new.

- **Persona**: New Graduate
- **Priority**: Could
- **Covers**: FR-501, FR-502, FR-505
- **Note**: suggesting which bullets to *drop* is out of MVP scope (FR-506) — I can already
  delete items myself during US-RESUME-02.

**Acceptance Criteria**
1. Given a proposed reorder of bullets within one entry, When I review it, Then I can accept or
   reject it.
2. Given a reorder suggestion, When it's generated, Then it uses a separate `ReorderSuggestion`
   record (entry ID, proposed order, rationale, status) and does not go through
   `verify_suggestion`.
3. Given an accepted reorder and accepted rewrites on the same entry, When I export, Then
   bullets appear in the accepted order, each using its accepted text — the two are applied
   independently.

---

### US-SUGGEST-07 — Trust that a reorder never loses or duplicates a bullet
**As a** user, **I want** any proposed reorder to be guaranteed to contain exactly my existing
bullets, **so that** I never end up with a missing, duplicated, or foreign bullet from
accepting a reorder.

- **Persona**: New Graduate
- **Priority**: Could
- **Covers**: FR-503, FR-504

**Acceptance Criteria**
1. Given a proposed bullet order, When it's checked, Then it must be an exact permutation of
   that entry's existing bullet IDs — no missing, duplicate, or foreign IDs.
2. Given a proposed order fails that check, When it's evaluated, Then it is dropped before I
   ever see it (this permutation rule is property-tested per FR-504).

---

### US-SUGGEST-08 — Choose a writing style, including keeping my own voice
**As a** user, **I want** to choose how my suggestions and cover letter are written, including
an option to match my own writing sample, **so that** the output sounds like something I would
actually say.

- **Persona**: International Student
- **Priority**: Should
- **Covers**: FR-601, FR-602, FR-603, FR-604
- **Cross-referenced by**: US-COVER-01 — the style chosen here also applies to the cover
  letter, so that story does not repeat this criterion.

**Acceptance Criteria**
1. Given the style options, When I choose, Then I can pick `natural`, `professional`, or
   `keep_my_voice`.
2. Given I choose `keep_my_voice`, When I'm prompted, Then I may paste a writing sample up to
   3,000 characters.
3. Given I provide a writing sample, When it's used in a prompt, Then it is wrapped in its own
   tag and treated as untrusted data to analyze, never as instructions.
4. Given I've chosen a style, When suggestions or the cover letter are generated, Then that
   same style applies to both.

---

## Cover Letter

### US-COVER-01 — Get a cover letter draft built only from confirmed content
**As a** user, **I want** a cover letter draft built from my settled resume content and
confirmed facts, **so that** I get a useful starting draft without anything made up.

- **Persona**: New Graduate
- **Priority**: Must
- **Covers**: FR-701, FR-702, PR-2.1, PR-2.2, PR-2.3, NFR-505
- **Note**: uses the style chosen in US-SUGGEST-08 (FR-604); see US-FOUNDATION-04 for what
  happens if this LLM call fails.

**Acceptance Criteria**
1. Given my final resume content (accepted/edited suggestions, original text for rejected,
   pending, or unsuggested lines, and accepted bullet order) plus any confirmed facts, When the
   draft is generated, Then it uses only that material — no other source.
2. Given the draft is generated, When I view it, Then it is three to four short paragraphs on
   one page: opening, why this role, relevant evidence, closing.
3. Given the draft is generated, When it is produced, Then it avoids empty phrases and
   repeated JD-keyword stuffing and preserves my real seniority level and tone (PR-2.1–PR-2.3);
   this is checked manually during the truthfulness evaluation set (NFR-505).

---

### US-COVER-02 — Personalize the letter without the system guessing
**As a** user, **I want** to optionally supply a company name and role title, and have the
assistant never guess them on its own, **so that** the letter is only ever personalized with
details I actually gave it.

- **Persona**: New Graduate
- **Priority**: Must
- **Covers**: FR-703, FR-704

**Acceptance Criteria**
1. Given I provide or confirm a company name and role title, When the letter is drafted, Then
   those details are used; if I don't, Then the assistant does not infer them from the JD.
2. Given no company name or role title was supplied, When the letter opens, Then it uses a
   generic opening such as "Dear Hiring Manager."

---

### US-COVER-03 — Edit the draft and see it formatted consistently
**As a** user, **I want** to edit the cover letter draft and have it formatted like a plain
business letter, **so that** it looks professional and consistent with my resume.

- **Persona**: International Student
- **Priority**: Must
- **Covers**: FR-705, FR-706

**Acceptance Criteria**
1. Given the draft, When I review it, Then I can edit it before export.
2. Given the draft is ready for export, When it is rendered, Then my name and contact details
   from the resume appear at the top, with no recipient address block and no placeholders.

---

## Export

### US-EXPORT-01 — Choose what to export and in what format
**As a** user, **I want** to choose whether to export my resume, my cover letter, or both, and
in DOCX or PDF, **so that** I get exactly the files I need for a given application.

- **Persona**: New Graduate
- **Priority**: Must
- **Covers**: FR-801, FR-802

**Acceptance Criteria**
1. Given the export step, When I choose, Then I can select resume, cover letter, or both, and
   DOCX or PDF.
2. Given I select both resume and cover letter, When they are exported, Then I receive two
   separate files in one `.zip`.

---

### US-EXPORT-02 — Export a clean, ATS-friendly resume
**As a** user, **I want** my exported resume to use a plain, applicant-tracking-system-friendly
layout, **so that** it reads correctly wherever I submit it.

- **Persona**: Student Seeking an Internship
- **Priority**: Must
- **Covers**: FR-803, FR-806, FR-807

**Acceptance Criteria**
1. Given my exported resume, When I open it, Then it is single-column and plain: name and
   contact at the top, a fixed section order (Education, Experience, Projects, Skills), one
   font, bold section headers, and no color, tables, or icons.
2. Given the export format I chose, When the file is generated, Then DOCX is written with
   `python-docx` and PDF with `fpdf2` — preserving my original upload's layout is out of scope.
3. Given the fixed section order, When I export, Then section reordering is not available
   (out of MVP scope).

---

### US-EXPORT-03 — Export only content I've confirmed
**As a** user, **I want** my exported files to contain only my final resume content and
cover letter, in a consistent format, **so that** nothing unconfirmed or inconsistently
formatted ends up in what I submit.

- **Persona**: International Student
- **Priority**: Must
- **Covers**: FR-805, FR-804

**Acceptance Criteria**
1. Given my final resume content (accepted/edited suggestion text, original text for rejected,
   pending, or unsuggested lines, and accepted reordering) and, if selected, the cover letter,
   When I export, Then the files contain exactly that — nothing else.
2. Given both files are exported, When I open the cover letter, Then it uses the same font and
   plain business-letter layout as the resume (per US-COVER-03 / FR-705) — the two are one
   specification, not two conflicting ones.

---

### US-EXPORT-04 — Know if anything hasn't been reviewed before I export
**As a** user, **I want** a heads-up if I still have unreviewed suggestions when I reach
export, **so that** I know what I'm submitting without being blocked from exporting if that's
what I want.

- **Persona**: New Graduate
- **Priority**: Must
- **Covers**: FR-808, FR-412
- **Note**: this notice is informational and non-blocking, distinct from the blocking
  staleness check in US-FOUNDATION-02 (FR-904).

**Acceptance Criteria**
1. Given one or more suggestions are still `pending` when I reach export, When the export
   screen loads, Then I see a notice such as "N suggestions have not been reviewed."
2. Given that notice is shown, When I proceed to export anyway, Then export is not blocked, and
   the still-pending lines export with their original text.

---

## Story Count Summary

| Group | Stories | Must | Should | Could |
|---|---|---|---|---|
| Foundation Unit | 4 | 4 | 0 | 0 |
| Resume Pipeline | 2 | 2 | 0 | 0 |
| JD & Matching | 5 | 5 | 0 | 0 |
| Suggestions & Verification | 8 | 4 | 2 | 2 |
| Cover Letter | 3 | 3 | 0 | 0 |
| Export | 4 | 4 | 0 | 0 |
| **Total** | **26** | **22** | **2** | **2** |

Every functional and relevant non-functional requirement group in `requirements.md` §5–§6 is
covered by at least one story above, or is explicitly named in "How to Read This Document" as
internal-only and therefore out of scope for stories.
