# Vision Document: Storywell

## 1. Executive Summary

Storywell is a resume tailoring assistant for students looking for internships and new graduates looking for their first full time job in the North American job market. A user uploads their resume and pastes a job description (JD). The assistant compares the two, suggests line by line edits, asks follow up questions when the resume is missing information the JD asks for, and drafts a matching cover letter. Every suggestion is based only on facts from the original resume or facts the user has confirmed. The goal is to help people present their real experience clearly for each job, without making anything up.

## 2. Problem Statement

Students and new graduates are expected to tailor their resume and cover letter to every job they apply for. Doing this well takes a lot of time. Many of them have limited work experience and do not know which parts of it match a JD, how to describe that experience clearly, or how to write in a natural professional style. This is even harder for international students who are writing in their second language.

General AI chatbots can rewrite a resume, but they often make things up. They add skills the user never had, invent numbers, or turn "took part in a project" into "led a project." They also produce generic, keyword stuffed text that sounds machine written. A user who sends this kind of resume risks being caught in an interview. Our product solves this by treating truthfulness as a hard rule: the assistant only uses facts it can trace back to the user, and it asks instead of guessing.

## 3. Target Users

| User type | Description | What they need |
|---|---|---|
| Student looking for an internship | Undergraduate or graduate students applying for internships in North America | Help matching limited experience (courses, class projects, part time work) to a JD and describing it with confidence |
| New graduate | Recent graduates applying for their first full time job in North America | A fast way to pick and reword their most relevant experience for each application |
| International student | Students and new graduates whose first language is not English | Natural, clear English wording that still sounds like them, without inflated language |

## 4. Success Metrics

| Metric | Target |
|---|---|
| Fabricated facts in output | 0 unsupported claims across our evaluation set of 8 to 10 resume and JD pairs |
| Suggestions with a visible source | 100% of rewrite suggestions show the original line or the user's confirmed answer they are based on |
| Verification catches violations | Our automatic checks flag every invented number and every inflated verb (for example "participated" to "led") in our test cases |
| JD requirements checked | Every requirement extracted from the JD is labeled "found", "needs confirmation", or "not found" |
| Missing information handled | When a JD skill has no evidence in the resume, the assistant asks the user before using it, every time |
| Time to tailor | A user can go from upload to export in about 10 minutes |
| Export works | Confirmed content exports to DOCX and PDF without errors |

## 5. Full Scope Vision

At maturity, the product could include these areas. This section is aspirational and is not all planned for this course.

- **Resume tailoring:** rewrite suggestions, JD requirement checks, and follow up questions (the MVP core).
- **Cover letters:** drafts for each application based on confirmed facts.
- **Resume from scratch:** a guided interview for users who have no resume yet.
- **Application tracking:** saved versions of each tailored resume and cover letter, linked to the job.
- **Interview preparation:** practice questions based on the JD and the user's real experience.
- **Writing style learning:** the assistant learns the user's voice from their past writing.
- **More formats:** scanned resume support (OCR) and keeping the original resume layout on export.

## 6. MVP Scope: Features IN

| Feature | Description | Why it is in the MVP |
|---|---|---|
| Resume input | Upload a PDF (text based only) or DOCX file, or paste plain text | Users need a way to give us their real experience |
| JD input | Paste the job description as text | Tailoring is not possible without the target job |
| JD requirement check | List each requirement from the JD and label it "found", "needs confirmation", or "not found", with the matching resume evidence | Shows the user where they match and where the gaps are, instead of a single match score |
| Rewrite suggestions | Line by line suggestions shown next to the original line, with the source of each change. Before a suggestion is shown, our own code checks that its sources exist and that it adds no new numbers or inflated verbs | This is the main value of the product, and the checks make sure it follows our truthfulness rule |
| Accept, reject, or edit | The user accepts, rejects, or edits each suggestion one by one | Keeps the user in control of their own resume |
| Follow up questions | When the JD asks for something the resume does not show, the assistant asks the user before using it. Unconfirmed information is never added | This is how we enforce the truthfulness rule. The LLM judges which requirements lack evidence and need a question, and the user's answers become confirmed facts that later steps can use |
| Writing style choice | The user picks a style: natural and clear, professional, or keep my voice. For keep my voice, the user can paste a short writing sample (up to 3,000 characters) | Different users and jobs need different tones, and a sample helps the output sound like the user |
| Cover letter draft | A cover letter draft built only from accepted resume content and confirmed facts, which the user can edit | A cover letter is usually needed with the resume for the same JD, so it reuses the same confirmed facts |
| Export | Export the confirmed resume content and cover letter as DOCX or PDF | Users need a file they can actually submit |

### Product rules

**Rule 1: Every statement must be based on real facts.**
- Only use the original resume and information the user has clearly added and confirmed.
- Never add new jobs, skills, certificates, duties, numbers, or skill levels.
- Wording can be improved, order can change, and relevant experience can be highlighted, but nothing can be exaggerated. For example, "took part in a project" cannot become "led a project."
- If the JD asks for a skill with no evidence in the resume, ask the user. If the user does not confirm it, it does not go into the resume or cover letter.
- Every suggestion shows the original text or the user's added information it is based on.

**Rule 2: The writing should be natural, specific, and sound like the user.**
- Avoid empty phrases, too much praise, and copying JD keywords over and over.
- Support each point with specific experience instead of piling up adjectives.
- Keep the user's real level and tone. Do not write a student as a senior manager.
- All generated content is an editable draft that the user reviews and confirms.
- We aim for writing that reads like a real person wrote it, but we do not promise it will pass any AI detection tool.

## 7. MVP Scope: Features OUT

| Feature | Reason | Target phase |
|---|---|---|
| Resume from scratch (guided interview) | A separate large flow. Our MVP focuses on users who already have a resume | Future |
| Auto applying to jobs | Outside our core problem and has legal and account risks | Not planned |
| Scraping job sites | Terms of service risks. Users paste the JD instead | Not planned |
| Visa sponsorship filtering | Needs outside data sources and is a separate problem | Future |
| Mock interviews | A different product area | Future |
| Scanned resume support (OCR) | Adds complexity. We only support text based PDFs | Future |
| User accounts and saved history | Not needed to tailor one application. Avoids storing personal data | Future |
| Keeping the original resume layout | Hard to do reliably. We export with one fixed template | Future |

## 8. Risks and Open Questions

### Risks

| Risk | Impact | How we plan to handle it |
|---|---|---|
| The LLM adds facts that are not in the resume | Breaks our core rule and could hurt the user | Strict prompts, output checks that each suggestion links to a source, and testing on sample resumes |
| Resumes contain personal information | Privacy concern | Keep resume text in memory only and clear it when the session ends. Never write it to disk or logs. Tell users their text is sent to the Anthropic API. Keep the API key out of the repository |
| PDF text extraction is messy | Bad input leads to bad suggestions | Support text based PDFs only (up to 5 MB and 4 pages). If almost no text is found, ask the user to paste their resume instead |
| API cost | Running a large model on every request could get expensive | Switch to a smaller Claude model with one config change if our budget gets tight |
| Output sounds generic or AI written | Users will not trust or use it | Style options, specific wording rules, and user review of every change |

### Open questions

- How should we handle a JD requirement the user partly has, for example "used SQL in one class"?
- What does the exported DOCX and PDF template look like?
- How many follow up questions are too many before the user gets tired?
