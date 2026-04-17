# Phase 6: AI Lesson Generation

**Status**: 🔄 IN PROGRESS — Chunk 6.5 complete, Chunk 6.6 next

**Architecture reference**: [ADR-001 — Python AI Microservice](../adr/ADR-001-python-ai-service.md) — read this before starting any chunk.

**Goal**: Build a Python AI microservice that handles all LLM interactions (lesson generation, quiz generation, AI coaching). The Node backend calls this service over HTTP and validates every response with Zod. Node is the source of truth; Python is best-effort.

**Repo**: `/Users/guypowell/Documents/Projects/learning-ai` (separate from Node monorepo)

---

## Architecture Overview

```
Node API (packages/api)
  └── AIServiceClient.ts          # HTTP client with retry + circuit breaker
        └── POST http://ai-service/generate/lesson
        └── POST http://ai-service/generate/quiz
        └── POST http://ai-service/coaching/message

Python AI Service (learning-ai/)
  ├── main.py                     # FastAPI app
  ├── routers/
  │   ├── generate.py             # /generate/lesson, /generate/quiz
  │   └── coaching.py             # /coaching/message
  ├── clients/
  │   ├── base.py                 # Abstract BaseModelClient
  │   ├── ollama_client.py        # Ollama (local dev)
  │   └── vertex_client.py        # Google Vertex AI (prod)
  ├── models/
  │   └── schemas.py              # Pydantic request/response models
  ├── prompts/
  │   ├── lesson_prompts.py       # Lesson generation prompt templates
  │   └── quiz_prompts.py         # Quiz generation prompt templates
  └── middleware/
      └── auth.py                 # X-Internal-API-Key validation

packages/shared/src/types/ai.ts   # Zod schemas (Node validates Python output against these)
```

---

## Chunk 6.1: Python AI Service — Foundation ✅ COMPLETE

### Implementation Summary (2026-04-14)

**All 6.1 deliverables shipped. 8 tests passing.**

#### Files created in `learning-ai/` (new repo, previously empty):

| File | Purpose |
|------|---------|
| `pyproject.toml` | uv project config — FastAPI, uvicorn, gunicorn, pydantic, httpx, google-cloud-aiplatform, python-dotenv; dev extras: pytest, pytest-asyncio |
| `uv.lock` | Pinned lockfile (61 packages) for reproducible Docker builds |
| `main.py` | FastAPI app entry point — loads .env, registers `APIKeyMiddleware`, mounts `/generate` and `/coaching` routers, exposes `GET /health` |
| `middleware/auth.py` | `APIKeyMiddleware` — rejects every request except `/health` with `401 {"detail": "Unauthorized"}` if `X-Internal-API-Key` header doesn't match `AI_SERVICE_API_KEY` env var; also rejects if env var is empty |
| `routers/generate.py` | Stub router — populated in Chunk 6.3 |
| `routers/coaching.py` | Stub router — populated in Chunk 6.4 |
| `.env.example` | Template env file — all vars documented, safe to commit |
| `.env` | Local dev env — `AI_PROVIDER=ollama`, `AI_SERVICE_API_KEY=dev-secret-change-me`, Ollama URLs/models, empty Vertex vars. **Not committed.** |
| `.gitignore` | Excludes `.env`, `.venv/`, `__pycache__/`, `.pytest_cache/` |
| `Dockerfile` | `python:3.11-slim` base, installs uv, runs `uv sync --frozen --no-dev`, starts with `gunicorn main:app -k uvicorn.workers.UvicornWorker -w 2` |
| `tests/test_health_and_auth.py` | 8 unit tests covering health + auth middleware (see below) |
| `{routers,clients,models,prompts,middleware,tests}/__init__.py` | Package markers |

#### Key decisions / gotchas:
- `uv init --no-workspace` was used (not `uv init` inside a workspace) — this is a standalone service repo
- Added `[tool.uv] package = false` to `pyproject.toml` — this is a service, not a distributable library, so uv must not try to build/install it as a package
- `load_dotenv()` is called **before** FastAPI imports in `main.py` so env vars are available when middleware initialises
- Docker CMD uses `uv run gunicorn ...` (not bare `gunicorn`) so it uses the venv managed by uv inside the container
- Starlette's `BaseHTTPMiddleware` is used — this is the standard FastAPI middleware pattern; response is a `JSONResponse` so it is serialisable

#### Docker (changes to `learning/docker-compose.yml`):
- `ollama` service added: `ollama/ollama:latest`, port `11434:11434`, volume `ollama_data`
- `ai-service` service added: builds from `learning-ai/` Dockerfile, port `8000:8000`, reads `learning-ai/.env` via `env_file`, overrides `AI_PROVIDER`/`OLLAMA_*` as Docker env vars, `depends_on: ollama`
- `ollama_data` volume added to `volumes:` block
- Commented-out placeholder ollama block removed

