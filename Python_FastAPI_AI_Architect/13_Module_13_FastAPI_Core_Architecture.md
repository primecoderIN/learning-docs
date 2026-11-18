# Module 13: FastAPI Core Architecture & Web Engineering

Welcome to **Module 13** of the Normora Python & AI Masterclass.

Now that you have mastered the Python asynchronous event loop in [Module 10](file:///c:/Users/Sanjeev%20Kumar/Codes/learning-docs/Python_FastAPI_AI_Architect/10_Module_10_Asynchronous_Python_and_Asyncio.md) and Pydantic v2 data validation in [Module 12](file:///c:/Users/Sanjeev%20Kumar/Codes/learning-docs/Python_FastAPI_AI_Architect/12_Module_12_Pydantic_v2_Data_Validation.md), we bring these two foundational pillars together into **FastAPI**—the fastest, most ergonomic Python web framework in existence.

For a software architect coming from the .NET ecosystem (C# 13, ASP.NET Core 10, Kestrel), FastAPI will feel instantly intuitive yet refreshingly concise. Where ASP.NET Core relies on heavy compile-time attributes, dependency injection service collections, and controller classes, FastAPI leverages Python's runtime type hints and ASGI (Asynchronous Server Gateway Interface) to achieve equivalent throughput, automatic OpenAPI documentation, and rock-solid validation with a fraction of the boilerplate.

In this module, you will master the end-to-end physical architecture of FastAPI:
* The **ASGI specification** and how **Uvicorn** drives worker processes over OS sockets.
* The complete **HTTP Request Lifecycle** inside FastAPI, visualised with Mermaid sequence diagrams.
* Modern application lifecycle management with **`lifespan` async context managers** (replacing legacy startup/shutdown events).
* Clean architecture with **`APIRouter`** modular monolith organization.
* Parameter binding: Path parameters, Query parameters, Request Bodies, Headers, and Cookies.
* File handling: Streaming file uploads with **`UploadFile`**, batch uploads, and `multipart/form-data` with **`Form()`**.
* Response models, status codes, and specialized responses (**`StreamingResponse`** for LLM tokens, **`FileResponse`**, **`JSONResponse`**).
* Enterprise API response envelopes (**`ApiResponse[T]`**) for seamless Angular 22 SPA integration.
* Asynchronous post-response processing with **`BackgroundTasks`**.
* Custom ASGI and Starlette middleware: ordering, request decoration, and correlation tracing.
* Enterprise security exception handling: **BOLA defense (404)**, **BFLA defense (403)**, and sanitized error envelopes.
* Production **OpenAPI / Swagger** customization and interactive documentation.
* Comprehensive **FastAPI vs ASP.NET Core** architectural comparison.
* **Common Mistakes & Anti-Patterns** in production.
* **Architectural Interview Questions & Answers**.

---

## ASP.NET Core vs FastAPI: Architectural Quick-Reference

Before diving into Python code, let's map the mental models between ASP.NET Core and FastAPI:

| Feature / Concept | ASP.NET Core 10 (.NET) | FastAPI (Python 3.12+) |
| :--- | :--- | :--- |
| **Web Server** | Kestrel (Managed C# Sockets / libuv) | Uvicorn (uvloop / libuv C extension / Windows IOCP) |
| **Server Interface** | `IHttpApplication<TContext>` / OWIN | ASGI (`scope, receive, send`) |
| **Modular Routing** | `[ApiController]`, `ControllerBase`, `MapGroup` | `APIRouter(prefix=..., tags=...)` |
| **Input Validation** | FluentValidation / `DataAnnotations` | Pydantic v2 `BaseModel` (compiled Rust engine) |
| **Parameter Binding** | `[FromRoute]`, `[FromQuery]`, `[FromBody]`, `[FromHeader]` | `Path()`, `Query()`, `Body()`, `Header()`, `Cookie()` |
| **File Uploads** | `IFormFile`, `IFormFileCollection`, `[FromForm]` | `UploadFile`, `File()`, `Form()` |
| **Response Types** | `ActionResult<T>`, `IResult`, `FileStreamResult` | `response_model=T`, `StreamingResponse`, `FileResponse` |
| **Dependency Injection** | `IServiceCollection`, `[FromServices]`, Scoped/Transient | `Depends(get_db_session)` (Hierarchical DAG) |
| **Middleware Pipeline** | `app.UseMiddleware<T>()`, `RequestDelegate` | `app.add_middleware(T)`, Starlette / Pure ASGI |
| **Exception Handling** | `IExceptionHandler`, `app.UseExceptionHandler()` | `@app.exception_handler(CustomException)` |
| **App Lifecycle** | `IHostedService`, `IHostApplicationLifetime` | `lifespan` (`asynccontextmanager`) |
| **Background Work** | `IHostedService`, `Channel<T>`, Hangfire | `BackgroundTasks` (lightweight), Celery / ARQ (heavy) |
| **API Documentation** | Swashbuckle / Microsoft.AspNetCore.OpenApi | Native OpenAPI 3.1.0 + Swagger UI (`/docs`) & ReDoc |

---

## 1. The ASGI Specification, Uvicorn & The FastAPI Request Lifecycle

### 1.1 From WSGI to ASGI

In the first two decades of Python web development (Django, Flask), web servers communicated with applications using **WSGI (Web Server Gateway Interface, PEP 3333)**. 

WSGI was strictly synchronous:
```python
# Legacy WSGI Signature (Synchronous, Thread-Blocking)
def wsgi_app(environ: dict, start_response: callable):
    start_response("200 OK", [("Content-Type", "text/plain")])
    return [b"Hello from legacy WSGI!"]
```
Because WSGI functions were synchronous `def`, the web server had to allocate a dedicated OS thread per concurrent HTTP request. If an endpoint made an external database query or waited for Google Gemini Flash to generate tokens, that thread sat idle and blocked. Handling 10,000 concurrent streaming connections required 10,000 threads, quickly crashing the OS via stack memory exhaustion.

**ASGI (Asynchronous Server Gateway Interface)** revolutionized Python web engineering by defining an asynchronous, event-driven contract:
```python
# Modern ASGI Signature (Asynchronous, Non-Blocking)
async def asgi_app(scope: dict, receive: callable, send: callable) -> None:
    ...
```

Let's break down the three ASGI primitives:
* **`scope`**: A Python dictionary containing connection metadata. It lives for the duration of the connection. For an HTTP request, it holds the HTTP method, path, query string, headers, client IP, and server port. For a WebSocket connection, it holds the subprotocol, path, and handshake headers.
* **`receive`**: An `async` callable used by the application to read incoming chunks from the client socket. Calling `await receive()` yields messages such as `{"type": "http.request", "body": b"...", "more_body": False}`.
* **`send`**: An `async` callable used by the application to push response headers and body bytes back to the OS socket. Calling `await send({"type": "http.response.start", "status": 200, "headers": [...]})` writes HTTP headers, and `await send({"type": "http.response.body", "body": b"..."})` streams body bytes.

### 1.2 Uvicorn: The High-Performance ASGI Engine

**Uvicorn** is an enterprise-grade ASGI web server written in Python and C. It binds to the OS network interface, accepts TCP connections, parses HTTP/1.1 and HTTP/2 wire protocols using the ultra-fast C library `httptools`, and delegates the incoming socket events to FastAPI's ASGI interface.

On Linux and macOS, Uvicorn utilizes **`uvloop`**—a drop-in replacement for Python's default `asyncio` event loop built on **`libuv`** (the same battle-tested C asynchronous I/O engine that powers Node.js). On Windows, it leverages high-performance I/O Completion Ports (IOCP).

```mermaid
flowchart LR
    Client["Client Browser / Angular 22 SPA"] <-->|HTTP/1.1, HTTP/2, SSE, WebSockets| Uvicorn["Uvicorn ASGI Server<br>(uvloop / libuv C Sockets)"]
    Uvicorn <-->|ASGI: (scope, receive, send)| Starlette["Starlette Core Engine<br>(Routing, Middleware, State)"]
    Starlette <-->|Validation, Serialization| Pydantic["Pydantic v2<br>(pydantic-core Rust Engine)"]
    Starlette <-->|Dependency Graph / DI| FastAPI["FastAPI Application Handlers<br>(Normora Endpoints)"]
```

### 1.3 The FastAPI HTTP Request Lifecycle

When an Angular 22 client sends a request to Normora (for instance, uploading a legal handbook to be embedded into pgvector), the request flows through a meticulously orchestrated pipeline:

```mermaid
sequenceDiagram
    autonumber
    actor Client as Angular 22 SPA
    participant Uvicorn as Uvicorn (ASGI Server)
    participant Middleware as Middleware Pipeline (CORS, Trace, Auth)
    participant Router as APIRouter (Route Matching)
    participant Depends as Dependency Graph (Depends)
    participant Pydantic as Pydantic v2 (Rust Validator)
    participant Handler as Route Handler (CQRS / Service)
    participant PostProcess as Response Serializer & BackgroundTasks

    Client->>Uvicorn: HTTP POST /api/v1/documents/ingest (Headers, Multipart Body)
    Uvicorn->>Middleware: Creates ASGI scope, receive, send
    Note over Middleware: Execute request phase<br>(Trace ID, Metrics, CORS check)
    Middleware->>Router: Match URL & HTTP Method
    Router->>Depends: Resolve Dependencies (Auth Token, DbSession, TenantContext)
    alt Security Check Fails
        Depends-->>Client: 401 Unauthorized / 403 Forbidden / 404 BOLA
    end
    Depends->>Pydantic: Parse & Validate Path, Query, Body schemas
    alt Validation Fails
        Pydantic-->>Client: 400 Bad Request (ApiResponse envelope)
    end
    Pydantic->>Handler: Invoke async def ingest_document(payload, db, tenant)
    Handler-->>PostProcess: Return domain entity / DTO
    Note over PostProcess: Pydantic filters sensitive fields via response_model<br>Serialize to JSON / Stream
    PostProcess->>Middleware: Execute response phase (Set headers, Execution-Time)
    Middleware->>Uvicorn: ASGI send(http.response.start & body)
    Uvicorn-->>Client: HTTP 201 Created (ApiResponse JSON)
    opt BackgroundTasks Queued
        PostProcess->>PostProcess: Execute BackgroundTasks (Chunking, Vector Embedding)
    end
```

---

### 5 Real-World Normora Examples: ASGI & Server Architecture

#### Example 1.1: Production Normora FastAPI Root App Definition
Here is the production configuration for Normora's root application, including custom documentation URLs, OpenAPI metadata, and root health diagnostics:

```python
from fastapi import FastAPI, status
from fastapi.responses import JSONResponse

# Create the master enterprise FastAPI application instance:
app = FastAPI(
    title="Normora Enterprise AI Knowledge Platform",
    description="Multi-tenant knowledge management, pgvector RAG, and conversational AI engine.",
    version="1.0.0",
    docs_url="/api/docs",           # Relocated from default /docs for security
    redoc_url="/api/redoc",         # Clean ReDoc documentation endpoint
    openapi_url="/api/openapi.json",# OpenAPI 3.1 specification path
    contact={
        "name": "Normora Platform Engineering",
        "email": "architecture@normora.ai",
    },
    license_info={
        "name": "Proprietary Commercial",
        "url": "https://normora.ai/terms",
    }
)

@app.get("/health", tags=["System Diagnostics"], status_code=status.HTTP_200_OK)
async def health_check() -> dict[str, str]:
    """
    Kubernetes / Container liveness probe endpoint.
    Returns HTTP 200 immediately to verify the ASGI loop is responsive.
    """
    return {
        "status": "HEALTHY",
        "service": "Normora API Gateway",
        "environment": "production"
    }
```

#### Example 1.2: Programmatic Uvicorn Multi-Worker Server Runner
In production, you can launch Uvicorn directly from Python code with multi-process workers, custom log configurations, and socket settings:

```python
import multiprocessing
import uvicorn

def run_production_server() -> None:
    # Determine CPU core count for worker scaling:
    cpu_cores = multiprocessing.cpu_count()
    workers_count = (2 * cpu_cores) + 1  # Standard gunicorn/uvicorn concurrency formula

    uvicorn_config = uvicorn.Config(
        app="main:app",
        host="0.0.0.0",
        port=8000,
        workers=workers_count,
        loop="uvloop",                # Highest performance C-based event loop
        http="httptools",             # Ultra-fast C HTTP parser
        proxy_headers=True,           # Trust X-Forwarded-For headers from reverse proxy
        forwarded_allow_ips="*",      # Cloudflare / Ingress Controller CIDR
        timeout_keep_alive=65,        # Keep-alive matching AWS ALB timeout (60s + 5s)
        log_level="info",
        access_log=True
    )
    server = uvicorn.Server(uvicorn_config)
    server.run()

if __name__ == "__main__":
    run_production_server()
```

#### Example 1.3: Raw ASGI Middleware for Tenant Boundary Inspection
A pure ASGI middleware bypasses framework overhead to inspect raw connection metadata before Starlette even routes the request:

```python
import time
from starlette.types import ASGIApp, Scope, Receive, Send

class RawAsgiTenantAuditMiddleware:
    """
    High-performance raw ASGI middleware.
    Inspects tenant headers and logs execution latency on all HTTP requests.
    """
    def __init__(self, app: ASGIApp) -> None:
        self.app = app

    async def __call__(self, scope: Scope, receive: Receive, send: Send) -> None:
        # Only intercept HTTP protocols (ignore lifespan and raw websockets here):
        if scope["type"] != "http":
            await self.app(scope, receive, send)
            return

        start_time = time.perf_counter()
        
        # Raw ASGI headers are tuples of lowercased byte strings: (b"header-name", b"value")
        headers = dict(scope.get("headers", []))
        raw_tenant = headers.get(b"x-tenant-id", b"anonymous").decode("utf-8")
        path = scope.get("path", "")
        method = scope.get("method", "")

        # Wrap the ASGI send callable to capture status code and append telemetry:
        async def custom_send(message: dict) -> None:
            if message["type"] == "http.response.start":
                duration_ms = (time.perf_counter() - start_time) * 1000.0
                print(
                    f"[ASGI AUDIT] Tenant: {raw_tenant} | {method} {path} "
                    f"| Status: {message['status']} | Duration: {duration_ms:.2f}ms"
                )
            await send(message)

        await self.app(scope, receive, custom_send)

# Registration on the app:
app.add_middleware(RawAsgiTenantAuditMiddleware)
```

#### Example 1.4: High-Performance CORS Configuration for Angular 22 SPA
Normora's Angular 22 SPA runs standalone components with SignalR and Duende BFF. Cross-Origin Resource Sharing (CORS) must be strictly configured:

```python
from fastapi.middleware.cors import CORSMiddleware

# Define authorized origins (Local dev, Staging, and Production Angular apps):
ALLOWED_ORIGINS = [
    "http://localhost:4200",               # Angular 22 local development server
    "https://preview.normora.ai",          # PR preview environments
    "https://app.normora.ai",              # Production enterprise client
]

app.add_middleware(
    CORSMiddleware,
    allow_origins=ALLOWED_ORIGINS,
    allow_credentials=True,                # Critical for passing Keycloak session cookies
    allow_methods=["GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"],
    allow_headers=[
        "Authorization",
        "Content-Type",
        "X-Tenant-ID",
        "X-Correlation-ID",
        "X-Requested-With",
        "Accept",
        "Origin"
    ],
    expose_headers=["X-Correlation-ID", "Content-Disposition", "Server-Timing"],
    max_age=3600                           # Cache preflight OPTIONS responses for 1 hour
)
```

#### Example 1.5: OpenTelemetry / W3C `traceparent` Propagation ASGI Middleware
In enterprise distributed systems, trace context must pass seamlessly between the .NET backend, FastAPI AI microservices, and external agents:

```python
import uuid
from starlette.types import ASGIApp, Scope, Receive, Send

class OpenTelemetryTracePropagationMiddleware:
    """
    Extracts W3C 'traceparent' header or generates a synthetic correlation ID
    to guarantee 100% observability across Normora microservices.
    """
    def __init__(self, app: ASGIApp) -> None:
        self.app = app

    async def __call__(self, scope: Scope, receive: Receive, send: Send) -> None:
        if scope["type"] == "http":
            headers = dict(scope.get("headers", []))
            
            # Check for W3C traceparent or legacy correlation id:
            trace_id = headers.get(b"traceparent") or headers.get(b"x-correlation-id")
            correlation_id = trace_id.decode("utf-8") if trace_id else f"normora-trace-{uuid.uuid4()}"
            
            # Store on ASGI state dictionary so downstream endpoints can access it:
            if "state" not in scope:
                scope["state"] = {}
            scope["state"]["correlation_id"] = correlation_id

            async def send_with_trace(message: dict) -> None:
                if message["type"] == "http.response.start":
                    # Echo correlation ID back to the client in response headers:
                    headers_list = list(message.get("headers", []))
                    headers_list.append((b"x-correlation-id", correlation_id.encode("utf-8")))
                    message["headers"] = headers_list
                await send(message)

            await self.app(scope, receive, send_with_trace)
        else:
            await self.app(scope, receive, send)

app.add_middleware(OpenTelemetryTracePropagationMiddleware)
```

---

## 2. Application Lifespan Events (`lifespan` with `asynccontextmanager`)

### 2.1 Why Legacy `@app.on_event` is Deprecated

In older FastAPI tutorials, you will see startup and shutdown logic written using decorators:
```python
# DEPRECATED IN MODERN FASTAPI:
@app.on_event("startup")
async def startup_event():
    ...

@app.on_event("shutdown")
async def shutdown_event():
    ...
```

> [!WARNING]
> `@app.on_event("startup")` and `@app.on_event("shutdown")` are formally deprecated in Starlette and FastAPI. They do not share state cleanly, make dependency testing difficult, and do not provide clean exception-handling boundaries during startup failures.

Modern FastAPI uses the **ASGI Lifespan Protocol** powered by Python's standard library **`contextlib.asynccontextmanager`**.

```mermaid
flowchart TD
    Start([Uvicorn Worker Starts]) --> InitResources[Initialize DB Pool, MinIO Client, Gemini Warm-up]
    InitResources --> YieldPoint{{yield app.state}}
    YieldPoint --> ServingRequests[Application Serves Requests<br>Normal HTTP / SSE Traffic]
    ServingRequests --> TermSignal([SIGTERM / SIGINT Received])
    TermSignal --> Teardown[Code After Yield Executes<br>Drain Workers, Close Connections, Flush Traces]
    Teardown --> ProcessExit([Process Terminates Gracefully])
```

The code executes in two distinct phases:
1. **Before the `yield`**: Runs once when Uvicorn boots the worker process, *before* accepting any incoming HTTP connections. If an exception occurs here (e.g., PostgreSQL is unreachable or MinIO credentials are invalid), the process exits immediately with a non-zero exit code, alerting Kubernetes to restart the pod.
2. **After the `yield`**: Runs when Uvicorn receives a termination signal (`SIGTERM` or `SIGINT`). This guarantees that open network sockets, background worker threads, and connection pools are closed gracefully without dropping in-flight HTTP requests.

---

### 5 Real-World Normora Examples: Lifespan Management

#### Example 2.1: PostgreSQL `asyncpg` Connection Pool & pgvector Extension Verification
When Normora boots, it must establish a resilient connection pool to PostgreSQL and verify that the `pgvector` extension is active:

```python
from contextlib import asynccontextmanager
from typing import AsyncGenerator
from fastapi import FastAPI
import asyncpg

@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncGenerator[None, None]:
    # --- STARTUP PHASE ---
    print("[LIFESPAN] Booting Normora: Initializing PostgreSQL connection pool...")
    
    # Create enterprise connection pool:
    app.state.db_pool = await asyncpg.create_pool(
        dsn="postgresql://normora_admin:SecurePass123!@localhost:5432/normora_db",
        min_size=10,
        max_size=50,
        command_timeout=30.0,
        max_inactive_connection_lifetime=300.0
    )
    
    # Verify pgvector extension is installed and active in database:
    async with app.state.db_pool.acquire() as conn:
        version = await conn.fetchval("SELECT extversion FROM pg_extension WHERE extname = 'vector';")
        if not version:
            raise RuntimeError("CRITICAL: pgvector extension is NOT installed in PostgreSQL database!")
        print(f"[LIFESPAN] pgvector extension verified active (Version: {version})")

    yield  # --- APPLICATION SERVES HTTP REQUESTS HERE ---

    # --- SHUTDOWN PHASE ---
    print("[LIFESPAN] Shutting down Normora: Closing database connection pool...")
    await app.state.db_pool.close()
    print("[LIFESPAN] Database pool successfully drained and closed.")

app = FastAPI(title="Normora Knowledge Engine", lifespan=lifespan)
```

#### Example 2.2: MinIO S3 Object Storage Client Verification
Normora uses MinIO for storing ingested PDF/DOCX files. On startup, the lifespan ensures the target tenant storage buckets exist:

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI
from miniopy_async import Minio

@asynccontextmanager
async def minio_lifespan(app: FastAPI):
    # --- STARTUP ---
    print("[LIFESPAN] Connecting to MinIO Object Storage cluster...")
    app.state.minio_client = Minio(
        endpoint="localhost:9000",
        access_key="normora-minio-admin",
        secret_key="SuperSecretMinioPasswordKey2026!",
        secure=False
    )
    
    # Ensure default system buckets exist:
    required_buckets = ["normora-documents", "normora-exports", "normora-branding"]
    for bucket in required_buckets:
        exists = await app.state.minio_client.bucket_exists(bucket)
        if not exists:
            await app.state.minio_client.make_bucket(bucket)
            print(f"[LIFESPAN] Created missing MinIO bucket: '{bucket}'")
            
    yield
    
    # --- SHUTDOWN ---
    print("[LIFESPAN] Disconnecting MinIO client...")
    # Clean up client sessions if necessary
```

#### Example 2.3: Hosted Google Gemini Flash Client Pool Initialization & Warm-Up
Cold starts in LLM APIs introduce latency. Normora warms up Google Gemini Flash during startup:

```python
import os
from contextlib import asynccontextmanager
from fastapi import FastAPI
from google import genai
from google.genai import types

@asynccontextmanager
async def gemini_lifespan(app: FastAPI):
    # --- STARTUP ---
    api_key = os.getenv("GEMINI_API_KEY")
    if not api_key:
        raise ValueError("CRITICAL: GEMINI_API_KEY environment variable is missing!")

    print("[LIFESPAN] Initializing Google GenAI Client for Gemini 2.5 Flash...")
    app.state.gemini_client = genai.Client(api_key=api_key)

    # Perform a lightweight warm-up query to establish TLS handshake:
    try:
        response = await app.state.gemini_client.aio.models.generate_content(
            model="gemini-2.5-flash",
            contents="ping",
            config=types.GenerateContentConfig(max_output_tokens=5)
        )
        print(f"[LIFESPAN] Gemini Flash warm-up successful. Latency handshake verified.")
    except Exception as exc:
        print(f"[LIFESPAN WARNING] Gemini warm-up ping failed: {exc}")

    yield

    # --- SHUTDOWN ---
    print("[LIFESPAN] Closing Gemini client connection sessions.")
```

#### Example 2.4: Keycloak OIDC JWKS Public Key Cache Preloading
Validating JWTs on every incoming request requires Keycloak's JSON Web Key Set (JWKS). Pre-fetching keys on startup eliminates latency spikes on the first user request:

```python
from contextlib import asynccontextmanager
import httpx
from fastapi import FastAPI

@asynccontextmanager
async def keycloak_lifespan(app: FastAPI):
    # --- STARTUP ---
    keycloak_jwks_url = "https://auth.normora.ai/realms/normora-platform/protocol/openid-connect/certs"
    print(f"[LIFESPAN] Preloading Keycloak OIDC JWKS from {keycloak_jwks_url}...")
    
    async with httpx.AsyncClient(timeout=10.0) as client:
        response = await client.get(keycloak_jwks_url)
        response.raise_for_status()
        app.state.jwks_keys = response.json()
        print(f"[LIFESPAN] Loaded {len(app.state.jwks_keys.get('keys', []))} cryptographic signing keys.")

    yield

    # --- SHUTDOWN ---
    app.state.jwks_keys.clear()
    print("[LIFESPAN] Cleared cryptographic key cache.")
```

#### Example 2.5: Graceful Shutdown with In-Flight Task Draining
Ensuring that long-running document chunking or embedding jobs are given time to complete before the OS kills the process:

```python
import asyncio
from contextlib import asynccontextmanager
from fastapi import FastAPI

@asynccontextmanager
async def full_production_lifespan(app: FastAPI):
    # Track all active background document processing tasks:
    app.state.active_background_tasks = set()
    print("[LIFESPAN] Background task registry initialized.")

    yield

    # --- GRACEFUL SHUTDOWN ---
    tasks_to_drain = list(app.state.active_background_tasks)
    if tasks_to_drain:
        print(f"[LIFESPAN] Graceful shutdown initiated: Draining {len(tasks_to_drain)} active background tasks...")
        # Wait up to 25 seconds for background document chunkers to finish:
        done, pending = await asyncio.wait(tasks_to_drain, timeout=25.0)
        if pending:
            print(f"[LIFESPAN WARNING] {len(pending)} tasks did not finish within timeout. Cancelling...")
            for task in pending:
                task.cancel()
    print("[LIFESPAN] All worker tasks drained. Server shutdown complete.")
```

---

## 3. Modular Architecture with `APIRouter` (The Modular Monolith Pattern)

### 3.1 Folder-Per-Module CQRS Organization

Normora is structured as a **Modular Monolith**. In ASP.NET Core, you would organize code into distinct Feature Folders with MediatR Handlers and Controllers. In FastAPI, we achieve identical modularity and boundary encapsulation using **`APIRouter`**.

```
server/
├── main.py                          # App root & router aggregation
├── core/
│   ├── config.py                    # Environment settings (Pydantic BaseSettings)
│   ├── security.py                  # JWT & Keycloak verification
│   └── database.py                  # asyncpg / SQLAlchemy engine
└── modules/
    ├── tenants/                     # Tenants Module
    │   ├── api/
    │   │   └── tenants_router.py    # APIRouter for /api/v1/tenants
    │   ├── schemas/                 # Pydantic v2 DTOs
    │   └── services/                # Business logic
    ├── documents/                   # Documents & Ingestion Module
    │   ├── api/
    │   │   └── documents_router.py  # APIRouter for /api/v1/documents
    │   └── services/                # Chunking & pgvector embedding
    └── conversations/               # RAG & Chat Module
        ├── api/
        │   └── chat_router.py       # APIRouter for /api/v1/conversations
        └── services/                # Gemini Flash streaming & citations
```

An `APIRouter` can have its own:
* **`prefix`**: e.g., `/api/v1/documents`
* **`tags`**: Grouping endpoints cleanly in Swagger UI
* **`dependencies`**: Module-wide security guards (e.g., verifying active tenant membership)
* **`responses`**: Default error models documented for every endpoint in the module

---

### 5 Real-World Normora Examples: APIRouter Organization

#### Example 3.1: Tenants Module Router with Memberships & Quotas
```python
from fastapi import APIRouter, status
from pydantic import BaseModel, Field

tenants_router = APIRouter(
    prefix="/api/v1/tenants",
    tags=["Tenants & Workspaces"]
)

class TenantSummaryDto(BaseModel):
    id: str
    name: str
    tier: str
    max_storage_gb: int

@tenants_router.get("/{tenant_id}", response_model=TenantSummaryDto)
async def get_tenant_by_id(tenant_id: str) -> TenantSummaryDto:
    """Retrieve workspace details and subscription quota for a given tenant."""
    return TenantSummaryDto(
        id=tenant_id,
        name="Acme Legal Partners",
        tier="Enterprise",
        max_storage_gb=500
    )

@tenants_router.post("", status_code=status.HTTP_201_CREATED, response_model=TenantSummaryDto)
async def provision_new_tenant(name: str = Field(min_length=3)) -> TenantSummaryDto:
    """Provision a brand new isolated tenant workspace."""
    return TenantSummaryDto(
        id="t-9872",
        name=name,
        tier="Pro",
        max_storage_gb=100
    )
```

#### Example 3.2: Documents Module Router with Ingestion & Inspection
```python
from fastapi import APIRouter, status
from pydantic import BaseModel

documents_router = APIRouter(
    prefix="/api/v1/documents",
    tags=["Document Ingestion & Knowledge Base"]
)

class DocumentDetailDto(BaseModel):
    document_id: str
    filename: str
    total_chunks: int
    is_embedded: bool

@documents_router.get("/{document_id}", response_model=DocumentDetailDto)
async def get_document_status(document_id: str) -> DocumentDetailDto:
    """Inspect processing status and pgvector chunk count for an ingested document."""
    return DocumentDetailDto(
        document_id=document_id,
        filename="Security_Compliance_2026.pdf",
        total_chunks=142,
        is_embedded=True
    )
```

#### Example 3.3: Conversations Module Router with RAG Sessions
```python
from fastapi import APIRouter, status
from pydantic import BaseModel, Field

conversations_router = APIRouter(
    prefix="/api/v1/conversations",
    tags=["Conversational AI & RAG"]
)

class StartSessionRequest(BaseModel):
    title: str = Field(default="New Conversation", max_length=100)
    system_prompt_override: str | None = None

class SessionResponseDto(BaseModel):
    session_id: str
    title: str
    created_at: str

@conversations_router.post("/sessions", status_code=status.HTTP_201_CREATED, response_model=SessionResponseDto)
async def create_chat_session(payload: StartSessionRequest) -> SessionResponseDto:
    """Initialize a stateful RAG conversational session."""
    return SessionResponseDto(
        session_id="conv-session-4512",
        title=payload.title,
        created_at="2026-11-18T16:00:00Z"
    )
```

#### Example 3.4: Admin Router with Global Module-Wide RBAC Dependency
You can enforce security checks on **all endpoints** inside an `APIRouter` by attaching dependencies to the router declaration:

```python
from fastapi import APIRouter, Depends, HTTPException, Header, status

async def verify_platform_superadmin(x_admin_key: str = Header(...)) -> None:
    # In production, validate cryptographic secret or Keycloak 'realm-admin' claim:
    if x_admin_key != "normora-superadmin-secret-2026":
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Forbidden: Platform superadmin privileges required."
        )

# All routes in this router require the 'verify_platform_superadmin' dependency:
admin_router = APIRouter(
    prefix="/api/v1/admin",
    tags=["Platform Administration"],
    dependencies=[Depends(verify_platform_superadmin)]
)

@admin_router.get("/metrics/tenants-usage")
async def get_all_tenants_storage_metrics() -> dict[str, int]:
    return {"total_tenants": 412, "active_storage_terabytes": 84}
```

#### Example 3.5: Assembling the Root App Router Tree in `main.py`
In `main.py`, mount each modular router into the primary FastAPI application:

```python
from fastapi import FastAPI
# from modules.tenants.api.tenants_router import tenants_router
# from modules.documents.api.documents_router import documents_router
# from modules.conversations.api.chat_router import conversations_router
# from modules.admin.api.admin_router import admin_router

app = FastAPI(title="Normora Modular Monolith")

# Mount modules with hierarchical clean URLs:
app.include_router(tenants_router)
app.include_router(documents_router)
app.include_router(conversations_router)
app.include_router(admin_router)

# All endpoints are now registered and documented in Swagger at /docs!
```

---

## 4. Parameter Binding Deep Dive: Path, Query, Header, Cookie, and Request Body

In ASP.NET Core, binding attributes (`[FromRoute]`, `[FromQuery]`, `[FromBody]`, `[FromHeader]`) tell the framework where to extract HTTP data.

In FastAPI, parameter binding is inferred from Python **type annotations** and explicit field functions (`Path()`, `Query()`, `Header()`, `Cookie()`, `Body()`):
* If a parameter is declared in the route path (`/tenants/{tenant_id}`), FastAPI binds it as a **Path parameter**.
* If a parameter is a scalar type (`str`, `int`, `float`, `bool`) and *not* in the path, FastAPI binds it as a **Query parameter**.
* If a parameter is a subclass of Pydantic's **`BaseModel`**, FastAPI binds it as a **JSON Request Body**.
* To extract headers or cookies, annotate with `Header()` or `Cookie()`.

---

### 5 Real-World Normora Examples: Parameter Binding

#### Example 4.1: Path Parameters with Constraints and Regex Validation
Validating Tenant IDs and Document IDs directly in the path:

```python
import uuid
from fastapi import APIRouter, Path

router = APIRouter()

@router.get("/api/v1/tenants/{tenant_id}/documents/{document_id}")
async def get_tenant_document_metadata(
    tenant_id: str = Path(
        ...,
        min_length=3,
        max_length=64,
        pattern=r"^[a-zA-Z0-9_\-]+$",
        description="Tenant identifier alphanumeric slug"
    ),
    document_id: uuid.UUID = Path(
        ...,
        description="Globally unique Document UUIDv4 identifier"
    )
):
    return {
        "tenant_id": tenant_id,
        "document_id": str(document_id),
        "status": "INDEXED"
    }
```

#### Example 4.2: Query Parameters for Semantic Vector Search
Handling query strings, optional filters, pagination, and multi-value lists:

```python
from fastapi import APIRouter, Query

router = APIRouter()

@router.get("/api/v1/documents/search")
async def semantic_search_knowledge_base(
    q: str = Query(
        ...,
        min_length=2,
        max_length=500,
        description="Natural language question or search query"
    ),
    similarity_threshold: float = Query(
        default=0.75,
        ge=0.0,
        le=1.0,
        description="pgvector cosine similarity cutoff threshold"
    ),
    top_k: int = Query(
        default=5,
        ge=1,
        le=50,
        description="Maximum number of context chunks to return"
    ),
    departments: list[str] = Query(
        default=["Legal", "Compliance"],
        description="Repeatable department filter (e.g. ?departments=Legal&departments=HR)"
    ),
    include_archived: bool = Query(
        default=False,
        description="Flag to include legacy archived documents"
    )
):
    return {
        "query": q,
        "cutoff": similarity_threshold,
        "limit": top_k,
        "filters": departments,
        "matches": 3
    }
```

#### Example 4.3: Request Body Binding with Nested Pydantic v2 Models
Binding and validating complex JSON payloads for document chunking:

```python
from fastapi import APIRouter, status
from pydantic import BaseModel, Field

router = APIRouter()

class ChunkingStrategyConfig(BaseModel):
    chunk_size: int = Field(default=512, ge=128, le=2048)
    chunk_overlap: int = Field(default=64, ge=0, le=256)
    split_by_headings: bool = True

class IngestKnowledgePayload(BaseModel):
    title: str = Field(min_length=3, max_length=200)
    department: str = Field(default="Engineering")
    raw_markdown: str = Field(min_length=20, description="Markdown content to chunk and vectorize")
    strategy: ChunkingStrategyConfig = Field(default_factory=ChunkingStrategyConfig)

@router.post("/api/v1/documents/ingest-markdown", status_code=status.HTTP_201_CREATED)
async def ingest_markdown_document(payload: IngestKnowledgePayload):
    # 'payload' is 100% validated by Pydantic before this function runs:
    return {
        "title": payload.title,
        "department": payload.department,
        "configured_chunk_size": payload.strategy.chunk_size,
        "status": "QUEUED"
    }
```

#### Example 4.4: Custom Headers (`X-Tenant-ID`, `X-Correlation-ID`, `Idempotency-Key`)
FastAPI automatically converts Python snake_case parameter names (`x_tenant_id`) to standard HTTP kebab-case (`X-Tenant-ID`):

```python
from fastapi import APIRouter, Header

router = APIRouter()

@router.post("/api/v1/conversations/messages")
async def send_chat_message(
    x_tenant_id: str = Header(..., description="Mandatory tenant isolation partition key"),
    x_correlation_id: str | None = Header(default=None, description="Client-side tracing ID"),
    idempotency_key: str | None = Header(default=None, description="Prevents duplicate billing turns")
):
    return {
        "tenant_id": x_tenant_id,
        "correlation_id": x_correlation_id,
        "idempotency_key": idempotency_key,
        "processed": True
    }
```

#### Example 4.5: Multi-Parameter Fusion (Path + Query + Header + Body)
Combining all parameter types cleanly into a single endpoint:

```python
from fastapi import APIRouter, Path, Query, Header, Body
from pydantic import BaseModel

router = APIRouter()

class ModelSettingsPayload(BaseModel):
    temperature: float = 0.2
    max_tokens: int = 4096

@router.put("/api/v1/tenants/{tenant_id}/ai-config")
async def update_tenant_ai_settings(
    tenant_id: str = Path(..., description="Tenant workspace target"),
    dry_run: bool = Query(default=False, description="Simulate without applying to database"),
    authorization: str = Header(..., description="Bearer JWT"),
    payload: ModelSettingsPayload = Body(...)
):
    return {
        "tenant": tenant_id,
        "dry_run_mode": dry_run,
        "new_temperature": payload.temperature,
        "authorized": authorization.startswith("Bearer ")
    }
```

---

## 5. Form Data & File Uploads (`UploadFile`, `File()`, `Form()`)

### 5.1 `bytes` vs `UploadFile`

When an Angular client uploads a 100MB PDF technical manual to Normora, using standard JSON with Base64 encoding inflates file size by ~33% and consumes massive memory. Instead, we use **`multipart/form-data`**.

FastAPI provides two ways to receive files:
1. **`bytes = File()`**: FastAPI reads the **entire file directly into server RAM**. If 20 users simultaneously upload 200MB video/audio files, the server will crash with an Out-of-Memory (OOM) error!
2. **`UploadFile = File()`**: FastAPI streams the file using a **spooled temporary file on disk** (`tempfile.SpooledTemporaryFile`). Small files (<1MB) stay in memory; larger files roll over directly to disk. It exposes an `async` file-like interface (`read()`, `write()`, `seek()`, `close()`), allowing you to stream bytes directly into MinIO S3 object storage without memory bloat.

| Feature | `bytes = File()` | `UploadFile = File()` |
| :--- | :--- | :--- |
| **Storage Location** | 100% in RAM | Spooled (RAM for small, Temp File on Disk for large) |
| **Memory Risk** | High Risk of OOM Crashes | Completely Safe for Multi-Gigabyte Files |
| **File Metadata** | None (only raw bytes) | Exposes `filename`, `content_type`, `headers` |
| **Streaming** | No | Yes (via `await file.read(chunk_size)`) |
| **ASP.NET Core Equivalent** | `byte[]` | `IFormFile` |

---

### 5 Real-World Normora Examples: File Uploads & Multipart Forms

#### Example 5.1: Single PDF Upload Streamed to MinIO Object Storage
```python
from fastapi import APIRouter, File, UploadFile, HTTPException, status

router = APIRouter(prefix="/api/v1/documents", tags=["Documents"])

# Maximum allowed upload size: 25 MB
MAX_FILE_SIZE_BYTES = 25 * 1024 * 1024

@router.post("/upload", status_code=status.HTTP_201_CREATED)
async def upload_document_for_ingestion(
    file: UploadFile = File(..., description="PDF or DOCX document to ingest")
) -> dict[str, str | int]:
    # 1. Validate MIME type:
    allowed_types = ["application/pdf", "application/vnd.openxmlformats-officedocument.wordprocessingml.document"]
    if file.content_type not in allowed_types:
        raise HTTPException(
            status_code=status.HTTP_415_UNSUPPORTED_MEDIA_TYPE,
            detail=f"Unsupported file format '{file.content_type}'. Only PDF and DOCX are allowed."
        )

    # 2. Stream chunked bytes to verify size without loading entire file to RAM:
    total_bytes_read = 0
    chunk_size = 1024 * 1024  # 1 MB chunk buffer
    
    while chunk := await file.read(chunk_size):
        total_bytes_read += len(chunk)
        if total_bytes_read > MAX_FILE_SIZE_BYTES:
            raise HTTPException(
                status_code=status.HTTP_413_REQUEST_ENTITY_TOO_LARGE,
                detail=f"File exceeds maximum allowed size of {MAX_FILE_SIZE_BYTES // (1024 * 1024)} MB."
            )
        # In production: Stream 'chunk' directly to MinIO via aiohttp / s3 client!

    # 3. Always close the file handle to release OS temporary files:
    await file.close()

    return {
        "filename": file.filename or "unknown.pdf",
        "content_type": file.content_type,
        "bytes_stored": total_bytes_read,
        "status": "STORED_IN_MINIO"
    }
```

#### Example 5.2: Batch Document Upload (`list[UploadFile]`)
Uploading an entire directory of company policies simultaneously:

```python
from fastapi import APIRouter, File, UploadFile, status

router = APIRouter(prefix="/api/v1/documents", tags=["Documents"])

@router.post("/batch-upload", status_code=status.HTTP_201_CREATED)
async def batch_upload_documents(
    files: list[UploadFile] = File(..., description="Collection of documents to ingest")
) -> dict[str, int | list[str]]:
    saved_filenames = []
    
    for upload in files:
        # Validate and persist each file:
        saved_filenames.append(upload.filename or "unnamed_document")
        await upload.close()

    return {
        "total_files_received": len(files),
        "manifest": saved_filenames
    }
```

#### Example 5.3: Multipart Upload Combining File with Metadata (`Form()`)
In enterprise applications, users upload a file alongside metadata fields (e.g. `department`, `classification_level`, `enable_ocr`):

```python
from fastapi import APIRouter, File, UploadFile, Form, status

router = APIRouter(prefix="/api/v1/documents", tags=["Documents"])

@router.post("/ingest-with-metadata", status_code=status.HTTP_201_CREATED)
async def ingest_document_with_form_metadata(
    file: UploadFile = File(..., description="Raw document file"),
    department: str = Form(..., description="Target department (e.g. Legal, HR)"),
    classification: str = Form(default="Internal", description="Confidential, Internal, Public"),
    enable_ocr: bool = Form(default=False, description="Run optical character recognition on scanned PDFs")
) -> dict[str, str | bool]:
    filename = file.filename or "unnamed"
    await file.close()

    return {
        "filename": filename,
        "department": department,
        "classification": classification,
        "enable_ocr": enable_ocr,
        "pipeline": "ENQUEUE_FOR_EXTRACTION"
    }
```

#### Example 5.4: Audio File Upload for Google Gemini Flash Audio Transcription
Uploading an audio recording of an executive meeting for direct transcription and conversational Q&A:

```python
from fastapi import APIRouter, File, UploadFile, HTTPException, status

router = APIRouter(prefix="/api/v1/audio", tags=["Audio & Speech"])

@router.post("/transcribe", status_code=status.HTTP_200_OK)
async def transcribe_meeting_audio(
    audio_file: UploadFile = File(..., description="Audio recording (WAV, MP3, M4A)")
) -> dict[str, str]:
    if audio_file.content_type not in ["audio/wav", "audio/mpeg", "audio/mp4", "audio/x-m4a"]:
        raise HTTPException(
            status_code=status.HTTP_415_UNSUPPORTED_MEDIA_TYPE,
            detail="Audio format not supported. Please upload WAV or MP3."
        )

    # In production: Read audio bytes and forward to Google Gemini Flash API for audio comprehension
    await audio_file.close()

    return {
        "filename": audio_file.filename or "meeting.wav",
        "transcript": "Normora Executive Board Meeting. Discussion on Q3 AI roadmap..."
    }
```

#### Example 5.5: Tenant Branding Asset Upload (Logo + Primary Color)
Allowing workspace administrators to customize their Angular 22 client UI theme:

```python
from fastapi import APIRouter, File, UploadFile, Form, HTTPException, status

router = APIRouter(prefix="/api/v1/tenants", tags=["Tenants"])

@router.post("/{tenant_id}/branding", status_code=status.HTTP_200_OK)
async def upload_tenant_branding(
    tenant_id: str,
    logo_image: UploadFile = File(..., description="Logo image (PNG or SVG)"),
    primary_color_hex: str = Form(..., pattern=r"^#(?:[0-9a-fA-F]{3}){1,2}$"),
    company_display_name: str = Form(..., min_length=2, max_length=50)
) -> dict[str, str]:
    if logo_image.content_type not in ["image/png", "image/svg+xml"]:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail="Logo must be an image/png or image/svg+xml."
        )
    
    await logo_image.close()

    return {
        "tenant_id": tenant_id,
        "company_name": company_display_name,
        "brand_color": primary_color_hex,
        "logo_status": "SAVED"
    }
```

---

## 6. Response Models, Status Codes, and Specialized Response Classes

### 6.1 The Power of `response_model`

In FastAPI, **`response_model`** is not just for Swagger documentation—it is an active **security firewall**:
1. **Data Filtering**: If your database entity contains `password_hash`, `stripe_customer_id`, or `internal_encryption_key`, omitting those fields from your Pydantic `response_model` guarantees they are **stripped from the JSON output**.
2. **Type Coercion**: Converts database ORM models, dictionaries, and dataclasses into compliant JSON types.
3. **OpenAPI Schema Generation**: Swagger UI accurately reflects the exact fields returned.

### 6.2 Specialized Response Classes

FastAPI endpoints do not only return JSON. Starlette provides specialized response classes:
* **`JSONResponse`**: Explicit JSON with manual status codes and headers.
* **`StreamingResponse`**: Streams async iterators (vital for Google Gemini Flash token-by-token generation and Server-Sent Events).
* **`FileResponse`**: High-performance asynchronous streaming of files from the server filesystem.
* **`HTMLResponse`**: Renders raw HTML strings (used for OAuth callbacks or diagnostics).

---

### 5 Real-World Normora Examples: Response Classes & Status Codes

#### Example 6.1: Data Scrubbing via `response_model`
Preventing sensitive database credentials from leaking to Angular clients:

```python
from fastapi import APIRouter
from pydantic import BaseModel, EmailStr

router = APIRouter(prefix="/api/v1/users", tags=["Users"])

# Internal database entity model (contains sensitive credentials):
class DbUserEntity:
    def __init__(self, user_id: str, email: str, password_hash: str, internal_tenant_key: str):
        self.user_id = user_id
        self.email = email
        self.password_hash = password_hash
        self.internal_tenant_key = internal_tenant_key

# Public response contract (excludes all secrets):
class UserPublicProfileDto(BaseModel):
    user_id: str
    email: EmailStr

@router.get("/{user_id}", response_model=UserPublicProfileDto)
async def get_user_profile(user_id: str):
    # Simulate DB fetch returning full entity with password hash:
    db_record = DbUserEntity(
        user_id=user_id,
        email="compliance_officer@normora.ai",
        password_hash="$2b$12$e8YkZ1XqG.5mKnOp9V9ZDe...",
        internal_tenant_key="sec_live_9812491284912"
    )
    # FastAPI automatically strips 'password_hash' and 'internal_tenant_key'!
    return db_record
```

#### Example 6.2: `StreamingResponse` for Gemini Flash LLM Token-by-Token RAG Chat
Streaming conversational answers to the Angular 22 client in real time:

```python
import asyncio
from typing import AsyncGenerator
from fastapi import APIRouter
from fastapi.responses import StreamingResponse

router = APIRouter(prefix="/api/v1/conversations", tags=["Conversations"])

async def mock_gemini_flash_token_stream(prompt: str) -> AsyncGenerator[str, None]:
    """Simulates token-by-token streaming from Google Gemini Flash."""
    tokens = [
        "Based ", "on ", "your ", "uploaded ", "handbook, ",
        "the ", "annual ", "remote-work ", "stipend ", "is ", "$1,500."
    ]
    for token in tokens:
        await asyncio.sleep(0.04)  # Simulate 40ms per-token LLM generation
        yield f"data: {token}\n\n"

@router.get("/sessions/{session_id}/stream")
async def stream_rag_conversation(session_id: str) -> StreamingResponse:
    """Streams RAG generation using Server-Sent Events (SSE)."""
    return StreamingResponse(
        mock_gemini_flash_token_stream("What is the remote work stipend?"),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "Connection": "keep-alive",
            "X-Accel-Buffering": "no"  # Prevents NGINX from buffering SSE stream
        }
    )
