# Requirements: Storywell

**Status**: Approved 2026-11-22 (change request of 2026-11-22 applied — see §15)
**Stage**: INCEPTION — Requirements Analysis
**Depth**: Comprehensive
**Date**: 2026-11-22

---

## 1. Intent Analysis Summary

| Dimension | Assessment |
|---|---|
| **User request** | "Using AI-DLC, I want to start a new project. Please read vision.md and technical-environment.md, then begin the AI-DLC workflow." |
| **Request type** | New Project (greenfield) |
| **Scope estimate** | System-wide — FastAPI backend, React frontend, LLM pipeline, deterministic verification layer, document export |
| **Complexity estimate** | Complex |
| **Request clarity** | Clear — two detailed input documents, plus 15 verification answers and 5 clarification answers |
| **Project type** | Greenfield (no existing code; Reverse Engineering skipped) |

**Why comprehensive depth**: the scope is system-wide; the truthfulness rule is a
correctness-critical constraint that cuts across every feature rather than living in one
module; the system handles personal data; and it depends on an external LLM whose output
cannot be trusted by default. Each of these raises the cost of a missed requirement.

**Source documents**: `vision.md`, `technical-environment.md`,
`requirement-verification-questions.md` (15 answers),
`requirements-clarification-questions.md` (5 answers).

---

## 2. Product Summary

Storywell is a **local-only web application** that helps students and new graduates in the
North American job market tailor a resume and cover letter to a specific job description.
Every suggestion must be traceable to the user's original resume or to a fact the user
explicitly confirmed.

**Processing pipeline** (text representation):

```
create session
  -> upload/paste resume -> parse to ResumeItems -> USER REVIEWS AND CORRECTS ITEMS
  -> paste JD -> extract requirements -> USER REVIEWS AND EDITS REQUIREMENTS
  -> match evidence -> requirement checklist (found / needs_confirmation / not_found)
  -> follow-up questions in rounds of 3 -> ConfirmedFacts
  -> generate rewrite + reorder suggestions -> deterministic verification
  -> user accepts / rejects / edits each suggestion
  -> cover letter draft from accepted content + confirmed facts
  -> export (resume and/or cover letter, DOCX and/or PDF)
```

---

## 3. Target Users

Carried from `vision.md` Section 3. Personas are deferred to the User Stories stage.

| User type | Primary need |
|---|---|
| Student seeking an internship | Match limited experience (courses, class projects, part-time work) to a JD and describe it confidently |
| New graduate | Quickly select and reword the most relevant experience per application |
| International student | Natural, clear English that still sounds like them, without inflated language |

---

## 4. Cross-Cutting Product Rules

These are the product's defining constraints. Every functional requirement below is
subordinate to them. Derived from `vision.md` Section 6 "Product rules".

### PR-1: Every statement must be traceable to a real fact

- **PR-1.1** Only the parsed resume (as corrected by the user per FR-130) and `ConfirmedFact`
  entries may be used as source material.
- **PR-1.2** The system must never add jobs, skills, certificates, duties, numbers, or skill
  levels that do not appear in a source.
- **PR-1.3** Wording may be improved, bullet order may change within an entry, and relevant
  experience may be highlighted — but nothing may be exaggerated.
- **PR-1.4** If a JD requirement has no evidence, the system must ask the user. Unconfirmed
  information must never reach the resume or the cover letter.
- **PR-1.5** Every suggestion shown to the user must display the original text or confirmed
  answer it derives from.
- **PR-1.6** **The LLM proposes; deterministic code verifies.** No LLM output reaches the UI
  as a clean suggestion without passing `verify_suggestion`.

### PR-2: Writing must be natural, specific, and sound like the user

- **PR-2.1** Avoid empty phrases, excessive praise, and repeated JD keyword stuffing.
- **PR-2.2** Support each point with specific experience rather than accumulated adjectives.
- **PR-2.3** Preserve the user's real seniority level and tone.
- **PR-2.4** All generated content is an editable draft the user reviews and confirms.
- **PR-2.5** No claim is made that output will evade any AI-detection tool.

---

## 5. Functional Requirements

> **Defined term — "final resume content"**
>
> The resume as the user has settled it, namely:
>
> - the **accepted or edited text** of every suggestion the user accepted or edited (FR-440);
> - the **original text** of every line whose suggestion was **rejected**, is still
>   **pending**, or that never received a suggestion (FR-412);
> - the **accepted bullet order** within each entry (FR-505).
>
> `ConfirmedFact` entries are additional confirmed source material alongside the final resume
> content. Both the cover letter (FR-701) and the export (FR-805) draw on final resume
> content, so **rejecting a suggestion never removes the user's original line** from either.

### 5.1 Session and Resume Input (FR-1xx)

