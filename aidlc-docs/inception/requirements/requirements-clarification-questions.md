# Requirements Clarification Questions — Storywell

I validated all 15 answers in `requirement-verification-questions.md`. The product decisions
are coherent and the reasoning behind them is clear. I found **five conflicts** that I cannot
resolve on my own, because each one would have me either contradict `technical-environment.md`
or silently change something you specified.

Please answer each question by filling in the letter choice after the `[Answer]:` tag.
If none of the options match, choose the last option (**Other**) and describe your preference.

**What validated cleanly** (no action needed): Q2 and Q11 agree on regeneration-on-change;
Q7 and Q9 apply the same "fix the inputs first" principle consistently; Q8's rejection of
drop-suggestions is consistent with Q7 giving users manual deletion; Q3, Q4, and Q10 agree on
a plain single-column format with no recipient block; Q12's foundation-unit request is
consistent with Q11's wizard shell. Your Security N/A mapping is accurate for 13 of 15 rules
and your stated rationales match the actual rule text.

---

## Conflict 1: PBT-08 requires CI, but the project has no CI

**Where this comes from.** You chose **B (Partial)** for property-based testing. I then loaded
`property-based-testing.md` and found that Partial mode is not a free choice of rules — it is
defined in the file as enforcing exactly **PBT-02, PBT-03, PBT-07, PBT-08, and PBT-09**, with
all other rules advisory. PBT-02, PBT-03, PBT-07 and PBT-09 match what you described
(round-trips, invariants, domain generators, Hypothesis for Python).

**The conflict is PBT-08.** It is enforced in Partial mode and blocking by default, and it
requires: *"CI integration: PBT MUST be included in the project's CI pipeline"* and
*"CI configuration logs the seed for every PBT run or uses a fixed seed."*

But `technical-environment.md` Section 6 states: *"No CI service for the MVP. Run `uv run
pytest` and `npm test` locally before opening a pull request."*

PBT-08 does provide an escape hatch — its verification criteria allow *"No PBT is excluded
from CI without documented justification."* So this is resolvable, but I need you to pick how.

### Clarification Question 1
How should PBT-08's CI requirement be satisfied?

A) Keep "no CI" and record a documented justification — PBT-08's seed-logging and shrinking
requirements are enforced for local runs (configure Hypothesis to print the seed on failure
and never disable shrinking), and the CI sub-requirement is marked "waived with documented
justification: no CI service in the MVP, per technical-environment.md Section 6"
*(recommended — satisfies the rule's own escape hatch and changes nothing about your stack)*

B) Add a minimal GitHub Actions workflow that runs `uv run pytest` with a fixed Hypothesis
seed, purely to satisfy PBT-08 — this adds CI to the project, which Section 6 currently rules
out

C) Downgrade property-based testing from Partial to No (option C on the original question),
dropping all PBT rules including PBT-08

X) Other (please describe after [Answer]: tag below)

[Answer]: A — Keep "no CI" as stated in technical-environment.md Section 6, and record the CI sub-requirement as waived with that documented justification. The parts of PBT-08 that matter for us still apply to local runs: Hypothesis prints the seed for every failure so it can be reproduced, and shrinking is never disabled. We run uv run pytest locally before every pull request.

---

## Conflict 2: SECURITY-08 marked N/A, but it contains the CORS rule you require

**Where this comes from.** Your Security answer marked **SECURITY-06 and SECURITY-08 as N/A
("no accounts or IAM")**. That rationale is correct for most of SECURITY-08 — there is no
authentication, so the deny-by-default, token-validation, and role-check bullets genuinely do
not apply.

**But SECURITY-08 also contains the CORS bullet**: *"CORS policy: Cross-origin resource
sharing MUST be restricted to explicitly allowed origins — never use
`Access-Control-Allow-Origin: *`."* And `technical-environment.md` Section 8 already mandates
exactly that: *"CORS: allow only `http://localhost:5173`."*

Marking SECURITY-08 entirely N/A would drop a control you have already specified yourself.
There is a second bullet worth a decision too: *"Object-level authorization ... prevent
IDOR."* Every endpoint is `/api/sessions/{sid}/...`, so the session ID is the only thing
protecting one session's resume from another's. Bound to `127.0.0.1` with a
`secrets.token_urlsafe(16)` ID this is acceptable, but it is an explicit decision rather than
an inapplicable rule.

### Clarification Question 2
How should SECURITY-08 be classified?

A) Partially applicable — enforce the CORS bullet (restricted to `http://localhost:5173`) and
treat the session ID as the session's capability token, documented as an accepted risk given
`127.0.0.1` binding; mark the authentication, token-validation, and role-check bullets N/A
*(recommended — keeps the CORS control you already specified)*

B) Partially applicable as in A, and additionally require that a session ID mismatch returns
404 rather than 403 so the API does not confirm whether a session ID exists

C) Fully N/A as originally answered — CORS is still implemented because
`technical-environment.md` requires it, but it is not tracked as SECURITY-08 compliance

X) Other (please describe after [Answer]: tag below)