**Model pull on first run** (run once after `docker-compose up`):
```bash
docker exec learning-ollama ollama pull llama3:8b
docker exec learning-ollama ollama pull mistral:7b
```

#### Node env (changes to `learning/packages/api/`):
- `.env.local` — replaced `OLLAMA_BASE_URL` line with `AI_SERVICE_URL=http://localhost:8000`, `AI_SERVICE_API_KEY=dev-secret-change-me`, `AI_SERVICE_TIMEOUT_MS=30000`
- `.env.example` — same vars added with placeholder value for `AI_SERVICE_API_KEY`

#### Tests (8 passing):
| Test | What it verifies |
|------|-----------------|
| `test_health_returns_200` | `/health` returns HTTP 200 |
| `test_health_returns_ok_status` | `/health` body is `{"status": "ok"}` |
| `test_health_does_not_require_api_key` | `/health` with no key → still 200 |
| `test_missing_api_key_returns_401` | No `X-Internal-API-Key` header → 401 |
| `test_wrong_api_key_returns_401` | Wrong key value → 401 |
| `test_valid_api_key_passes_auth` | Correct key → not 401 |
| `test_empty_api_key_env_var_returns_401` | Empty env var + any header → 401 |
| `test_401_response_body` | 401 body is `{"detail": "Unauthorized"}` |

---

## Chunk 6.1: Python AI Service — Foundation (original spec)

**Deliverable**: Python service running locally in Docker, health check passing, auth middleware working.

### 6.1.1 — Repo Setup

**Location**: `/Users/guypowell/Documents/Projects/learning-ai`

**Package manager**: `uv` (fast, lockfile-based). Init with `uv init`.

**Dependencies** (`pyproject.toml`):
```toml
[project]
name = "learning-ai"
requires-python = ">=3.11"
dependencies = [
  "fastapi>=0.115",
  "uvicorn[standard]>=0.32",
  "gunicorn>=23",
  "pydantic>=2.9",
  "httpx>=0.27",           # async HTTP client for Ollama
  "google-cloud-aiplatform>=1.70",  # Vertex AI SDK
  "python-dotenv>=1.0",
]

[project.optional-dependencies]
dev = ["pytest>=8", "pytest-asyncio>=0.24", "httpx>=0.27"]
```

**Environment variables** (`.env` for local, Render env for prod):
```
AI_PROVIDER=ollama                 # or: vertex
AI_SERVICE_API_KEY=<shared-secret> # same value in Node .env
OLLAMA_BASE_URL=http://ollama:11434
OLLAMA_DEFAULT_MODEL=llama3:8b
OLLAMA_ALT_MODEL=mistral:7b
VERTEX_PROJECT=<gcp-project-id>   # prod only
VERTEX_LOCATION=us-central1        # prod only
VERTEX_MODEL=gemini-1.5-flash      # prod only — confirm model at implementation time
```

**App entry point** (`main.py`):
```python
from fastapi import FastAPI
from routers import generate, coaching
from middleware.auth import APIKeyMiddleware

app = FastAPI(title="Learning AI Service")
app.add_middleware(APIKeyMiddleware)
app.include_router(generate.router, prefix="/generate")
app.include_router(coaching.router, prefix="/coaching")

@app.get("/health")
async def health():
    return {"status": "ok"}
```

**Dev start command**: `uvicorn main:app --reload --port 8000`
**Prod start command**: `gunicorn main:app -k uvicorn.workers.UvicornWorker -w 2 -b 0.0.0.0:8000`

### 6.1.2 — Auth Middleware

**File**: `middleware/auth.py`

**Logic**: Every request (except `/health`) must include `X-Internal-API-Key` header matching `AI_SERVICE_API_KEY` env var. Reject with `401` on mismatch or absence.

### 6.1.3 — Docker Integration

**Add to `learning-project/docker-compose.yml`**:

```yaml
ai-service:
  build:
    context: /Users/guypowell/Documents/Projects/learning-ai
    dockerfile: Dockerfile
  ports:
    - "8000:8000"
  environment:
    - AI_PROVIDER=ollama
    - OLLAMA_BASE_URL=http://ollama:11434
    - OLLAMA_DEFAULT_MODEL=llama3:8b
    - OLLAMA_ALT_MODEL=mistral:7b
  env_file:
    - /Users/guypowell/Documents/Projects/learning-ai/.env
  depends_on:
    - ollama

ollama:
  image: ollama/ollama
  ports:
    - "11434:11434"
  volumes:
    - ollama_data:/root/.ollama
```