| ID | Requirement | Source |
|---|---|---|
| FR-101 | The system shall create a session with a random ID (`secrets.token_urlsafe(16)`) via `POST /api/sessions`. | tech-env §4, §8 |
| FR-102 | Session state shall hold: parsed resume items, JD text, extracted requirements, requirement matches, confirmed facts, suggestions and their statuses, reorder suggestions, cover letter draft, style choice, and writing sample. | tech-env §4 |
| FR-103 | Sessions shall expire after 2 hours of inactivity. Restarting the server shall clear all sessions. | tech-env §4 |
| FR-110 | The system shall accept a resume as PDF upload, DOCX upload, or pasted plain text. | vision §6 |
| FR-111 | Uploads shall be validated by both file extension and file signature (`%PDF` for PDF, ZIP header for DOCX). | tech-env §8 |
| FR-112 | Uploads shall be rejected above 5 MB; PDFs shall be rejected above 4 pages. | tech-env §8 |
| FR-113 | Pasted resume text shall be capped at 20,000 characters; JD text at 15,000; writing sample at 3,000. | tech-env §8 |
| FR-114 | If extracted text is under 200 characters, the system shall reject the file as a likely scanned PDF and instruct the user to paste text instead. | tech-env §8 |
| FR-115 | Uploaded files shall be read into memory (`BytesIO`) and never written to disk or temp files. | tech-env §4, §7 |
| FR-120 | The system shall parse the resume into `ResumeItem` records, each with a stable ID, a section label (`education`/`experience`/`project`/`skills`/`other`), and text. | tech-env §5 |
| FR-130 | **The user shall be able to correct the parsed item list before the JD step** — editing text, changing section labels, merging, splitting, and deleting items. | Q7 = A |
| FR-131 | After this review, the corrected item list shall be treated as the original resume for all later steps, including matching, source attribution, verification, and export. | Q7 = A |

### 5.2 Job Description and Requirement Matching (FR-2xx)

| ID | Requirement | Source |
|---|---|---|
| FR-201 | The system shall accept the JD as pasted text. Scraping job sites is out of scope. | vision §6, §7 |
| FR-210 | The system shall extract a list of discrete requirements from the JD via the LLM, returned as a validated Pydantic model. | tech-env §3, §5 |
| FR-211 | **The user shall be able to add, edit, and delete extracted requirements before matching runs.** | Q9 = A |
| FR-212 | If the requirement list is changed after matching has run, matching shall run again on the updated list. | Q9 = A |
| FR-220 | The system shall produce a `RequirementMatch` per requirement with status `found`, `needs_confirmation`, or `not_found`, plus the evidence IDs supporting it. | tech-env §5 |
| FR-221 | **A requirement with only partial evidence shall be labelled `needs_confirmation`, not `found`**, and shall trigger a follow-up question about depth. | Q1 = B |
| FR-222 | **`evidence_ids` must be non-empty for `found`** (at least one evidence ID), **may be non-empty for `needs_confirmation`**, and must be empty only for `not_found`. The UI shall render the list as "evidence" for `found` and as "partial evidence" for `needs_confirmation`, displayed beside the follow-up question. | C3 = A, change request 2026-11-22 |
| FR-223 | Two invariants shall each be covered by a property-based test: (1) "`status == not_found` implies `evidence_ids` is empty"; (2) **"`status == found` implies `evidence_ids` is non-empty."** | C3 = A, change request 2026-11-22 |
| FR-224 | Every extracted requirement shall appear in the checklist with exactly one of the three statuses — no requirement may be silently omitted. | vision §4 |

### 5.3 Follow-Up Questions and Confirmed Facts (FR-3xx)

| ID | Requirement | Source |
|---|---|---|
| FR-301 | The system shall generate follow-up questions for requirements lacking sufficient evidence. The LLM judges which requirements need a question. | vision §6 |
| FR-310 | **Questions shall be presented in rounds of 3**, ordered by how strongly the JD emphasises each requirement, so the most important gaps come first. | Q2 = D |
| FR-311 | After each round the user shall choose "answer more" or "continue to suggestions". | Q2 = D |
| FR-312 | A requirement that is skipped, or never asked because the user moved on, shall remain unconfirmed and shall not be used in the resume or cover letter. | Q2 = D, PR-1.4 |
| FR-320 | A user answer shall be stored as a `ConfirmedFact` with the question asked and the user's answer **verbatim in their own words**. | tech-env §5 |
| FR-321 | A user may answer "no" to a question; this shall be recorded and shall not create usable source material. | tech-env §5 endpoint sketch |
| FR-330 | **A user shall be able to create a `ConfirmedFact` from the suggestions screen**, via a "this is true, let me confirm it" control on a flagged suggestion, typing the supporting detail in their own words. This uses the existing `POST /api/sessions/{sid}/facts` endpoint. | C4 = A |
| FR-331 | On creating such a fact, **only the originating suggestion shall be re-verified**; no other suggestion shall be regenerated, so the user's existing accept/reject decisions are preserved. | C4 = A |
| FR-332 | Re-verification under FR-331 shall add the new `ConfirmedFact` ID to that suggestion's `source_ids` before running `verify_suggestion`. | Derived from C4 = A |
| FR-333 | A fact created under FR-330 shall be available to the cover letter step as ordinary confirmed source material. | C4 = A |

