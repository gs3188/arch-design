# Platform Architecture — Technical Design Reference

> This is an architecture playbook to implement a micro services driven agentic AI business service. It is a domain-agnostic description of the system architecture.
> Intended as a design reference for engineers evaluating patterns, trade-offs, and evolution strategy.


---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [Service Topology](#2-service-topology)
3. [Communication Patterns](#3-communication-patterns)
4. [Frontend Architecture](#4-frontend-architecture)
5. [Backend Service Architecture](#5-backend-service-architecture)
6. [Data Layer](#6-data-layer)
7. [AI / LLM Orchestration Service](#7-ai--llm-orchestration-service)
8. [RAG Pipeline](#8-rag-pipeline)
9. [Background Processing & Worker Architecture](#9-background-processing--worker-architecture)
10. [Policy Driven Detection Service](#10-Policy-Driven-detection-service)
11. [Authentication & Authorization](#11-authentication--authorization)
12. [Middleware & Cross-Cutting Concerns](#12-middleware--cross-cutting-concerns)
13. [Infrastructure-as-Code](#13-infrastructure-as-code)
14. [Containerization & Orchestration](#14-containerization--orchestration)
15. [Shared Code & Cross-Service Contracts](#15-shared-code--cross-service-contracts)
16. [Security Posture](#16-security-posture)
17. [Evolution Strategy](#17-evolution-strategy)

---

## 1. System Overview

The platform is a **polyglot, service-oriented system** composed of:

- **3 frontend applications** (Next.js 14, App Router, SSR)
- **3 backend API services** (FastAPI, async Python)
- **1 background worker** (async daemon, queue-driven)
- **1 web crawler service** (FastAPI + background pipeline)
- **1 shared library** (installable Python package)
- **1 IaC module** (Terraform, cloud landing-zone pattern)

All services are containerized and orchestrated via Docker Compose for local development, with a cloud-native deployment target using hub-spoke networking and private endpoints.

```
┌──────────────────────────────────────────────────────┐
│                      BROWSERS                        │
└───────┬──────────────────┬───────────────┬───────────┘
        │                  │               │
   ┌────▼────┐       ┌────▼────┐     ┌────▼────┐
   │ App UI  │       │Console  │     │ Chat UI │
   │ (SSR)   │       │ (SSR)   │     │ (SSR)   │
   │ :3000   │       │ :3002   │     │ :3003   │
   └──┬───┬──┘       └────┬────┘     └────┬────┘
      │   │               │               │
      │   │    ┌──────────▼───────┐       │
      │   │    │  Policy DrivenSvc│       │
      │   │    │  :8020           │       │
      │   │    └──────────────────┘       │
      │   │                               │
      │   └──────────┐      ┌─────────────┘
      ▼              ▼      ▼
   ┌──────┐     ┌───────────────┐     ┌──────────┐   ┌──────────┐
   │ Core │     │ AI/LLM        │◄─── │  Worker  │   │ Crawler  │
   │ API  │     │ Orchestrator  │queue│ (daemon) │   │  :8030   │
   │:8000 │     │    :8010      │     └─────┬────┘   └─────┬────┘
   └──┬───┘     └──────┬────────┘           │              │
      │                │                    │              │
      └───────┬────────┴───────────┬────────┘──────────────┘
              │                    │
       ┌──────▼──────┐     ┌──────▼──────┐
       │ PostgreSQL  │     │ Cloud Queue │
       │ + pgvector  │     │  (async)    │
       └─────────────┘     └─────────────┘
```

---

## 2. Service Topology

| Service | Type | Runtime | Port | Role |
|---------|------|---------|------|------|
| **App UI** | Next.js SSR | Node 20 | 3000 | Primary user-facing frontend |
| **Console UI** | Next.js SSR | Node 20 | 3002 | Internal operations console |
| **Chat UI** | Next.js SSR | Node 20 | 3003 | Conversational AI interface |
| **Core API** | FastAPI | Python 3.13 | 8000 | Primary REST API, CRUD, auth, payments |
| **Policy Driven Service** | FastAPI | Python 3.12 | 8020 | NLP-based Policy Driven detection & policy-driven redaction |
| **AI Orchestrator** | FastAPI | Python 3.12 | 8010 | Multi-provider LLM routing, RAG retrieval, streaming |
| **Worker** | Async daemon | Python 3.12 | — | Queue-driven ingestion: crawl → chunk → embed → store |
| **Crawler** | FastAPI + bg tasks | Python 3.12 | 8030 | Batch web crawling, classification, optional embedding |
| **Shared Lib** | Python package | — | — | Reusable RAG primitives (chunker, embeddings, vector store) |
| **IaC** | Terraform | — | — | Cloud landing zone: networking, governance, data services |

### Design Principles

- **Single responsibility per service** — each owns its schema, migrations, and API surface.
- **No direct browser-to-backend calls** — all backend traffic proxied through SSR route handlers.
- **Shared-nothing between services** — communication only via HTTP or message queues.
- **Schema-per-service in shared DB** — logical isolation on a single PostgreSQL instance.

---

## 3. Communication Patterns

### 3.1 Synchronous HTTP (Dominant Pattern)

Frontend route handlers act as a **backend-for-frontend (BFF) proxy layer**:

```
Browser → Next.js /api/* route handler → Backend service /v1/*
```

Each frontend maintains a **typed server-side HTTP client** (one per backend it communicates with). These clients:
- Are restricted to server-side execution (enforced via `import 'server-only'` or env-var access)
- Forward auth tokens from session cookies
- Map backend errors into frontend-appropriate HTTP responses
- Provide typed method signatures matching the backend's OpenAPI contracts

**Service-to-service HTTP:** Backend services call each other via dedicated client wrappers (e.g., Core API → Policy Driven Service, Core API → AI Orchestrator).

### 3.2 Server-Sent Events (SSE Streaming)

The AI Orchestrator exposes a streaming endpoint returning `text/event-stream`. The data flow:

```
Browser → Chat UI /api/stream → AI Orchestrator /v1/chat/stream
```

The frontend client transforms the raw SSE `ReadableStream` into a typed stream, parsing `data: {json}` lines and handling `[DONE]` termination. Mid-generation cancellation is supported via `AbortSignal` propagation.

### 3.3 Async Message Queue

```
AI Orchestrator ──enqueue──► Cloud Queue ──poll──► Worker (daemon)
```

- **Producer:** AI Orchestrator enqueues ingestion jobs (configurable: queue vs. inline).
- **Consumer:** Worker polls with configurable batch size, visibility timeout, and exponential back-off.
- **Message dispatch:** Non-blocking — each message spawns an `asyncio.Task` with a global concurrency cap.
- **Dead-letter handling:** Messages exceeding a dequeue threshold are routed to a configurable dead-letter strategy (log, secondary queue, or database).
- **Graceful shutdown:** SIGTERM/SIGINT → drain in-flight tasks within a configurable timeout.

### 3.4 Communication Matrix

| From | To | Protocol | Pattern |
|------|----|----------|---------|
| Browser | Any Frontend | HTTPS | Request/Response |
| Frontend route handlers | Core API | HTTP | Sync proxy |
| Frontend route handlers | AI Orchestrator | HTTP / SSE | Sync proxy + streaming |
| Frontend route handlers | Policy Driven Service | HTTP | Sync proxy |
| AI Orchestrator | Cloud Queue | Cloud SDK | Async enqueue |
| Worker | Cloud Queue | Cloud SDK | Async poll + dispatch |
| Crawler | External Spider API | HTTP | Streaming JSON |
| Core API | External Payment Provider | HTTP (SDK) | Sync |
| Core API | External Email Service | HTTP (Graph API) | Sync |

---

## 4. Frontend Architecture

### 4.1 Shared Stack

All three frontends share:
- **Next.js 14** with App Router (React Server Components by default)
- **Tailwind CSS** for styling
- **TypeScript** with strict mode
- Standalone output mode for containerized deployment

### 4.2 BFF Proxy Pattern

Every frontend enforces server-side-only backend communication:

```typescript
// Server-side client (never imported in client components)
class BackendClient {
  private baseUrl: string; // from process.env (server-only)

  async getResource(id: string, authToken?: string): Promise<Resource> {
    const res = await fetch(`${this.baseUrl}/v1/resources/${id}`, {
      headers: authToken ? { Authorization: `Bearer ${authToken}` } : {},
    });
    if (!res.ok) throw new ApiError(res.status, await res.json());
    return res.json();
  }
}
```

Route handlers in `/api/*` act as thin adapters: extract session/auth, call backend client, forward response.

### 4.3 State Management Patterns

| Concern | Approach | Source of Truth |
|---------|----------|-----------------|
| Auth/session | NextAuth.js, JWT in httpOnly cookie | Backend (validated on each API call) |
| Shopping cart | React Context + localStorage reference ID | Backend (reconciled after mutations) |
| Chat state | `useReducer` within component tree | Local (ephemeral) |
| Catalog data | ISR / `force-cache` at fetch level | Backend (cache-invalidated) |
| Transactional data | `cache: 'no-store'` | Backend (always fresh) |

### 4.4 Frontend-Specific Concerns

| Frontend | Auth Model | Backend Target | Unique Patterns |
|----------|-----------|----------------|-----------------|
| App UI | NextAuth (OAuth + credentials) | Core API, AI Orchestrator | Cart context, protected routes via middleware, Stripe.js client |
| Console UI | NextAuth (OTP-email) | Policy Driven Service | Prisma for local user DB, document upload/download workflows |
| Chat UI | None (service auth header) | AI Orchestrator | SSE streaming, markdown rendering, source citation display |

---

## 5. Backend Service Architecture

### 5.1 Layered Architecture (Core API)

```
┌─────────────┐
│   Routers   │  ← Thin: request validation, dependency injection, response shaping
├─────────────┤
│  Services   │  ← Business logic, orchestration, cross-entity workflows
├─────────────┤
│Repositories │  ← Data access, ORM queries, no business logic
├─────────────┤
│   Models    │  ← SQLAlchemy ORM definitions
├─────────────┤
│  Schemas    │  ← Pydantic request/response models, enums, error types
└─────────────┘
```

**Router pattern (thin delegation):**

```python
@router.post("/v1/resources/{id}/items")
async def add_item(
    id: str,
    item: AddItemSchema,
    svc: ResourceService = Depends(),
):
    return await svc.add_item(id, item)
```

Routers never contain business logic. All orchestration, validation beyond schema, and side-effects live in the service layer.

### 5.2 Dependency Injection

FastAPI's `Depends()` used pervasively:

- **DB sessions:** `async_sessionmaker` → `get_db` dependency
- **Auth context:** Bearer token extraction → user resolution → role checking
- **Services:** Injected with DB session, current user, and config
- **Rate limiter:** Key function resolves user ID or falls back to IP

### 5.3 Configuration

All Python services use `pydantic-settings` with `.env` file loading:

```python
class Settings(BaseSettings):
    database_url: str
    redis_url: str = "redis://localhost:6379"
    debug: bool = False

    model_config = SettingsConfigDict(env_file=".env")
```

Singleton pattern: settings loaded once at import time, cached.

---

## 6. Data Layer

### 6.1 Database Architecture

Single PostgreSQL 16 instance with pgvector extension, logically partitioned by schema:

```
PostgreSQL Instance (:5432)
├── database: primary_db
│   ├── schema: public      ← Core API (ORM models, migrations)
│   └── extensions: uuid-ossp, pgcrypto
├── database: ai_rag_db
│   ├── schema: rag          ← AI Orchestrator + Worker (vector store)
│   ├── schema: spider        ← Crawler (separate vector namespace)
│   └── extensions: pgvector
├── database: Policy Driven_db
│   └── schema: Policy Driven           ← Policy Driven Service (job tracking, audit)
└── database: console_auth_db
    └── schema: console        ← Console UI (Prisma, user/session models)
```

### 6.2 Migration Strategy

- **Python services:** Alembic with async driver (asyncpg). Auto-migration on container startup via entrypoint scripts.
- **Console UI:** Prisma Migrate for its own auth database.
- **Vector tables:** Created by Alembic in the AI service; HNSW indexes managed declaratively.

### 6.3 Repository Pattern

The Core API uses a generic base repository providing standard CRUD:

```python
class BaseRepository(Generic[T]):
    async def get(self, id: str) -> Optional[T]: ...
    async def list(self, filters: dict, skip: int, limit: int) -> List[T]: ...
    async def create(self, obj: T) -> T: ...
    async def update(self, id: str, data: dict) -> T: ...
    async def delete(self, id: str) -> bool: ...
```

Entity-specific repositories extend this with custom queries. All DB access is async (`AsyncSession`).

### 6.4 Vector Storage

Dual-backend retrieval (selectable per request):

| Backend | Index Type | Search Method | Use Case |
|---------|-----------|---------------|----------|
| **pgvector** | HNSW (cosine) | Approximate nearest neighbor | Default, self-hosted |
| **Cloud Search** | Hybrid (BM25 + vector) | Keyword + semantic | Optional, managed service |

Vector dimensions configurable (1536 for cloud embeddings, 768 for local). Content-hash deduplication prevents redundant storage.

---

## 7. AI / LLM Orchestration Service

### 7.1 Multi-Provider Router

```
┌──────────────────────────────────────────────┐
│              Provider Router                 │
│                                              │
│  ┌──────────┐ ┌──────────┐ ┌──────────────┐  │
│  │Provider A│ │Provider B│ │Provider C    │  │
│  │(OpenAI)  │ │(Anthropic│ │(Azure OpenAI)│  │
│  └──────────┘ └──────────┘ └──────────────┘  │
│  ┌──────────────┐                            │
│  │ Local (self- │                            │
│  │  hosted)     │                            │
│  └──────────────┘                            │
└──────────────────────────────────────────────┘
```

All providers implement an abstract `LLMProvider` interface:

```python
class LLMProvider(ABC):
    @abstractmethod
    async def chat(self, messages, model, **kwargs) -> ChatResponse: ...

    @abstractmethod
    async def chat_stream(self, messages, model, **kwargs) -> AsyncIterator[str]: ...

    @abstractmethod
    async def embed(self, texts, model) -> List[List[float]]: ...
```

Providers are initialized at startup, availability checked dynamically. Default provider configurable; fallback routing possible.

### 7.2 Multi-Stage Processing Pipeline

The AI service implements a **deterministic multi-stage pipeline** for complex query processing:

```
Input
  → Abuse Gate           (block disallowed queries)
  → Intent Classifier    (categorize query type)
  → Mode Router          (select processing mode)
  → Fact Extractor       (extract structured data, generate clarifications)
  → Goal Resolver        (determine user objective from conversation context)
  → Severity Triage      (assess urgency)
  → Retrieval Router     (decide retrieval strategy)
  → Context Sanitizer    (clean retrieved context for LLM consumption)
  → Rule Synthesizer     (apply domain-specific rules)
  → Guardrail Checker    (final safety/quality gate)
  → Response Composer    (assemble structured output)
Output
```

Each stage is a standalone service class with a single responsibility, testable in isolation. The pipeline is stateless — all context passed explicitly through a `ConversationState` dataclass.

### 7.3 Retrieval-Augmented Generation (RAG)

```
Query → Embed → Vector Search → Score & Filter → Context Window Assembly → LLM
```

- **Embedding:** Cloud-hosted (1536d) or self-hosted (768d), with retry/backoff.
- **Retrieval:** Two-pass scoring — initial similarity threshold (permissive), then relevance gating (strict).
- **Context budget:** Retrieved chunks are capped at a token limit to fit within model context windows.
- **Source attribution:** Retrieved chunks carry source URL, title, and relevance scores for citation in responses.

---

## 8. RAG Pipeline

### 8.1 Shared Library Architecture

A shared Python package provides reusable RAG primitives consumed by the AI Orchestrator, Worker, and Crawler:

```
┌───────────────────────────────────────┐
│           Shared RAG Library          │
│                                       │
│  TextChunker     → token-based split  │
│  EmbeddingClient → API w/ retry       │
│  PgVectorStore   → HNSW + dedup       │
│  WebCrawler      → rate-limited HTTP  │
│  Models          → SQLAlchemy ORM     │
│  Database        → async connection   │
└───────────────────────────────────────┘
        ▲            ▲            ▲
        │            │            │
   AI Orchestrator  Worker     Crawler
```

**Critical design constraint:** Schema name and embedding dimensions are module-level constants read from environment variables at import time. All consumers must set these before importing the models module.

### 8.2 Ingestion Flow

```
Source (URL / Document / Spider Output)
  → Crawl / Parse (extract text content)
  → Classify (document type via multi-signal heuristics)
  → Chunk (token-based, configurable size/overlap)
  → Embed (batch API call with retry)
  → Store (pgvector with content-hash dedup)
  → Optional: Archive raw content to blob storage
```

### 8.3 Chunking Strategy

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| Chunk size | 1000 tokens | Balance retrieval precision vs. context |
| Overlap | 128 tokens | Preserve cross-boundary context |
| Minimum size | 150 tokens | Filter noise/fragments |
| Skip threshold | 200 chars | Discard near-empty pages |
| Tokenizer | tiktoken | Accurate token counting for model compatibility |

---

## 9. Background Processing & Worker Architecture

### 9.1 Worker Daemon

The Worker is a **standalone async daemon** (no HTTP server). Its lifecycle:

```
Startup → Load config → Init shared resources → Register handlers → Start polling → [Run] → Shutdown
```

**Handler-based dispatch:** Each message type maps to a handler class:

| Handler | Trigger | Action |
|---------|---------|--------|
| Crawl URL | `crawl_url` | Fetch single URL → chunk → embed → store |
| Ingest Document | `ingest_doc` | Process uploaded document → chunk → embed → store |
| Discover Links | `discover_links` | Parse page for links → enqueue children |
| Discover Sitemap | `discover_sitemap` | Parse sitemap.xml → enqueue URLs |
| Crawl Site | `crawl_site` | Deep crawl via external spider API (fan-out) |
| Process Spider Page | `process_spider_page` | Process individual crawled page result |

### 9.2 Concurrency Model

```
Queue Consumer (single poller)
  → asyncio.Task per message (bounded by semaphore)
    → HTTP concurrency limiter (global + per-host)
    → Embedding rate limiter (token bucket)
```

- **Task concurrency:** Configurable cap (default: 8 concurrent tasks).
- **HTTP limiting:** Global and per-host semaphores prevent overwhelming targets.
- **Embedding rate:** Token-bucket limiter prevents API quota exhaustion.

### 9.3 Crawler Service

Operates as a FastAPI service with a background task pipeline:

```
POST /v1/crawl → Enqueue batch → Return job ID immediately

Background:
  → Spider API (streaming JSON response, object-by-object parsing)
  → Classify each page (multi-signal: URL heuristics + keyword anchors + metadata)
  → Optional LLM tie-breaker for ambiguous classifications
  → Optional: embed via shared library pipeline
  → Archive to blob storage
```

---

## 10. Policy Driven Detection Service

### 10.1 Architecture

```
Input Text
  → NLP Engine (spaCy, transformer model)
  → Entity Recognition
  │   ├── Built-in recognizers (names, emails, phones, SSNs, credit cards, IPs, URLs)
  │   ├── Custom recognizers (addresses, regional IDs, domain-specific patterns)
  │   └── Gendered language detector (pronouns, titles, relationship terms)
  → Policy Application
  │   ├── Per-entity-type operator selection (replace, mask, hash, redact)
  │   ├── Threshold filtering by detection mode
  │   └── Unicode block character rendering for visual output
  → Output: redacted text + entity spans + audit metadata
```

### 10.2 Detection Modes

| Mode | Threshold | NLP Features | Use Case |
|------|-----------|-------------|----------|
| `fast` | High | Basic NER | Real-time chat redaction |
| `balanced` | Medium | Standard NER | Default for most workflows |
| `strict` | Low | NER + coreference resolution | Regulatory compliance |
| `sensitive` | Lowest | NER + coreference + fuzzy | Maximum recall |

### 10.3 Policy System

Named, versioned redaction policies define per-entity-type operators:

```python
# Example policy definition
{
    "name": "safe_v1",
    "operators": {
        "PERSON":        {"type": "replace", "value": "[REDACTED_NAME]"},
        "EMAIL_ADDRESS": {"type": "mask", "chars_to_mask": 6},
        "PHONE_NUMBER":  {"type": "redact"},
        "LOCATION":      {"type": "replace", "value": "[LOCATION]"},
    },
    "default_operator": {"type": "replace", "value": "█████"}
}
```

### 10.4 Document Processing Pipeline

```
Upload → Format Detection → Parser (PDF/DOCX/TXT/Image+OCR)
  → Text Extraction → Policy Driven Detection → Policy Application
  → Redacted Output (text + optionally re-rendered document)
```

Parser selection via factory pattern. External OCR service integration for image-based documents.

### 10.5 Coreference Resolution

AI-assisted pronoun resolution for `strict` and `sensitive` modes:

```
"John Smith went to the store. He bought milk."
  → Coreference: "He" → "John Smith"
  → Both "John Smith" and "He" marked as PERSON entities
```

Model loaded once at service startup, shared across requests.

---

## 11. Authentication & Authorization

### 11.1 Authentication Patterns

| Layer | Mechanism | Token Type | Storage |
|-------|-----------|------------|---------|
| **Frontend ↔ User** | NextAuth.js (OAuth/credentials/OTP) | JWT | httpOnly cookie |
| **Frontend → Backend** | Bearer token forwarding | JWT (HS256) | Header |
| **Service → Service** | Shared secret | Static | `X-Service-Auth` header |
| **Backend → User resolution** | JWT decode + DB lookup | — | — |

### 11.2 Authorization Model

```python
# Role-based access control via dependency injection
class RoleChecker:
    def __init__(self, allowed_roles: List[str]):
        self.allowed_roles = allowed_roles

    def __call__(self, current_user: User = Depends(get_current_user)):
        if current_user.role not in self.allowed_roles:
            raise HTTPException(403)
        return current_user

# Usage in router
@router.get("/v1/admin/users", dependencies=[Depends(RoleChecker(["admin"]))])
async def list_users(): ...
```

### 11.3 Route Protection

Frontend middleware intercepts navigation to protected routes, checking JWT validity before allowing access. Unauthenticated users are redirected to login.

---

## 12. Middleware & Cross-Cutting Concerns

### 12.1 Middleware Stack (Core API)

Applied in order on every request:

```
Request →
  1. CORS (origin-restricted)
  2. Security Headers (CSP, HSTS, X-Frame-Options, Permissions-Policy)
  3. Audit Logging (structured, with automatic Policy Driven field redaction)
  4. Rate Limiting (tiered: public / authenticated / AI / auth-endpoints)
→ Router
```

### 12.2 Rate Limiting

```
┌─────────────────────────────────────────────┐
│              Rate Limiter                   │
│                                             │
│  Key: user_id (authenticated) or IP         │
│  Backend: Redis (preferred) or in-memory    │
│  Strategy: Fixed window                     │
│                                             │
│  Tiers:                                     │
│    Public endpoints:   60 req/min           │
│    Authenticated:     120 req/min           │
│    AI/compute-heavy:   20 req/min           │
│    Auth endpoints:     10 req/min           │
└─────────────────────────────────────────────┘
```

### 12.3 Structured Error Envelope

All backend services return errors in a consistent envelope:

```json
{
  "error": {
    "code": "RESOURCE_NOT_FOUND",
    "message": "Human-readable description",
    "details": { "resource_id": "abc123" }
  }
}
```

Each service defines typed exception hierarchies (e.g., `AppException` base → `NotFoundError`, `ValidationError`, `ProviderError`) that middleware catches and serializes.

### 12.4 Audit Logging

- Structured JSON logs with request ID correlation.
- Automatic redaction of configured sensitive fields (`password`, `token`, `ssn`, `credit_card`).
- Auth endpoint bodies are never logged.
- Health check and documentation paths are excluded.

---

## 13. Infrastructure-as-Code

### 13.1 Cloud Landing Zone

Three-module Terraform composition:

```
┌─────────────────────────────────────────────────────┐
│                    Platform Module                  │
│  Management groups, policies, RBAC, monitoring,     │
│  central resource groups, private DNS zones         │
├─────────────────────────────────────────────────────┤
│                  Connectivity Module                │
│  Hub-spoke VNet, NAT gateway, VNet peering,         │
│  subnet partitioning (app, data, private endpoints) │
├─────────────────────────────────────────────────────┤
│          Control-Plane / Data-Plane Modules         │
│  Workload-specific resources (compute, data, AI)    │
└─────────────────────────────────────────────────────┘
```

### 13.2 Networking Model

- **Hub-spoke topology** with address space management via CIDR subnet calculations.
- **Private endpoints subnet** for all data services (databases, caches, queues).
- **NAT gateway** for controlled outbound traffic.
- **Multi-region support** via parameterized connectivity configuration.
- **Optional bastion host** for secure administrative access.

### 13.3 Environment Strategy

- Environment-parameterized (`prod`, `nonprod`) with separate variable files.
- Supports separate subscriptions for platform vs. workload resources.
- Budget alerts configurable with per-environment thresholds.
- Remote state backend with per-environment state files.

---

## 14. Containerization & Orchestration

### 14.1 Build Strategy

All services use **multi-stage Docker builds**:

```dockerfile
# Stage 1: Builder (install dependencies, compile assets)
FROM python:3.12-slim AS builder
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Stage 2: Production (minimal image, copy artifacts)
FROM python:3.12-slim
COPY --from=builder /usr/local/lib/python3.12/site-packages /usr/local/lib/python3.12/site-packages
COPY ./app ./app
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0"]
```

Frontend services use a Node builder stage, then copy the Next.js standalone output to an Alpine runtime image.

### 14.2 Local Orchestration

Docker Compose defines the full stack on a shared bridge network:

- All services share a single PostgreSQL instance (logically separated by database/schema).
- Redis available for rate limiting and caching.
- Health checks with appropriate start periods for each service type.
- Entrypoint scripts run database migrations before starting the application.
- Host networking available for local LLM server access from containers.

### 14.3 Service Dependencies

```
PostgreSQL ← Core API, Policy Driven Service, AI Orchestrator, Worker, Crawler, Console UI
Cloud Queue ← AI Orchestrator (producer), Worker (consumer)
Blob Storage ← Core API (documents), Worker (archive), Crawler (output)
External Spider API ← Crawler
External LLM APIs ← AI Orchestrator, Policy Driven Service (coreference)
Email Service ← Core API, Console UI
Payment Provider ← Core API
```

---

## 15. Shared Code & Cross-Service Contracts

### 15.1 Shared RAG Library

Installed as a local Python package by services needing RAG capabilities. Contains:

| Module | Purpose |
|--------|---------|
| Models | SQLAlchemy ORM for vector tables |
| Database | Async connection management |
| VectorStore | Abstract interface + pgvector implementation |
| Chunker | Token-based text splitting |
| Crawler | HTTP fetcher with rate limiting + robots.txt |
| Embeddings | API client with retry/backoff |

**Design constraint:** Schema name and embedding dimensions are read from environment at module import time. All consumers must set these before importing.

### 15.2 Shared Configuration Data

A canonical data file (JSON) with typed wrappers in both TypeScript and Python, consumed by all services for configuration consistency. A sync utility ensures the wrappers stay in sync with the source data.

### 15.3 API Contract Consistency

- **Backend schemas:** Pydantic models define all request/response shapes.
- **Frontend types:** TypeScript interfaces mirror Pydantic schemas.
- **Error envelope:** Identical structure across all services.
- **Versioning:** All API paths prefixed with `/v1`.

---

## 16. Security Posture

| Concern | Implementation |
|---------|---------------|
| **Transport** | HSTS in production, TLS termination at load balancer |
| **XSS** | Content Security Policy (`default-src 'none'`), X-XSS-Protection |
| **Clickjacking** | X-Frame-Options: DENY, CSP frame-ancestors: none |
| **SSRF** | Blocked hosts/IP ranges for RAG crawler (private, metadata endpoints) |
| **Rate Limiting** | Per-user/IP, tiered by endpoint sensitivity, Redis-backed |
| **Auth Tokens** | JWT HS256, httpOnly cookies, short-lived access + long-lived refresh |
| **Service Auth** | Shared secret via custom header (optional, configurable) |
| **Audit** | Structured logging with automatic sensitive field redaction |
| **Data Privacy** | Dedicated Policy Driven service, configurable deletion grace periods |
| **Sensitive Logging** | Auth endpoint bodies never logged, field-level redaction |
| **Browser Features** | Permissions-Policy disabling camera, mic, geolocation, payment, USB |
| **DB Access** | Per-service schema isolation, no cross-schema queries |

---

## 17. Evolution Strategy

### Current State (MVP)

- Monorepo with clear service boundaries.
- Python (FastAPI) for all backend services — rapid iteration, shared ecosystem.
- Single PostgreSQL instance with schema-per-service isolation.
- Docker Compose for local development; cloud IaC for production.

### Planned Evolution

```
Phase 1 (Current)          Phase 2                    Phase 3
─────────────────          ──────────                 ──────────
Python FastAPI services    Extract hot paths to Go    Full Go microservices
Single PostgreSQL          Per-service databases      Per-service databases
Docker Compose (local)     Container orchestration    Kubernetes / managed
Cloud queue (single)       Event-driven (pub/sub)     Event mesh
Shared RAG library         RAG as standalone service  RAG service + API
```

### Migration-Safe Design Decisions

1. **Service boundaries are strict** — no shared ORM models or direct DB access across services.
2. **Communication is HTTP + queue** — easily replaceable with gRPC, event mesh, or service mesh.
3. **Schema-per-service** — straightforward to split into per-service databases.
4. **Typed API contracts** — provider-agnostic interfaces enable implementation swaps.
5. **Abstract vector store interface** — backend-agnostic retrieval (pgvector today, dedicated vector DB tomorrow).
6. **LLM provider abstraction** — adding new providers requires only implementing the base interface.

---

*This document describes architectural patterns and technical design decisions. It is intended as a living reference — update it as the system evolves.*