**Model pull on first run** (add to docker-compose startup notes or a setup script):
```bash
docker exec ollama ollama pull llama3:8b
docker exec ollama ollama pull mistral:7b
```

**`Dockerfile`** (in `learning-ai/`):
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY pyproject.toml uv.lock ./
RUN pip install uv && uv sync --frozen --no-dev
COPY . .
CMD ["gunicorn", "main:app", "-k", "uvicorn.workers.UvicornWorker", "-w", "2", "-b", "0.0.0.0:8000"]
```

### 6.1.4 — Node Environment

**Add to `packages/api/.env.local`**:
```
AI_SERVICE_URL=http://localhost:8000
AI_SERVICE_API_KEY=<shared-secret>
AI_SERVICE_TIMEOUT_MS=30000
```

**Tests**: Health check endpoint returns 200. Auth middleware returns 401 on missing/wrong key, 200 on valid key.

---

## Chunk 6.2: Model Abstraction Layer ✅ COMPLETE

### Implementation Summary (2026-04-14)

**All 6.2 deliverables shipped. 22 tests passing (14 new + 8 from 6.1).**

#### Files created / updated in `learning-ai/`:

| File | Purpose |
|------|---------|
| `models/schemas.py` | All Pydantic request and output models: `LessonRequest`, `QuizRequest`, `CoachingRequest`, `LessonOutput`, `QuizOutput`, `CoachingOutput` |
| `clients/base.py` | `ModelOutputError` exception + `BaseModelClient` abstract class with `generate_lesson`, `generate_quiz`, `coaching_reply` abstract methods |
| `clients/ollama_client.py` | `OllamaClient` — POST to `/api/generate` (generate) and `/api/chat` (coaching) with `format:"json"` and `stream:false`; raises `ModelOutputError` on Pydantic validation failure |
| `clients/vertex_client.py` | `VertexClient` — uses `vertexai.GenerativeModel.generate_content_async()` with `GenerationConfig(response_mime_type="application/json")`; raises `ModelOutputError` on Pydantic validation failure |
| `clients/__init__.py` | `get_model_client()` factory — reads `AI_PROVIDER` env var; lazy-imports client class to avoid SDK init at startup |
| `tests/test_clients.py` | 14 unit tests (see below) |

#### Key decisions / patterns:
- `ModelOutputError` lives in `clients/base.py` — imported by routers in Chunk 6.3 to map to 502
- `OllamaClient` uses `/api/generate` for lessons/quiz (prompt in, JSON string out via `response` field), `/api/chat` for coaching (messages in, JSON string out via `message.content` field)
- `VertexClient.coaching_reply` flattens the messages list into a `"Role: content\n..."` string before calling the generate endpoint (Vertex does not natively support chat history in the same way as Ollama)
- Lazy imports in `get_model_client()` — `VertexClient` (which imports `vertexai`) is only imported when `AI_PROVIDER=vertex`; avoids Google SDK overhead in local dev
- Both clients use `model_validate_json(raw)` (Pydantic v2) — catches both JSON parse errors and schema validation errors in one call
- All model config read from env vars in `__init__` — testable via `monkeypatch.setenv`

#### Tests (14 new, all passing):

| Class | Test | What it verifies |
|-------|------|-----------------|
| `TestOllamaClient` | `test_generate_lesson_returns_lesson_output` | Success path — returns typed `LessonOutput` |
| `TestOllamaClient` | `test_generate_lesson_sends_correct_payload` | Request to Ollama has correct model, format, stream fields |
| `TestOllamaClient` | `test_generate_quiz_returns_quiz_output` | Success path — returns typed `QuizOutput` |
| `TestOllamaClient` | `test_coaching_reply_returns_coaching_output` | Success path — returns typed `CoachingOutput` |
| `TestOllamaClient` | `test_generate_lesson_raises_model_output_error_on_schema_mismatch` | Incomplete JSON → `ModelOutputError` |
| `TestOllamaClient` | `test_generate_quiz_raises_model_output_error_on_invalid_json` | Non-JSON string → `ModelOutputError` |
| `TestOllamaClient` | `test_coaching_reply_raises_model_output_error_on_schema_mismatch` | Missing `message` field → `ModelOutputError` |
| `TestVertexClient` | `test_generate_lesson_returns_lesson_output` | Success path — mocked `generate_content_async` |
| `TestVertexClient` | `test_generate_quiz_returns_quiz_output` | Success path |
| `TestVertexClient` | `test_coaching_reply_returns_coaching_output` | Success path |
| `TestVertexClient` | `test_generate_lesson_raises_model_output_error_on_schema_mismatch` | Invalid output → `ModelOutputError` |
| `TestGetModelClient` | `test_returns_ollama_client_by_default` | `AI_PROVIDER=ollama` → `OllamaClient` |
| `TestGetModelClient` | `test_returns_ollama_client_when_provider_not_set` | Env var absent → `OllamaClient` |
| `TestGetModelClient` | `test_returns_vertex_client_for_vertex_provider` | `AI_PROVIDER=vertex` → `VertexClient` |

---

## Chunk 6.2: Model Abstraction Layer (original spec)

**Deliverable**: Python service can call Ollama locally and Vertex AI in prod, switching via env var. Structured JSON output enforced by prompt + Pydantic validation.

### 6.2.1 — Abstract Base Client

**File**: `clients/base.py`

```python
from abc import ABC, abstractmethod
from models.schemas import LessonOutput, QuizOutput, CoachingOutput

