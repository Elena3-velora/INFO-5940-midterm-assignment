# Requirements Verification Questions — Storywell

Please answer each question by filling in the letter choice after the `[Answer]:` tag.
If none of the options match what you want, choose the last option (**Other**) and describe
your preference after the `[Answer]:` tag.

**Context**: `vision.md` and `technical-environment.md` are detailed and gave me the stack,
the MVP scope, the product rules, and the prohibited libraries. The questions below cover
(a) the three open questions you listed in `vision.md` Section 8, (b) behavioural decisions
that are not yet pinned down and would change the design, and (c) the AI-DLC extension
opt-ins.

---

## Section A: Open Questions from vision.md

## Question 1
**Partial matches.** `RequirementMatch.status` has three values: `found`,
`needs_confirmation`, `not_found`. How should a requirement the user *partly* has be
handled — for example, the JD asks for SQL and the resume says "used SQL in one database
course"?

A) Label it `found` and let the evidence line speak for itself — the user sees the course
bullet as the evidence and judges whether it is enough

B) Label it `needs_confirmation` and ask a follow-up question about depth ("How much SQL
have you written outside class?"), so the user can add context that becomes a confirmed fact

C) Add a fourth status, `partial`, with its own colour in the checklist, and ask a follow-up
question as well

D) Label it `needs_confirmation` only when the evidence is coursework or a class project;
treat work or internship evidence as `found`

X) Other (please describe after [Answer]: tag below)

[Answer]: B — Partial evidence should not be treated as a full match. Asking a follow-up keeps us within our truthfulness rule, and the user's answer becomes a confirmed fact that later steps can use. The checklist should still show the course bullet as partial evidence next to the question.

## Question 2
**Follow-up question volume.** When a JD lists many requirements with no resume evidence,
the assistant could generate a long list of follow-up questions and tire the user out. How
should this be limited?

A) Hard cap per session — ask at most 5 questions, chosen from the requirements the JD
emphasises most, and skip the rest

B) Hard cap per session — ask at most 10 questions, same prioritisation

C) No cap, but present them as a skippable list: every gap gets a question, each one has a
"skip this" control, and the user can stop answering at any point and move on

D) Ask in rounds — show 3 questions at a time, and after each round let the user choose
"answer more" or "continue to suggestions"

X) Other (please describe after [Answer]: tag below)

[Answer]: D — Ask in rounds of 3, ordered by how much the JD emphasises each requirement, so the most important gaps are asked first. After each round the user chooses "answer more" or "continue to suggestions". Any requirement that is skipped or never asked stays unconfirmed, so it is not used in the resume or the cover letter.

## Question 3
**Export template.** `fpdf2` and `python-docx` write from one fixed template. What should
that template look like?

A) Single column, plain and ATS-friendly — name and contact at the top, then sections in a
fixed order (Education, Experience, Projects, Skills), one font, bold section headers, no
colour, no tables, no icons

B) Single column and ATS-friendly as in A, but the user can reorder the sections before
exporting

C) Two column layout with a sidebar for skills and contact details, for a more designed look

D) Match the look of the uploaded resume as closely as the libraries allow

X) Other (please describe after [Answer]: tag below)

[Answer]: A — A single-column, plain, ATS-friendly template is the safest choice for the North American job market, and matching the original layout (D) is out of scope in our vision. Section reordering (B) can be a future feature. The cover letter export should use the same font and a standard plain business-letter format.

---

## Section B: Behaviour Not Yet Specified

## Question 4
**Export packaging.** The `POST /api/sessions/{sid}/export` endpoint exports confirmed
content. What should a single export call return?

A) One file containing the resume only — the cover letter is a separate export call and a
separate file

B) Two separate files in one download (a .zip containing resume and cover letter)

C) One file containing the resume and the cover letter as separate pages in the same
document

D) The user picks what to include (resume, cover letter, or both) and the format (DOCX or
PDF) for each export call

X) Other (please describe after [Answer]: tag below)

