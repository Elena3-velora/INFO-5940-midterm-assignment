# AI-DLC State Tracking

## Project Information
- **Project Name**: Storywell
- **Project Type**: Greenfield
- **Start Date**: 2026-11-22T23:06:02Z
- **Current Stage**: INCEPTION - Requirements Analysis (awaiting approval)
- **Rule Details Directory**: `.aidlc-rule-details/`

## Workspace State
- **Existing Code**: No
- **Programming Languages**: None found (Markdown documents only)
- **Build System**: None found
- **Project Structure**: Empty (documentation and AI-DLC rules only)
- **Reverse Engineering Needed**: No
- **Workspace Root**: `/Users/janniejane/Desktop/INFO-5940-midterm-assignment`

## Team
Per `README.md` "Team Members". All three use Claude Code.

| Name | NetID | Role |
|---|---|---|
| Xinyi Wang | xw785 | Project Planning & Requirements |
| Linghui Zhou | lz633 | Application Development — owns the foundation unit and final integration |
| Yunzhu Qiu | yq337 | AI Development Tools, Collaboration Plan & Documentation |

Ownership of the five feature slices is deferred to Units Generation.

## Input Documents
- `vision.md` — product vision for Storywell (problem, users, MVP scope in/out, product rules, risks)
- `technical-environment.md` — mandated stack, constraints, prohibited libraries, code patterns

## Code Location Rules
- **Application Code**: Workspace root (NEVER in aidlc-docs/)
- **Documentation**: aidlc-docs/ only
- **Structure patterns**: See code-generation.md Critical Rules
- **Planned layout** (per technical-environment.md Section 5): `backend/app/...`, `backend/tests/...`, `frontend/src/...`

## Extension Configuration

| Extension | Enabled | Mode | Decided At | Rules File Loaded |
|---|---|---|---|---|
| Security Baseline | Yes | Scoped to local single-user deployment | Requirements Analysis | Yes |
| Resiliency Baseline | No | — | Requirements Analysis | No (never loaded — opted out) |
| Property-Based Testing | Yes | Partial | Requirements Analysis | Yes |

### Security Baseline — Rule Scope
User opted in with per-rule scoping for a local-only, single-user app bound to `127.0.0.1`
with no accounts, no database, and no persistent storage.

**Enforced**: SECURITY-03 (structured logging; never log resume/JD content), SECURITY-04,
SECURITY-05, SECURITY-09, SECURITY-10 (dependency pinning via lock files), SECURITY-11
(secure design, excluding rate limiting), SECURITY-12 (no hardcoded credentials), SECURITY-13
(safe deserialization), SECURITY-15.

**N/A**: SECURITY-01 (no data stores), SECURITY-02 and SECURITY-07 (no cloud network),
SECURITY-06 (no IAM), SECURITY-12 password/MFA/login rules (no authentication), SECURITY-11
rate limiting (no public endpoints), SECURITY-10 vulnerability scanning and SBOM,
SECURITY-14 (no production deployment).

**SECURITY-08 — RESOLVED** (Clarification Question 2 = B): Partially applicable. CORS bullet
enforced (`http://localhost:5173` only); unknown/mismatched session ID returns 404; session ID
treated as the session's capability token, documented as an accepted risk given `127.0.0.1`
binding and `secrets.token_urlsafe(16)` IDs. Authentication, token-validation, and role-check
bullets are N/A (no accounts).

### Property-Based Testing — Partial Mode
Per `property-based-testing.md`, Partial mode enforces exactly **PBT-02, PBT-03, PBT-07,
PBT-08, PBT-09**; all other PBT rules are advisory (non-blocking). Framework: Hypothesis with
pytest (matches PBT-09's recommended Python framework).

**PBT-08 — RESOLVED** (Clarification Question 1 = A): Shrinking never disabled; Hypothesis
prints the seed on every failure for replay. The CI sub-requirement is **waived with
documented justification** — no CI service in the MVP, per `technical-environment.md`
Section 6. PBT-08's own verification criteria permit exclusion from CI with documented
justification.

### Resiliency Baseline — Skipped
Opted out. Full rules file never loaded. Resiliency rules are skipped at every subsequent
stage; each skip is logged in `audit.md` per the conditional-enforcement requirement.

## Stage Progress

### INCEPTION PHASE
- [x] Workspace Detection — COMPLETED 2026-11-22T23:06:02Z (Greenfield)
- [ ] Reverse Engineering — SKIPPED (greenfield; no existing codebase)
- [x] Requirements Analysis — ARTIFACTS COMPLETE, awaiting user approval
  - [x] Intent analysis (clear / new project / system-wide / complex)
  - [x] Depth determination (comprehensive)
  - [x] Completeness analysis across all 6 mandated areas
  - [x] Extension opt-in prompts presented (Step 5.1)
  - [x] `requirement-verification-questions.md` — 15 questions, all answered
  - [x] Contradiction/ambiguity validation — 5 conflicts found
  - [x] `requirements-clarification-questions.md` — 5 questions, all answered
  - [x] Re-validation after clarifications — no new conflicts
  - [x] `requirements.md` — 73 FRs, 39 NFRs, 11 product-rule clauses, 5 assumptions
  - [x] Change request from team review (5 changes) applied 2026-11-22 — see requirements.md §15
- [ ] User Stories — WILL EXECUTE (assessment below)
- [ ] Workflow Planning — pending
- [ ] Application Design — not yet assessed
- [ ] Units Generation — not yet assessed (decomposition approach captured in requirements §11)

### CONSTRUCTION PHASE
- [ ] Per-Unit Loop — pending Units Generation
- [ ] Build and Test — pending

### OPERATIONS PHASE
- [ ] Operations — PLACEHOLDER

## User Stories Assessment
**Decision**: EXECUTE. Matches five "ALWAYS Execute" high-priority indicators in CLAUDE.md:
new user-facing features; changes affecting user workflows; three distinct user types
(internship-seeking student, new graduate, international student); complex business
requirements needing acceptance criteria (the truthfulness rule); and cross-functional team
collaboration (three developers owning vertical slices per requirements §11).

## Requirements Artifacts
- `aidlc-docs/inception/requirements/requirement-verification-questions.md` (15 Q&A)
- `aidlc-docs/inception/requirements/requirements-clarification-questions.md` (5 Q&A)
- `aidlc-docs/inception/requirements/requirements.md` (the requirements document)

## Data Model Changes Agreed During Requirements Analysis
These depart from `technical-environment.md` Section 5 as specified:
1. `RequirementMatch.evidence_ids` — relaxed from "empty unless `found`" to "may be non-empty
   for `found` and `needs_confirmation`; must be empty only for `not_found`" (C3 = A).
2. New type `ReorderSuggestion` — separate from `Suggestion`, outside `verify_suggestion`,
   guarded by an exact-permutation check (C5 = A).