class BaseModelClient(ABC):
    @abstractmethod
    async def generate_lesson(self, prompt: str) -> LessonOutput: ...

    @abstractmethod
    async def generate_quiz(self, prompt: str) -> QuizOutput: ...

    @abstractmethod
    async def coaching_reply(self, messages: list[dict]) -> CoachingOutput: ...
```

### 6.2.2 — Ollama Client

**File**: `clients/ollama_client.py`

**Logic**:
- POST to `{OLLAMA_BASE_URL}/api/generate` with `format: "json"` to force structured output
- Default model: `OLLAMA_DEFAULT_MODEL` (llama3:8b)
- Parse response JSON → validate with Pydantic model
- Raise `ModelOutputError` if Pydantic validation fails (Node will handle fallback)

### 6.2.3 — Vertex AI Client

**File**: `clients/vertex_client.py`

**Logic**:
- Use `google-cloud-aiplatform` SDK with `GenerativeModel`
- Configure with `VERTEX_PROJECT`, `VERTEX_LOCATION`, `VERTEX_MODEL`
- Use `response_schema` parameter to enforce JSON structure (Vertex supports controlled generation)
- Parse → validate with Pydantic

### 6.2.4 — Client Factory

**File**: `clients/__init__.py`

```python
def get_model_client() -> BaseModelClient:
    provider = os.getenv("AI_PROVIDER", "ollama")
    if provider == "vertex":
        return VertexClient()
    return OllamaClient()
```

**Tests**: Unit test each client with mocked HTTP/SDK responses. Test Pydantic validation failure raises `ModelOutputError`.

---

## Chunk 6.3: Generation Endpoints ✅ COMPLETE

### Implementation Summary (2026-04-14)

**All 6.3 deliverables shipped. 30 tests passing (8 new + 22 from 6.1–6.2).**

#### Files created / updated in `learning-ai/`:

| File | Purpose |
|------|---------|
| `prompts/lesson_prompts.py` | `build_lesson_prompt(request: LessonRequest) -> str` — injects topic, skill level, user context (goal/learning_style/completed_lessons) into a structured prompt; includes exact JSON schema; per-level instructions in `_SKILL_LEVEL_INSTRUCTIONS` dict |
| `prompts/quiz_prompts.py` | `build_quiz_prompt(request: QuizRequest) -> str` — injects lesson content and skill level into MCQ prompt; includes exact JSON schema |
| `routers/generate.py` | `POST /lesson` → `LessonRequest` → `LessonOutput`; `POST /quiz` → `QuizRequest` → `QuizOutput`; both call `get_model_client()` then the appropriate client method; catch `ModelOutputError` → HTTP 502 |
| `tests/test_generate.py` | 8 unit tests across `TestGenerateLessonEndpoint` and `TestGenerateQuizEndpoint` (see below) |

#### Key decisions / patterns:
- Prompt builders are pure functions in `prompts/` — easy to iterate without touching router logic
- Each prompt ends with a strict instruction: "Return ONLY a JSON object... no markdown, no explanation, no text before or after" — reduces hallucinated wrappers
- Router patches `routers.generate.get_model_client` in tests (point-of-use patching, same as client tests)
- `LessonOutput(**VALID_LESSON)` construction in tests verifies Pydantic v2 coerces nested `quiz` dict to `QuizOutput` at fixture time
- `build_lesson_prompt` falls back to "beginner" instructions for unrecognised skill levels via `.get(level, default)`

#### Tests (8 new, all passing):

| Class | Test | What it verifies |
|-------|------|-----------------|
| `TestGenerateLessonEndpoint` | `test_returns_lesson_output` | 200 with typed `LessonOutput` JSON, correct fields |
| `TestGenerateLessonEndpoint` | `test_passes_topic_and_level_in_prompt` | Prompt passed to client contains topic and skill level |
| `TestGenerateLessonEndpoint` | `test_returns_502_on_model_output_error` | `ModelOutputError` → 502 with error detail |
| `TestGenerateLessonEndpoint` | `test_returns_401_without_api_key` | No API key → 401 |
| `TestGenerateQuizEndpoint` | `test_returns_quiz_output` | 200 with typed `QuizOutput` JSON, 4 options |
| `TestGenerateQuizEndpoint` | `test_passes_lesson_content_in_prompt` | Lesson content appears in prompt |
| `TestGenerateQuizEndpoint` | `test_returns_502_on_model_output_error` | `ModelOutputError` → 502 with error detail |
| `TestGenerateQuizEndpoint` | `test_returns_401_without_api_key` | No API key → 401 |

---

## Chunk 6.3: Generation Endpoints (original spec)

**Deliverable**: `/generate/lesson` and `/generate/quiz` endpoints return validated structured JSON.

### 6.3.1 — Pydantic Schemas

**File**: `models/schemas.py`

```python
from pydantic import BaseModel