```

#### Example 6.3: `FileResponse` for Exported Audit Log Archives
Securely streaming a generated compliance report file to the user's browser:

```python
import os
from fastapi import APIRouter, HTTPException, status
from fastapi.responses import FileResponse

router = APIRouter(prefix="/api/v1/compliance", tags=["Compliance"])

@router.get("/exports/{export_id}/download")
async def download_compliance_export(export_id: str) -> FileResponse:
    file_path = f"/var/normora/exports/{export_id}.pdf"
    
    # In real world: check path traversal security and file existence:
    if not os.path.exists(file_path):
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="Export archive has expired or does not exist."
        )

    return FileResponse(
        path=file_path,
        filename=f"Normora_Audit_Report_{export_id}.pdf",
        media_type="application/pdf",
        headers={"Content-Disposition": f"attachment; filename=Normora_Audit_{export_id}.pdf"}
    )
```

#### Example 6.4: Resource Creation with `201 Created` and `Location` Header
```python
from fastapi import APIRouter, Response, status
from pydantic import BaseModel

router = APIRouter(prefix="/api/v1/conversations", tags=["Conversations"])

class CreateSessionDto(BaseModel):
    session_id: str
    title: str

@router.post("/sessions", status_code=status.HTTP_201_CREATED, response_model=CreateSessionDto)
async def create_chat_session(response: Response) -> CreateSessionDto:
    new_session_id = "session-7781"
    
    # Set standard REST Location header pointing to the new resource:
    response.headers["Location"] = f"/api/v1/conversations/sessions/{new_session_id}"
    
    return CreateSessionDto(session_id=new_session_id, title="Architecture Review")
