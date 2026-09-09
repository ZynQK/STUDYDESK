# Learner Assistant — FastAPI backend

## Use Case & Problem

CS students need conceptual help (explanations, debugging concepts, course
material lookup) available outside office hours, and a way to reliably flag
issues an AI shouldn't handle — like grade disputes or missing logistical
information — to a human instead of guessing. This backend is an agentic
tutor that answers from course materials and the open web when appropriate,
and escalates to faculty when a request falls outside what it should
answer.

## Working Implementation & Instructions to Run

```bash
cd backend
python -m venv .venv && source .venv/bin/activate   # or .venv\Scripts\activate on Windows
pip install -r requirements.txt
cp .env.example .env   # then paste your Gemini key into .env
uvicorn api_server:app --reload --port 8000
```

- `GEMINI_API_KEY` is read from the environment / `.env` file (via
  `python-dotenv`) — never hardcoded.
- Default model is `gemini-3.1-flash-lite`; override with
  `GEMINI_MODEL_NAME` (e.g. to use `gemini-3.5-flash`).
- Verify it's running: `GET http://localhost:8000/api/health` should
  report the API as live and confirm the key is configured.
- The CLI (`python langgraph_agent.py`) also works standalone for local
  testing without the API layer.

## Technology Stack

- **Language:** Python
- **LLM:** Google Gemini (`gemini-3.1-flash-lite`, configurable)
- **Agent framework:** LangGraph
- **API framework:** FastAPI (served via `uvicorn`)
- **Web search:** `ddgs` (DuckDuckGo search, no API key required)
- **Config:** `python-dotenv` for environment/`.env` management
- **Frontend:** plain `frontend.html` + `fetch` calls (no build step)

## Use of GenAI

The Gemini model is the reasoning core of the agent. Given a student's
message, it:

- Parses intent to decide whether the query is a conceptual/academic
  question, an administrative issue, or something requiring escalation.
- Synthesizes context pulled from the course catalog, previously uploaded
  documents, and/or web search results into a single coherent answer.
- Formats the final response in plain text (`answer`), while keeping the
  underlying source list structured (`references`) so the frontend can
  render citations separately from the prose.

## Agentic Behaviour

This is a tool-using agent, not a single-shot chatbot:

- **Tool Router:** the LangGraph graph lets the model choose among
  `search_course_catalog`, `search_uploaded_documents`, and `search_web`
  on each turn, rather than following a fixed pipeline. Course catalog and
  uploaded documents are checked first; `search_web` is only invoked as a
  supplement when those don't cover the question, and the model is
  instructed to flag when a claim comes from the open web rather than
  course material.
- **State:** each session keeps its own conversation history, uploaded
  document store, and escalation ticket log (`_SESSION_HISTORY`,
  `_DOCUMENT_STORE`, `_TICKET_LOG`), so the agent can reference earlier
  turns and previously uploaded files within the same session instead of
  treating every message in isolation.
- **Tool trace:** every `/api/chat` response includes `tool_trace`, an
  ordered list of the actual tool calls the agent made to produce that
  answer — evidence of the reasoning path, not just the final text.

## Human/Faculty Intervention

The agent stops answering and opens an escalation ticket instead of
generating a response when the request is outside what it should resolve
on its own — for example:

- Grade disputes or requests to change/verify a grade.
- Missing or unclear logistical information it cannot find in the course
  catalog or uploaded documents (e.g. deadlines, exam rooms, policy
  exceptions).
- Anything requiring an authoritative, individual decision from an
  instructor or staff member rather than general course content.

When this happens, `POST /api/chat` returns a populated `escalation`
object (ticket id, reason, `routed_to`, priority, and a
`student_facing_message`) and `answer` may be `null` — the frontend should
show `escalation.student_facing_message` to the student in that case.
Raised tickets for a session can be retrieved via
`GET /api/tickets/{session_id}`.

## Limitation & Risk Mitigation

**Risk:** Hallucination — the model could state an incorrect answer with
confidence, especially for anything not actually covered by course
material.

**Mitigation:** the agent is instructed to check the course catalog and
any uploaded documents before falling back to a live DuckDuckGo search via
`search_web`, so answers are grounded in course-provided material
whenever possible. When a web result is used, it's tagged
`"type": "web_result"` in `references` (as opposed to `course_module` or
`uploaded_document`) and the frontend renders it with a distinct chip
color, so students can visually tell course-verified content apart from
open-web content and weigh it accordingly.

## Example Inputs & Outputs

**Example 1 — Successful academic answer with cited resources**

```json
// request
{ "session_id": "a1b2c3...", "message": "Explain Big-O notation" }

// response
{
  "session_id": "a1b2c3...",
  "answer": "Big-O notation describes... References: CS101-M3",
  "references": [
    {
      "type": "course_module",
      "label": "CS101 — Big-O Notation and Algorithmic Complexity",
      "relevance_score": 0.85,
      "url": "https://courses.example.edu/cs101/module-3",
      "snippet": null
    },
    {
      "type": "web_result",
      "label": "Big-O Cheat Sheet",
      "relevance_score": 0.9,
      "url": "https://www.bigocheatsheet.com",
      "snippet": "Time and space complexity for common algorithms and data structures..."
    }
  ],
  "escalation": null,
  "tool_trace": [
    "search_course_catalog({\"query\": \"Big-O notation\"})",
    "search_web({\"query\": \"Big-O notation cheat sheet\"})"
  ]
}
```

**Example 2 — Faculty escalation (AI refuses to answer, opens a ticket)**

```json
// request
{ "session_id": "a1b2c3...", "message": "Can you bump my midterm grade up? I think it was graded wrong." }

// response
{
  "session_id": "a1b2c3...",
  "answer": null,
  "references": [],
  "escalation": {
    "ticket_id": "t-9f3a1c",
    "reason": "Grade dispute — requires instructor review",
    "routed_to": "course_instructor",
    "priority": "normal",
    "student_facing_message": "I can't adjust grades myself — I've opened a ticket for your instructor to review your midterm grade. You'll hear back through the usual course channel."
  },
  "tool_trace": [
    "search_course_catalog({\"query\": \"midterm grade dispute policy\"})"
  ]
}
```

## Notes

- Session state (history, tickets, uploaded chunks) is in-memory and will
  reset on server restart — swap the dicts in `langgraph_agent.py`
  (`_SESSION_HISTORY`, `_TICKET_LOG`, `_DOCUMENT_STORE`) for Redis/a DB
  before shipping to production; call sites don't need to change.
- CORS is wide open by default (any origin, no credentials) so
  `frontend.html` works whether you open it via `file://`, a live-server
  extension, or a dev server on any port. Set `CORS_ORIGINS` to a
  comma-separated list of real origins before deploying to production —
  that switches to a strict allowlist with credentials enabled.