class LessonRequest(BaseModel):
    skill_id: str
    skill_level: str  # beginner | intermediate | advanced
    topic: str
    user_context: dict  # goal, learning_style, completed_lessons count
    tier: str           # free | starter | pro | premium (for model routing only)

class QuizRequest(BaseModel):
    lesson_content: str
    skill_level: str
    tier: str

class CoachingRequest(BaseModel):
    messages: list[dict]     # [{role: user|assistant, content: str}]
    lesson_context: str
    user_context: dict
    tier: str

class KeyTakeaway(BaseModel):
    point: str

class QuizOutput(BaseModel):
    question: str
    options: list[str]       # exactly 4 items
    correct_answer: str      # must be one of options
    explanation: str

class LessonOutput(BaseModel):
    title: str
    content: str
    estimated_minutes: int
    key_takeaways: list[str]
    quiz: QuizOutput

class CoachingOutput(BaseModel):
    message: str
    suggestions: list[str] = []
```

### 6.3.2 — Prompt Templates

**File**: `prompts/lesson_prompts.py`

Prompt templates for each skill level (beginner/intermediate/advanced). Each prompt:
- Instructs the model to return **only valid JSON** matching the `LessonOutput` schema
- Includes the topic, user context, and skill level
- Specifies the 3-minute target length
- Lists the exact JSON fields expected

**File**: `prompts/quiz_prompts.py`

Prompts that take lesson content and generate an MCQ matching `QuizOutput` schema.

### 6.3.3 — Generate Router

**File**: `routers/generate.py`

```python
@router.post("/lesson", response_model=LessonOutput)
async def generate_lesson(request: LessonRequest):
    client = get_model_client()
    prompt = build_lesson_prompt(request)
    try:
        output = await client.generate_lesson(prompt)
    except ModelOutputError as e:
        raise HTTPException(status_code=502, detail=str(e))
    return output

@router.post("/quiz", response_model=QuizOutput)
async def generate_quiz(request: QuizRequest):
    client = get_model_client()
    prompt = build_quiz_prompt(request)
    try:
        output = await client.generate_quiz(prompt)
    except ModelOutputError as e:
        raise HTTPException(status_code=502, detail=str(e))
    return output
```

**502 on ModelOutputError**: This tells Node the service is up but the model output was unusable — Node triggers the fallback.

**Tests**: Unit test each endpoint with mocked client. Test 502 on ModelOutputError. Test 401 on missing API key.

---

## Chunk 6.4: AI Coaching Endpoint ✅ COMPLETE

### Implementation Summary (2026-04-14)

**All 6.4 deliverables shipped. 36 tests passing (6 new + 30 from 6.1–6.3).**

#### Files created / updated in `learning-ai/`:

| File | Purpose |
|------|---------|
| `routers/coaching.py` | `POST /message` → `CoachingRequest` → `CoachingOutput`; prepends system message with `lesson_context` + `user_context`; calls `client.coaching_reply(messages)`; catches `ModelOutputError` → HTTP 502 |
| `tests/test_coaching.py` | 6 unit tests across `TestCoachingMessageEndpoint` (see below) |

#### Key decisions / patterns:
- `_build_messages(request)` is a module-level helper that prepends a `{"role": "system", ...}` message before the conversation history — this keeps the router function clean and the helper easy to test indirectly via captured `call_args`
- System message template uses `.format()` rather than an f-string so the JSON example braces (`{{`, `}}`) can be escaped once at definition time
- `user_context.get("skill_level", "beginner")` — coaching system prompt includes skill level so the tone and complexity of coaching replies adapts to the learner
- Tests patch `routers.coaching.get_model_client` (point-of-use), same pattern as `routers.generate`

#### Tests (6 new, all passing):

| Class | Test | What it verifies |
|-------|------|-----------------|
| `TestCoachingMessageEndpoint` | `test_returns_coaching_output` | 200 with correct `message` and `suggestions` fields |
| `TestCoachingMessageEndpoint` | `test_passes_lesson_context_in_system_message` | System message (first in list) contains `lesson_context` text |
| `TestCoachingMessageEndpoint` | `test_passes_conversation_history` | User message from request is second in messages list; list length is 2 |
| `TestCoachingMessageEndpoint` | `test_injects_user_goal_in_system_message` | `user_context.goal` appears in system message content |
| `TestCoachingMessageEndpoint` | `test_returns_502_on_model_output_error` | `ModelOutputError` → 502 with error detail |
| `TestCoachingMessageEndpoint` | `test_returns_401_without_api_key` | No API key → 401 |

---

## Chunk 6.4: AI Coaching Endpoint (original spec)

**Deliverable**: `/coaching/message` endpoint for Pro/Premium conversational AI coaching.

**Note**: Node enforces tier access (Pro/Premium only) before calling this endpoint. Python does not enforce tier gating — that is Node's responsibility.

### 6.4.1 — Coaching Router

**File**: `routers/coaching.py`

```python
@router.post("/message", response_model=CoachingOutput)
async def coaching_message(request: CoachingRequest):
    client = get_model_client()
    try:
        output = await client.coaching_reply(request.messages)
    except ModelOutputError as e:
        raise HTTPException(status_code=502, detail=str(e))
    return output
