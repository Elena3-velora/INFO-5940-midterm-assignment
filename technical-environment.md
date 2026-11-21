# Technical Environment: Storywell

Storywell is a local web app that helps students and new graduates in the North American job market tailor a resume and cover letter to a specific job description (JD). Every suggestion must be traceable to the user's original resume or to facts the user explicitly confirmed. This document defines the stack, constraints, and code patterns AI-DLC should follow.

## 1. Language and Package Manager

| Area | Choice | Notes |
|---|---|---|
| Backend language | Python 3.12+ | Type hints required on all public functions |
| Backend package manager | `uv` only | `pyproject.toml` for project and tool config; commit `uv.lock` |
| Frontend language | TypeScript (strict mode) | No plain `.js` source files in `frontend/src` |
| Frontend runtime | Node.js 22 LTS | |
| Frontend package manager | `npm` | Commit `package-lock.json` |

## 2. Frameworks and Libraries

### Backend (required)

| Library | Purpose |
|---|---|
| FastAPI | REST API under `/api` |
| Pydantic v2 | Request/response models and LLM structured output schemas |
| `anthropic` (official Python SDK) | All LLM calls |
| `pdfplumber` | Extract the text layer from PDF resumes |
| `python-docx` | Read DOCX resumes and write DOCX exports |
| `fpdf2` | Write PDF exports from a fixed template |
| `uvicorn` | Local ASGI server |
| `python-multipart` | File uploads in FastAPI |
| `python-dotenv` | Load `.env` in local development |

### Frontend (required)

| Library | Purpose |
|---|---|
| React 19 + Vite | Single-page app |
| Native `fetch` | API calls, wrapped in `frontend/src/api/client.ts` |
| React `useState` / `useReducer` | UI state; no global state library |
| Plain CSS modules | Styling |

## 3. LLM Provider and Model

| Item | Choice |
|---|---|
| Provider | Anthropic Claude API (hosted) |
| Primary model | `claude-opus-5-5` |
| Fallback model if budget is tight | `claude-sonnet-5-5` (change one config value, `LLM_MODEL`) |
| Output format | Structured outputs via `client.messages.parse(..., output_format=<PydanticModel>)` |
| Effort | Set explicitly per call (`output_config={"effort": ...}`): `medium` for extraction and matching, `high` for rewriting and cover letters |

Rules for LLM use:

- All LLM calls go through one module, `backend/app/llm/client.py`. No other module imports `anthropic`. This keeps the provider swappable (for example, to a local Ollama model later) without touching services.
- Every LLM output is parsed into a Pydantic model. Free-text parsing of LLM responses is not allowed.
- Check `stop_reason` before using a response. `refusal` and `max_tokens` are treated as errors and surfaced to the user as "could not generate, please retry"; nothing partial is shown.
- The LLM proposes; deterministic code verifies. Source checks (Section 9, `verify_suggestion`) run on every suggestion before it reaches the UI.
- Unit tests never call the real API. A `FakeLLMClient` returns fixed Pydantic objects.

## 4. Cloud Provider and Deployment Model

- **Local only.** The product runs on each teammate's laptop for development and the demo. No cloud hosting.
- Backend: `uv run uvicorn app.main:app --host 127.0.0.1 --port 8000 --reload` (from `backend/`).
- Frontend: `npm run dev` on `localhost:5173`, proxying `/api` to port 8000 via Vite config.
- The only external network call is to the Anthropic API.
- **No database.** Session state (parsed resume, JD, confirmed facts, suggestions and their accept/reject status) lives in an in-memory store keyed by a random session ID, expiring after 2 hours of inactivity. Restarting the server clears everything. This matches the out-of-scope decision "no saved accounts or history".
- Uploaded files are read into memory (`BytesIO`) and never written to disk.

## 5. Architecture and Project Structure

### API style

REST + JSON, all routes under `/api`, versionless for the MVP. Errors return `{"error": {"code": str, "message": str}}` with an appropriate HTTP status.

Initial endpoint sketch (Application Design may refine):

