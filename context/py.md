# Python Project Context

_Maintained by the `/py` agent. Updated incrementally as work is done._

---

## Tooling

- **Package manager**: `uv` — installed at `~/.local/bin/uv`. Always prefix commands with `export PATH="$HOME/.local/bin:$PATH"` in bash if not already on PATH.
- **Test runner**: `pytest` — run with `uv run pytest`
- **Async test mode**: `pytest-asyncio` with `asyncio_mode = "auto"` (set in `pyproject.toml`) — no `@pytest.mark.asyncio` decorators needed
- **Dev server**: `uvicorn main:app --reload --port 8000`
- **Prod server**: `gunicorn main:app -k uvicorn.workers.UvicornWorker -w 2 -b 0.0.0.0:8000`
- **In Docker**: `uv run gunicorn ...` (uses the venv managed by uv)

---

## Project Structure

**Repo**: `/Users/guypowell/Documents/Projects/learning-ai` (standalone, separate from Node monorepo at `learning/`)

```
learning-ai/
├── main.py                     # FastAPI app entry point
├── middleware/
│   └── auth.py                 # APIKeyMiddleware — X-Internal-API-Key validation
├── routers/
│   ├── generate.py             # /generate/lesson, /generate/quiz
│   └── coaching.py             # /coaching/message — prepends system msg with lesson_context + user_context
├── clients/
│   ├── __init__.py             # get_model_client() factory — reads AI_PROVIDER env var
│   ├── base.py                 # BaseModelClient (abstract) + ModelOutputError
│   ├── ollama_client.py        # OllamaClient — /api/generate + /api/chat
│   └── vertex_client.py        # VertexClient — Vertex AI GenerativeModel
├── models/
│   └── schemas.py              # Pydantic request/response models
├── prompts/
│   ├── lesson_prompts.py       # build_lesson_prompt() — skill-level + learning-style-aware lesson prompt
│   └── quiz_prompts.py         # build_quiz_prompt() — MCQ prompt from lesson content
├── tests/
│   ├── test_health_and_auth.py # 8 tests — health + auth middleware
│   ├── test_clients.py         # 14 tests — OllamaClient, VertexClient, factory
│   ├── test_generate.py        # 8 tests — /generate/lesson and /generate/quiz endpoints
│   ├── test_coaching.py        # 6 tests — /coaching/message endpoint
│   └── test_lesson_prompts.py  # 20 tests — build_lesson_prompt() style variant selection
├── pyproject.toml
├── uv.lock
├── Dockerfile
├── .env                        # Local dev — NOT committed
└── .env.example                # Template — committed
```

---

## Patterns

- `load_dotenv()` is called **before** FastAPI is imported in `main.py` — ensures env vars are available at middleware init time
- `ModelOutputError` (in `clients/base.py`) is raised by both clients on Pydantic validation failure; routers catch it and return HTTP 502
- `get_model_client()` uses lazy imports — `VertexClient` (which imports `vertexai`) is only loaded when `AI_PROVIDER=vertex`
- Ollama: `/api/generate` for lesson/quiz (JSON string in `response` field), `/api/chat` for coaching (JSON string in `message.content` field); both use `format:"json"` and `stream:false`
- Vertex: `generate_content_async()` with `GenerationConfig(response_mime_type="application/json")`; coaching flattens messages to `"Role: content\n..."` string
- Use `model_validate_json(raw)` (Pydantic v2) to parse + validate in one call — catches both JSON syntax errors and schema mismatches
- Middleware uses `starlette.middleware.base.BaseHTTPMiddleware` — standard FastAPI pattern
- 401 responses use `JSONResponse(status_code=401, content={"detail": "Unauthorized"})` — body matches FastAPI error shape
- `[tool.uv] package = false` in `pyproject.toml` — this is a service, not a library; prevents uv from trying to build/install the project itself
- All packages have `__init__.py` files (even if empty) — required for pytest discovery

---

## Dependencies

| Package | Version | Use |
|---------|---------|-----|
| `fastapi` | >=0.115 | Web framework |
| `uvicorn[standard]` | >=0.32 | ASGI dev server |
| `gunicorn` | >=23 | Prod process manager |
| `pydantic` | >=2.9 | Request/response validation |
| `httpx` | >=0.27 | Async HTTP client for Ollama calls; also used in tests as ASGI transport |
| `google-cloud-aiplatform` | >=1.70 | Vertex AI SDK (prod) |
| `python-dotenv` | >=1.0 | `.env` loading |
| `pytest` | >=8 | Test runner |
| `pytest-asyncio` | >=0.24 | Async test support |

---

## Gotchas

- `uv` is not on the system PATH by default — installed to `~/.local/bin/uv`. Must export PATH before running.
- `uv init --no-workspace` used (not plain `uv init`) because this repo must NOT join any uv workspace.
- Python 3.14 is the system default (Homebrew). The project requires `>=3.11` and uv resolves to 3.14.4.
- `pytest-asyncio==1.3.0` was installed (major version bump from 0.x to 1.x happened after Aug 2025).
- `starlette==1.0.0` was installed (also major bump). API is compatible with middleware patterns used.

---

## Lesson Generation Scripts (learning monorepo, not learning-ai)

The active Python surface is `learning/scripts/`: `lesson_config.py` (curriculum source of truth), `generate-lessons.py` (Claude Batch API), `generate-lessons-local.py` (Ollama fallback — stale, do not extend). The `learning-ai` FastAPI service documented above is **archived** — realtime generation was abandoned for offline batch.

**Planned work (2026-06-11, not started)**: `docs/persistent-ids-lessons.md` in learning-project specs persistent IDs (`trackId`/`levelId`/`topicId`/`lessonNumber`) threaded through `lesson_config.py` and `generate-lessons.py`, plus two new scripts (`backfill-lesson-ids.py`, `repair-orphan-lessons.py`). Read that spec before touching any generation script — it includes a root-cause fix (stamp lesson metadata from the batch manifest, never trust model JSON output for known fields) and a mandatory sort-by-(topicId, lessonIndex) before writing JSON.

---

## Change Log

| Date | What | Why |
|------|------|-----|
| 2026-04-14 | Chunk 6.1 — repo setup, auth middleware, Dockerfile, docker-compose, Node env, 8 tests | Phase 6 kick-off |
| 2026-04-14 | Chunk 6.2 — models/schemas.py, clients/base.py, ollama_client.py, vertex_client.py, factory, 14 tests | Model abstraction layer |
| 2026-04-14 | Chunk 6.3 — prompts/lesson_prompts.py, prompts/quiz_prompts.py, routers/generate.py, 8 tests | Generation endpoints |
| 2026-04-14 | Chunk 6.4 — routers/coaching.py, tests/test_coaching.py, 6 tests | Coaching endpoint |
| 2026-04-14 | Chunk 6.5 — Node integration: Zod schemas in shared, AIServiceClient (retry+circuit breaker), LessonGenerationService (fallback chain), requirePlan middleware, CoachingController, 21 tests | Node→Python HTTP layer |
| 2026-04-17 | Chunk 7.4.3 — `build_lesson_prompt()` learning style variant selection: `visual-concise` (250w, bullets, 2min), `detailed-narrative` (500w, prose, 5min), `reinforcement` (recap/spaced-rep, 3min), `general` fallback (400w, 3min); `_SCHEMA_TEMPLATE` with `{content_hint}` / `{minutes}` substitution; 20 new tests in `tests/test_lesson_prompts.py` | Phase 7 personalization — Python prompt variants |