```

**Context**: Pass lesson content + conversation history in `messages`. Python assembles a system prompt with the user's lesson context and returns a coaching response.

**Tests**: Unit test with mocked client. Test conversation history is passed correctly. Test 502 on malformed model output.

---

## Chunk 6.5: Node Integration ✅ COMPLETE

### Implementation Summary (2026-04-14)

**All 6.5 deliverables shipped. 63 tests passing (21 new + 42 from 6.1–6.4).**

#### Files created / updated in `packages/`:

| File | Purpose |
|------|---------|
| `packages/shared/src/types/ai.ts` | Zod schemas (`LessonOutputSchema`, `QuizOutputSchema`, `CoachingOutputSchema`) + request types (`LessonRequest`, `CoachingRequest`) |
| `packages/shared/src/index.ts` | Added `export * from './types/ai'` |
| `packages/api/src/services/AIServiceClient.ts` | HTTP client with AbortController timeout, 3-attempt exponential backoff retry (150ms/300ms/600ms), opossum circuit breaker (50% threshold, 30s reset, 5 req volume), Zod validation; typed `AIServiceError` with `kind` field |
| `packages/api/src/services/LessonGenerationService.ts` | Fallback chain: AI → DB cache stub → static `Lesson` → 503; maps Prisma `Lesson` to `LessonOutput` shape; logs every fallback activation |
| `packages/api/src/middleware/plan.ts` | `requirePlan(minimumPlan)` middleware — queries subscription, checks plan hierarchy (free=0 < starter=1 < pro=2 < premium=3), attaches `req.planName`, returns 403 on insufficient plan |
| `packages/api/src/controllers/CoachingController.ts` | `POST /api/coaching/message` — validates body, passes tier from `req.planName`, maps `AIServiceError` → 503 |
| `packages/api/src/routes/coaching.ts` | Express router: `authMiddleware` + `requirePlan('pro')` + `POST /message` |
| `packages/api/src/index.ts` | Registered `coachingRoutes` at `/api/coaching` |

#### Key decisions / patterns:
- `AIServiceError` carries a `kind` field (`timeout` / `circuit_open` / `validation` / `http_error` / `network_error`) — lets upstream code distinguish transient from permanent failures without parsing error messages
- Retry logic: retries on 5xx and network errors; does NOT retry on 4xx or `AIServiceError` (already typed); 3 attempts max
- Circuit breaker wraps `_fetchWithRetry` (the full retry stack) — opossum counts a failure when all retries are exhausted; `timeout: false` disables opossum's own timeout so AbortController is the sole timeout mechanism
- `LessonGenerationService._getCachedLesson` is a stub returning `null` — the `GeneratedLesson` model is added in Chunk 6.6; the fallback chain structure is complete
- `requirePlan` uses `subscriptionService.getUserSubscription()` which already falls back to free plan for users with no subscription record
- `zod` added as runtime dep to `packages/shared`; `opossum` + `@types/opossum` added to `packages/api`

#### Tests (21 new, all passing):

| File | Tests | What they cover |
|------|-------|----------------|
| `services/__tests__/AIServiceClient.test.ts` | 8 | Success path, retry on 5xx, no retry on 4xx, 3-attempt exhaustion, Zod validation failure, circuit-open rejection; coachingMessage success + validation failure |
| `services/__tests__/LessonGenerationService.test.ts` | 4 | AI success, static fallback on AIServiceError, 503 when no fallback, console.warn log on fallback |
| `middleware/__tests__/plan.test.ts` | 5 | Pass-through on matching plan, pass-through on higher plan, 403 on lower plan, 401 with no user, next(err) on DB error |
| `controllers/__tests__/CoachingController.test.ts` | 6 | 200 success, tier passed to AI service, 503 on AIServiceError, 400 missing messages, 400 missing lesson_context, 401 no user |

---

## Chunk 6.5: Node Integration (original spec)

**Deliverable**: Node API calls Python service with retry, circuit breaker, timeout, and fallback. All responses Zod-validated.

### 6.5.1 — Zod Schemas

**File**: `packages/shared/src/types/ai.ts`

```typescript
import { z } from 'zod'