### 5.4 Rewrite Suggestions and Verification (FR-4xx)

| ID | Requirement | Source |
|---|---|---|
| FR-401 | The system shall generate line-by-line rewrite suggestions, each shown next to its original line with its source displayed. | vision §6, PR-1.5 |
| FR-410 | **Suggestion scope**: every resume line that is evidence for a JD requirement shall receive a suggestion; additionally, lines the assistant judges to be weakly worded shall receive suggestions. | Q6 = C |
| FR-411 | The two kinds shall be presented as **separate, visually distinct groups**, so JD-related changes remain the focus. | Q6 = C |
| FR-412 | A line shall carry its **original text** (as corrected in FR-130) into the final resume content when it received no suggestion, when its suggestion was **rejected**, or when its suggestion is still **pending**. Rejecting a suggestion keeps the original line; it does not delete it. | Q6 = C, change request 2026-11-22 |
| FR-420 | Every suggestion shall be verified by `verify_suggestion(proposed, source_ids, known)` before reaching the UI. | tech-env §3, §9, PR-1.6 |
| FR-421 | Verification shall flag numbers in the proposed text that appear in no source (`unsupported_number:<n>`). | tech-env §9 |
| FR-422 | Verification shall flag verbs implying greater seniority than any source verb (`verb_inflation:<verb>`). | tech-env §9 |
| FR-423 | Verification shall reject suggestions with empty `source_ids`, or any `source_id` not present in the session (`invalid_source_ids`). | tech-env §9 |
| FR-430 | **Suggestions flagged `invalid_source_ids` shall be dropped from the list.** The UI shall display a count of withheld suggestions (e.g. "2 suggestions were withheld because they could not be traced to your resume") without exposing the untraceable text. | Q5 = X |
| FR-431 | **Suggestions flagged `unsupported_number` or `verb_inflation` shall be shown with a warning badge naming the flag**, and **shall not be acceptable as-is** — the user must edit them first. | Q5 = X |
| FR-432 | If a suggestion flagged under FR-431 is cleared of all flags by re-verification following FR-330/FR-332, it shall become an ordinary suggestion and shall be acceptable without further editing. | Derived from Q5 = X + C4 = A |
| FR-440 | The user shall accept, reject, or edit each suggestion individually, with status tracked as `pending`/`accepted`/`rejected`/`edited`. | vision §6, tech-env §5 |
| FR-441 | Text the user types themselves (an edit) is the user's own words and is therefore confirmed by authorship. | Q5 = X, C4 = A |

### 5.5 Bullet Reordering (FR-5xx)

| ID | Requirement | Source |
|---|---|---|
| FR-501 | The system shall be able to suggest **reordering bullets within a single entry**, which the user accepts or rejects. | Q8 = B, PR-1.3 |
| FR-502 | **Reorder suggestions shall use a separate `ReorderSuggestion` type** (entry ID, proposed bullet-ID order, rationale, status) on its own endpoint, outside `verify_suggestion`, because a reorder changes no text. | C5 = A |
| FR-503 | **A proposed order must be an exact permutation of that entry's existing bullet IDs** — no missing, duplicate, or foreign IDs. A reorder suggestion failing this check shall be dropped. | C5 = A |
| FR-504 | The permutation rule in FR-503 shall be covered by a property-based test. | C5 = A |
| FR-505 | Reorders and rewrites shall be independent: on export, bullets appear in the accepted order, each using its accepted text. | C5 = A |
| FR-506 | Suggesting which bullets to **drop** is out of MVP scope; users delete items themselves during the FR-130 review. | Q8 = B |
| FR-507 | Section order shall remain fixed, matching the export template. | Q8 = B, Q3 = A |

### 5.6 Writing Style (FR-6xx)

| ID | Requirement | Source |
|---|---|---|
| FR-601 | The user shall choose a style: `natural`, `professional`, or `keep_my_voice` (`StyleChoice`). | vision §6, tech-env §5 |
| FR-602 | For `keep_my_voice`, the user may paste a writing sample up to 3,000 characters. | vision §6, tech-env §8 |
| FR-603 | The writing sample shall be treated as untrusted data, wrapped in its own `<writing_sample>` tag in the prompt. | tech-env §8 |
| FR-604 | The style choice shall apply to both rewrite suggestions and the cover letter. | tech-env §5 endpoint sketch |

### 5.7 Cover Letter (FR-7xx)

