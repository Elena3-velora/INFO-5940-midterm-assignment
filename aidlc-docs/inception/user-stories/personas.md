# Personas: Storywell

**Stage**: INCEPTION — User Stories
**Date**: 2026-11-22 (revised after story review)
**Basis**: `vision.md` §2 "Problem Statement", §3 "Target Users", §6 "Product rules";
`requirements.md` §3, §4, PR-2

> These are **fictional archetypes**, not real individuals, and each one represents a broad
> category of users, not one specific major, degree, or target job. Storywell serves students
> and new graduates of **any major, applying to any role**, in the North American job market.
> The scenario under each persona is **one representative example** of the kind of gap or
> wording problem they run into — not an exhaustive description of their experience. Per
> Q4 = X of `story-generation-plan.md`, the three personas are exactly the three user types
> named in `requirements.md` §3 / `vision.md` §3; no new personas were added, and every goal
> and pain point below traces to `vision.md` or `requirements.md`.

---

## Persona 1: Alex — The Student Seeking an Internship

**Who they are**: An undergraduate or graduate student, of any major, applying for
internships in North America (`vision.md` §3). Their experience is mostly coursework, class
projects, and part-time work.

**Narrative**: Alex is applying for their first real internship and doesn't have much
full-time work to point to yet — some classes, a group project, maybe a part-time job in a
different field. Every job posting asks for things Alex isn't sure they can claim, and every
general AI writing tool Alex has tried either leaves the resume thin or quietly invents
experience Alex doesn't have.

**Representative scenario**: A JD Alex is applying to asks for "experience with SQL." Alex's
resume only has one line that's related: "Used SQL in a database systems course." That's real
experience, but it's one class, not job experience — exactly the kind of partial evidence
that should be labeled `needs_confirmation`, not `found`, and should prompt a follow-up
question about how deep that experience actually goes (`requirements.md` FR-221).

**Goals**:
- Match limited experience (courses, class projects, part-time work) to a JD and describe it
  with confidence (`vision.md` §3).
- See exactly where their resume already matches the JD and where the real gaps are, instead
  of one vague match score (`vision.md` §6, JD requirement check).
- Be asked before anything is assumed about them, rather than have a tool guess and get it
  wrong (`vision.md` §6, follow-up questions).

**Pain points**:
- Doesn't know which parts of their limited experience match a JD, or how to describe that
  experience clearly (`vision.md` §2).
- Worried that rewriting their resume with a general AI chatbot means it will "add skills the
  user never had... or invent numbers" — and that sending an exaggerated resume risks being
  caught in an interview (`vision.md` §2).

**Representative quote** (illustrative, not a real user statement):
> "I've only got one class that touches SQL — I don't know if that counts as 'experience' or
> if I'm supposed to say I don't have it."

---

## Persona 2: Jordan — The New Graduate

**Who they are**: A recent graduate, of any major, applying for their first full-time job in
North America (`vision.md` §3).

**Narrative**: Jordan is sending out applications to several different roles in the same
week — not all in the same field or at the same level — and each job description emphasizes
something different. Jordan needs their resume to feel freshly targeted every time, without
rewriting it from scratch for every posting. Speed matters as much as accuracy.

**Representative scenario**: In one week, Jordan applies to three different roles. One JD
emphasizes data analysis, another emphasizes client communication, and a third emphasizes a
tool Jordan has only touched briefly. The same resume needs to be re-tailored three separate
times, with a different set of lines highlighted and a different set of gaps surfaced each
time — tailoring "to every job you apply for" is explicitly called out as the time-consuming
part of the problem (`vision.md` §2).

**Goals**:
- A fast way to pick and reword their most relevant experience for each application
  (`vision.md` §3).
- Go from upload to export in about 10 minutes (`vision.md` §4, success metric).
- Highlight the experience most relevant to a given JD by reordering bullets, without inventing
  anything new (`requirements.md` PR-1.3, FR-501).

**Pain points**:
- Tailoring a resume and cover letter well for every job takes a lot of time, especially when
  applying to several different kinds of roles (`vision.md` §2, problem statement).
- Needs the output to still sound specific and real, not generic or keyword-stuffed, the way
  general AI chatbots tend to write (`vision.md` §2).

**Representative quote** (illustrative, not a real user statement):
> "I'm applying to three different kinds of roles this week — I need to retarget my resume for
> each one in minutes, not start over every time."

---

## Persona 3: Sam — The International Student

**Who they are**: A student or new graduate, of any major, whose first language is not
English (`vision.md` §3).

**Narrative**: Writing a resume and cover letter in English is already harder for Sam than
for a native speaker, and the stakes of sounding "off" — too stiff, too generic, or
inadvertently overstated — feel higher. Sam wants the writing to read naturally without
losing their own voice.

**Representative scenario**: Sam's resume has a line that reads "Took part in a class project
building a mobile app." Sam is worried that a rewriting tool will turn that into "Led a mobile
app project" — exaggerating Sam's actual role just to sound stronger. This is the exact
example the product rules name directly: "'took part in a project' cannot become 'led a
project'" (`vision.md` §6, Rule 1).

**Goals**:
- Natural, clear English wording that still sounds like them, without inflated language
  (`vision.md` §3).
- Keep their real seniority level and tone rather than being rewritten as more senior than they
  are (`requirements.md` PR-2.3).
- Optionally preserve their own voice directly, by supplying a writing sample
  (`requirements.md` FR-601, FR-602).

**Pain points**:
- Writing in a second language is explicitly called out as harder than the general problem of
  tailoring a resume (`vision.md` §2).
- General AI tools tend to produce "generic, keyword stuffed text that sounds machine written"
  (`vision.md` §2) — exactly the opposite of sounding like themselves.
- Specifically worried about a tool inflating what they actually did, e.g. "took part in" a
  project becoming "led" it (`vision.md` §6, Rule 1).

**Representative quote** (illustrative, not a real user statement):
> "I only took part in that project — I didn't lead it. I want it to sound polished, but I
> don't want it to say something that isn't true."

---

## Persona-to-Story Mapping

Each story in `stories.md` is tagged with exactly one **primary persona** (the user whose
stated need the story most directly serves), per Q1/Q4 of `story-generation-plan.md`. Every
persona is primary for at least one story.

| Persona | Primary stories |
|---|---|
| **Alex — Student Seeking an Internship** | US-FOUNDATION-03, US-FOUNDATION-04, US-RESUME-01, US-MATCH-03, US-MATCH-04, US-MATCH-05, US-SUGGEST-05, US-EXPORT-02 |
| **Jordan — New Graduate** | US-FOUNDATION-01, US-FOUNDATION-02, US-MATCH-01, US-MATCH-02, US-SUGGEST-04, US-SUGGEST-06, US-SUGGEST-07, US-COVER-01, US-COVER-02, US-EXPORT-01, US-EXPORT-04 |
| **Sam — International Student** | US-RESUME-02, US-SUGGEST-01, US-SUGGEST-02, US-SUGGEST-03, US-SUGGEST-08, US-COVER-03, US-EXPORT-03 |

This mapping reflects which persona's stated need each story most directly serves; it does not
mean other personas never use that feature — e.g., every persona uploads a resume
(US-RESUME-01), but Alex's limited-experience concern is the sharpest fit for that story's
purpose. The mapping is unchanged by this revision; only the persona names and scenarios were
added.