[Answer]: B — Treat SECURITY-08 as partially applicable. Enforce the CORS bullet (allow only http://localhost:5173). Treat the session ID as the session's capability token and document it as an accepted risk, given the 127.0.0.1 binding and secrets.token_urlsafe(16) IDs. An unknown or mismatched session ID always returns 404, which already matches the example endpoint in technical-environment.md Section 9, so the API never confirms whether a session ID exists. The authentication, token-validation, and role-check bullets are N/A because there are no accounts.

---

## Conflict 3: Partial evidence has nowhere to live in the data model

**Where this comes from.** Your answer to **Q1** chose B and added: *"The checklist should
still show the course bullet as partial evidence next to the question."*

But the core data model in `technical-environment.md` Section 5 says:

```python
class RequirementMatch(BaseModel):
    requirement: str
    status: Literal["found", "needs_confirmation", "not_found"]
    evidence_ids: list[str]  # ResumeItem or ConfirmedFact ids; empty unless "found"
```

`evidence_ids` is specified as **empty unless `found`**. A `needs_confirmation` requirement
therefore cannot carry the course bullet you want displayed. I can see the intent clearly, but
the fix changes a model you specified, so I would rather not choose for you.

### Clarification Question 3
How should partial evidence be represented?

A) Relax the constraint — `evidence_ids` may be non-empty for `found` **and**
`needs_confirmation`, and must be empty only for `not_found`. The UI renders the same list as
"evidence" for `found` and "partial evidence" for `needs_confirmation`
*(recommended — smallest change, one invariant to test)*

B) Add a separate field, `partial_evidence_ids: list[str]`, keeping `evidence_ids` strictly
for `found`. Makes the distinction explicit in the data rather than inferred from `status`

C) Keep the model exactly as specified — partial evidence is not shown next to the follow-up
question, and the question text alone carries the context

X) Other (please describe after [Answer]: tag below)

[Answer]: A — Relax the constraint: evidence_ids may be non-empty for found and needs_confirmation, and must be empty only for not_found. The UI shows the list as "evidence" for found and as "partial evidence" for needs_confirmation, next to the follow-up question. The rule "not_found implies empty evidence_ids" should be covered by a property-based test.

---

## Conflict 4: Correcting a flagged number needs a path that does not exist yet

**Where this comes from.** Your **Q5** answer says that when a suggestion is flagged for an
unsupported number and *"a flagged number is actually true, the user can add it as a confirmed
fact through a follow-up question, so later suggestions can use it."*

Your **Q2** answer defines follow-up questions as a bounded flow: rounds of 3, ordered by JD
emphasis, driven by requirement gaps, and occurring **before** the suggestions step in the
Q11 wizard order.

So the mechanism Q5 relies on does not exist at the point Q5 needs it. The user is on the
suggestions screen, past the questions step, looking at a flagged "40%" that is genuinely
true. There is no requirement-gap question waiting to capture it.

### Clarification Question 4
How does a user confirm a fact while reviewing suggestions?

A) Allow user-initiated facts on the suggestions screen — a flagged suggestion gets a
"this is true, let me confirm it" control that creates a `ConfirmedFact` via the existing
`POST /api/sessions/{sid}/facts` endpoint, with the user typing the supporting detail in their
own words. Only that one suggestion is re-verified; nothing else regenerates
*(recommended — narrow, reuses the existing endpoint, keeps the user in control)*

B) Same as A, but after any new confirmed fact the suggestions step offers "regenerate
suggestions" so later suggestions can use the new fact, as your Q5 wording implies

C) No new path — the user edits the flagged suggestion by hand to include the number (the
edited text is the user's own words, so it is confirmed by authorship). No `ConfirmedFact` is
created and the cover letter cannot reuse the number

X) Other (please describe after [Answer]: tag below)

[Answer]: A — A flagged suggestion gets a "this is true, let me confirm it" control that creates a ConfirmedFact through the existing POST /api/sessions/{sid}/facts endpoint, with the user typing the supporting detail in their own words. Only that one suggestion is re-verified. The new fact is stored in the session, so the cover letter step can use it later; this is what we meant by "later suggestions can use it" in Q5. We do not want to regenerate all suggestions (B), because that would discard the accept and reject decisions the user has already made.

---

## Conflict 5: Reorder suggestions do not fit the Suggestion model

**Where this comes from.** Your **Q8** answer puts reordering bullets within one entry in the
MVP, accepted or rejected per suggestion. But the `Suggestion` model is shaped for text
rewriting — `original_id`, `original_text`, `proposed_text`, and `source_ids` feeding
`verify_suggestion(proposed, source_ids, known)`.

A reorder changes no text, so: there is no `proposed_text` to verify, the number and
verb-inflation checks have nothing to check, and a reorder concerns several bullets at once
rather than one `original_id`. Forcing it into `Suggestion` would mean a suggestion that
trivially passes verification while carrying no verifiable content — which weakens the
guarantee that passing verification means something.

### Clarification Question 5
How should reorder suggestions be modelled?

A) A separate type, `ReorderSuggestion` (entry ID, proposed bullet-ID order, rationale,
status), on its own endpoint, explicitly outside `verify_suggestion` because it changes no
text. Documented as safe-by-construction: a permutation of existing bullet IDs cannot
introduce a fact
*(recommended — keeps the verification guarantee meaningful and is easy to test: output is
always a permutation of the input)*

B) Extend `Suggestion` with a `kind: Literal["rewrite", "reorder"]` discriminator and make the
text fields optional for reorders, keeping one list and one accept/reject UI

C) Drop reordering from the MVP after all (option A on the original Q8) and revisit it later

X) Other (please describe after [Answer]: tag below)

[Answer]: A (with additions) — Use a separate ReorderSuggestion type (entry ID, proposed bullet-ID order, rationale, status) on its own endpoint, outside verify_suggestion. "Safe by construction" must still be checked: the proposed order must be an exact permutation of that entry's existing bullet IDs (no missing, duplicate, or foreign IDs), otherwise the reorder suggestion is dropped. This permutation rule should be a property-based test. Reorders and rewrites are independent: on export, bullets appear in the accepted order, each using its accepted text.

---

**When you have filled in every `[Answer]:` tag, let me know. I will re-validate for
consistency and then write `requirements.md`.**
