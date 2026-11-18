# Module 13: FastAPI Core Architecture & Web Engineering

Welcome to **Module 13** of the Normora Python & AI Masterclass.

Now that you have mastered the Python async event loop (Module 10) and Pydantic v2 data contracts (Module 12), we bring them together into **FastAPI**—the fastest, most ergonomic Python web framework in existence.

In this module, you will learn the physical architecture of FastAPI:
* The **ASGI (Asynchronous Server Gateway Interface)** specification and how Uvicorn runs worker processes.
* The complete **HTTP Request Lifecycle** inside FastAPI.
* Parameter binding: Path parameters, Query parameters, Request Bodies, Headers, and Cookies.
* Response serialization and status codes.
* Global exception handling and overriding validation error envelopes.

Every concept is followed by **at least 5 real-world Normora examples**.

---

## 1. The ASGI Specification & Uvicorn Web Server

In the older Python web generation (Flask, Django), servers used **WSGI (Web Server Gateway Interface)**. WSGI was strictly synchronous: one request per worker thread. 

FastAPI is built on **ASGI (Asynchronous Server Gateway Interface)**.  
An ASGI application is simply a single async callable with the signature:
```python
async def app(scope: dict, receive: callable, send: callable):
    ...
```
* **`scope`**: A dictionary containing connection metadata (HTTP method, headers, path, client IP).
* **`receive`**: An async function used to read incoming request body bytes or WebSocket frames.
* **`send`**: An async function used to stream response headers, status codes, and body chunks back to the client.

**Uvicorn** is the lightning-fast ASGI web server that manages OS sockets and feeds them into FastAPI's ASGI engine.

```mermaid
flowchart LR
    Client["Client Browser / Angular SPA"] <-->|HTTP / SSE / WebSocket| Uvicorn["Uvicorn ASGI Server<br>(uvloop / IOCP OS Sockets)"]
    Uvicorn <-->|ASGI: (scope, receive, send)| Starlette["Starlette Core<br>(Routing & Middleware)"]
    Starlette <-->|Validation & Contracts| Pydantic["Pydantic v2<br>(Rust Validation Engine)"]
    Starlette <-->|Dependency Graph| FastAPI["FastAPI Application Handlers"]
```

### 5 Real-World Normora Examples:

#### Example 1.1: The Minimal Production Normora FastAPI App
```python
from fastapi import FastAPI

app = FastAPI(
    title="Normora AI Engine API",
    version="1.0.0",
    description="Multi-tenant knowledge management and conversational AI API"
)

@app.get("/health", tags=["System"])
async def health_check():
    return {"status": "HEALTHY", "service": "Normora API Gateway"}
```

#### Example 1.2: Running with Uvicorn in Production
In production on a multi-core cloud VM, you run Uvicorn with multiple worker processes (one worker process per CPU core):
```bash
# Terminal command:
uvicorn main:app --host 0.0.0.0 --port 8000 --workers 4 --loop uvloop
```

#### Example 1.3: Inspecting the ASGI `scope` Object in Custom Middleware
```python
from starlette.types import ASGIApp, Scope, Receive, Send

class RequestLoggingAsgiMiddleware:
    def __init__(self, app: ASGIApp):
        self.app = app

    async def __call__(self, scope: Scope, receive: Receive, send: Send):
        if scope["type"] == "http":
            method = scope["method"]
            path = scope["path"]
            print(f"[ASGI AUDIT] Incoming {method} request to '{path}'")
        await self.app(scope, receive, send)

app.add_middleware(RequestLoggingAsgiMiddleware)
```

#### Example 1.4: Enabling CORS for the Angular 22 Frontend
```python
from fastapi.middleware.cors import CORSMiddleware

# Authorize your local Angular client and production domain:
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:4200", "https://app.normora.ai"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

#### Example 1.5: Organizing Routes with `APIRouter` (Modular Monolith Style)
In Normora, we organize routes into separate modules (`tenants`, `documents`, `conversations`):
```python
from fastapi import APIRouter