| Method | Path | Purpose |
|---|---|---|
| POST | `/api/sessions` | Create a session |
| POST | `/api/sessions/{sid}/resume` | Upload PDF/DOCX or paste text |
| POST | `/api/sessions/{sid}/job-description` | Paste JD text |
| POST | `/api/sessions/{sid}/requirements/match` | Build the JD requirement checklist |
| POST | `/api/sessions/{sid}/facts` | Answer a follow-up question (creates a confirmed fact or a "no") |
| POST | `/api/sessions/{sid}/suggestions` | Generate rewrite suggestions; body includes `style` (`StyleChoice`) and optional `writing_sample` |
| PATCH | `/api/sessions/{sid}/suggestions/{suggestion_id}` | Accept, reject, or edit a suggestion |
| POST | `/api/sessions/{sid}/cover-letter` | Generate a cover letter draft from accepted content and confirmed facts; body includes `style` and optional `writing_sample` |
| POST | `/api/sessions/{sid}/export` | Export confirmed content as DOCX or PDF |

### Processing flow

`parse resume -> extract JD requirements -> match evidence -> ask follow-up questions -> generate suggestions -> verify sources -> user accepts/rejects -> cover letter -> export`

### Core data model (Pydantic)

```python
class ResumeItem(BaseModel):
    id: str                  # e.g. "exp2.b3" (experience 2, bullet 3)
    section: Literal["education", "experience", "project", "skills", "other"]
    text: str

class ConfirmedFact(BaseModel):
    id: str                  # e.g. "fact1"
    question: str            # what the system asked
    answer: str              # the user's own words, stored verbatim

class RequirementMatch(BaseModel):
    requirement: str
    status: Literal["found", "needs_confirmation", "not_found"]
    evidence_ids: list[str]  # ResumeItem or ConfirmedFact ids; empty unless "found"

class Suggestion(BaseModel):
    id: str
    original_id: str
    original_text: str
    proposed_text: str
    source_ids: list[str]    # must be non-empty and must all exist in the session
    rationale: str
    status: Literal["pending", "accepted", "rejected", "edited"] = "pending"
    flags: list[str] = []    # e.g. "unsupported_number:40%", "verb_inflation:led"

class StyleChoice(StrEnum):
    natural = "natural"
    professional = "professional"
    keep_my_voice = "keep_my_voice"
```

### Directory layout

```
backend/
  app/
    main.py              # FastAPI app, router registration
    api/                 # route handlers only; no business logic
    services/            # parsing, matching, suggestions, cover letter, export
    verification/        # deterministic truthfulness checks
    llm/                 # client.py (only Anthropic import), prompts/
    models/              # Pydantic models
    store.py             # in-memory session store with TTL
  tests/
    unit/
    api/
    fixtures/            # sample resumes (.pdf/.docx/.txt) and JDs
    eval/                # truthfulness evaluation set (real API, opt-in)
frontend/
  src/
    api/                 # typed API client
    components/
    pages/
aidlc-docs/              # AI-DLC artifacts only; no application code here
```

## 6. Testing

| Area | Tooling | Gate |
|---|---|---|
| Backend unit + API tests | `pytest`, `pytest-asyncio`, FastAPI `TestClient` | Must pass before merging to `main` |
| Backend coverage | `pytest-cov` | ≥ 80% line coverage for `app/services` and `app/verification` |
| Backend lint/format/types | `ruff` (lint + format), `mypy` | No errors |
| Frontend tests | `vitest` + React Testing Library | Core flows: upload, accept/reject, export button state |
| Frontend lint/types | `eslint`, `tsc --noEmit` | No errors |
| Truthfulness eval | `pytest -m eval` against 8-10 resume/JD pairs in `tests/eval/` | Run manually before each milestone; calls the real API |

Testing rules:

- Unit and API tests use `FakeLLMClient`; they never require an API key or network.
- Every truthfulness rule has at least one test that proves a violation is caught:
  - a number in `proposed_text` that does not appear in any source is flagged;
  - a verb that raises seniority (for example "participated" to "led") is flagged;
  - a skill the user did not confirm never appears in suggestions or the cover letter;
  - a suggestion with empty or unknown `source_ids` is rejected.
- Eval runs report, per pair: number of suggestions, number flagged by verification, and any unsupported claim found on manual review.
- No CI service for the MVP. Run `uv run pytest` and `npm test` locally before opening a pull request.

## 7. Do NOT Use

