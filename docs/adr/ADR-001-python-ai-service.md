# ADR-001 — Python AI Microservice

**Ticket Reference:** Phase 6 (AI Lesson Generation) / Phase 7 (AI Personalization)
**Status:** accepted
**Date:** 2026-04-14

---

## Context

The original Phase 6 plan embedded all AI tooling directly in the Node/Express backend (`packages/api`): an OllamaClient, OpenAIClient, LLMClient, and ModelRouter, all written in TypeScript. This approach has three structural problems:

1. **Wrong ecosystem.** The Python AI/ML ecosystem (LangChain, Pydantic, Vertex AI SDKs, Hugging Face) is far richer and better maintained than its Node equivalents. Building in TypeScript creates friction for every AI library, model, or technique added in future.
2. **Tight coupling.** Embedding model calls in the Node service means any model change, prompt tweak, or provider swap requires a Node deployment. Separation allows the AI layer to evolve independently.
3. **Testability.** Prompt engineering and model output parsing are best tested in Python with Python tooling.

The new design extracts all AI interactions into a dedicated Python microservice. Node remains the system of record; Python is a best-effort AI capability layer.

---

## Decision

**All AI interactions (lesson generation, quiz generation, AI coaching) are handled by a dedicated Python microservice.** The Node backend calls this service over HTTP and validates every response with Zod before storing or serving it.

### Service Location
- Repo: `/Users/guypowell/Documents/Projects/learning-ai`
- This is a separate repository from the monorepo.

### Stack
| Concern | Choice | Rationale |
|---------|--------|-----------|
| Framework | FastAPI | Async, Pydantic-native, OpenAPI spec auto-generated |
| Server (dev) | uvicorn | Single-worker, hot-reload |
| Server (prod) | uvicorn + gunicorn | Gunicorn manages workers; uvicorn is the ASGI worker class |
| Internal validation | Pydantic v2 | Shape validation inside Python before returning to Node |
| Package management | uv | Fast, lockfile-based |

### Model Strategy

| Environment | Provider | Models |
|-------------|----------|--------|
| Local (dev) | Ollama | `llama3:8b` (default), `mistral:7b` (available) |
| Production | Google Vertex AI | Online model (configured via env) |

The Python service selects the provider based on `AI_PROVIDER` env var (`ollama` or `vertex`). Node never specifies a model — it passes tier and context; Python routes accordingly.

### Inter-Service Authentication
Node authenticates to Python using a pre-shared `X-Internal-API-Key` header. The key is stored in env vars on both sides (`AI_SERVICE_API_KEY`). This is sufficient for MVP (services are not publicly reachable). Pre-mTLS for production hardening is a Phase 11 concern.

### Response Strategy: Batch (not streaming)
All responses are returned as complete JSON payloads, not streamed tokens. Rationale:
- Zod can only validate a complete, parseable JSON object — streaming breaks this invariant
- Retry and circuit breaker logic is simpler on atomic responses
- SSE/WebSocket infrastructure adds complexity for marginal UX benefit (a loading spinner is acceptable for MVP lesson generation)
- Streaming can be added in a future phase once the batch contract is stable

### Reliability Patterns (Node side)

**Timeout**: 30 seconds per AI service call, enforced via `AbortController`. LLM generation can take 5–25 seconds; 30s gives headroom without blocking forever.

**Retry**: 3 attempts, exponential backoff with jitter: ~150ms → ~300ms → ~600ms. Only retry on network errors and 5xx responses. Do not retry on 4xx (bad request) or Zod validation failure (retrying a malformed shape will not fix it).

**Circuit breaker**: Use `opossum` (Node.js circuit breaker library). Configuration:
- Opens after 5 consecutive failures within 10 seconds
- Half-open probe after 30-second cool-down
- Closes on first successful probe response
- While open: skip AI service call, go directly to fallback

**Fallback hierarchy** (Node executes when AI service is unavailable or Zod validation fails):
1. Serve the most recent DB-cached generated lesson for this user + skill + day
2. If no cache: serve static seeded lesson content
3. If no static content: return 503 with user-facing message ("AI generation unavailable, try again shortly")
4. Log all fallback events for monitoring

### Zod Contract Location
Zod schemas that validate Python responses live in `packages/shared/src/types/ai.ts` and are exported from `@learning/shared`. This ensures the contract is a single source of truth accessible to both the Node API and (if needed) the web/mobile clients.

---

## Architectural Constraints for Dev Agent

These rules are **non-negotiable** when implementing Phase 6:

1. **No AI code in Node.** Node has no model clients, no prompt templates, no LLM calls. Only an HTTP client to the Python service.
2. **Zod validates all Python responses.** Parse with `schema.safeParse()`. On failure: log the raw response, trigger fallback. Never pass unvalidated model output to the DB or client.
3. **Python is best-effort.** All business logic (lesson count, streak, subscription enforcement) stays in Node services. Python only generates text.
4. **Internal API key on every request.** The Node HTTP client must always attach `X-Internal-API-Key`. Python must reject requests without a valid key with `401`.
5. **Batch only.** No streaming endpoints in Phase 6. Any streaming work requires a new ADR.
6. **Fallback must exist before the AI path goes live.** Implement and test the fallback (DB cache → static) before wiring up the Python service call. This ensures a safe rollout.
7. **Pydantic inside Python.** All Python endpoints must validate their own output with Pydantic before returning. This is defence-in-depth — Pydantic catches shape errors before Node/Zod sees them.
8. **Model env switching only.** No runtime model selection by users or Node. Provider is set via `AI_PROVIDER` env var at service startup.
9. **Docker for local.** Python service runs as a Docker container in `docker-compose.yml` alongside Postgres and Redis. Ollama also runs in Docker with `llama3:8b` and `mistral:7b` pre-pulled.
10. **Render for prod.** Python service deploys as a separate Render web service. Use `gunicorn -k uvicorn.workers.UvicornWorker` as the start command.

---

## Consequences

**Benefits**:
- Python AI ecosystem fully accessible (Vertex AI SDK, LangChain, future fine-tuning tooling)
- AI layer evolves independently of Node — prompt changes need no Node deployment
- Clear trust boundary: Node enforces contracts, Python generates content
- Easier to swap models or providers (env var change, no code change in Node)

**Trade-offs**:
- Additional service to run locally, deploy, and monitor
- HTTP call adds latency (~5–50ms overhead, negligible vs LLM generation time)
- Two codebases to maintain (Node monorepo + Python AI repo)
- More complex local dev setup (Docker Compose manages this)

---

## Alternatives Considered

**Keep AI in Node (TypeScript).** Rejected: TypeScript AI library ecosystem is immature, Vertex AI Node SDK lags Python SDK, prompt engineering tooling is Python-native.

**Streaming responses.** Rejected for Phase 6: breaks Zod validation invariant, adds SSE infrastructure, marginal UX benefit. Revisit post-MVP.

**GraphQL/gRPC between Node and Python.** Rejected: REST HTTP is simpler, sufficient for this call volume, and consistent with existing API patterns. gRPC is a future option if performance becomes a concern.

**Single repo (Python in `packages/ai`).** Rejected: Python and Node toolchains conflict in a pnpm workspace. Separate repo keeps dependency graphs clean and allows independent CI.
