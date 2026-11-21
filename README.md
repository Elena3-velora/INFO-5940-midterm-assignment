# Storywell

A resume and cover letter tailoring assistant for students and new graduates in the North American job market. INFO 5940 team project.

## Team Members

| Name | Cornell NetID | Role |
|---|---|---|
| Xinyi Wang | xw785 | Project Planning & Requirements |
| Linghui Zhou | lz633 | Application Development |
| Yunzhu Qiu | yq337 | AI Development Tools, Collaboration Plan & Documentation |

## 1. Problem and Why We Use LLMs

Students and new graduates are expected to tailor their resume and cover letter for every job they apply to. This takes a lot of time, and many of them have limited work experience and are not sure which parts of it match a job description (JD) or how to describe it well. It is even harder for international students who are writing in their second language. Our product, Storywell, is a resume tailoring assistant for students looking for internships and new graduates looking for their first job in the North American job market. The user uploads their resume and pastes a JD, and the assistant helps them present their real experience clearly for that job.

An LLM is a good fit for this problem because the task is mostly about understanding and rewriting language. Matching a JD to a resume is not just keyword matching. "Built dashboards for a class project" can be real evidence for "experience with data visualization" even though the words are different. An LLM can read both documents, find these connections, and rewrite a line in clearer English while keeping its meaning. At the same time, LLMs are known to make things up, and general chatbots often add skills or numbers the user never had. That is why our product is built around one hard rule: every statement must come from the original resume or from information the user has confirmed. We use the LLM for what it does well, and we design the product to control what it does badly.

## 2. How LLMs Fit Into the Product

The LLM powers four main features. First, it reads the JD and pulls out each requirement. Second, it checks the resume for evidence of each requirement and labels it "found", "needs confirmation", or "not found". Third, it writes line by line rewrite suggestions, each shown next to the original line and the source it is based on, so the user can accept, reject, or edit it. Fourth, it drafts a cover letter using only the facts the user has confirmed. Users can also choose a writing style: natural and clear, professional, or keep my voice, and can paste a short writing sample so the output sounds more like them.

Our design follows one principle: the LLM proposes, and our own code verifies. The assistant works as a step by step pipeline: parse the resume, extract JD requirements, match evidence, ask follow up questions, generate suggestions, verify sources, and let the user accept, reject, or edit. The LLM makes the judgment calls along the way, such as deciding which requirements have no evidence and need a question. For example, if the JD asks for SQL and the resume never mentions it, the assistant asks the user whether they have used SQL instead of guessing. The user's answer is saved as a confirmed fact that later steps can use. If the user does not confirm it, SQL will not appear in the resume or cover letter. After the LLM writes a suggestion, deterministic checks make sure every source it cites really exists, and flag any number or stronger verb (for example "participated" changed to "led") that is not in the source. We build this pipeline ourselves instead of using an agent framework like LangChain, so every step stays easy to trace and test. We plan to use the Anthropic Claude API, with Claude Opus 5.5 as our main model and Claude Sonnet 5.5 as a lower cost backup if our budget gets tight.

## 3. Project Scope

By the final project deadline, our product will be a working local web app that covers the full tailoring flow: upload a resume (text based PDF, DOCX, or pasted text), paste a JD, see the requirement check, review and accept, reject, or edit rewrite suggestions, answer follow up questions, get a cover letter draft, and export the confirmed content as DOCX or PDF. This is more than one small feature. It includes file handling, a multi step LLM pipeline with follow up questions and automatic truthfulness checks, a review interface, and document export, which together make a complete product a real user could use for an application.

We also kept the scope realistic for the time we have. Several features are out of scope for the MVP: writing a resume from scratch, auto applying to jobs, scraping job sites, visa sponsorship filtering, mock interviews, scanned resume support, user accounts and saved history, and keeping the original resume layout. These features are listed in our vision document with the reason each one was left out. Focusing on one complete flow means we can spend our time on the hardest and most important part of the product, which is making the suggestions useful while never making anything up.

## 4. AI Development Tools

- **AI development lifecycle:** AWS AI-DLC v1.0.1
- **AI coding agents:** Claude Code (Linghui, Xinyi, and Yunzhu) and OpenAI Codex (Yunzhu, as an alternative)
- **Version control:** Git and GitHub

We run the AI-DLC workflow once, together, in Claude Code. One teammate operates the agent during our shared sessions while all three of us answer its questions and review and approve each stage. The AI-DLC v1.0.1 rules are installed for both agents we use: `CLAUDE.md` for Claude Code and `AGENTS.md` for OpenAI Codex, with the shared rule details in `.aidlc-rule-details/`.

During the Construction phase, we use the coding agents to generate code, debug, and write tests. All AI-generated code is reviewed and tested by team members before it is merged into the main branch.

## 5. Collaboration Plan

### 5.1 Meetings and Communication

- **Meeting frequency:** We meet twice a week, about one hour each time.
- **Meeting format:** All meetings are in person.
- **Between meetings:** We use a WeChat group for day-to-day communication, plus GitHub Issues and pull requests to track tasks and review changes.

### 5.2 Division of Work

- **Xinyi Wang:** product proposal and scope (README sections 1 to 3) and the vision document.
- **Linghui Zhou:** technical environment document, and leading application development in the Construction phase.
- **Yunzhu Qiu:** AI-DLC setup and record, the AI tools and collaboration sections (README sections 4 and 5).
- **All three:** attend every AI-DLC working session, answer the agent's questions, review each artifact before approval, and test the product together before the final submission.

### 5.3 Collaborative Development Workflow

- **Shared Planning:** Team members jointly discuss requirements, establish priorities, and agree on implementation approaches before development.
- **Parallel Development:** Individual tasks are developed independently where possible, with clearly defined interfaces to minimize dependencies.
- **Regular Integration:** Team members regularly share code updates through GitHub and integrate completed components incrementally.
- **Collective Review:** All members participate in reviewing functionality, identifying issues, and validating the final product.

### 5.4 Human-AI Collaboration

- Use AI coding agents to support code generation, debugging, and testing.
- Provide clear task descriptions and project context to AI agents.
- Review and validate AI-generated outputs before incorporating them into the project.
- Ensure human oversight of technical decisions and final implementation.

### 5.5 Progress Monitoring and Risk Management

- Track development progress against agreed milestones.
- Communicate blockers and dependencies as soon as they arise.
- Prioritize critical features and resolve integration issues early.
- Adjust task priorities collaboratively when unexpected challenges occur.