| Prohibited | Reason | Use Instead |
|---|---|---|
| OCR libraries (`pytesseract`, `easyocr`, cloud OCR) | Scanned resumes are out of scope | Detect missing text layer and ask the user to paste text |
| Scraping/browser automation (`selenium`, `playwright`, `beautifulsoup4`, `scrapy`) | No crawling of job sites; JDs come from the user | JD pasted as text |
| Databases and ORMs (SQLite, Postgres, SQLAlchemy, Redis) | No accounts or saved history | In-memory session store (`store.py`) |
| Auth libraries (`fastapi-users`, `authlib`, JWT libraries) | Single local user, no accounts | Bind server to `127.0.0.1` |
| Editing the original PDF/DOCX layout (`PyMuPDF` write APIs, `pdfrw`) | Restoring the original layout is out of scope | Export confirmed text with a fixed template (`python-docx`, `fpdf2`) |
| LangChain, LlamaIndex, other agent frameworks | Hide prompts and make source tracing harder to verify | Anthropic SDK + our own pipeline in `services/` |
| OpenAI SDK or OpenAI-compatible shims in product code | One provider for the MVP. This applies only to the product's runtime code; teammates may still use any coding agent (for example, OpenAI Codex) to write code | `anthropic` SDK in `llm/client.py` only |
| Parsing LLM output with regex or `json.loads` on free text | Fragile; bypasses validation | `client.messages.parse` with a Pydantic `output_format` |
| `eval()`, `exec()`, `pickle` | Security risk | Pydantic models; plain dicts in memory |
| `requests` | Blocks the async event loop | `httpx` (only where an HTTP client is needed, e.g. tests) |
| Flask, Django, Streamlit | One web stack | FastAPI + React |
| Redux, MobX, Zustand | Unneeded for the MVP's state size | React `useState` / `useReducer` |
| Large UI kits (MUI, Ant Design) | Extra weight; the UI is small | Plain CSS modules |
| pip, poetry, pipenv | One package manager | `uv` |
| black, flake8, isort | Replaced by one tool | `ruff` |
| Writing uploads or resume text to disk or temp files | Resume content is personal data | `BytesIO` in memory |
| `print()` for diagnostics | Can leak resume content to logs | `logging` with IDs and lengths only |
| Cloud deployment (AWS, Vercel, Render, etc.) | Product runs locally | Local `uvicorn` + Vite dev server |

## 8. Security Basics

- **Authentication:** none; single local user. The backend binds to `127.0.0.1` only, so it is not reachable from other machines. Session IDs are random (`secrets.token_urlsafe(16)`).
- **Secrets:** `ANTHROPIC_API_KEY` lives in `backend/.env`, loaded with `python-dotenv`. `.env` is in `.gitignore`; only `.env.example` (with an empty value) is committed. Never log or return the key.
- **Personal data:** resume text, JD text, confirmed facts, and writing samples stay in memory only and expire with the session. Logs record session IDs, item IDs, character counts, and timings, never content. The UI states that resume text is sent to the Anthropic API for processing.
- **Upload validation:**
  - Accept only `.pdf` and `.docx`, checked by extension and file signature (`%PDF` / ZIP header).
  - Maximum file size 5 MB; PDFs at most 4 pages.
  - If extracted text is under 200 characters, reject as a likely scanned PDF and ask the user to paste text.
  - Pasted resume text ≤ 20,000 characters; JD text ≤ 15,000 characters; writing sample ≤ 3,000 characters.
- **Prompt injection:** resume, JD, and writing-sample text are untrusted data. Prompts wrap each in its own XML tag (`<resume>`, `<job_description>`, `<writing_sample>`) and the system prompt states that text inside these tags is content to analyze, never instructions. All LLM output is schema-validated, and every `source_id` must exist in the session before a suggestion is shown.
- **Truthfulness as a safety property:** content that fails verification is never shown as a clean suggestion; it is either dropped or shown with its flags so the user can decide.
- **CORS:** allow only `http://localhost:5173`.

## 9. Example Code Patterns

### Endpoint

```python
# backend/app/api/requirements.py
from fastapi import APIRouter, Depends, HTTPException

from app.llm.client import LLMClient, LLMError, get_llm_client
from app.models.requirements import MatchResponse
from app.services.requirements import match_requirements
from app.store import SessionNotFound, get_store, SessionStore

router = APIRouter(prefix="/api/sessions/{sid}/requirements", tags=["requirements"])


@router.post("/match", response_model=MatchResponse)
async def match(
    sid: str,
    store: SessionStore = Depends(get_store),
    llm: LLMClient = Depends(get_llm_client),
) -> MatchResponse:
    try:
        session = store.get(sid)
    except SessionNotFound:
        raise HTTPException(status_code=404, detail="Session not found or expired")

    if not session.resume_items or not session.jd_text:
        raise HTTPException(status_code=400, detail="Upload a resume and a JD first")

    try:
        matches = await match_requirements(
            llm, session.resume_items, session.confirmed_facts, session.jd_text
        )
    except LLMError:
        raise HTTPException(status_code=502, detail="Could not analyze the JD. Please retry.")

    session.requirement_matches = matches
    return MatchResponse(matches=matches)
```