```

#### Example 6.5: `204 No Content` for Document Purge
When purging a document and deleting its vector embeddings from pgvector, return `204 No Content`:

```python
from fastapi import APIRouter, Response, status

router = APIRouter(prefix="/api/v1/documents", tags=["Documents"])

@router.delete("/{document_id}", status_code=status.HTTP_204_NO_CONTENT)
async def purge_document(document_id: str) -> Response:
    # 1. Delete rows from PostgreSQL documents table
    # 2. Delete vectors from pgvector chunk table
    # 3. Delete raw PDF from MinIO bucket
    print(f"[PURGE] Document {document_id} completely eliminated across all stores.")
    
    # Must return an empty Response with HTTP 204:
    return Response(status_code=status.HTTP_204_NO_CONTENT)
```

---

## 7. Standardizing API Response Envelopes (`ApiResponse[T]`)

In an enterprise platform like Normora, API responses should never return bare JSON arrays or ad-hoc dictionaries. The Angular 22 frontend needs a standardized envelope so that HTTP interceptors can handle success states, extract payload data, and display toast notifications uniformly.

The Normora Standard Envelope:
```json
{
  "success": true,
  "data": { ... },
  "message": "Document ingested successfully.",
  "errors": null
}
```

---

### 5 Real-World Normora Examples: Standard Envelopes

#### Example 7.1: The Generic `ApiResponse[DataT]` Contract
Using Python 3.12+ / Pydantic v2 generics:

```python
from typing import Generic, TypeVar
from pydantic import BaseModel