| ID | Requirement | Source |
|---|---|---|
| FR-701 | The system shall generate a cover letter draft from **final resume content and confirmed facts** (see the defined term at the head of Section 5). This includes accepted and edited suggestions, original lines whose suggestion was rejected or is pending, and lines that never received a suggestion — so the whole resume remains available as source material, not only the parts that happened to be rewritten. No other source may be used. | vision §6, PR-1.4, change request 2026-11-22 |
| FR-702 | **Shape**: three to four short paragraphs on one page — opening, why this role, relevant evidence, closing. | Q10 = D |
| FR-703 | The user may supply a company name and role title. **These shall be used only if the user provides or confirms them; the assistant must not infer them from the JD.** | Q10 = D |
| FR-704 | If not supplied, the letter shall use a generic opening such as "Dear Hiring Manager". | Q10 = D |
| FR-705 | On export, the user's name and contact details from the resume shall appear at the top, with **no recipient address block and no placeholders**. | Q10 = D |
| FR-706 | The draft shall be editable by the user before export. | vision §6, PR-2.4 |

### 5.8 Export (FR-8xx)

| ID | Requirement | Source |
|---|---|---|
| FR-801 | **The user shall choose what to export** (resume, cover letter, or both) **and the format** (DOCX or PDF). | Q4 = D |
| FR-802 | **When both are selected they shall be returned as two separate files in one `.zip`**, because application systems typically request them as separate uploads. | Q4 = D |
| FR-803 | **The resume template shall be single-column, plain, and ATS-friendly**: name and contact at the top; fixed section order (Education, Experience, Projects, Skills); one font; bold section headers; no colour, no tables, no icons. | Q3 = A |
| FR-804 | The cover letter export shall use the same font as the resume template and a standard plain business-letter format, **as specified in FR-705**: the user's name and contact details at the top, **no recipient address block, and no placeholders**. "Standard plain business-letter format" here means FR-705's layout — the two requirements describe the same format and must not be read as conflicting. | Q3 = A, Q10 = D, change request 2026-11-22 |
| FR-805 | Export shall contain only confirmed content: the **final resume content** (see the defined term at the head of Section 5) and, if selected, the cover letter. Concretely: accepted and edited suggestions in their accepted text; original text for lines whose suggestion was rejected, is pending, or never received one; and accepted reorderings. | PR-1.1, FR-412, FR-505, change request 2026-11-22 |
| FR-808 | If any suggestions are still **pending** when the user reaches the export step, the UI shall display a notice before exporting — for example "N suggestions have not been reviewed". The notice is **informational and does not block export** (unlike the staleness block in FR-904); those lines export with their original text per FR-412. | change request 2026-11-22 |
| FR-806 | DOCX shall be written with `python-docx` and PDF with `fpdf2`. Preserving the original upload's layout is out of scope. | tech-env §2, §7 |
| FR-807 | Section reordering before export is **out of MVP scope** (future feature). | Q3 = A |

### 5.9 Wizard Flow and Staleness (FR-9xx)

| ID | Requirement | Source |
|---|---|---|
| FR-901 | The UI shall be a **linear wizard, one step per screen**, with a visible progress indicator and back navigation. | Q11 = A |
| FR-902 | Step order: resume upload → parsed resume review → JD input → requirement list review → requirement checklist → follow-up questions → suggestions → cover letter → export. | Q11 = A |
| FR-903 | **If the user goes back and changes the resume or the requirement list, dependent results (matching, suggestions, cover letter) shall be marked out of date and must be regenerated before export.** | Q11 = A |
| FR-904 | The export step shall be blocked while any dependent result is out of date. | Derived from FR-903 |

---

## 6. Non-Functional Requirements

### 6.1 Privacy and Data Handling (NFR-1xx)

| ID | Requirement | Source |
|---|---|---|
| NFR-101 | Resume text, JD text, confirmed facts, and writing samples shall live **in memory only** and expire with the session. | vision §8, tech-env §4, §8 |
| NFR-102 | No resume content shall be written to disk, temp files, or logs. | tech-env §7, §8 |
| NFR-103 | Logs shall record session IDs, item IDs, character counts, and timings — **never content**. `print()` is prohibited for diagnostics. | tech-env §7, §8 |
| NFR-104 | The UI shall state that resume text is sent to the Anthropic API for processing. | vision §8, tech-env §8 |
| NFR-105 | No user accounts and no saved history. | vision §7 |

### 6.2 Security (NFR-2xx)

| ID | Requirement | Source |
|---|---|---|
| NFR-201 | The backend shall bind to `127.0.0.1` only. | tech-env §8 |
| NFR-202 | **CORS shall allow only `http://localhost:5173`.** Wildcard origins are prohibited. | tech-env §8, SECURITY-08 (C2 = B) |
| NFR-203 | **An unknown or mismatched session ID shall always return 404**, so the API never confirms whether a session ID exists. | C2 = B, tech-env §9 |
| NFR-204 | The session ID is treated as the session's **capability token**. This is a documented accepted risk, mitigated by `127.0.0.1` binding and `secrets.token_urlsafe(16)` IDs. | C2 = B |
| NFR-205 | `ANTHROPIC_API_KEY` shall live in `backend/.env`, git-ignored; only `.env.example` with an empty value is committed. The key shall never be logged or returned. | tech-env §8 |
| NFR-206 | Resume, JD, and writing-sample text are **untrusted data**. Prompts shall wrap each in its own XML tag (`<resume>`, `<job_description>`, `<writing_sample>`), and the system prompt shall state that text inside these tags is content to analyse, never instructions. | tech-env §8 |
| NFR-207 | Every `source_id` shall be confirmed to exist in the session before a suggestion is shown. | tech-env §8, FR-423 |
| NFR-208 | `eval()`, `exec()`, and `pickle` are prohibited. | tech-env §7 |
| NFR-209 | Error responses shall not expose stack traces, internal paths, or framework versions. | SECURITY-09 |
| NFR-210 | Truthfulness is treated as a **safety property**: content failing verification is never shown as a clean suggestion. | tech-env §8 |