export const QuizOutputSchema = z.object({
  question: z.string().min(1),
  options: z.array(z.string()).length(4),
  correct_answer: z.string().min(1),
  explanation: z.string().min(1),
})

export const LessonOutputSchema = z.object({
  title: z.string().min(1),
  content: z.string().min(1),
  estimated_minutes: z.number().int().positive(),
  key_takeaways: z.array(z.string()).min(1),
  quiz: QuizOutputSchema,
})

export const CoachingOutputSchema = z.object({
  message: z.string().min(1),
  suggestions: z.array(z.string()).default([]),
})

export type LessonOutput = z.infer<typeof LessonOutputSchema>
export type QuizOutput = z.infer<typeof QuizOutputSchema>
export type CoachingOutput = z.infer<typeof CoachingOutputSchema>
```

Export these from `packages/shared/src/index.ts`.

### 6.5.2 — AI Service HTTP Client

**File**: `packages/api/src/services/AIServiceClient.ts`

**Dependencies**: `opossum` (circuit breaker — `npm install opossum @types/opossum`)

**Structure**:
```typescript
// Timeout: AbortController with AI_SERVICE_TIMEOUT_MS (default 30000)
// Retry: 3 attempts, exponential backoff with jitter (~150ms, ~300ms, ~600ms)
//        Retry on: network error, 5xx. Do NOT retry on: 4xx, Zod validation failure.
// Circuit breaker: opossum — opens after 5 failures in 10s, half-open after 30s
// All responses: parse with schema.safeParse() — on failure, log raw response + throw AIServiceError