DataT = TypeVar("DataT")

class ApiResponse(BaseModel, Generic[DataT]):
    success: bool
    data: DataT | None = None
    message: str | None = None
    errors: list[str] | None = None

    @classmethod
    def ok(cls, data: DataT, message: str = "Operation successful") -> "ApiResponse[DataT]":
        return cls(success=True, data=data, message=message, errors=None)

    @classmethod
    def fail(cls, message: str, errors: list[str] | None = None) -> "ApiResponse[None]":
        return cls(success=False, data=None, message=message, errors=errors)
```

#### Example 7.2: Returning Wrapped Single-Resource DTO
```python
from fastapi import APIRouter
from pydantic import BaseModel

router = APIRouter(prefix="/api/v1/tenants", tags=["Tenants"])

class TenantDetailDto(BaseModel):
    tenant_id: str
    company_name: str
    active_users: int

@router.get("/current", response_model=ApiResponse[TenantDetailDto])
async def get_current_tenant_workspace() -> ApiResponse[TenantDetailDto]:
    dto = TenantDetailDto(
        tenant_id="t-apex-01",
        company_name="Apex Healthcare",
        active_users=128
    )
    return ApiResponse.ok(data=dto, message="Tenant context retrieved.")
```

#### Example 7.3: Paginated Search Response Envelope (`PagedResult[T]`)
```python
from typing import Generic, TypeVar
from fastapi import APIRouter
from pydantic import BaseModel