[Answer]: D — The user picks what to export (resume, cover letter, or both) and the format (DOCX or PDF). When both are selected, they are returned as two separate files in one .zip, because most application systems ask for the resume and cover letter as separate uploads. Merging them into one document (C) would force users to split the file themselves.

## Question 5
**Flagged suggestions.** `technical-environment.md` Section 8 says content that fails
verification "is either dropped or shown with its flags so the user can decide." Which one
should the MVP do?

A) Drop it silently — the user only ever sees suggestions that passed every check

B) Drop it from the suggestion list, but show a count ("2 suggestions were withheld because
they could not be traced to your resume") so the behaviour is visible without exposing
unverified text

C) Show it with a visible warning badge naming the flag ("adds a number not in your resume:
40%"), and let the user accept, reject, or edit it

D) Show it with a warning badge as in C, but it cannot be accepted as-is — the user must
edit it before it can be accepted

X) Other (please describe after [Answer]: tag below)

[Answer]: X — Handle flags by type. (1) Suggestions with invalid or missing source IDs are dropped from the list, and the UI shows a count as in B ("2 suggestions were withheld because they could not be traced to your resume"). (2) Suggestions flagged for an unsupported number or verb inflation are shown with a warning badge naming the flag, and they cannot be accepted as-is; the user must edit them first, as in D. If a flagged number is actually true, the user can add it as a confirmed fact through a follow-up question, so later suggestions can use it.

## Question 6
**Suggestion scope.** Which resume lines should get rewrite suggestions?

A) Every line in the resume, so the whole document gets a pass

B) Only lines that are evidence for a JD requirement, so the user reviews a short, focused
list

C) Lines that are evidence for a JD requirement, plus any line the assistant judges to be
weakly worded, with the two kinds visually distinguished

D) The user chooses the scope before generation (whole resume, or JD-relevant lines only)

X) Other (please describe after [Answer]: tag below)

[Answer]: C — The core of the product is tailoring to the JD, so JD-evidence lines always get suggestions. Lines the assistant judges to be weakly worded also get suggestions, because many of our target users write in their second language, but they are shown in a separate, visually distinct group so the JD-related changes stay the focus. Lines with no suggestion are kept exactly as in the original resume when exporting.

## Question 7
**Reviewing the parsed resume.** PDF and DOCX extraction can split or merge lines wrongly.
After upload, should the user be able to correct the parsed `ResumeItem` list before
anything else happens?

A) Yes — show the parsed items and let the user edit text, fix section labels, merge, split,
and delete items before the JD step

B) Show the parsed items read-only as a confirmation step ("does this look right?"), with
re-upload or paste-as-text as the only remedy if it is wrong

C) No review step — go straight to the JD step and rely on the per-line accept/reject/edit
controls later to absorb any parsing problems

X) Other (please describe after [Answer]: tag below)

[Answer]: A — The parsed ResumeItem list is the source of truth for every later step: JD matching, source IDs on suggestions, and verification all depend on it, so parsing errors must be fixed before the JD step. Users are only correcting their own resume text, so this does not break our truthfulness rule. After this review, the corrected items are treated as the original resume for all later steps and for export.

## Question 8
**Reordering and section-level changes.** Product Rule 1 says "order can change, and
relevant experience can be highlighted." Is that in the MVP, or is the MVP line-by-line
rewriting only?

A) Line-by-line rewriting only — reordering and dropping content are out of the MVP

B) Line-by-line rewriting, plus the assistant may suggest reordering bullets within a single
entry, which the user accepts or rejects

C) Line-by-line rewriting, plus the assistant may suggest both reordering and which bullets
to drop as less relevant to this JD

D) Rewriting as in A, plus the user can manually reorder and drop items themselves before
export, with no suggestions from the assistant

X) Other (please describe after [Answer]: tag below)