### 6.3 Reliability and LLM Handling (NFR-3xx)

| ID | Requirement | Source |
|---|---|---|
| NFR-301 | All LLM calls shall go through `backend/app/llm/client.py`. **No other module may import `anthropic`.** | tech-env §3 |
| NFR-302 | Every LLM output shall be parsed into a Pydantic model via `client.messages.parse(..., output_format=...)`. Free-text parsing, regex extraction, and `json.loads` on free text are prohibited. | tech-env §3, §7 |
| NFR-303 | `stop_reason` shall be checked before use. `refusal` and `max_tokens` are errors surfaced as "could not generate, please retry" — **nothing partial is shown**. | tech-env §3 |
| NFR-304 | All external calls shall have explicit error handling, and error paths shall **fail closed**. | SECURITY-15 |
| NFR-305 | A global error handler shall be configured at the application entry point. | SECURITY-15 |
| NFR-306 | Effort shall be set per call: `medium` for extraction and matching, `high` for rewriting and cover letters. | tech-env §3 |

### 6.4 Performance and Cost (NFR-4xx)

| ID | Requirement | Source |
|---|---|---|
| NFR-401 | A user shall be able to go from upload to export in **about 10 minutes**. | vision §4 |
| NFR-402 | The primary model shall be `claude-opus-5-5`, switchable to `claude-sonnet-5-5` by changing one config value (`LLM_MODEL`). | tech-env §3, vision §8 |
| NFR-403 | The only external network call shall be to the Anthropic API. | tech-env §4 |

### 6.5 Testability and Quality Gates (NFR-5xx)

| ID | Requirement | Source |
|---|---|---|
| NFR-501 | Unit and API tests shall use a `FakeLLMClient` returning fixed Pydantic objects, and shall **never require an API key or network access**. | tech-env §3, §6 |
| NFR-502 | Backend coverage shall be **≥ 80% line coverage for `app/services` and `app/verification`**. | tech-env §6 |
| NFR-503 | `ruff` (lint + format) and `mypy` shall report no errors. `eslint` and `tsc --noEmit` shall report no errors. | tech-env §6 |
| NFR-504 | Every truthfulness rule shall have at least one test proving a violation is caught: an unsupported number is flagged; a seniority-raising verb is flagged; an unconfirmed skill never appears in suggestions or the cover letter; a suggestion with empty or unknown `source_ids` is rejected. | tech-env §6 |
| NFR-505 | A truthfulness evaluation set of 8–10 resume/JD pairs shall run via `pytest -m eval` against the real API, manually before each milestone. | tech-env §6, vision §4 |
| NFR-506 | Eval runs shall report per pair: number of suggestions, number flagged by verification, and any unsupported claim found on manual review. | tech-env §6 |
| NFR-507 | Frontend tests (`vitest` + React Testing Library) shall cover upload, accept/reject, and export button state. | tech-env §6 |
| NFR-508 | **No CI service for the MVP.** `uv run pytest` and `npm test` are run locally before opening a pull request. | tech-env §6, C1 = A |

### 6.6 Maintainability (NFR-6xx)

| ID | Requirement | Source |
|---|---|---|
| NFR-601 | Type hints shall be required on all public Python functions. TypeScript shall be strict; no plain `.js` sources in `frontend/src`. | tech-env §1 |
| NFR-602 | API route handlers shall contain **no business logic**; logic lives in `services/`. | tech-env §5 |
| NFR-603 | Deterministic truthfulness checks shall be isolated in `app/verification/` — security-critical logic in a dedicated module. | tech-env §5, SECURITY-11 |
| NFR-604 | Dependencies shall be pinned via committed lock files (`uv.lock`, `package-lock.json`). | tech-env §1, SECURITY-10 |
| NFR-605 | Provider swappability shall be preserved by the single-module LLM boundary (NFR-301), so a future local model requires no service changes. | tech-env §3 |

### 6.7 Prohibited Technologies (NFR-7xx)