ItemT = TypeVar("ItemT")

class PagedResult(BaseModel, Generic[ItemT]):
    items: list[ItemT]
    total_count: int
    page: int
    page_size: int
    total_pages: int

class DocumentChunkMatchDto(BaseModel):
    chunk_id: str
    similarity_score: float
    excerpt: str

router = APIRouter(prefix="/api/v1/search", tags=["Search"])

@router.get("", response_model=ApiResponse[PagedResult[DocumentChunkMatchDto]])
async def search_chunks() -> ApiResponse[PagedResult[DocumentChunkMatchDto]]:
    items = [
        DocumentChunkMatchDto(chunk_id="chk-1", similarity_score=0.89, excerpt="ISO 27001 Access Control Policy..."),
        DocumentChunkMatchDto(chunk_id="chk-2", similarity_score=0.84, excerpt="MFA requirements for cloud admins...")
    ]
    paged_data = PagedResult(
        items=items,
        total_count=42,
        page=1,
        page_size=10,
        total_pages=5
    )
    return ApiResponse.ok(data=paged_data, message="Search executed.")
```

#### Example 7.4: Standardized Error Envelope with Machine-Readable Codes
```python
from fastapi import APIRouter, status
from fastapi.responses import JSONResponse

router = APIRouter(prefix="/api/v1/billing", tags=["Billing"])

@router.post("/upgrade")
async def upgrade_tenant_subscription():
    # Simulating business rule validation failure:
    error_envelope = ApiResponse.fail(
        message="Subscription upgrade rejected: Invalid payment method.",
        errors=["CARD_EXPIRED", "PAYMENT_METHOD_DECLINED"]
    )
    return JSONResponse(
        status_code=status.HTTP_402_PAYMENT_REQUIRED,
        content=error_envelope.model_dump()
    )
```

#### Example 7.5: Consuming the Envelope in Angular 22 (Client-Side Interceptor Context)
Here is how the Angular 22 TypeScript frontend pairs with FastAPI's `ApiResponse[T]`:

```typescript
// Angular 22 TypeScript Data Contract:
export interface ApiResponse<T> {
  success: boolean;
  data: T | null;
  message: string | null;
  errors: string[] | null;
}

// Angular 22 Standalone HTTP Interceptor:
export const apiResponseInterceptor: HttpInterceptorFn = (req, next) => {
  return next(req).pipe(
    map(event => {
      if (event instanceof HttpResponse && event.body) {
        const body = event.body as ApiResponse<any>;
        if (!body.success) {
          throw new Error(body.message || 'Operation failed');
        }
      }
      return event;
    })
  );
};
```

---

## 8. Background Tasks (`BackgroundTasks`)

### 8.1 How `BackgroundTasks` Works

In ASP.NET Core, if you want to perform work *after* returning an HTTP response, you might push a message to an `IHostedService` via a `System.Threading.Channels.Channel<T>`.

In FastAPI, Starlette provides a built-in mechanism called **`BackgroundTasks`**. 
When you declare `background_tasks: BackgroundTasks` as an endpoint parameter, FastAPI executes the queued functions **immediately after the HTTP response has been sent to the client socket**.

```mermaid
sequenceDiagram
    autonumber
    actor Client as Angular 22 Client
    participant FastAPI as FastAPI Route Handler
    participant ClientSocket as Uvicorn HTTP Socket
    participant Worker as BackgroundTask Runner

    Client->>FastAPI: POST /api/v1/documents/upload (PDF)
    FastAPI->>FastAPI: Save raw file to MinIO
    FastAPI->>FastAPI: background_tasks.add_task(chunk_and_embed_document, doc_id)
    FastAPI->>ClientSocket: Send HTTP 202 Accepted {"status": "QUEUED"}
    ClientSocket-->>Client: Response Arrives at Client (UI Updates Immediately!)
    Note over Worker: Background execution starts AFTER socket flush
    Worker->>Worker: 1. Extract text from PDF
    Worker->>Worker: 2. Split text into 512-token chunks
    Worker->>Worker: 3. Call Google Gemini Embedding API
    Worker->>Worker: 4. Insert vectors into PostgreSQL pgvector
