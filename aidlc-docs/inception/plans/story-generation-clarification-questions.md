# Story Generation — Clarification Question

I detected one structural gap between your answers that needs resolving before generation.

## Gap: Which slice owns the LLM-refusal story?

**Q1 = D** requires every story to belong to **exactly one** of the six groups (foundation unit
or one of the five feature slices).

**Q5 = C** gives an enumerated, "only"-framed list of the foundation unit's user-visible
stories: wizard navigation/progress (FR-901/FR-902), out-of-date marking (FR-903), the
export-blocked notice (FR-904), and the privacy notice (NFR-104). By that "only" framing,
nothing else belongs to the foundation unit.

**Q6 = C** separately requires a **dedicated** story for the LLM refusal / max-tokens error
(NFR-303), since it is a cross-cutting failure mode. But NFR-303 is not on Q5's foundation
list, and the LLM calls that can trigger it happen inside three different feature slices'
generation steps: JD extraction and matching (FR-210, FR-220), rewrite-suggestion generation
(FR-401), and cover letter generation (FR-701). There is no single slice it obviously belongs
to under your Q1/Q5 rules as stated.

### Clarification Question 1
Which group should own the single dedicated NFR-303 (LLM refusal/max-tokens) story?

A) **Foundation unit** (`US-FOUNDATION-xx`) — all LLM calls go through the one client wrapper
(NFR-301), and the retry/error message ("could not generate, please retry," nothing partial
shown) is identical regardless of which slice's call failed. This slightly extends Q5's list
rather than keeping it as a strict "only."

B) **JD/matching slice** (`US-MATCH-xx`) — it is the first slice in wizard order (FR-902) where
an LLM call can fail, so it owns the one dedicated story; the suggestions/verification and
cover letter slice stories cross-reference it instead of duplicating it.

C) **No single dedicated story** — despite Q6's wording, give each of the three affected
slices (JD/matching, suggestions/verification, cover letter) its own copy of the same
Given/When/Then refusal criterion, attached to that slice's existing generation story, so each
slice owner's test suite covers it without a cross-slice reference.

D) Other (please describe after [Answer]: tag below)

[Answer]: A — Assign the dedicated NFR-303 story to the Foundation unit (US-FOUNDATION-xx). Every LLM call goes through the one client wrapper (NFR-301), where stop_reason is checked, and the "could not generate, please retry" behaviour with nothing partial shown is identical no matter which slice's call failed. We explicitly extend our Question 5 list to include it, because it is user-visible foundation behaviour, which is the rule Q5 was based on. The JD/matching, suggestions/verification, and cover letter stories cross-reference this story instead of repeating it.