[Answer]: B — Our Product Rule 1 allows changing order to highlight relevant experience, and reordering bullets within one entry does this without adding or removing any facts. The user accepts or rejects each reorder suggestion. Suggesting what to drop (C) is out of the MVP because a wrong judgment could remove experience the user cares about; users can already delete items themselves in the parsed-resume review step (Question 7). Section order stays fixed, matching the export template in Question 3.

## Question 9
**Editing the JD requirement list.** The assistant extracts requirements from the pasted JD.
Extraction can miss a requirement or invent an over-specific one. Should the user be able to
change that list?

A) Yes — the user can add, edit, and delete requirements before the matching step runs

B) The user can delete requirements they consider irrelevant, but not add or edit

C) No — the extracted list is read-only; a bad extraction is handled by re-pasting the JD

X) Other (please describe after [Answer]: tag below)

[Answer]: A — Like the parsed resume review in Question 7, the requirement list is an input to every later step, so the user should be able to add missed requirements, fix over-specific ones, and delete irrelevant ones before matching runs. This only changes how the job is described, not the user's own facts, so it does not affect our truthfulness rule. Re-pasting the JD (C) would likely produce the same extraction errors. If the list is changed after matching, matching runs again on the updated list.

## Question 10
**Cover letter shape.** What should the generated draft be?

A) Three to four short paragraphs on one page, no addresses or date block — opening, why
this role, relevant evidence, closing

B) A full formal business letter including date, recipient block, and salutation, with
placeholders where the user has not supplied details

C) Same body as A, but the user picks the length (short — about 150 words, or standard —
about 300 words)

D) Same body as A, and the user can also supply the company name and role title to be woven
into the opening

X) Other (please describe after [Answer]: tag below)

[Answer]: D — Three to four short paragraphs on one page (opening, why this role, relevant evidence, closing). The user can enter the company name and role title, and they are only used if the user provides or confirms them; the assistant must not guess them from the JD. If they are not provided, the letter uses a generic opening such as "Dear Hiring Manager". On export, the user's name and contact details from the resume appear at the top, with no recipient address block or placeholders, which matches the plain format in Question 3.

## Question 11
**UI flow.** `frontend/src/pages/` suggests more than one page. How should the user move
through the steps?

A) A linear wizard with one step per screen (resume, JD, checklist, questions, suggestions,
cover letter, export) and a visible progress indicator; back navigation allowed

B) A single long scrolling page where each section unlocks as the previous one completes

C) A two pane layout — inputs and the requirement checklist on the left, the resume with
suggestions on the right — with the cover letter and export as separate tabs

X) Other (please describe after [Answer]: tag below)

[Answer]: A — A linear wizard with a visible progress indicator and back navigation, which also matches our earlier prototype. The steps are: resume upload, parsed resume review (Question 7), JD input, requirement list review (Question 9), requirement checklist, follow-up questions, suggestions, cover letter, and export. If the user goes back and changes the resume or the requirement list, the later results (matching, suggestions, cover letter) are marked out of date and must be regenerated before export.

## Question 12
**Development parallelism.** This affects how I decompose the work into units in the Units
Generation stage.

A) Solo developer — sequence the units for one person working through them in order

B) Small team (2–4 people) — decompose so that backend and frontend units can be built in
parallel, with the API contract fixed first

C) Small team as in B, and I want the vertical slices split by feature (resume pipeline,
JD/matching, suggestions/verification, cover letter, export) so people can own a slice each

X) Other (please describe after [Answer]: tag below)

[Answer]: C (with additions) — We are a team of three, and each of us uses Claude Code, so we want vertical slices by feature (resume pipeline, JD/matching, suggestions/verification, cover letter, export), each covering both backend and frontend, so one person can own a slice end to end. Please add a shared foundation unit that is built first: the Pydantic data models, the in-memory session store, the LLM client wrapper, the API contract, and the app shell with wizard navigation. The feature slices then build on it in parallel. Linghui owns the foundation unit and final integration.

---

## Section C: AI-DLC Extension Opt-Ins

These three questions decide which AI-DLC extension rule sets are enforced as hard
constraints for the rest of the workflow. Answering "No" means the extension's rules are
never loaded.