```

> [!NOTE]
> `BackgroundTasks` is ideal for lightweight post-response tasks (audit logging, email dispatch, cache invalidation, or fire-and-forget ingestion queuing). For mission-critical, durable multi-hour processing jobs that must survive server restarts, use an external distributed task queue like Celery, ARQ, or RabbitMQ.

---

### 5 Real-World Normora Examples: Background Tasks

#### Example 8.1: Asynchronous Document Chunking & Vectorization after HTTP 202
```python
from fastapi import APIRouter, BackgroundTasks, status
from pydantic import BaseModel
import asyncio

router = APIRouter(prefix="/api/v1/documents", tags=["Documents"])

async def run_document_chunking_pipeline(document_id: str, tenant_id: str) -> None:
    """Simulates background chunking, embedding generation, and pgvector persistence."""
    print(f"[BACKGROUND TASK] Starting text extraction for doc '{document_id}'...")
    await asyncio.sleep(2.0)  # Extract text
    print(f"[BACKGROUND TASK] Generating embeddings via Google Gemini...")
    await asyncio.sleep(1.5)  # Embedding API call
    print(f"[BACKGROUND TASK] Persisted 85 vectors to pgvector for tenant '{tenant_id}'. Complete!")

class IngestResponse(BaseModel):
    document_id: str
    status: str

@router.post("/ingest-async", status_code=status.HTTP_202_ACCEPTED, response_model=IngestResponse)
async def queue_document_ingestion(
    document_id: str,
    tenant_id: str,
    background_tasks: BackgroundTasks
) -> IngestResponse:
    # Queue task for background execution:
    background_tasks.add_task(run_document_chunking_pipeline, document_id, tenant_id)
    
    # HTTP 202 is returned to Angular immediately without waiting for chunking!
    return IngestResponse(document_id=document_id, status="QUEUED_FOR_PROCESSING")
```

#### Example 8.2: Audit Logging Security Events to PostgreSQL
Recording compliance events without penalizing user request latency:

```python
from fastapi import APIRouter, BackgroundTasks, Request

router = APIRouter(prefix="/api/v1/tenants", tags=["Tenants"])

async def record_security_audit_log(tenant_id: str, action: str, ip_address: str) -> None:
    # In production: write to security_audit_logs table via asyncpg
    print(f"[AUDIT LOG] Tenant: {tenant_id} | Action: {action} | IP: {ip_address}")

@router.post("/{tenant_id}/api-keys/rotate")
async def rotate_tenant_api_key(
    tenant_id: str,
    request: Request,
    background_tasks: BackgroundTasks
) -> dict[str, str]:
    client_ip = request.client.host if request.client else "unknown"
    
    # Queue audit log writing:
    background_tasks.add_task(record_security_audit_log, tenant_id, "API_KEY_ROTATED", client_ip)
    
    return {"status": "SUCCESS", "new_key": "sec_live_981273981274"}
```

#### Example 8.3: Sending Member Invitation Email via Keycloak/SMTP
```python
from fastapi import APIRouter, BackgroundTasks, status
from pydantic import BaseModel, EmailStr
import asyncio

router = APIRouter(prefix="/api/v1/tenants", tags=["Tenants"])

class InviteMemberRequest(BaseModel):
    email: EmailStr
    role: str

async def dispatch_invitation_email(recipient_email: str, workspace_name: str) -> None:
    print(f"[EMAIL SERVICE] Connecting to SMTP server...")
    await asyncio.sleep(0.5)
    print(f"[EMAIL SERVICE] Invitation email successfully sent to {recipient_email} for '{workspace_name}'")

@router.post("/{tenant_id}/members/invite", status_code=status.HTTP_200_OK)
async def invite_workspace_member(
    tenant_id: str,
    payload: InviteMemberRequest,
    background_tasks: BackgroundTasks
) -> dict[str, str]:
    background_tasks.add_task(dispatch_invitation_email, payload.email, "Acme Corp Workspace")
    return {"message": f"Invitation queued for {payload.email}"}
```

#### Example 8.4: Recalculating and Caching Tenant Vector Storage Quota
```python
from fastapi import APIRouter, BackgroundTasks

router = APIRouter(prefix="/api/v1/documents", tags=["Documents"])

async def recalculate_tenant_vector_usage(tenant_id: str) -> None:
    print(f"[QUOTA SERVICE] Recalculating pgvector index footprint for tenant {tenant_id}...")
    # SQL: SELECT SUM(pg_column_size(embedding)) FROM document_chunks WHERE tenant_id = ...
    print(f"[QUOTA SERVICE] Updated Redis quota cache for {tenant_id}: 42.8 MB used.")

@router.delete("/{document_id}/chunks")
async def delete_document_chunks(
    document_id: str,
    tenant_id: str,
    background_tasks: BackgroundTasks
) -> dict[str, str]:
    # Delete chunks from DB...
    background_tasks.add_task(recalculate_tenant_vector_usage, tenant_id)
    return {"message": f"Chunks for document {document_id} purged."}
```

#### Example 8.5: Webhook Push Notification to Angular SignalR Hub
Notifying frontend clients via an external SignalR or WebSocket gateway:

```python
import httpx
from fastapi import APIRouter, BackgroundTasks

router = APIRouter(prefix="/api/v1/webhooks", tags=["Notifications"])

async def notify_signalr_hub(channel: str, event_name: str, payload: dict) -> None:
    async with httpx.AsyncClient() as client:
        try:
            await client.post(
                f"http://signalr-service:5000/api/broadcast/{channel}",
                json={"event": event_name, "payload": payload},
                timeout=3.0
            )
        except Exception as e:
            print(f"[SIGNALR GATEWAY ERROR] Failed to push event: {e}")

@router.post("/trigger-report")
async def trigger_report_generation(tenant_id: str, background_tasks: BackgroundTasks):
    background_tasks.add_task(
        notify_signalr_hub,
        f"tenant-{tenant_id}",
        "REPORT_GENERATION_STARTED",
        {"tenant_id": tenant_id}
    )
    return {"status": "STARTED"}
```

---

## 9. Custom Middleware: Ordering, Execution Flow & ASGI

### 9.1 The Middleware Onion Model

Middleware wraps around your application like the layers of an onion. 

When a request arrives, it travels **downwards** through each middleware from outermost to innermost until it reaches the route handler. When the route handler returns a response, the response travels **upwards** in reverse order through each middleware.

```mermaid
flowchart TD
    subgraph Request Journey (Descending)
        Req[Incoming HTTP Request] --> M1_Req["Middleware 1: Correlation ID (Generate Trace ID)"]
        M1_Req --> M2_Req["Middleware 2: Timing Benchmark (Start Clock)"]
        M2_Req --> M3_Req["Middleware 3: Tenant Resolver (Extract X-Tenant-ID)"]
        M3_Req --> Endpoint["FastAPI Route Handler (Business Logic)"]
    end

    subgraph Response Journey (Ascending)
        Endpoint --> M3_Res["Middleware 3: Tenant Resolver (Clean Scope)"]
        M3_Res --> M2_Res["Middleware 2: Timing Benchmark (Append Server-Timing Header)"]
        M2_Res --> M1_Res["Middleware 1: Correlation ID (Append X-Correlation-ID Header)"]
        M1_Res --> Res[Final HTTP Response to Client]
    end
```

> [!IMPORTANT]
> In FastAPI, middlewares are executed in the **reverse order of their registration**! The last middleware added with `app.add_middleware()` is the first one to receive the incoming request.

---

### 5 Real-World Normora Examples: Custom Middleware

#### Example 9.1: Correlation ID Middleware (`X-Correlation-ID`)
Ensures every request has a tracking ID attached to `request.state` and returned in the HTTP response headers:

```python
import uuid
from fastapi import Request, Response
from starlette.middleware.base import BaseHTTPMiddleware, RequestResponseEndpoint

class CorrelationIdMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next: RequestResponseEndpoint) -> Response:
        # 1. Extract incoming correlation ID or generate a new UUIDv4:
        correlation_id = request.headers.get("X-Correlation-ID") or f"normora-{uuid.uuid4()}"
        
        # 2. Store on request.state so downstream handlers and dependencies can read it:
        request.state.correlation_id = correlation_id
        
        # 3. Process the inner application:
        response: Response = await call_next(request)
        
        # 4. Inject correlation ID into response headers:
        response.headers["X-Correlation-ID"] = correlation_id
        return response

app.add_middleware(CorrelationIdMiddleware)
```

#### Example 9.2: HTTP Execution Time & Performance Benchmark Middleware
Appends standard W3C `Server-Timing` headers so the Angular client can monitor backend latency:

```python
import time
from fastapi import Request, Response
from starlette.middleware.base import BaseHTTPMiddleware, RequestResponseEndpoint

class ServerTimingBenchmarkMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next: RequestResponseEndpoint) -> Response:
        start_time = time.perf_counter()
        
        response: Response = await call_next(request)
        
        duration_ms = (time.perf_counter() - start_time) * 1000.0
        response.headers["Server-Timing"] = f"total;dur={duration_ms:.2f}"
        response.headers["X-Response-Time-Ms"] = f"{duration_ms:.2f}"
        return response

app.add_middleware(ServerTimingBenchmarkMiddleware)
```

#### Example 9.3: Tenant Context Resolution Middleware
Validates that incoming requests specify a valid tenant header and stores it on `request.state`:

```python
from fastapi import Request, Response, status
from fastapi.responses import JSONResponse
from starlette.middleware.base import BaseHTTPMiddleware, RequestResponseEndpoint

class TenantHeaderResolutionMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next: RequestResponseEndpoint) -> Response:
        # Skip tenant check for public endpoints:
        public_paths = ["/health", "/api/docs", "/api/redoc", "/api/openapi.json"]
        if request.url.path in public_paths:
            return await call_next(request)

        tenant_id = request.headers.get("X-Tenant-ID")
        if not tenant_id:
            return JSONResponse(
                status_code=status.HTTP_400_BAD_REQUEST,
                content={
                    "success": False,
                    "message": "Missing mandatory 'X-Tenant-ID' header.",
                    "errors": ["MISSING_TENANT_HEADER"]
                }
            )

        # Attach tenant ID to request state:
        request.state.tenant_id = tenant_id
        return await call_next(request)

app.add_middleware(TenantHeaderResolutionMiddleware)
```

#### Example 9.4: Global Security Headers Middleware
Enforces OWASP recommended HTTP security headers (HSTS, CSP, X-Frame-Options):

```python
from fastapi import Request, Response
from starlette.middleware.base import BaseHTTPMiddleware, RequestResponseEndpoint

class SecurityHardeningHeadersMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next: RequestResponseEndpoint) -> Response:
        response: Response = await call_next(request)
        
        # Enforce strict transport and browser isolation policies:
        response.headers["Strict-Transport-Security"] = "max-age=31536000; includeSubDomains; preload"
        response.headers["X-Content-Type-Options"] = "nosniff"
        response.headers["X-Frame-Options"] = "DENY"
        response.headers["X-XSS-Protection"] = "1; mode=block"
        response.headers["Content-Security-Policy"] = "default-src 'self'; frame-ancestors 'none';"
        response.headers["Referrer-Policy"] = "strict-origin-when-cross-origin"
        
        return response