# server/Modules/Documents/API/documents_router.py
documents_router = APIRouter(prefix="/api/v1/documents", tags=["Documents"])

@documents_router.get("/")
async def list_documents():
    return [{"id": "doc-1", "title": "Handbook.pdf"}]

# In main.py:
app.include_router(documents_router)
```

---

## 2. Parameter Binding (Path, Query, Body & Headers)

FastAPI automatically parses and binds incoming HTTP data based on your type annotations and parameter declarations:
* **Path Parameters (`Path`)**: Extracted from URL path (`/tenants/{tenant_id}`).
* **Query Parameters (`Query`)**: Extracted from URL query string (`?page=1&limit=10`).
* **Request Body**: Extracted from incoming JSON payload (validated against a Pydantic model).
* **Headers (`Header`)**: Extracted from HTTP request headers.

### 5 Real-World Normora Examples:

#### Example 2.1: Path Parameters for Tenant Resource Lookup
```python
from fastapi import Path

@app.get("/api/v1/tenants/{tenant_id}")
async def get_tenant_details(
    tenant_id: str = Path(..., description="Unique tenant workspace identifier", min_length=3)
):
    return {"tenant_id": tenant_id, "name": "Acme Corp", "status": "ACTIVE"}
```

#### Example 2.2: Query Parameters for Document Search and Pagination
```python
from fastapi import Query

@app.get("/api/v1/documents/search")
async def search_company_knowledge(
    q: str = Query(..., min_length=2, max_length=200, description="Semantic search query"),
    department: str | None = Query(default=None, description="Optional department filter"),
    page: int = Query(default=1, ge=1, description="Page number"),
    page_size: int = Query(default=10, ge=1, le=50, description="Results per page")
):
    return {
        "query": q,
        "department": department,
        "page": page,
        "results": [f"Result matching '{q}'"]
    }
```

#### Example 2.3: Request Body Binding with Pydantic
```python
from pydantic import BaseModel, Field

class IngestDocumentDto(BaseModel):
    title: str = Field(min_length=1, max_length=255)
    department: str = Field(default="General")
    raw_content: str = Field(min_length=10)

@app.post("/api/v1/documents/ingest", status_code=201)
async def ingest_document(payload: IngestDocumentDto):
    # 'payload' is fully validated and parsed by Pydantic:
    return {
        "success": True,
        "message": f"Document '{payload.title}' queued for chunking and embedding."
    }
```

#### Example 2.4: Extracting Custom Headers (`X-Tenant-ID` and Correlation ID)
```python
from fastapi import Header

@app.get("/api/v1/ai/status")
async def get_ai_quota_status(
    x_tenant_id: str = Header(..., description="Mandatory tenant isolation header"),
    x_correlation_id: str | None = Header(default=None)
):
    return {
        "tenant_id": x_tenant_id,
        "correlation_id": x_correlation_id,
        "gemini_quota_remaining": 450000
    }
```

#### Example 2.5: Combining Path, Query, Body, and Headers in a Single Endpoint
```python
@app.put("/api/v1/tenants/{tenant_id}/branding")
async def update_tenant_branding(
    tenant_id: str = Path(...),
    dry_run: bool = Query(default=False),
    theme_payload: dict = ...,
    authorization: str = Header(...)
):
    return {
        "tenant_id": tenant_id,
        "dry_run": dry_run,
        "updated_theme": theme_payload
    }
```

---

## 3. Response Models & Status Codes

In FastAPI:
* `status_code`: Sets the HTTP status (e.g. `201 Created`, `204 No Content`).
* `response_model`: Defines the exact Pydantic schema used to serialize the response. **FastAPI automatically filters out any sensitive fields that are not in the response model!**

### 5 Real-World Normora Examples:

#### Example 3.1: Filtering Sensitive Data with `response_model`
Even if your database returns a password hash or internal secret, the response model strips it:
```python
class InternalUserModel:
    def __init__(self, user_id: str, email: str, password_hash: str):
        self.user_id = user_id
        self.email = email
        self.password_hash = password_hash

class UserPublicResponse(BaseModel):
    user_id: str
    email: str