## Question: Security Extensions
Should security extension rules be enforced for this project?

A) Yes — enforce all SECURITY rules as blocking constraints (recommended for
production-grade applications)

B) No — skip all SECURITY rules (suitable for PoCs, prototypes, and experimental projects)

X) Other (please describe after [Answer]: tag below)

[Answer]: X — Enforce the security baseline, but treat it according to our deployment model: a local-only, single-user app bound to 127.0.0.1, with no accounts, no database, and no persistent storage. The following rules apply and should be enforced: SECURITY-03 (structured logging, never logging resume or JD content), SECURITY-04, SECURITY-05, SECURITY-09, SECURITY-10 dependency pinning via lock files, SECURITY-11 secure design (except rate limiting), SECURITY-12 no hardcoded credentials (API key in .env only), SECURITY-13 safe deserialization, and SECURITY-15. Please mark these as N/A because they do not fit a local single-user app: SECURITY-01 (no data stores), SECURITY-02 and SECURITY-07 (no cloud network), SECURITY-06 and SECURITY-08 (no accounts or IAM), SECURITY-12 password, MFA, and login rules (no authentication), SECURITY-11 rate limiting (no public endpoints), and SECURITY-10 vulnerability scanning and SBOM plus SECURITY-14 alerting and monitoring (no production deployment).

## Question: Resiliency Extensions
Should the resiliency baseline be applied to this project?

**What this extension is.** Enabling it applies a set of **directional, design-time best
practices** for building resilient systems, derived from the **AWS Well-Architected
Framework (Reliability Pillar)** and resilience-review guidance. It steers requirements,
design, and code toward fault tolerance, high availability, observability, and
recoverability — covering 15 practice areas across business goals, change management,
observability, high availability, disaster recovery, and continuous improvement.

**What this extension is NOT.** Enabling it does **not** make your workload
production-ready, nor does it certify or guarantee any availability, RTO, or RPO target. It
is a **starting point** that scaffolds good resiliency decisions early — it is not a
substitute for a formal **AWS Well-Architected Review** of the built system.

Treat the output as a well-grounded **first draft of your resiliency posture** to build on
and validate — not a finished, production-certified result.

A) Yes — apply the resiliency baseline as directional best practices and design-time
guidance (recommended for business-critical workloads, as an informed starting point that
you can validate and harden before go-live)

B) No — skip the resiliency baseline (suitable for PoCs, prototypes, and experimental
projects where rapid iteration matters more than reliability)

X) Other (please describe after [Answer]: tag below)

[Answer]: B — Storywell is a local-only, single-user app built for a course demo, so high availability, disaster recovery, and multi-region concerns do not apply and would add design work outside our MVP. The basic resilience we do need is already covered in technical-environment.md: LLM failures, refusals, and max-token cutoffs are shown to the user as a retryable error with no partial output, and the SDK retries transient API errors.

## Question: Property-Based Testing Extension
Should property-based testing (PBT) rules be enforced for this project?

A) Yes — enforce all PBT rules as blocking constraints (recommended for projects with
business logic, data transformations, serialization, or stateful components)

B) Partial — enforce PBT rules only for pure functions and serialization round-trips
(suitable for projects with limited algorithmic complexity)

C) No — skip all PBT rules (suitable for simple CRUD applications, UI-only projects, or thin
integration layers with no significant business logic)

X) Other (please describe after [Answer]: tag below)

[Answer]: B — Partial enforcement fits our project. Our most important logic is the deterministic verification in app/verification (unsupported numbers, verb inflation, source ID checks), which are pure functions and good targets for properties, for example "a number copied from the source is never flagged" and "a suggestion identical to its source has no flags". Pydantic model serialization round-trips should also be covered. Use hypothesis with pytest as the PBT framework. Stateful PBT for the session store is not needed for the MVP.


---

**When you have filled in every `[Answer]:` tag, let me know and I will validate the answers
for contradictions and then write the requirements document.**