app.add_middleware(SecurityHardeningHeadersMiddleware)
```

#### Example 9.5: Sliding-Window Rate Limiting Middleware
Protects the Google Gemini Flash API budget by rate-limiting calls per tenant:

```python
import time
from collections import defaultdict
from fastapi import Request, Response, status
from fastapi.responses import JSONResponse
from starlette.middleware.base import BaseHTTPMiddleware, RequestResponseEndpoint

class InMemorySlidingWindowRateLimiter(BaseHTTPMiddleware):
    def __init__(self, app, max_requests_per_minute: int = 120):
        super().__init__(app)
        self.max_requests = max_requests_per_minute
        # Dictionary mapping tenant_id -> list of timestamps:
        self.request_history: dict[str, list[float]] = defaultdict(list)

    async def dispatch(self, request: Request, call_next: RequestResponseEndpoint) -> Response:
        tenant_id = request.headers.get("X-Tenant-ID", "global")
        now = time.time()
        one_minute_ago = now - 60.0

        # Filter out requests older than 60 seconds:
        self.request_history[tenant_id] = [
            ts for ts in self.request_history[tenant_id] if ts > one_minute_ago
        ]

        if len(self.request_history[tenant_id]) >= self.max_requests:
            return JSONResponse(
                status_code=status.HTTP_429_TOO_MANY_REQUESTS,
                content={
                    "success": False,
                    "message": "Tenant rate limit exceeded. Please throttle requests.",
                    "errors": ["RATE_LIMIT_EXCEEDED"]
                },
                headers={"Retry-After": "60"}
            )

        self.request_history[tenant_id].append(now)
        return await call_next(request)

app.add_middleware(InMemorySlidingWindowRateLimiter, max_requests_per_minute=100)
```

---

## 10. Global Exception Handling & Enterprise Security Envelopes

### 10.1 Security Principles: BOLA & BFLA Defense

In Normora:
* **BOLA Defense (`BolaException`)**: Broken Object-Level Authorization occurs when a user accesses an entity ID belonging to a different tenant. **We map `BolaException` to `404 Not Found` (never 403 Forbidden)**. Returning 403 reveals that the resource exists, enabling attacker enumeration attacks!
* **BFLA Defense (`BflaException`)**: Broken Function-Level Authorization occurs when a user has a valid tenant membership but lacks permissions for an operation (e.g., a Member trying to delete a Tenant). Maps to **`403 Forbidden`**.
* **Request Validation Errors (`RequestValidationError`)**: Overridden to return Normora's standard `ApiResponse` envelope.
* **Sanitized 500 Catch-All**: Never leak Python tracebacks, database connection strings, or internal file paths to the client!

---

### 5 Real-World Normora Examples: Exception Handlers

#### Example 10.1: Production `BolaException` Handler (Preventing Resource Enumeration)
```python
from fastapi import FastAPI, Request, status
from fastapi.responses import JSONResponse

class BolaException(Exception):
    """Raised when an object exists, but does not belong to the calling tenant."""
    def __init__(self, resource_type: str = "Resource"):
        self.resource_type = resource_type

app = FastAPI()

@app.exception_handler(BolaException)
async def bola_security_exception_handler(request: Request, exc: BolaException) -> JSONResponse:
    # 1. Log the attempted security violation internally with caller IP:
    client_ip = request.client.host if request.client else "unknown"
    print(
        f"[SECURITY AUDIT ALERT] BOLA violation attempted on {request.method} {request.url.path} "
        f"from IP: {client_ip} | Resource: {exc.resource_type}"
    )

    # 2. Return HTTP 404 NOT FOUND (simulating non-existence):
    return JSONResponse(
        status_code=status.HTTP_404_NOT_FOUND,
        content={
            "success": False,
            "data": None,
            "message": f"{exc.resource_type} not found.",
            "errors": ["NOT_FOUND"]
        }
    )
```

#### Example 10.2: Production `BflaException` Handler
```python
class BflaException(Exception):
    """Raised when the user lacks the required RBAC role (e.g. Member attempting Admin operation)."""
    def __init__(self, required_role: str, action: str):
        self.required_role = required_role
        self.action = action

@app.exception_handler(BflaException)
async def bfla_security_exception_handler(request: Request, exc: BflaException) -> JSONResponse:
    return JSONResponse(
        status_code=status.HTTP_403_FORBIDDEN,
        content={
            "success": False,
            "data": None,
            "message": f"Forbidden: Action '{exc.action}' requires '{exc.required_role}' privileges.",
            "errors": ["INSUFFICIENT_PERMISSIONS"]
        }
    )
```

#### Example 10.3: Overriding FastAPI's Default `RequestValidationError`
By default, FastAPI returns validation errors in an internal format. We override it to match our standardized `ApiResponse` envelope:

```python
from fastapi.exceptions import RequestValidationError

@app.exception_handler(RequestValidationError)
async def validation_exception_handler(request: Request, exc: RequestValidationError) -> JSONResponse:
    # Extract clean field-level error messages:
    error_details = []
    for error in exc.errors():
        field_loc = " -> ".join(str(loc) for loc in error["loc"] if loc != "body")
        error_details.append(f"{field_loc}: {error['msg']}")

    return JSONResponse(
        status_code=status.HTTP_400_BAD_REQUEST,
        content={
            "success": False,
            "data": None,
            "message": "Input validation failed on incoming request payload.",
            "errors": error_details
        }
    )
```

#### Example 10.4: Database Concurrency & Transient Error Handler
Handling PostgreSQL connection timeouts and lock deadlocks gracefully:

```python
import asyncpg

@app.exception_handler(asyncpg.PostgresError)
async def postgres_transient_error_handler(request: Request, exc: asyncpg.PostgresError) -> JSONResponse:
    print(f"[DATABASE ERROR] Query failed on {request.url.path}: {exc}")
    
    return JSONResponse(
        status_code=status.HTTP_503_SERVICE_UNAVAILABLE,
        content={
            "success": False,
            "data": None,
            "message": "The database is temporarily unable to complete your request. Please retry.",
            "errors": ["DATABASE_TRANSIENT_FAILURE"]
        },
        headers={"Retry-After": "3"}
    )
```

#### Example 10.5: Unhandled Critical Exception Catch-All Handler
Guarantees that an uncaught exception never leaks database connection strings or stack traces:

```python
@app.exception_handler(Exception)
async def global_catchall_unhandled_exception_handler(request: Request, exc: Exception) -> JSONResponse:
    # 1. Log the full traceback to server console / OpenTelemetry:
    print(f"[CRITICAL UNHANDLED EXCEPTION] Path: {request.url.path} | Error: {type(exc).__name__}: {exc}")

    # 2. Return a strictly sanitized response:
    return JSONResponse(
        status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
        content={
            "success": False,
            "data": None,
            "message": "An unexpected internal server error occurred. Our engineering team has been alerted.",
            "errors": ["INTERNAL_SERVER_ERROR"]
        }
    )
```

---

## 11. OpenAPI Customization & Interactive Documentation

FastAPI generates **OpenAPI 3.1.0** specifications automatically from your Python type hints.

You can customize:
* Security schemes (Keycloak Bearer JWT OIDC).
* Rich markdown descriptions for route tags.
* Endpoint deprecation notices (`deprecated=True`).
* Swagger UI interactive behavior (collapsing endpoints, display duration).

---

### 5 Real-World Normora Examples: OpenAPI Customization

#### Example 11.1: Enterprise OpenAPI Schema with Keycloak OIDC Security Scheme
```python
from fastapi import FastAPI
from fastapi.openapi.utils import get_openapi

app = FastAPI(title="Normora Platform API")

def custom_openapi():
    if app.openapi_schema:
        return app.openapi_schema
    
    openapi_schema = get_openapi(
        title="Normora Enterprise AI Knowledge API",
        version="1.0.0",
        description="""
        ## Normora Platform API
        Welcome to the official Normora Conversational AI and Document RAG Engine.
        * **Authentication**: All endpoints require a valid Keycloak Bearer JWT.
        * **Tenant Isolation**: Mandatory `X-Tenant-ID` header required on all calls.
        """,
        routes=app.routes,
    )
    
    # Define Keycloak Bearer JWT security scheme:
    openapi_schema["components"]["securitySchemes"] = {
        "KeycloakBearerAuth": {
            "type": "http",
            "scheme": "bearer",
            "bearerFormat": "JWT",
            "description": "Enter your Keycloak access token."
        }
    }
    # Apply security globally to all endpoints:
    openapi_schema["security"] = [{"KeycloakBearerAuth": []}]
    
    app.openapi_schema = openapi_schema
    return app.openapi_schema

app.openapi = custom_openapi
```

#### Example 11.2: Rich Tag Metadata with Group Descriptions
```python
tags_metadata = [
    {
        "name": "Tenants",
        "description": "Tenant workspace provisioning, seat management, quotas, and membership RBAC.",
        "externalDocs": {
            "description": "Tenant Isolation Whitepaper",
            "url": "https://normora.ai/docs/tenancy",
        },
    },
    {
        "name": "Documents",
        "description": "Document ingestion pipeline: text extraction, semantic chunking, and pgvector embeddings.",
    },
    {
        "name": "Conversations",
        "description": "RAG conversational chat sessions, streaming tokens, citations, and tool execution.",
    },
]

app = FastAPI(openapi_tags=tags_metadata)
```

#### Example 11.3: Documenting Deprecated Endpoints with Sunset Notices
```python
from fastapi import APIRouter

router = APIRouter(prefix="/api/v1/search", tags=["Search"])

@router.get(
    "/legacy-keyword-search",
    deprecated=True,
    summary="Legacy keyword full-text search (Deprecated)",
    description="""
    > [!WARNING]
    > **Sunset Notice**: This endpoint is deprecated and will be removed in v2.0.0.
    > Please migrate to `/api/v1/documents/search` for semantic vector search.
    """
)
async def legacy_keyword_search(keyword: str):
    return {"warning": "Migrate to semantic search", "results": []}
```

#### Example 11.4: Documenting Multiple Status Codes & Custom Responses
```python
from fastapi import APIRouter, status
from pydantic import BaseModel

router = APIRouter(prefix="/api/v1/documents", tags=["Documents"])

class DocumentDto(BaseModel):
    id: str
    title: str

class ErrorResponseDto(BaseModel):
    success: bool = False
    message: str
    errors: list[str]

@router.get(
    "/{document_id}",
    response_model=DocumentDto,
    responses={
        status.HTTP_200_OK: {
            "description": "Document metadata successfully retrieved."
        },
        status.HTTP_404_NOT_FOUND: {
            "model": ErrorResponseDto,
            "description": "Document not found or belongs to another tenant (BOLA defense)."
        },
        status.HTTP_403_FORBIDDEN: {
            "model": ErrorResponseDto,
            "description": "User lacks READ permissions for this document."
        }
    }
)
async def get_document(document_id: str):
    return DocumentDto(id=document_id, title="Architecture.pdf")
```

#### Example 11.5: Conditional Documentation Disabling in Production
In hardened enterprise environments, public exposure of Swagger UI can be completely disabled via environment flags:

```python
import os
from fastapi import FastAPI