@app.get("/api/v1/users/{user_id}", response_model=UserPublicResponse)
async def get_user_profile(user_id: str):
    # Simulating database record with sensitive data:
    db_user = InternalUserModel(user_id, "admin@normora.ai", "$2b$12$SuperSecretHash...")
    # FastAPI filters out 'password_hash' automatically!
    return db_user
```

#### Example 3.2: Standard `201 Created` on Resource Creation
```python
from fastapi import status

class CreateWorkspaceResponse(BaseModel):
    workspace_id: str
    status: str

@app.post("/api/v1/workspaces", status_code=status.HTTP_201_CREATED, response_model=CreateWorkspaceResponse)
async def create_workspace():
    return {"workspace_id": "WS-900", "status": "INITIALIZED"}
```

#### Example 3.3: `204 No Content` on Document Deletion
```python
from fastapi import Response

@app.delete("/api/v1/documents/{document_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_document(document_id: str):
    print(f"Purged document {document_id} from pgvector and storage.")
    return Response(status_code=status.HTTP_204_NO_CONTENT)
```

#### Example 3.4: Returning a Paginated List Response Model
```python
class DocumentListItemDto(BaseModel):
    id: str
    title: str
    department: str

@app.get("/api/v1/documents", response_model=list[DocumentListItemDto])
async def list_all_documents():
    return [
        {"id": "d1", "title": "Handbook.pdf", "department": "HR", "internal_storage_key": "s3://..."},
        {"id": "d2", "title": "Security.pdf", "department": "Legal", "internal_storage_key": "s3://..."}
    ]
```

#### Example 3.5: Dynamic Status Codes with `Response` parameter
```python
@app.post("/api/v1/conversations/turns")
async def execute_turn(response: Response, is_new_session: bool = False):
    if is_new_session:
        response.status_code = status.HTTP_201_CREATED
    else:
        response.status_code = status.HTTP_200_OK
    return {"reply": "Normora assistant response."}
```

---

## 4. Standardizing API Response Envelopes (`ApiResponse[T]`)

In enterprise platforms like Normora, all API responses must follow a consistent JSON envelope structure so the frontend interceptors in Angular can parse responses uniformly.

```json
{
  "success": true,
  "data": { ... },
  "message": "Operation completed successfully.",
  "errors": null
}
```

### 5 Real-World Normora Examples:

#### Example 4.1: The Generic `ApiResponse[T]` Model
```python
from typing import Generic, TypeVar

DataT = TypeVar("DataT")

class ApiResponse(BaseModel, Generic[DataT]):
    success: bool
    data: DataT | None = None
    message: str | None = None
    errors: list[str] | None = None

    @classmethod
    def ok(cls, data: DataT, message: str = "Success") -> "ApiResponse[DataT]":
        return cls(success=True, data=data, message=message, errors=None)

    @classmethod
    def fail(cls, message: str, errors: list[str] | None = None) -> "ApiResponse[None]":
        return cls(success=False, data=None, message=message, errors=errors)
```

#### Example 4.2: Using `ApiResponse` in an Endpoint
```python
class TenantSummaryDto(BaseModel):
    tenant_id: str
    name: str

@app.get("/api/v1/tenant/current", response_model=ApiResponse[TenantSummaryDto])
async def get_current_tenant():
    data = TenantSummaryDto(tenant_id="T-10", name="Apex Legal")
    return ApiResponse.ok(data=data, message="Tenant context retrieved.")
```

#### Example 4.3: Standardized List Response Envelope
```python
@app.get("/api/v1/departments/all", response_model=ApiResponse[list[str]])
async def list_departments():
    depts = ["Engineering", "Legal", "HR", "Finance"]
    return ApiResponse.ok(data=depts)
```

#### Example 4.4: Consistent Error Envelope Response
```python
@app.get("/api/v1/check-status", response_model=ApiResponse[None])
async def check_status():
    return ApiResponse.fail(message="System in maintenance mode.", errors=["DB_MIGRATION_IN_PROGRESS"])
```

#### Example 4.5: Automatic OpenAPI Documentation of the Generic Envelope
FastAPI automatically documents `ApiResponse[TenantSummaryDto]` in the generated Swagger UI at `/docs`, with full type definitions and examples!

---

## 5. Global Exception Handling (Normora Security Standards)

In Normora:
* **No Raw Stack Traces:** Stack traces belong strictly in structured server logs, never in HTTP responses.
* **BOLA Defense (`BolaException`):** Maps globally to **`404 Not Found`** to prevent resource enumeration.
* **BFLA Defense (`BflaException`):** Maps globally to **`403 Forbidden`**.

### 5 Real-World Normora Examples:

#### Example 5.1: Global `BolaException` Handler (Preventing ID Enumeration)
```python
from fastapi import Request
from fastapi.responses import JSONResponse

class BolaException(Exception):
    def __init__(self, resource: str = "Resource"):
        self.resource = resource

@app.exception_handler(BolaException)
async def bola_exception_handler(request: Request, exc: BolaException):
    # Log the security violation internally with caller IP:
    print(f"[SECURITY ALERT] BOLA violation attempted on {request.url.path} from {request.client.host}")
    # Return 404 to the client to simulate non-existence:
    return JSONResponse(
        status_code=404,
        content={"success": False, "message": f"{exc.resource} not found.", "errors": ["NOT_FOUND"]}
    )
```

#### Example 5.2: Global `BflaException` Handler
```python
class BflaException(Exception):
    def __init__(self, action: str):
        self.action = action

@app.exception_handler(BflaException)
async def bfla_exception_handler(request: Request, exc: BflaException):
    return JSONResponse(
        status_code=403,
        content={"success": False, "message": f"Forbidden: Insufficient privileges for {exc.action}.", "errors": ["FORBIDDEN"]}
    )
```

#### Example 5.3: Overriding FastAPI's Default `RequestValidationError`
By default, FastAPI returns validation errors in an internal format. We override it to match our standardized `ApiResponse` contract:
```python
from fastapi.exceptions import RequestValidationError

@app.exception_handler(RequestValidationError)
async def validation_exception_handler(request: Request, exc: RequestValidationError):
    error_messages = [f"{err['loc'][-1]}: {err['msg']}" for err in exc.errors()]
    return JSONResponse(
        status_code=400,
        content={
            "success": False,
            "message": "Validation failed on incoming request payload.",
            "errors": error_messages
        }
    )
```

#### Example 5.4: Centralized Rate Limit (429) Exception Handler
```python
class QuotaExceededException(Exception):
    pass

@app.exception_handler(QuotaExceededException)
async def quota_exception_handler(request: Request, exc: QuotaExceededException):
    return JSONResponse(
        status_code=429,
        content={
            "success": False,
            "message": "Monthly AI token budget has been exhausted. Please contact workspace administrator.",
            "errors": ["QUOTA_EXCEEDED"]
        }
    )
```

#### Example 5.5: Global Catch-All Unhandled Exception Handler
Guarantees that an uncaught 500 error never leaks database connection strings or stack traces:
```python
@app.exception_handler(Exception)
async def generic_catchall_handler(request: Request, exc: Exception):
    print(f"[UNHANDLED CRITICAL EXCEPTION] Path: {request.url.path} | Error: {exc}")
    return JSONResponse(
        status_code=500,
        content={
            "success": False,
            "message": "An unexpected internal server error occurred. Please try again later.",
            "errors": ["INTERNAL_SERVER_ERROR"]
        }
    )
```

---

## Summary & Next Step

You have mastered:
1. The ASGI specification, Uvicorn worker management, and FastAPI application structure.
2. Binding Path, Query, Body, and Header parameters.
3. Controlling status codes and filtering response data with `response_model`.
4. Creating standardized `ApiResponse[T]` envelopes.
5. Implementing global security exception handlers (`BolaException` -> 404, `BflaException` -> 403).

In **Module 14**, we master FastAPI's crown jewel: **Dependency Injection (`Depends`) and Tenant Security Architecture**!