### LLM call (the only place `anthropic` is imported)

```python
# backend/app/llm/client.py
import anthropic
from pydantic import BaseModel

from app.config import settings


class LLMError(Exception):
    pass


class LLMClient:
    def __init__(self) -> None:
        self._client = anthropic.AsyncAnthropic()  # reads ANTHROPIC_API_KEY

    async def structured[T: BaseModel](
        self, *, system: str, user: str, schema: type[T], effort: str = "medium"
    ) -> T:
        response = await self._client.messages.parse(
            model=settings.llm_model,            # "claude-opus-5-5"
            max_tokens=16000,
            system=system,
            messages=[{"role": "user", "content": user}],
            output_format=schema,
            output_config={"effort": effort},
        )
        if response.stop_reason in ("refusal", "max_tokens") or response.parsed_output is None:
            raise LLMError(f"LLM call failed: stop_reason={response.stop_reason}")
        return response.parsed_output
```

### Function (deterministic verification)

```python
# backend/app/verification/checks.py
import re

NUMBER = re.compile(r"\d+(?:[.,]\d+)?%?")

# Higher level = more seniority/ownership implied.
VERB_LEVEL = {
    "assisted": 1, "participated": 1, "contributed": 1, "supported": 1, "helped": 1,
    "built": 2, "developed": 2, "created": 2, "implemented": 2, "designed": 2,
    "led": 3, "managed": 3, "owned": 3, "directed": 3, "spearheaded": 3,
}


def find_unsupported_numbers(proposed: str, sources: list[str]) -> list[str]:
    """Numbers in the proposed text that appear in none of the source texts."""
    source_text = " ".join(sources)
    return [n for n in NUMBER.findall(proposed) if n not in source_text]


def find_verb_inflation(proposed: str, sources: list[str]) -> list[str]:
    """Verbs in the proposed text that imply more seniority than any source verb."""
    def max_level(text: str) -> int:
        words = re.findall(r"[a-z]+", text.lower())
        return max((VERB_LEVEL.get(w, 0) for w in words), default=0)

    source_level = max((max_level(s) for s in sources), default=0)
    return [
        w for w in re.findall(r"[a-z]+", proposed.lower())
        if VERB_LEVEL.get(w, 0) > max(source_level, 1)
    ]


def verify_suggestion(proposed: str, source_ids: list[str], known: dict[str, str]) -> list[str]:
    """Return flags; an empty list means the suggestion passed all checks."""
    if not source_ids or any(sid not in known for sid in source_ids):
        return ["invalid_source_ids"]
    sources = [known[sid] for sid in source_ids]
    flags = [f"unsupported_number:{n}" for n in find_unsupported_numbers(proposed, sources)]
    flags += [f"verb_inflation:{v}" for v in find_verb_inflation(proposed, sources)]
    return flags
```

### Test

```python
# backend/tests/unit/test_checks.py
from app.verification.checks import (
    find_unsupported_numbers,
    find_verb_inflation,
    verify_suggestion,
)


def test_flags_invented_metric():
    assert find_unsupported_numbers(
        "Cut report time by 40% by automating weekly reports",
        ["Automated weekly sales reports in Excel"],
    ) == ["40%"]


def test_allows_number_from_source():
    assert find_unsupported_numbers(
        "Built dashboards used by 3 teams", ["Built Tableau dashboards for 3 teams"]
    ) == []


def test_flags_participated_to_led():
    assert find_verb_inflation(
        "Led a 4-person capstone project", ["Participated in a 4-person capstone project"]
    ) == ["led"]


def test_allows_same_level_rewording():
    assert find_verb_inflation(
        "Developed a Python script to clean survey data",
        ["Built a Python script that cleaned survey data"],
    ) == []


def test_rejects_unknown_source_id():
    known = {"exp1.b1": "Built a Python script that cleaned survey data"}
    assert verify_suggestion("Wrote SQL queries", ["exp9.b9"], known) == ["invalid_source_ids"]
```