ENVIRONMENT = os.getenv("NORMORA_ENV", "production")

# Disable Swagger UI and OpenAPI schema in production if required by security compliance:
enable_docs = ENVIRONMENT != "production"

app = FastAPI(
    title="Normora Engine",
    docs_url="/docs" if enable_docs else None,
    redoc_url="/redoc" if enable_docs else None,
    openapi_url="/openapi.json" if enable_docs else None
)
```

---

## 12. Comprehensive Architectural Comparison: FastAPI vs ASP.NET Core

Here is a side-by-side comparison of a production Document Ingestion Endpoint in both languages:

### ASP.NET Core 10 C# Implementation:
```csharp
namespace Normora.Modules.Documents.API;

[ApiController]
[Route("api/v1/documents")]
[Authorize]
public class DocumentsController(IMediator mediator) : ControllerBase
{
    [HttpPost("ingest")]
    [Consumes("multipart/form-data")]
    [ProducesResponseType(typeof(ApiResponse<IngestDocumentResultDto>), StatusCodes.Status202Accepted)]
    [ProducesResponseType(typeof(ApiResponse<object>), StatusCodes.Status400BadRequest)]
    public async Task<IActionResult> Ingest(
        [FromForm] IngestDocumentCommand command, 
        CancellationToken cancellationToken)
    {
        var result = await mediator.Send(command, cancellationToken);
        return Accepted(ApiResponse<IngestDocumentResultDto>.Ok(result, "Queued for processing."));
    }
}
```

### FastAPI Python 3.12+ Implementation:
```python
from fastapi import APIRouter, UploadFile, File, Form, BackgroundTasks, status, Depends
from pydantic import BaseModel

router = APIRouter(prefix="/api/v1/documents", tags=["Documents"])

class IngestResultDto(BaseModel):
    document_id: str
    status: str

@router.post(
    "/ingest",
    status_code=status.HTTP_202_ACCEPTED,
    response_model=ApiResponse[IngestResultDto]
)
async def ingest_document(
    background_tasks: BackgroundTasks,
    file: UploadFile = File(..., description="PDF or DOCX document"),
    department: str = Form(default="General"),
    tenant_context: TenantContext = Depends(get_current_tenant)
) -> ApiResponse[IngestResultDto]:
    document_id = await save_raw_file_to_minio(file, tenant_context.tenant_id)
    background_tasks.add_task(process_chunking_and_embedding, document_id, tenant_context.tenant_id)
    
    result = IngestResultDto(document_id=document_id, status="QUEUED")
    return ApiResponse.ok(data=result, message="Queued for processing.")
```

---

## 13. Common Mistakes & Anti-Patterns in FastAPI

> [!CAUTION]
> ### Anti-Pattern 1: Blocking the Async Event Loop with Synchronous I/O
> **The Mistake:** Using `requests.get()`, `time.sleep()`, or synchronous database drivers (`psycopg2`) inside an `async def` endpoint.
> ```python
> # CRITICAL BUG: This halts the entire Uvicorn event loop for ALL concurrent users!
> @app.get("/api/v1/ai/status")
> async def check_ai_status():
>     time.sleep(2.0)                 # BLOCKS EVENT LOOP!
>     resp = requests.get("https://api.gemini.google.com") # BLOCKS EVENT LOOP!
>     return resp.json()
> ```
> **The Fix:** Always use asynchronous alternatives (`asyncio.sleep()`, `httpx.AsyncClient()`, `asyncpg`):
> ```python
> @app.get("/api/v1/ai/status")
> async def check_ai_status():
>     await asyncio.sleep(2.0)        # Non-blocking yield!
>     async with httpx.AsyncClient() as client:
>         resp = await client.get("https://api.gemini.google.com")
>         return resp.json()
> ```

> [!CAUTION]
> ### Anti-Pattern 2: The `def` vs `async def` Threadpool Trap
> **The Mistake:** Not knowing that FastAPI executes regular `def` functions in a background threadpool (`starlette.concurrency.run_in_threadpool`), while `async def` functions run directly on the main event loop.
> * If an endpoint does CPU-bound work or calls a legacy blocking library: define it as **`def`** so FastAPI offloads it to a worker thread.
> * If an endpoint uses `await`: define it as **`async def`**.
> * Never call blocking code inside `async def`!

> [!CAUTION]
> ### Anti-Pattern 3: Exfiltrating Sensitive Database Fields by Omitting `response_model`
> **The Mistake:** Returning ORM models directly without declaring a strict `response_model`.
> ```python
> # DANGEROUS: Leaks password_hash and internal API keys to client JSON!
> @app.get("/api/v1/members/{member_id}")
> async def get_member(member_id: str):
>     return await db.fetch_user(member_id)
> ```
> **The Fix:** Always specify a dedicated Pydantic DTO as `response_model`:
> ```python
> @app.get("/api/v1/members/{member_id}", response_model=MemberPublicDto)
> async def get_member(member_id: str):
>     return await db.fetch_user(member_id)
> ```

> [!CAUTION]
> ### Anti-Pattern 4: Memory Exhaustion via `.read()` on Large File Uploads
> **The Mistake:** Calling `await file.read()` without a chunk size on user uploads.
> ```python
> # CRITICAL VULNERABILITY: An attacker uploads a 5GB ISO, crashing the server!
> @app.post("/upload")
> async def upload(file: UploadFile = File(...)):
>     content = await file.read()     # Loads entire 5GB into RAM! OOM Crash!
> ```
> **The Fix:** Read in small buffers (e.g., 1MB) and stream directly to disk or MinIO:
> ```python
> @app.post("/upload")
> async def upload(file: UploadFile = File(...)):
>     while chunk := await file.read(1024 * 1024):  # 1MB buffer
>         await minio_stream.write(chunk)
> ```

> [!CAUTION]
> ### Anti-Pattern 5: Using In-Memory `BackgroundTasks` for Mission-Critical Durable Jobs
> **The Mistake:** Assuming `BackgroundTasks` is a replacement for Celery or RabbitMQ.
> If the Kubernetes pod restarts or crashes while a `BackgroundTask` is executing, **the task is lost forever**. Use `BackgroundTasks` only for transient, non-critical post-processing. Use Celery / ARQ / Redis for durable financial or document processing pipelines.

---

## 14. Architectural Interview Questions & Answers

### Q1: What is the ASGI specification, and how does FastAPI's concurrency model differ from ASP.NET Core Kestrel?
**Answer:** ASGI (Asynchronous Server Gateway Interface) is the standard asynchronous interface between Python web servers (like Uvicorn) and Python web applications (like FastAPI). Its callable signature is `async def app(scope, receive, send)`. 

While ASP.NET Core Kestrel uses managed C# OS threads combined with the .NET ThreadPool and I/O Completion Ports, Uvicorn on Linux runs a single-threaded C event loop (`uvloop` based on `libuv`). Concurrency is cooperative and non-blocking: I/O operations yield control back to the event loop via `await`. Multi-core CPU scaling in Python is achieved by running multiple worker processes (typically 1 worker per CPU core), whereas Kestrel runs a single process with multi-threaded parallel execution.

### Q2: Why does Normora return HTTP 404 instead of 403 when a BOLA violation occurs?
**Answer:** In multi-tenant systems, Broken Object-Level Authorization (BOLA) occurs when Tenant A attempts to access an entity ID belonging to Tenant B. If the API returns `403 Forbidden`, it confirms to the attacker that the resource ID exists, enabling brute-force ID enumeration attacks. Returning `404 Not Found` completely hides the existence of the resource, making enumeration impossible.

### Q3: What is the difference between `bytes = File()` and `UploadFile = File()`?
**Answer:** Declaring `bytes = File()` forces FastAPI to read the entire upload payload directly into RAM, creating an acute risk of Out-of-Memory (OOM) denial-of-service crashes on large files. `UploadFile` utilizes a spooled temporary file on disk (`SpooledTemporaryFile`), storing files larger than 1MB on the filesystem. It exposes an async stream interface (`await file.read(chunk_size)`), allowing safe streaming of multi-gigabyte files into object storage like MinIO.

### Q4: How does middleware execution ordering work in FastAPI?
**Answer:** FastAPI uses the Starlette middleware onion architecture. Middlewares execute in the **reverse order of their declaration**. When a request arrives, the last middleware added via `app.add_middleware()` executes first on the request phase. When the response returns from the endpoint, it flows back up through the middlewares in reverse order.

### Q5: Why is the `lifespan` context manager preferred over legacy startup/shutdown events?
**Answer:** Legacy `@app.on_event("startup")` and `@app.on_event("shutdown")` are deprecated because they lack shared state encapsulation and do not provide clean exception boundaries. The modern `lifespan` pattern uses Python's standard `asynccontextmanager`. Code before `yield` initializes resources (database pools, MinIO buckets, Gemini clients) and stores them on `app.state`. Code after `yield` executes during graceful shutdown, ensuring in-flight tasks and connection pools are properly closed.

### Q6: When should you use FastAPI's `BackgroundTasks` versus an external queue like Celery or RabbitMQ?
**Answer:** `BackgroundTasks` runs in-memory inside the same Uvicorn worker process immediately after sending the HTTP response. It is ideal for lightweight, non-durable operations like audit logging, email dispatch, or cache invalidation. However, because it is in-memory, tasks are lost if the server process restarts. For mission-critical, long-running, or retry-heavy tasks (such as 100-page PDF OCR and vectorization), an external persistent message broker with dedicated worker processes is required.

### Q7: What happens when an endpoint is declared as `def` instead of `async def` in FastAPI?
**Answer:** When an endpoint is declared as regular `def`, FastAPI does not run it on the main event loop. Instead, it offloads the function call to an external threadpool (`starlette.concurrency.run_in_threadpool`). This prevents blocking operations (like legacy synchronous database queries or heavy CPU computations) from halting the async event loop for other concurrent requests. Conversely, `async def` functions run directly on the event loop and must strictly avoid blocking calls.

---

## Summary & Next Step

You have mastered the foundational core architecture of FastAPI:
1. **ASGI & Uvicorn**: The asynchronous socket engine powering high-throughput Python web services.
2. **Lifespan Management**: Safe resource initialization and teardown via `asynccontextmanager`.
3. **Modular Monolith Architecture**: Organizing clean CQRS modules with `APIRouter`.
4. **Parameter Binding**: Seamless extraction of Path, Query, Header, Cookie, and Request Body parameters.
5. **Streaming File Uploads**: Memory-safe multipart uploads with `UploadFile` and `Form()`.
6. **Response Serialization**: Data security scrubbing with `response_model` and real-time streaming with `StreamingResponse`.
7. **Enterprise Response Envelopes**: Standardizing JSON payloads with `ApiResponse[T]`.
8. **Background Tasks**: Fire-and-forget post-response processing with `BackgroundTasks`.
9. **Custom Middleware**: The execution onion, correlation IDs, and rate limiting.
10. **Global Exception Handling**: Defensive security against BOLA (404) and BFLA (403).
11. **OpenAPI Customization**: Custom security schemes and interactive Swagger UI documentation.

In **Module 14**, we dive deep into FastAPI's crown jewel: [Module 14: FastAPI Dependency Injection and Tenant Security](file:///c:/Users/Sanjeev%20Kumar/Codes/learning-docs/Python_FastAPI_AI_Architect/14_Module_14_Dependency_Injection_and_Security.md)!