| ID | Requirement |
|---|---|
| NFR-701 | The following are prohibited in product runtime code, per `technical-environment.md` Section 7: OCR libraries; scraping/browser automation; databases and ORMs; auth libraries; original-layout editing APIs; LangChain, LlamaIndex, and other agent frameworks; the OpenAI SDK and OpenAI-compatible shims; regex or `json.loads` parsing of free-text LLM output; `eval()`/`exec()`/`pickle`; `requests`; Flask, Django, Streamlit; Redux, MobX, Zustand; large UI kits; pip, poetry, pipenv; black, flake8, isort; writing uploads or resume text to disk; `print()` for diagnostics; cloud deployment. |
| NFR-702 | The prohibition on the OpenAI SDK applies to **product runtime code only**. Teammates may use any coding agent, including OpenAI Codex, to write code. |

---

## 7. Data Model Decisions

These refine the core model in `technical-environment.md` Section 5. Application Design will
finalise signatures.

### 7.1 Change to `RequirementMatch` (C3 = A)

The original comment constrained `evidence_ids` to be "empty unless `found`". This is relaxed:

| Status | `evidence_ids` | UI label |
|---|---|---|
| `found` | **must be non-empty** (at least one ID; invariant, property-tested per FR-223 — changed 2026-11-22) | "evidence" |
| `needs_confirmation` | **may be non-empty** (changed) | "partial evidence" |
| `not_found` | **must be empty** (invariant, property-tested per FR-223) | — |

### 7.2 New type: `ReorderSuggestion` (C5 = A)

Separate from `Suggestion`, carrying entry ID, proposed bullet-ID order, rationale, and
status. It does **not** pass through `verify_suggestion` — a reorder changes no text, so the
number and verb-inflation checks have nothing to examine. Safety comes instead from the
exact-permutation check in FR-503, which is property-tested per FR-504.

**Rationale**: routing reorders through `verify_suggestion` would produce suggestions that
pass verification while carrying no verifiable content, weakening the guarantee that a passing
verification means something.

### 7.3 Unchanged

`ResumeItem`, `ConfirmedFact`, `Suggestion`, and `StyleChoice` are used as specified in
`technical-environment.md` Section 5.

---

## 8. Extension Compliance Configuration

| Extension | Status | Mode |
|---|---|---|
| Security Baseline | **Enabled** | Scoped to local single-user deployment |
| Resiliency Baseline | **Disabled** | Rules file never loaded; skips logged in `audit.md` |
| Property-Based Testing | **Enabled** | Partial (PBT-02, PBT-03, PBT-07, PBT-08, PBT-09) |

### 8.1 Security Baseline — Per-Rule Scope

| Rule | Status | Rationale |
|---|---|---|
| SECURITY-01 Encryption at rest/transit | **N/A** | No persistence store; in-memory session state only |
| SECURITY-02 Access logging on intermediaries | **N/A** | No load balancer, API gateway, or CDN |
| SECURITY-03 Application-level logging | **Enforced** | `logging` configured; IDs/counts/timings only, never content (NFR-103). Centralized-log-service sub-bullet N/A — local-only deployment |
| SECURITY-04 HTTP security headers | **Enforced** | CSP, `X-Content-Type-Options`, `X-Frame-Options`, `Referrer-Policy`. HSTS sub-bullet N/A — local HTTP dev server has no TLS |
| SECURITY-05 Input validation | **Enforced** | FR-111 to FR-114; Pydantic schemas on every endpoint |
| SECURITY-06 Least-privilege IAM | **N/A** | No cloud IAM |
| SECURITY-07 Restrictive network config | **N/A** | No cloud network; `127.0.0.1` binding is the local analogue (NFR-201) |
| SECURITY-08 Application access control | **Partially enforced** (C2 = B) | CORS bullet enforced (NFR-202); 404-on-unknown-session enforced (NFR-203); session ID as capability token documented as accepted risk (NFR-204). Authentication, token-validation, and role-check bullets N/A — no accounts |
| SECURITY-09 Hardening | **Enforced** | No default credentials; generic error responses (NFR-209); current framework versions |
| SECURITY-10 Supply chain | **Partially enforced** | Dependency pinning via lock files enforced (NFR-604); trusted-registry and no-unused-dependency bullets enforced. Vulnerability scanning and SBOM N/A — no production deployment |
| SECURITY-11 Secure design | **Partially enforced** | Verification isolated in a dedicated module (NFR-603); prompt-injection misuse case addressed (NFR-206). Rate limiting N/A — no public endpoints |
| SECURITY-12 Authentication and credentials | **Partially enforced** | No-hardcoded-credentials bullet enforced (NFR-205). Password, MFA, session-cookie, and brute-force bullets N/A — no authentication |
| SECURITY-13 Integrity verification | **Partially enforced** | Safe deserialization enforced (NFR-208, Pydantic validation). SRI N/A — Vite bundles locally, no external CDN scripts. CI/CD pipeline bullet N/A — no CI |
| SECURITY-14 Alerting and monitoring | **N/A** | No production deployment |
| SECURITY-15 Exception handling | **Enforced** | Explicit error handling on all external calls; fail closed; global error handler (NFR-304, NFR-305) |