class AIServiceClient {
  async generateLesson(request: LessonRequest): Promise<LessonOutput>
  async generateQuiz(request: QuizRequest): Promise<QuizOutput>
  async coachingMessage(request: CoachingRequest): Promise<CoachingOutput>
}
```

**Error handling**: Throw a typed `AIServiceError` on timeout, circuit open, or Zod failure. `LessonGenerationService` catches this and executes the fallback chain.

### 6.5.3 — Updated LessonGenerationService

**File**: `packages/api/src/services/LessonGenerationService.ts`

**Fallback chain** (in order):
1. Call `AIServiceClient.generateLesson()` — if success, cache result in DB + Redis, return
2. On `AIServiceError`: check DB for cached `GeneratedLesson` for this user+skill+day — if found, return it
3. If no cache: fetch a seeded static lesson from the `Lesson` table (first lesson for skill path)
4. If no static: return `503` with message `"AI generation temporarily unavailable"`
5. Log every fallback activation with reason (timeout / circuit open / Zod failure / no cache)

**Pass to Python service**: `{ skill_id, skill_level, topic, user_context: { goal, learningStyle, completedLessons }, tier }`

### 6.5.4 — Coaching Integration

**File**: `packages/api/src/controllers/CoachingController.ts` (new)
**Route**: `POST /api/coaching/message` (protected, requires Pro/Premium tier — enforced by `requirePlan('pro')` middleware)

Call `AIServiceClient.coachingMessage()`. On `AIServiceError`: return `503`. Never fall back to static content for coaching — it is inherently dynamic.

**Tests**:
- Unit test `AIServiceClient`: mock HTTP, test retry logic, test circuit breaker state transitions, test Zod validation failure path
- Unit test `LessonGenerationService`: test each step of the fallback chain
- Unit test `CoachingController`: test tier enforcement, test 503 on AI service error

---

## Chunk 6.6: Schema Updates ✅ COMPLETE

### Implementation Summary (2026-04-17)

**All 6.6 deliverables shipped. 100 backend tests passing.**

#### Schema changes (`prisma/schema.prisma`):
- `Lesson`: added `isAiGenerated Boolean @default(false)` and `generatedModel String?`
- `UserProgress`: added `completionTime Int?`, `revisitCount Int @default(0)`, `rating Int?` (Phase 7 engagement fields added here since 6.6 migration ran first)
- `GeneratedLesson` model: `(userId, skillId)` unique cache key, `generatedContent Json` stores `LessonOutput`, `coachingMessage String?` (Phase 7, ADR-002), cascade on user delete
- `UserSkillRating` model: Elo ratings per user+skill, cascade on user delete
- `Skill`: added `userSkillRatings UserSkillRating[]` backrelation
- `User`: added `generatedLessons GeneratedLesson[]` and `skillRatings UserSkillRating[]` backrelations

#### `LessonGenerationService` updates:
- `_getCachedLesson(userId, skillId)` — queries `GeneratedLesson` by `userId_skillId` composite unique, deserialises `generatedContent` JSON
- `_cacheLesson(userId, skillId, model, content)` — upserts `GeneratedLesson` after successful AI call; cache write failure is non-fatal (wrapped in try/catch)
- Cache write called immediately after successful AI generation before returning

#### Tests (8 tests in `LessonGenerationService.test.ts`, all passing):
| Test | What it verifies |
|------|-----------------|
| `caches generated lesson after successful AI call` | `prisma.generatedLesson.upsert` called with correct key |
| `serves cached lesson when AI fails` | Cache hit short-circuits static fallback |
| `logs cache hit when cached lesson is served` | `console.warn` contains "Serving cached lesson" |
| `does not throw when cache write fails` | Lesson still returned even if DB write errors |

**Gotcha**: `GeneratedLesson` uses `skillId` (not `lessonId`) as the cache key — matches `GenerateLessonOptions.skillId` without requiring callers to know the specific `lessonId`. The spec's `lessonId` FK approach was not used because `LessonGenerationService` doesn't receive a `lessonId` in its options.

## Chunk 6.6: Schema Updates (original spec)

**Deliverable**: DB schema supports AI-generated lesson caching and audit log.

### 6.6.1 — Prisma Schema Changes

**File**: `prisma/schema.prisma`

**Update `Lesson` model**:
```prisma
model Lesson {
  // ... existing fields ...
  isAiGenerated  Boolean  @default(false)
  generatedModel String?  // e.g. "llama3:8b", "gemini-1.5-flash" — for audit
}
```

**Add `GeneratedLesson` (cache + audit log)**:
```prisma
model GeneratedLesson {
  id          String   @id @default(cuid())
  userId      String
  lessonId    String
  model       String   // model name used for generation
  generatedAt DateTime @default(now())
  user        User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  lesson      Lesson   @relation(fields: [lessonId], references: [id])

  @@unique([userId, lessonId])
  @@index([userId])
}
```

**Remove `LessonTemplate` from schema** (if previously added — templates live in Python service code, not the DB).

### 6.6.2 — Migration

```bash
# Dev
pnpm db:push

# Before prod deployment: create named Prisma migration
npx prisma migrate dev --name add-ai-generation
```

**Tests**: Verify `GeneratedLesson` upsert works correctly (unique constraint on userId+lessonId).

---

## Environment Reference

| Variable | Node API | Python Service |
|----------|----------|----------------|
| `AI_SERVICE_URL` | `http://localhost:8000` | — |
| `AI_SERVICE_API_KEY` | `<shared-secret>` | `<shared-secret>` |
| `AI_SERVICE_TIMEOUT_MS` | `30000` | — |
| `AI_PROVIDER` | — | `ollama` (local) / `vertex` (prod) |
| `OLLAMA_BASE_URL` | — | `http://ollama:11434` |
| `OLLAMA_DEFAULT_MODEL` | — | `llama3:8b` |
| `OLLAMA_ALT_MODEL` | — | `mistral:7b` |
| `VERTEX_PROJECT` | — | `<gcp-project-id>` |
| `VERTEX_LOCATION` | — | `us-central1` |
| `VERTEX_MODEL` | — | `gemini-1.5-flash` |

---

## Deployment: Render (Production)

**Python AI service on Render**:
- New web service pointing to `learning-ai` repo
- Build command: `pip install uv && uv sync --frozen`
- Start command: `gunicorn main:app -k uvicorn.workers.UvicornWorker -w 2 -b 0.0.0.0:$PORT`
- Set all `VERTEX_*` env vars and `AI_SERVICE_API_KEY` in Render dashboard
- Set `AI_PROVIDER=vertex`
- Render private networking: Node API calls Python service via internal URL (no public internet hop)

**Node API on Render**:
- Set `AI_SERVICE_URL` to the Python service's internal Render URL
- Set `AI_SERVICE_API_KEY` to the same shared secret

---

## Next Phase

[Phase 7: AI Personalization & Adaptive Learning](./phase-7-ai-personalization.md)