**Note**: `security-baseline.md` contains an "Appendix: OWASP Reference Mapping" whose own
inline TODO questions whether the cited OWASP Top 10 2025 edition exists. The appendix is
labelled for human reviewers and is not a rule, so no AI-DLC artifact cites that mapping.

### 8.2 Property-Based Testing — Partial Mode

| Rule | Status | Application |
|---|---|---|
| PBT-02 Round-trip | **Enforced** | Pydantic model serialization round-trips |
| PBT-03 Invariants | **Enforced** | Verification invariants: "a number copied from a source is never flagged"; "a suggestion identical to its source has no flags"; "`not_found` implies empty `evidence_ids`" (FR-223); "`found` implies non-empty `evidence_ids`" (FR-223, change request 2026-11-22); "a proposed reorder is always an exact permutation of the entry's bullet IDs" (FR-504) |
| PBT-07 Generator quality | **Enforced** | Domain generators required for `ResumeItem`, `ConfirmedFact`, `RequirementMatch`, `Suggestion`, `ReorderSuggestion` — not raw primitives; centralized as reusable test utilities |
| PBT-08 Shrinking and reproducibility | **Enforced, CI sub-requirement waived** (C1 = A) | Shrinking never disabled; Hypothesis prints the seed on every failure for replay. **Documented justification for the CI waiver**: no CI service in the MVP, per `technical-environment.md` Section 6. PBT-08's verification criteria permit exclusion from CI with documented justification |
| PBT-09 Framework selection | **Enforced** | Hypothesis with pytest — matches PBT-09's recommended Python framework. Added as a dev dependency |
| PBT-01, PBT-04, PBT-05, PBT-06, PBT-10 | **Advisory** (non-blocking in Partial mode) | PBT-06 stateful testing of the session store explicitly not required for the MVP |

---

## 9. Traceability to Vision Success Metrics

| Vision metric (§4) | Target | Requirements delivering it |
|---|---|---|
| Fabricated facts in output | 0 unsupported claims across 8–10 pairs | PR-1, FR-420 to FR-431, NFR-504, NFR-505 |
| Suggestions with a visible source | 100% | PR-1.5, FR-401, FR-423 |
| Verification catches violations | All invented numbers and inflated verbs in test cases | FR-421, FR-422, NFR-504, PBT-03 |
| JD requirements checked | Every requirement labelled | FR-220, FR-224 |
| Missing information handled | Always ask before using | FR-301, FR-310, FR-312, PR-1.4 |
| Time to tailor | ~10 minutes upload to export | NFR-401, FR-901 |
| Export works | DOCX and PDF without errors | FR-801 to FR-806 |

---

## 10. Scope Boundary

### In scope (MVP)
Resume input (PDF/DOCX/paste); parsed-item review and correction; JD paste; requirement
extraction with user editing; requirement checklist with three statuses and partial evidence;
follow-up questions in rounds; confirmed facts (including user-initiated from the suggestions
screen); rewrite suggestions with deterministic verification and flag handling; within-entry
bullet reordering; accept/reject/edit; three writing styles with optional writing sample;
cover letter draft; DOCX/PDF export with selectable contents.

### Out of scope
Resume from scratch (guided interview); auto-applying to jobs; scraping job sites; visa
sponsorship filtering; mock interviews; OCR / scanned resumes; user accounts and saved
history; preserving the original resume layout; section reordering before export (FR-807);
drop-bullet suggestions (FR-506); stateful PBT of the session store; CI pipeline (NFR-508);
cloud deployment.

---

## 11. Development Organisation

From Q12 = C with additions. This informs Units Generation.

- **Team**: three developers, each using Claude Code — **Xinyi Wang** (xw785),
  **Linghui Zhou** (lz633), and **Yunzhu Qiu** (yq337), per `README.md` "Team Members".
- **Decomposition**: a **shared foundation unit built first**, then **vertical feature slices
  in parallel**, each slice covering both backend and frontend so one person owns it end to
  end.
- **Foundation unit** (built first): Pydantic data models, in-memory session store, LLM client
  wrapper, API contract, and the app shell with wizard navigation. **Owned by Linghui Zhou,
  who also owns final integration.**
- **Feature slices**: resume pipeline; JD/matching; suggestions/verification; cover letter;
  export. **Ownership of these five slices is deferred to Units Generation** (A-5).

**Note for Units Generation**: the suggestions/verification slice depends on the
`POST /api/sessions/{sid}/facts` endpoint (FR-330), which otherwise belongs to the
JD/matching slice. This cross-slice dependency needs an explicit owner in the unit dependency
map.

---

## 12. Assumptions

| # | Assumption | Basis |
|---|---|---|
| A-1 | **CONFIRMED by the team on 2026-11-22.** A flagged suggestion cleared of all flags by re-verification becomes acceptable without editing (FR-432). | Originally derived from Q5 + C4 = A; the "must edit" constraint attaches to flagged suggestions, so once the flag is gone it is an ordinary suggestion. No longer an open assumption — FR-432 is a confirmed requirement. |
| A-2 | Re-verification under FR-331 adds the new fact ID to the suggestion's `source_ids` (FR-332). | Mechanically required for C4 = A to work — verification would otherwise still fail |
| A-3 | The export step is blocked while dependent results are stale (FR-904). | Derived from Q11's "must be regenerated before export" |
| A-4 | "Weakly worded" line detection (FR-410) is an LLM judgement, not a deterministic rule. | Consistent with the LLM-proposes/code-verifies split; no deterministic definition was specified |
| A-5 | **RESOLVED on 2026-11-22.** The team is **Xinyi Wang** (xw785), **Linghui Zhou** (lz633), and **Yunzhu Qiu** (yq337). Linghui Zhou owns the foundation unit and final integration. **Slice ownership beyond the foundation unit will be decided during Units Generation.** | Names confirmed against `README.md` "Team Members"; ownership assignment deferred by the team |

---

## 13. Deferred to Later Stages

| Item | Stage |
|---|---|
| Component boundaries, service signatures, endpoint refinement | Application Design |
| Unit decomposition and dependency map (per Section 11) | Units Generation |
| Prompt text and per-prompt schemas | Functional Design (per unit) |
| "Testable Properties" sections per component (PBT-01, advisory) | Functional Design (per unit) |
| Concrete `.env.example`, template typography, page geometry | Code Generation |

---

## 14. Summary

Storywell's requirements are shaped by one dominant constraint: **the LLM proposes and
deterministic code verifies**. Everything else follows from protecting that boundary — the
parsed-item review (FR-130) exists so source attribution is reliable; the flag-handling split
(FR-430/FR-431) exists so an untraceable suggestion is never shown while a correctable one
stays visible; the separate `ReorderSuggestion` type exists so passing verification keeps its
meaning; and the 80% coverage floor targets precisely `app/services` and `app/verification`.

The clarification round resolved five conflicts, of which three were the same underlying
issue: the product decisions extended past what the specified Pydantic model could express.
Two artifacts changed as a result — `RequirementMatch.evidence_ids` was relaxed, and
`ReorderSuggestion` was added.

**Counts**: 9 functional requirement groups (73 requirements), 7 non-functional groups
(39 requirements), 2 cross-cutting product rules (11 clauses), 5 assumptions (2 since
resolved).

---

## 15. Revision History

### 2026-11-22 — Change request after team review (5 changes)

| # | Change | Requirements touched |
|---|---|---|
| 1 | **Cover letter may use the final resume content**, not only accepted suggestions. Introduced the defined term "final resume content" at the head of Section 5 so the cover letter and export requirements share one definition instead of restating it. | New defined term; FR-701 |
| 2 | **Rejected and pending suggestions export with their original text**, and a non-blocking notice shows the pending count at the export step. | FR-412, FR-805, new FR-808 |
| 3 | **Team named**: Xinyi Wang, Linghui Zhou, Yunzhu Qiu. Slice ownership deferred to Units Generation. | A-5 resolved; Section 11 |
| 4 | **A-1 confirmed** — a suggestion cleared of all flags by re-verification is acceptable without editing. FR-432 is now a confirmed requirement rather than a derived assumption. | A-1 resolved; FR-432 |
| 5 | **FR-804 now cites FR-705 explicitly** so the cover letter format reads as one specification rather than two possibly conflicting ones. | FR-804 |

**Why change 1 mattered**: as originally written, FR-701 restricted the cover letter to
"accepted resume content", which would have excluded every line the user left alone or whose
suggestion they rejected — on a resume where few lines need rewriting, that is most of the
document. The defined term fixes this at the root: rejecting a suggestion now explicitly keeps
the original line as usable source material, in both the cover letter and the export.

**Truthfulness is unaffected** by changes 1 and 2. Original resume lines are source material
under PR-1.1 by virtue of being in the user's own resume, so using an un-rewritten line in the
cover letter introduces no unverified content. FR-312 still holds: an unconfirmed JD
requirement remains unusable.

### 2026-11-22 — Change request after User Stories review (`found` evidence invariant)

| # | Change | Requirements touched |
|---|---|---|
| 1 | **`found` now requires at least one evidence ID.** Previously `evidence_ids` only had to be non-empty "for `found` and `needs_confirmation`" without requiring it for `found` — a `found` match with zero evidence was not actually excluded. A `found` requirement with no evidence would mean the checklist asserts a match the user cannot see any support for, which the traceability rule (PR-1) requires every shown claim to have. | FR-222, FR-223 (new invariant), §7.1 table, §8.2 PBT-03 row |

**Why this mattered**: a status of `found` is the strongest claim the checklist makes about a
requirement — stronger than `needs_confirmation`. Leaving `evidence_ids` only "may be
non-empty" for `found` meant the schema allowed the strongest claim to carry the weakest
support. The fix adds the missing invariant symmetrically to the existing `not_found` one:
`not_found` implies empty, `found` implies non-empty, both property-tested per FR-223.
`needs_confirmation` is unchanged — it may or may not carry partial evidence, by design.
