# Module 12: Data Validation with Pydantic v2 (The Rust Data Engine)

Welcome to **Module 12** of the Normora Python & AI Masterclass.

Generative AI applications are inherently non-deterministic: Large Language Models (LLMs) such as Google Gemini Flash output unstructured or semi-structured text, hallucinated JSON fields, or malformed tool call arguments. In a modern enterprise platform like **Normora**—which integrates multi-tenant knowledge management, pgvector similarity search, and automated conversational RAG pipelines—unvalidated data entering your domain logic will trigger catastrophic runtime failures, database corruption, or security vulnerabilities like Broken Object Level Authorization (BOLA).

**Pydantic v2** is the high-performance defensive boundary wall between unpredictable external data (HTTP JSON payloads, Keycloak OIDC claims, MinIO document metadata, LLM outputs) and your strictly typed Python application layer. Completely re-engineered from the ground up with a compiled **Rust engine (`pydantic-core`)**, Pydantic v2 parses and validates complex nested data structures at sub-millisecond speeds—up to 20x faster than Pydantic v1.

If you come from the .NET world, think of Pydantic as combining the best capabilities of **C# Records**, **FluentValidation**, **System.ComponentModel.DataAnnotations**, and **System.Text.Json source generators** into a single cohesive, high-speed framework.

---

## The Rust Data Engine Architecture

Before writing schemas, let us examine how Pydantic v2 processes data under the hood. In standard Python, type hints (such as `name: str` or `seats: int`) are purely informational annotations ignored by the Python runtime at execution. Pydantic bridges this gap by passing raw inputs directly to its underlying Rust engine, **`pydantic-core`**.

```mermaid
flowchart TD
    A["Untrusted Input: HTTP Request / Gemini JSON / MinIO Payload"] --> B["FastAPI Endpoint / Pipeline Worker"]
    B --> C["pydantic-core (Rust Engine)"]
    C --> D{"Schema Validation & Type Coercion"}
    D -- "Invalid Types / Failed Constraints" --> E["ValidationError (Structured 422 JSON Response)"]
    D -- "Valid Payload" --> F["Typed Pydantic BaseModel Instance"]
    F --> G["CQRS Handler / Database (PostgreSQL + pgvector)"]
```

### Parsing vs. Validating
A critical architectural philosophy of Pydantic is that it is a **data parsing engine**, not just a validator:
- **Validation** merely checks if data is valid (e.g., asserting that `active_seats` is of type integer).
- **Parsing** inspects the incoming data and coerces compatible types into their target domain representation (e.g., converting the incoming string `"45"` or float `45.0` into a native Python integer `45`).

If coercion fails (e.g., attempting to parse `"unlimited"` as an integer), Pydantic halts execution and generates a comprehensive `ValidationError` reporting the exact line, field name, and error message.

---

## Normora Pydantic Schema Hierarchy

Within the Normora architecture, schemas are organized into modular domains matching our CQRS architecture. Here is the class hierarchy that powers tenant management, document ingestion, and conversational RAG:

```mermaid
classDiagram
    class NormoraBaseModel {
        +ConfigDict model_config
    }
    class TenantCreateRequest {
        +str name
        +str slug
        +str subscription_plan
        +int active_seats
        +bool enforce_mfa
    }
    class TenantResponseDto {
        +str tenant_id
        +str company_name
        +int active_seats
        +datetime created_at
        +bool is_quota_exceeded
    }
    class DocumentChunkSchema {
        +str chunk_id
        +str document_id
        +str text_content
        +int token_count
        +list[float] embedding
        +int character_count
    }
    class StreamPacket {
        <<interface>>
        +str packet_type
    }
    class TextStreamPacket {
        +str packet_type = "text"
        +str token
    }
    class CitationStreamPacket {
        +str packet_type = "citation"
        +str document_id
        +int page_number
    }
    class AppSettings {
        +str ENVIRONMENT
        +SecretStr GEMINI_API_KEY
        +str POSTGRES_HOST
        +int POSTGRES_PORT
    }

    NormoraBaseModel <|-- TenantCreateRequest
    NormoraBaseModel <|-- TenantResponseDto
    NormoraBaseModel <|-- DocumentChunkSchema
    NormoraBaseModel <|-- StreamPacket
    StreamPacket <|-- TextStreamPacket
    StreamPacket <|-- CitationStreamPacket
```

---

## 1. `BaseModel`, Fields & Type Coercion

In Pydantic, data contracts are defined by subclassing **`BaseModel`**. Model attributes are declared using standard Python type annotations, enriched with the **`Field`** function for fine-grained validation bounds, descriptions, and defaults.

### Teaching from Scratch: Python Type Annotations for .NET Developers
In C#, you declare typed properties inside a class or record:
```csharp
public record CreateTenantRequest(
    [Required, StringLength(100, MinimumLength = 3)] string Name,
    [Range(1, 5000)] int ActiveSeats = 1
);
```

In Python, the equivalent is achieved using `BaseModel` and `Field`:
```python
from pydantic import BaseModel, Field

class CreateTenantRequest(BaseModel):
    name: str = Field(min_length=3, max_length=100)
    active_seats: int = Field(default=1, ge=1, le=5000)
```

The arguments to `Field()` mirror enterprise validation constraints:
- `default`: The default value when omitted.
- `default_factory`: A zero-argument callable that produces dynamic defaults (e.g., `list`, `dict`, `datetime.utcnow`, `uuid4`). **Never pass mutable defaults directly like `tags: list = []`**.
- `min_length`, `max_length`: String and collection size boundaries.
- `ge` (greater or equal), `gt` (greater than): Numeric minimum limits.
- `le` (less or equal), `lt` (less than): Numeric maximum limits.
- `pattern`: Regular expression validation (regex).
- `description`: Explanatory text surfaced directly in FastAPI OpenAPI / Swagger documentation.
- `exclude`: Excludes this property from outbound serialization (e.g., sensitive passwords or salts).

> [!NOTE]
> When Pydantic instantiates a model, it runs inputs through its Rust validator. You can instantiate models using keyword arguments (`CreateTenantRequest(name="Acme", active_seats=5)`) or by unpacking a dictionary (`CreateTenantRequest(**payload)`).

---

### 5 Real-World Normora Examples

#### Example 1.1: The `CreateTenantRequest` Schema
Validates incoming tenant registration payloads from the Normora onboarding portal.
```python
from pydantic import BaseModel, Field, ValidationError

class CreateTenantRequest(BaseModel):
    name: str = Field(
        min_length=3,
        max_length=100,
        description="Legal enterprise name of the customer organization.",
        examples=["Acme Healthcare Technologies"]
    )
    subscription_plan: str = Field(
        default="Pro",
        description="Subscription tier: Starter, Pro, or Enterprise."
    )
    active_seats: int = Field(
        default=5,
        ge=1,
        le=5000,
        description="Number of licensed user seats. Must be between 1 and 5000."
    )
    enforce_mfa: bool = Field(
        default=True,
        description="Whether Multi-Factor Authentication is mandated for all tenant users."
    )

# Validating payload with automatic string-to-integer coercion:
incoming_payload = {
    "name": "Normora Global Corp",
    "subscription_plan": "Enterprise",
    "active_seats": "120",  # String '120' is safely coerced to integer 120
    "enforce_mfa": True
}

tenant_request = CreateTenantRequest(**incoming_payload)
print(f"Validated Tenant: {tenant_request.name}")
print(f"Seats: {tenant_request.active_seats} (Python Type: {type(tenant_request.active_seats).__name__})")
```

#### Example 1.2: Catching `ValidationError` on Malformed Document Uploads
When validation fails, Pydantic raises `ValidationError`. FastAPI automatically catches this exception and converts it into an HTTP 422 Unprocessable Entity response containing granular error paths.
```python
invalid_tenant_data = {
    "name": "No",       # Too short (min_length=3)
    "active_seats": 0   # Below minimum boundary (ge=1)
}

try:
    CreateTenantRequest(**invalid_tenant_data)
except ValidationError as exc:
    print(f"Total Errors Encountered: {exc.error_count()}")
    for err in exc.errors():
        field_location = " -> ".join(str(loc) for loc in err["loc"])
        error_message = err["msg"]
        error_type = err["type"]
        print(f"Field [{field_location}]: {error_message} (Rule: {error_type})")
```

#### Example 1.3: Document Upload Metadata with Safe Default Factories
In Python, setting default mutable objects like `tags: list[str] = []` causes state leakage across all instances. Pydantic prevents this with `default_factory`.
```python
from uuid import UUID, uuid4
from datetime import datetime, timezone

class DocumentUploadMetadata(BaseModel):
    document_id: UUID = Field(
        default_factory=uuid4,
        description="Unique identifier for the document in PostgreSQL."
    )
    filename: str = Field(min_length=1, max_length=255)
    department: str = Field(default="Engineering")
    is_confidential: bool = False
    uploaded_at: datetime = Field(
        default_factory=lambda: datetime.now(timezone.utc)
    )
    tags: list[str] = Field(
        default_factory=list,
        description="Taxonomy tags used for tenant metadata filtering."
    )

doc1 = DocumentUploadMetadata(filename="Architecture_Design.pdf", tags=["rag", "pgvector"])
doc2 = DocumentUploadMetadata(filename="Employee_Handbook.pdf")

# Notice doc2 has an empty list independent of doc1:
print(f"Doc 1 ({doc1.filename}): tags={doc1.tags}")
print(f"Doc 2 ({doc2.filename}): tags={doc2.tags}")
```

#### Example 1.4: RAG Semantic Search Query Request
Validates queries submitted to the conversational AI search engine before generating embeddings via Google Gemini Flash.
```python
class SearchQueryRequest(BaseModel):
    query: str = Field(
        min_length=3,
        max_length=1000,
        description="Natural language question submitted by the user."
    )
    similarity_threshold: float = Field(
        default=0.72,
        ge=0.0,
        le=1.0,
        description="Cosine similarity cutoff for pgvector HNSW search."
    )
    top_k: int = Field(
        default=5,
        ge=1,
        le=50,
        description="Maximum number of context chunks to retrieve."
    )
    enable_reranking: bool = Field(
        default=True,
        description="Whether to run cross-encoder reranking on retrieved chunks."
    )

search_req = SearchQueryRequest(query="What is our maternity leave policy?", top_k="8")
print(f"Search Query: '{search_req.query}', Top K: {search_req.top_k}")
```

#### Example 1.5: Tenant Membership Assignment (RBAC Contract)
Enforces membership assignments when mapping Keycloak user identities to Normora workspaces.
```python
from typing import Literal

class TenantMembershipAssignment(BaseModel):
    tenant_id: UUID
    user_id: UUID
    role: Literal["TenantAdmin", "KnowledgeManager", "StandardUser", "Guest"] = Field(
        description="Role-Based Access Control tier in accordance with BFLA policies."
    )
    allowed_departments: list[str] = Field(
        default_factory=lambda: ["General"],
        min_length=1,
        description="Departments this user has authorization to query in RAG pipelines."
    )

assignment = TenantMembershipAssignment(
    tenant_id=uuid4(),
    user_id=uuid4(),
    role="KnowledgeManager",
    allowed_departments=["Legal", "Compliance", "Security"]
)
print(f"Assigned user to tenant as {assignment.role} with access to: {assignment.allowed_departments}")
```

---

## 2. `model_config` Power Options

In Pydantic v2, model-level behaviors are configured using the **`model_config`** class attribute with a **`ConfigDict`** instance. This replaces the legacy Pydantic v1 inner `class Config:`.

### Essential Configuration Flags
1. **`strict=True`**: Disables type coercion. If a field is typed as an `int`, passing `"50"` will fail with a `ValidationError`. This is essential for safety-critical AI pipelines and LLM structured outputs where guessing types causes subtle silent bugs.
2. **`frozen=True`**: Makes the model instance completely immutable and hashable. Once constructed, attempting to modify any field (`instance.name = "new"`) raises a `ValidationError`. Equivalent to a C# `readonly record struct`.
3. **`extra='forbid'` / `'ignore'` / `'allow'`**:
   - `'ignore'` (default): Silently drops extra attributes present in the input JSON.
   - `'forbid'`: Raises a validation error if the client sends unexpected fields. Essential for preventing **Mass Assignment / Over-Posting** security vulnerabilities.
   - `'allow'`: Stores extra attributes dynamically in `__pydantic_extra__`.
4. **`str_strip_whitespace=True`**: Automatically trims leading and trailing whitespace from all string fields before validation occurs.
5. **`populate_by_name=True`**: Allows populating fields using either their Python attribute name (`tenant_id`) or their configured alias (`tenantId`).
6. **`alias_generator`**: A function (like `to_camel`) that automatically converts all field names into camelCase for frontend frameworks like Angular 22.

---

### 5 Real-World Normora Examples

#### Example 2.1: Strict Mode for LLM Structured Output (`strict=True`)
When Google Gemini Flash returns structured JSON scoring document relevance, we cannot tolerate arbitrary string-to-float coercions.
```python
from pydantic import BaseModel, ConfigDict, Field, ValidationError

class GeminiRerankScore(BaseModel):
    model_config = ConfigDict(strict=True)

    chunk_id: str
    relevance_score: float = Field(ge=0.0, le=1.0)
    reasoning: str

# Valid strict input:
valid_rerank = GeminiRerankScore(
    chunk_id="chunk_8941",
    relevance_score=0.94,
    reasoning="Matches ISO27001 audit requirements precisely."
)
print(f"Strict Score Validated: {valid_rerank.relevance_score}")

# Malformed input from LLM returning string for float:
try:
    GeminiRerankScore(
        chunk_id="chunk_8942",
        relevance_score="0.95",  # Under strict=True, string "0.95" will NOT be coerced!
        reasoning="Failed strict parse."
    )
except ValidationError as e:
    print(f"Strict validation prevented bad score: {e.errors()[0]['msg']}")
```

#### Example 2.2: Immutable Document Chunk Embedding (`frozen=True`)
Vector embeddings generated from document chunks must never be mutated downstream by background workers.
```python
from uuid import UUID, uuid4

class DocumentChunkEmbedding(BaseModel):
    model_config = ConfigDict(frozen=True)

    chunk_id: UUID
    document_id: UUID
    embedding: tuple[float, ...]  # Tuples are immutable, matching frozen model semantics
    token_count: int

frozen_chunk = DocumentChunkEmbedding(
    chunk_id=uuid4(),
    document_id=uuid4(),
    embedding=(0.0125, -0.0452, 0.8812),
    token_count=512
)

try:
    # Attempting to mutate a frozen model:
    frozen_chunk.token_count = 256  # type: ignore
except ValidationError as err:
    print(f"Immutability guard triggered: {err.errors()[0]['msg']}")

# Frozen models are hashable! They can be stored in sets or used as dictionary keys:
chunk_cache = {frozen_chunk: "Cached in Redis memory"}
print(f"Cache key lookup successful: {chunk_cache[frozen_chunk]}")
```

#### Example 2.3: Preventing Mass-Assignment Attacks (`extra='forbid'`)
When updating tenant subscription plans, malicious clients might attempt to inject unapproved attributes (such as `is_super_admin: true`). Setting `extra='forbid'` blocks this.
```python
class UpdateTenantPlanRequest(BaseModel):
    model_config = ConfigDict(extra="forbid")

    subscription_tier: Literal["Starter", "Pro", "Enterprise"]
    auto_renew: bool

# Attacker attempts over-posting:
malicious_payload = {
    "subscription_tier": "Enterprise",
    "auto_renew": True,
    "is_super_admin": True  # Unrecognized field
}

try:
    UpdateTenantPlanRequest(**malicious_payload)
except ValidationError as exc:
    print(f"Mass-assignment blocked: {exc.errors()[0]['msg']} on field '{exc.errors()[0]['loc'][0]}'")
```

#### Example 2.4: Automatic Whitespace Trimming (`str_strip_whitespace=True`)
Users frequently copy-paste department names, document titles, or search queries containing accidental padding.
```python
class DocumentTaggingRequest(BaseModel):
    model_config = ConfigDict(str_strip_whitespace=True)

    department_name: str
    category_slug: str

tag_req = DocumentTaggingRequest(
    department_name="   Human Resources Department   ",
    category_slug="   hr-benefits-2026   "
)

print(f"Cleaned Department: '{tag_req.department_name}'")
print(f"Cleaned Category Slug: '{tag_req.category_slug}'")
assert tag_req.department_name == "Human Resources Department"
assert tag_req.category_slug == "hr-benefits-2026"
```

#### Example 2.5: Full Angular 22 camelCase Interoperability
In Python, standard naming is `snake_case` (`active_seat_count`). In Angular 22, the convention is `camelCase` (`activeSeatCount`). Pydantic bridges this effortlessly.
```python
from pydantic.alias_generators import to_camel

class AngularCompatibleModel(BaseModel):
    model_config = ConfigDict(
        alias_generator=to_camel,
        populate_by_name=True
    )

class TenantMembershipDto(AngularCompatibleModel):
    tenant_id: str
    company_name: str
    active_seat_count: int
    is_subscription_active: bool

# 1. Parsing incoming JSON from Angular SPA (camelCase):
incoming_angular_json = {
    "tenantId": "ten_9941",
    "companyName": "Apex Financial Services",
    "activeSeatCount": 85,
    "isSubscriptionActive": True
}
dto = TenantMembershipDto(**incoming_angular_json)
print(f"Parsed into Python snake_case: {dto.tenant_id} -> {dto.company_name}")

# 2. Emitting outgoing JSON for Angular SPA (camelCase):
json_for_angular = dto.model_dump_json(by_alias=True)
print(f"Emitted for Angular: {json_for_angular}")
```

---

## 3. Deep Validation: `@field_validator` vs `@model_validator`

When standard type checks and `Field()` bounds are insufficient, Pydantic v2 provides two levels of custom validation:

```mermaid
flowchart TD
    Raw["Raw Input Data: dict"] --> VBefore["@field_validator(mode='before')\nPre-parse cleaning & transforms"]
    VBefore --> CoreParse["pydantic-core Rust Engine\nType Coercion & Field Constraints"]
    CoreParse --> VAfter["@field_validator(mode='after')\nPost-parse business rules"]
    VAfter --> MAfter["@model_validator(mode='after')\nCross-field dependencies on self"]
    MAfter --> Validated["Validated Model Instance"]
```

### The Validator Modes
1. **`@field_validator`**:
   - `mode='after'` (default): Runs *after* Pydantic's core validation. The input value is guaranteed to be of the declared Python type. Must be declared as a `@classmethod`.
   - `mode='before'`: Runs *before* Pydantic parses or coerces the value. The input can be anything (string, None, dict). Ideal for sanitizing input strings or stripping null bytes.
   - `mode='wrap'`: Wraps the entire validation flow, allowing pre-processing, calling the next validator, and handling exceptions.
2. **`@model_validator`**:
   - `mode='after'` (default): Runs on the fully parsed and typed model instance (`self`). Perfect for **cross-field validation** (e.g., verifying that `end_date > start_date` or that `max_tokens` matches the selected subscription tier).
   - `mode='before'`: Runs on the raw input dictionary before any field parsing starts. Useful for reshaping or renaming incoming JSON structures from third-party APIs like Keycloak.

---

### 5 Real-World Normora Examples

#### Example 3.1: `@field_validator(mode='after')` — Tenant Slug Validation
Enforces that organization slugs are lowercase, alphanumeric, and do not collide with reserved routing paths.
```python
from pydantic import BaseModel, field_validator

class TenantSlugRequest(BaseModel):
    company_name: str
    slug: str

    @field_validator("slug")
    @classmethod
    def validate_tenant_slug(cls, value: str) -> str:
        clean_slug = value.strip().lower()
        
        # Disallow reserved routing slugs
        reserved_slugs = {"admin", "api", "auth", "billing", "system", "health"}
        if clean_slug in reserved_slugs:
            raise ValueError(f"The slug '{clean_slug}' is a reserved system path.")
        
        # Ensure only lowercase alphanumeric characters and hyphens
        if not clean_slug.replace("-", "").isalnum():
            raise ValueError("Slug must contain only alphanumeric characters and hyphens.")
            
        return clean_slug

# Valid request:
req = TenantSlugRequest(company_name="Apex Healthcare", slug="APEX-HEALTHCARE")
print(f"Validated clean slug: {req.slug}")

# Invalid reserved slug:
try:
    TenantSlugRequest(company_name="System Admin", slug="admin")
except ValidationError as e:
    print(f"Validation error: {e.errors()[0]['msg']}")
```

#### Example 3.2: `@field_validator(mode='before')` — File Ingestion Extension Whitelist
Cleans raw filenames and verifies extensions against an allowed document whitelist before parsing.
```python
class DocumentIngestionRequest(BaseModel):
    filename: str
    tenant_id: str

    @field_validator("filename", mode="before")
    @classmethod
    def sanitize_and_whitelist_filename(cls, raw_val: Any) -> str:
        if not isinstance(raw_val, str):
            raise ValueError("Filename must be a valid string.")
        
        sanitized = raw_val.strip().replace("\x00", "")  # Strip null-byte injections
        allowed_extensions = (".pdf", ".docx", ".txt", ".md")
        
        if not any(sanitized.lower().endswith(ext) for ext in allowed_extensions):
            raise ValueError(
                f"Unsupported file format '{sanitized}'. Allowed formats: {allowed_extensions}"
            )
        return sanitized

doc = DocumentIngestionRequest(filename="  Security_Compliance_2026.PDF  ", tenant_id="ten_01")
print(f"Sanitized filename: '{doc.filename}'")
```

#### Example 3.3: `@model_validator(mode='after')` — Cross-Field Plan Quota Enforcement
Validates that token allocations do not exceed subscription tier allowances.
```python
from pydantic import model_validator

class WorkspaceQuotaConfig(BaseModel):
    subscription_plan: Literal["Free", "Pro", "Enterprise"]
    monthly_token_quota: int
    max_active_documents: int

    @model_validator(mode="after")
    def verify_plan_quota_alignment(self) -> "WorkspaceQuotaConfig":
        if self.subscription_plan == "Free":
            if self.monthly_token_quota > 100_000:
                raise ValueError("Free tier cannot exceed 100,000 monthly tokens.")
            if self.max_active_documents > 25:
                raise ValueError("Free tier cannot exceed 25 active documents.")
        elif self.subscription_plan == "Pro":
            if self.monthly_token_quota > 5_000_000:
                raise ValueError("Pro tier cannot exceed 5,000,000 monthly tokens.")
        return self

# Valid Pro configuration:
pro_config = WorkspaceQuotaConfig(
    subscription_plan="Pro",
    monthly_token_quota=2_000_000,
    max_active_documents=500
)
print("Pro quota aligned successfully.")

# Invalid Free configuration:
try:
    WorkspaceQuotaConfig(
        subscription_plan="Free",
        monthly_token_quota=500_000,  # Exceeds 100k
        max_active_documents=10
    )
except ValidationError as e:
    print(f"Cross-field quota error: {e.errors()[0]['msg']}")
```

#### Example 3.4: `@model_validator(mode='after')` — Audit Log Date Range Bounds
Ensures temporal queries adhere to strict business rules: `end_date` must be after `start_date`, and queries cannot exceed 90 days.
```python
from datetime import datetime, timezone, timedelta

class AuditLogDateRangeQuery(BaseModel):
    start_date: datetime
    end_date: datetime

    @model_validator(mode="after")
    def validate_audit_window(self) -> "AuditLogDateRangeQuery":
        if self.end_date < self.start_date:
            raise ValueError("end_date cannot be earlier than start_date.")
        
        max_window = timedelta(days=90)
        if (self.end_date - self.start_date) > max_window:
            raise ValueError("Audit log queries cannot span more than 90 continuous days.")
            
        return self

valid_query = AuditLogDateRangeQuery(
    start_date=datetime(2026, 1, 1, tzinfo=timezone.utc),
    end_date=datetime(2026, 2, 1, tzinfo=timezone.utc)
)
print("Audit date range validated.")
```

#### Example 3.5: `@model_validator(mode='before')` — Normalizing Keycloak JWT Claims
External identity providers like Keycloak often provide nested or inconsistently named claims. A `mode='before'` validator reshapes raw dictionaries into our domain format.
```python
class KeycloakUserPrincipal(BaseModel):
    user_id: str
    email: str
    tenant_id: str
    roles: list[str]

    @model_validator(mode="before")
    @classmethod
    def map_keycloak_claims(cls, data: Any) -> Any:
        if isinstance(data, dict):
            # Normalize 'sub' claim to 'user_id'
            if "sub" in data and "user_id" not in data:
                data["user_id"] = data["sub"]
            
            # Extract nested tenant from custom realm claim
            if "custom_attributes" in data and "tenant_id" in data["custom_attributes"]:
                data["tenant_id"] = data["custom_attributes"]["tenant_id"]
            
            # Flatten Keycloak realm_access.roles
            if "realm_access" in data and isinstance(data["realm_access"], dict):
                data["roles"] = data["realm_access"].get("roles", [])
                
        return data

# Raw claim dictionary received from Keycloak OIDC:
raw_keycloak_jwt = {
    "sub": "kc_user_550e8400",
    "email": "sarah.architect@normora.ai",
    "custom_attributes": {"tenant_id": "ten_apex_99"},
    "realm_access": {"roles": ["TenantAdmin", "KnowledgeManager"]}
}

principal = KeycloakUserPrincipal.model_validate(raw_keycloak_jwt)
print(f"Principal Mapped: {principal.user_id} -> Tenant: {principal.tenant_id}")
print(f"Assigned Roles: {principal.roles}")
```

---

## 4. Computed Fields (`@computed_field`)

In C#, if you want a calculated property on a record, you define a getter:
```csharp
public record DocumentChunk(string TextContent)
{
    public int CharacterCount => TextContent.Length;
}
```
However, in standard C# serialization or older Pydantic v1, calculated properties are omitted from serialized JSON unless custom converters are manually configured.

In Pydantic v2, the **`@computed_field`** decorator solves this completely:
1. It exposes the property as a real field during `.model_dump()` and `.model_dump_json()`.
2. It automatically adds the computed field to the generated **OpenAPI / Swagger JSON Schema** so frontend clients know it exists.
3. It can specify `return_type`, descriptions, and formatting tags.

---

### 5 Real-World Normora Examples

#### Example 4.1: `DocumentChunkDto` with Token and Character Metrics
Exposes text metrics dynamically calculated from the ingested chunk content.
```python
from pydantic import BaseModel, computed_field

class DocumentChunkDto(BaseModel):
    chunk_id: str
    document_id: str
    text_content: str

    @computed_field(description="Total characters in this text chunk.")
    @property
    def character_count(self) -> int:
        return len(self.text_content)

    @computed_field(description="Rough token estimate for Google Gemini Flash (~4 chars per token).")
    @property
    def estimated_tokens(self) -> int:
        return max(1, len(self.text_content) // 4)

chunk = DocumentChunkDto(
    chunk_id="chk_101",
    document_id="doc_88",
    text_content="Normora utilizes PostgreSQL with pgvector HNSW indexing for sub-second similarity lookups."
)

print(chunk.model_dump())
# Output includes: 'character_count': 97, 'estimated_tokens': 24
```

#### Example 4.2: `TenantUsageSummary` with Quota Exhaustion Alerts
Calculates whether a tenant has exceeded their monthly token quota.
```python
class TenantUsageSummary(BaseModel):
    tenant_id: str
    monthly_token_allowance: int
    tokens_consumed: int

    @computed_field
    @property
    def remaining_tokens(self) -> int:
        return max(0, self.monthly_token_allowance - self.tokens_consumed)

    @computed_field
    @property
    def quota_utilization_percentage(self) -> float:
        if self.monthly_token_allowance == 0:
            return 100.0
        return round((self.tokens_consumed / self.monthly_token_allowance) * 100.0, 2)

    @computed_field
    @property
    def is_quota_exceeded(self) -> bool:
        return self.tokens_consumed >= self.monthly_token_allowance

usage = TenantUsageSummary(
    tenant_id="ten_44",
    monthly_token_allowance=1_000_000,
    tokens_consumed=875_400
)
print(f"Remaining Tokens: {usage.remaining_tokens}")
print(f"Utilization: {usage.quota_utilization_percentage}%")
print(f"Exceeded? {usage.is_quota_exceeded}")
```

#### Example 4.3: `ConversationSessionResponse` Dynamic Metrics
Calculates message count and extracts unique source document IDs cited across the conversation.
```python
class Citation(BaseModel):
    document_id: str
    page_number: int

class ConversationMessage(BaseModel):
    message_id: str
    role: str
    citations: list[Citation] = []

class ConversationSessionResponse(BaseModel):
    session_id: str
    messages: list[ConversationMessage]

    @computed_field
    @property
    def total_messages(self) -> int:
        return len(self.messages)

    @computed_field
    @property
    def cited_document_ids(self) -> list[str]:
        doc_ids = {
            cit.document_id 
            for msg in self.messages 
            for cit in msg.citations
        }
        return sorted(list(doc_ids))

session = ConversationSessionResponse(
    session_id="conv_900",
    messages=[
        ConversationMessage(message_id="m1", role="user"),
        ConversationMessage(
            message_id="m2",
            role="assistant",
            citations=[
                Citation(document_id="doc_hr_01", page_number=2),
                Citation(document_id="doc_legal_05", page_number=12)
            ]
        )
    ]
)
print(f"Session Serialized: {session.model_dump()}")
```

#### Example 4.4: `VectorSearchHit` Relevance Percentage
Normalizes a raw cosine similarity score (typically `0.0` to `1.0`) into an intuitive UI display string.
```python
class VectorSearchHit(BaseModel):
    chunk_id: str
    cosine_similarity: float
    snippet: str

    @computed_field
    @property
    def relevance_match_badge(self) -> str:
        pct = round(self.cosine_similarity * 100, 1)
        if pct >= 85.0:
            return f"High Match ({pct}%)"
        elif pct >= 70.0:
            return f"Moderate Match ({pct}%)"
        return f"Weak Match ({pct}%)"

hit = VectorSearchHit(chunk_id="chk_5", cosine_similarity=0.914, snippet="Keycloak OIDC integration...")
print(f"Relevance: {hit.relevance_match_badge}")
```

#### Example 4.5: `TenantStorageMetrics` Gigabyte Converter
Converts raw storage bytes stored in MinIO into gigabytes and checks capacity limits.
```python
class TenantStorageMetrics(BaseModel):
    tenant_id: str
    storage_allocated_bytes: int
    storage_used_bytes: int

    @computed_field
    @property
    def storage_used_gb(self) -> float:
        return round(self.storage_used_bytes / (1024 ** 3), 3)

    @computed_field
    @property
    def is_nearing_capacity(self) -> bool:
        # Warning when storage exceeds 90%
        return (self.storage_used_bytes / self.storage_allocated_bytes) >= 0.90

storage = TenantStorageMetrics(
    tenant_id="ten_acme",
    storage_allocated_bytes=100 * 1024 * 1024 * 1024,   # 100 GB
    storage_used_bytes=94 * 1024 * 1024 * 1024          # 94 GB
)
print(f"Storage Used: {storage.storage_used_gb} GB")
print(f"Alert Required? {storage.is_nearing_capacity}")
```

---

## 5. Serialization & Custom Serializers

Pydantic models are serialized into Python native types or JSON strings using two primary methods:
- **`.model_dump()`**: Serializes the model into a standard Python `dict`.
- **`.model_dump_json()`**: Serializes the model directly into a UTF-8 JSON `str` using the compiled Rust engine (drastically faster than Python's `json.dumps()`).

### Serialization Control Flags
- `by_alias=True`: Serializes fields using their configured aliases (e.g., camelCase for Angular 22).
- `exclude_unset=True`: Serializes **only** the fields that were explicitly passed during model initialization. Indispensable for HTTP `PATCH` endpoints to prevent overwriting omitted fields with defaults.
- `exclude_none=True`: Omits fields whose values are `None`.
- `exclude={"field_a", "field_b"}`: Explicitly omits specific fields from the output.

### Custom Serializers with `@field_serializer` and `@model_serializer`
When standard JSON serialization is insufficient (e.g., custom datetime formatting, truncating high-precision vector embeddings, or encrypting fields), Pydantic v2 introduces dedicated decorators:
- **`@field_serializer`**: Customizes serialization for a specific field.
- **`@model_serializer`**: Takes control of serializing the entire model instance (e.g., wrapping all responses in a standardized API envelope).

---

### 5 Real-World Normora Examples

#### Example 5.1: `@field_serializer` for Angular-Compatible ISO8601 Timestamps
Ensures all UTC datetimes are serialized with an explicit trailing `Z` suffix required by Angular 22 DatePipes.
```python
from datetime import datetime, timezone
from pydantic import BaseModel, field_serializer

class AuditEventDto(BaseModel):
    event_id: str
    action: str
    occurred_at: datetime

    @field_serializer("occurred_at")
    def serialize_utc_datetime(self, dt: datetime, _info) -> str:
        # Guarantee ISO 8601 with explicit Zulu 'Z' suffix
        if dt.tzinfo is None:
            dt = dt.replace(tzinfo=timezone.utc)
        return dt.astimezone(timezone.utc).strftime("%Y-%m-%dT%H:%M:%S.%fZ")

audit = AuditEventDto(
    event_id="evt_501",
    action="TENANT_PROVISIONED",
    occurred_at=datetime.now(timezone.utc)
)

print("Serialized Audit JSON:")
print(audit.model_dump_json(indent=2))
```

#### Example 5.2: `@field_serializer` for Vector Embedding Precision Truncation
1536-dimensional float vectors from Google Gemini Flash waste enormous bandwidth if serialized at 64-bit precision during debugging or logging. We can round them to 4 decimal places.
```python
class VectorChunkExportDto(BaseModel):
    chunk_id: str
    embedding: list[float]

    @field_serializer("embedding")
    def round_embedding_precision(self, emb: list[float], _info) -> list[float]:
        return [round(val, 4) for val in emb]

chunk_export = VectorChunkExportDto(
    chunk_id="chk_882",
    embedding=[0.0123456789, -0.987654321, 0.44556677]
)

print(chunk_export.model_dump())
# {'chunk_id': 'chk_882', 'embedding': [0.0123, -0.9877, 0.4456]}
```

#### Example 5.3: Excluding Sensitive Credentials from Outbound DTOs
Prevents Keycloak client secrets or internal salts from leaking into client-facing responses.
```python
class KeycloakClientRegistration(BaseModel):
    client_id: str
    realm_name: str
    redirect_uri: str
    client_secret: str = Field(exclude=True)  # NEVER emitted in model_dump()

kc_reg = KeycloakClientRegistration(
    client_id="normora-spa-client",
    realm_name="normora-enterprise",
    redirect_uri="https://app.normora.ai/auth/callback",
    client_secret="secret_xyz_super_confidential"
)

serialized_payload = kc_reg.model_dump()
print(f"Keys present in serialized output: {list(serialized_payload.keys())}")
assert "client_secret" not in serialized_payload
```

#### Example 5.4: Sparse CQRS Updates via `exclude_unset=True`
When executing a partial HTTP `PATCH`, we only want to update columns that the client explicitly provided.
```python
class PatchTenantSettingsRequest(BaseModel):
    name: str | None = None
    subscription_plan: str | None = None
    enforce_mfa: bool | None = None
    max_seats: int | None = None

# Client only submitted 'enforce_mfa':
patch_request = PatchTenantSettingsRequest(enforce_mfa=True)

# exclude_unset=True ignores default None values that weren't sent by client:
cqrs_update_data = patch_request.model_dump(exclude_unset=True)
print(f"PostgreSQL UPDATE dictionary: {cqrs_update_data}")
assert cqrs_update_data == {"enforce_mfa": True}
```

#### Example 5.5: `@model_serializer(mode='wrap')` for Standardized Response Envelopes
Wraps all outbound API models into a centralized Normora JSON envelope with status and timestamps.
```python
from pydantic import model_serializer

class TenantResponsePayload(BaseModel):
    tenant_id: str
    organization_name: str
    status: str

    @model_serializer(mode="wrap")
    def wrap_with_envelope(self, handler) -> dict:
        # handler(self) returns the standard dictionary serialization
        original_dict = handler(self)
        return {
            "success": True,
            "data": original_dict,
            "metadata": {
                "server_time": datetime.now(timezone.utc).isoformat(),
                "api_version": "v1.2"
            }
        }

tenant_dto = TenantResponsePayload(
    tenant_id="ten_001",
    organization_name="HealthSync Inc.",
    status="Active"
)

print(tenant_dto.model_dump_json(indent=2))
```

---

## 6. Polymorphic Schemas with Discriminated Unions

In modern AI systems, real-time responses and background jobs are **polymorphic**:
- An SSE / SignalR stream packet might be a **text chunk**, a **citation reference**, or an **agent tool execution request**.
- A search result might be a **dense vector match** or a **sparse keyword match**.

### Why Standard Unions Fail
If you define a standard union:
```python
Packet = Union[TextPacket, CitationPacket, ToolPacket]
```
Pydantic attempts to match the payload against `TextPacket` first. If any field matches, it coerces the data into `TextPacket`, frequently producing incorrect matches, slow trial-and-error parsing, and unhelpful error messages.

### The Solution: Discriminated Unions
A **Discriminated Union** uses a single discriminator field (e.g., `type: Literal["text"]`) to enable instant $O(1)$ schema lookup in Rust.

```mermaid
flowchart TD
    JSON["Incoming Stream JSON\n{'type': 'citation', 'document_id': 'doc_1', 'page': 4}"] --> Inspect{"Inspect Discriminator\nField: 'type'"}
    Inspect -- "type == 'text'" --> T["TextStreamPacket"]
    Inspect -- "type == 'citation'" --> C["CitationStreamPacket"]
    Inspect -- "type == 'tool_call'" --> TC["ToolCallStreamPacket"]
```

---

### 5 Real-World Normora Examples

#### Example 6.1: SignalR / SSE Streaming Packets
Powers real-time conversational streaming from Google Gemini Flash to the Angular 22 chat interface.
```python
from typing import Annotated, Literal, Union
from pydantic import BaseModel, Field, TypeAdapter

class TextStreamPacket(BaseModel):
    type: Literal["text"]
    token: str

class CitationStreamPacket(BaseModel):
    type: Literal["citation"]
    document_id: str
    page_number: int
    snippet: str

class ToolCallStreamPacket(BaseModel):
    type: Literal["tool_call"]
    tool_name: str
    arguments: dict

class ErrorStreamPacket(BaseModel):
    type: Literal["error"]
    error_code: str
    message: str

# Define Discriminated Union:
StreamPacket = Annotated[
    Union[TextStreamPacket, CitationStreamPacket, ToolCallStreamPacket, ErrorStreamPacket],
    Field(discriminator="type")
]

class EventEnvelope(BaseModel):
    event_id: str
    payload: StreamPacket

# 1. Parsing a citation packet:
citation_json = {
    "event_id": "evt_01",
    "payload": {
        "type": "citation",
        "document_id": "doc_cyber_policy",
        "page_number": 14,
        "snippet": "MFA is required for all external access."
    }
}
envelope = EventEnvelope.model_validate(citation_json)
print(f"Parsed Type: {type(envelope.payload).__name__}")
assert isinstance(envelope.payload, CitationStreamPacket)
print(f"Citation Document: {envelope.payload.document_id}, Page: {envelope.payload.page_number}")
```

#### Example 6.2: Multi-Channel Notification Dispatch
Enables tenant administrators to configure alert notifications across email, Slack, and generic webhooks.
```python
class EmailNotificationConfig(BaseModel):
    channel: Literal["email"]
    destination_address: str
    send_digest: bool

class SlackNotificationConfig(BaseModel):
    channel: Literal["slack"]
    webhook_url: str
    channel_name: str

class WebhookNotificationConfig(BaseModel):
    channel: Literal["webhook"]
    endpoint_url: str
    signing_secret: str

NotificationConfig = Annotated[
    Union[EmailNotificationConfig, SlackNotificationConfig, WebhookNotificationConfig],
    Field(discriminator="channel")
]

class TenantNotificationRule(BaseModel):
    rule_id: str
    target: NotificationConfig

slack_rule = TenantNotificationRule.model_validate({
    "rule_id": "rule_slack_alerts",
    "target": {
        "channel": "slack",
        "webhook_url": "https://hooks.slack.com/services/T00/B00/X00",
        "channel_name": "#security-alerts"
    }
})
print(f"Configured Channel: {slack_rule.target.channel} -> {slack_rule.target.channel_name}")
```

#### Example 6.3: Hybrid Search Retrieval Candidates
Discriminates between dense vector retrieval (from pgvector) and sparse lexical retrieval (from PostgreSQL BM25 full-text search).
```python
class DenseVectorCandidate(BaseModel):
    retrieval_method: Literal["dense"]
    chunk_id: str
    cosine_distance: float
    vector_dimension: int = 1536

class SparseLexicalCandidate(BaseModel):
    retrieval_method: Literal["sparse"]
    chunk_id: str
    bm25_score: float
    matched_terms: list[str]

SearchCandidate = Annotated[
    Union[DenseVectorCandidate, SparseLexicalCandidate],
    Field(discriminator="retrieval_method")
]

class SearchResultsBatch(BaseModel):
    query_id: str
    candidates: list[SearchCandidate]

batch = SearchResultsBatch.model_validate({
    "query_id": "qry_88",
    "candidates": [
        {"retrieval_method": "dense", "chunk_id": "chk_1", "cosine_distance": 0.082},
        {"retrieval_method": "sparse", "chunk_id": "chk_2", "bm25_score": 14.8, "matched_terms": ["retention", "policy"]}
    ]
})

for candidate in batch.candidates:
    if candidate.retrieval_method == "dense":
        print(f"Dense Match: {candidate.chunk_id} (Distance: {candidate.cosine_distance})")
    else:
        print(f"Sparse Match: {candidate.chunk_id} (BM25: {candidate.bm25_score})")
```

#### Example 6.4: Dynamic Generative UI Components (A2UI)
Allows Google Gemini Flash to select and parameterize UI cards rendered on the Angular 22 frontend.
```python
class MetricCardComponent(BaseModel):
    component_type: Literal["metric_card"]
    title: str
    value: str
    trend: Literal["up", "down", "neutral"]

class DataTableComponent(BaseModel):
    component_type: Literal["data_table"]
    headers: list[str]
    rows: list[list[str]]

class CitationAccordionComponent(BaseModel):
    component_type: Literal["citation_accordion"]
    document_title: str
    page_reference: int
    excerpt: str

UiComponent = Annotated[
    Union[MetricCardComponent, DataTableComponent, CitationAccordionComponent],
    Field(discriminator="component_type")
]

class AgentUiResponse(BaseModel):
    assistant_reply: str
    ui_widgets: list[UiComponent]

gemini_payload = {
    "assistant_reply": "Here is the summary of your tenant seat consumption:",
    "ui_widgets": [
        {
            "component_type": "metric_card",
            "title": "Active Seats",
            "value": "84 / 100",
            "trend": "up"
        }
    ]
}

ui_response = AgentUiResponse.model_validate(gemini_payload)
print(f"Rendered Widget: {ui_response.ui_widgets[0].component_type}")
```

#### Example 6.5: Document Pipeline Extraction Outcomes
Represents polymorphic ingestion results generated by asynchronous document chunking workers.
```python
class SuccessExtractionResult(BaseModel):
    status: Literal["success"]
    document_id: str
    extracted_chunks_count: int
    total_tokens_computed: int

class CorruptedFileExtractionResult(BaseModel):
    status: Literal["corrupted_file"]
    document_id: str
    detected_mime_type: str
    recovery_attempted: bool

class OcrFallbackExtractionResult(BaseModel):
    status: Literal["ocr_fallback"]
    document_id: str
    scanned_pages_processed: int
    confidence_score: float

ExtractionOutcome = Annotated[
    Union[SuccessExtractionResult, CorruptedFileExtractionResult, OcrFallbackExtractionResult],
    Field(discriminator="status")
]

outcome_adapter = TypeAdapter(ExtractionOutcome)
result = outcome_adapter.validate_python({
    "status": "ocr_fallback",
    "document_id": "doc_scanned_receipt",
    "scanned_pages_processed": 3,
    "confidence_score": 0.88
})

print(f"Worker Pipeline Outcome: {result.status} (Pages: {result.scanned_pages_processed})")
```

---

## 7. Validating Non-Model Types with `TypeAdapter`

In Pydantic v1, if you wanted to validate a standalone primitive type—such as a `list[UUID]`, a `dict[str, TenantDto]`, or a `Literal["admin", "user"]`—you were forced to construct a boilerplate dummy model:
```python
# The clumsy Pydantic v1 workaround:
class DummyListWrapper(BaseModel):
    items: list[int]
```

In Pydantic v2, **`TypeAdapter`** eliminates this boilerplate. It allows you to validate, parse, and serialize **any** arbitrary Python type structure directly using the Rust engine.

### Key `TypeAdapter` Methods:
- `.validate_python(data)`: Validates raw Python dictionaries, lists, or primitives.
- `.validate_json(json_str)`: Parses and validates a JSON string directly without calling `json.loads()`.
- `.dump_python(instance)`: Serializes into native Python objects.
- `.dump_json(instance)`: Serializes into a JSON byte string.

---

### 5 Real-World Normora Examples

#### Example 7.1: Validating Batch Vector Embeddings (`list[list[float]]`)
Validates a batch of high-dimensional embeddings returned from the Google Gemini embedding endpoint.
```python
from pydantic import TypeAdapter, ValidationError

# Create adapter for a list of 1536-dimensional float vectors
EmbeddingBatchAdapter = TypeAdapter(list[list[float]])

raw_embeddings = [
    [0.012, -0.045, 0.881],
    [0.104, 0.229, -0.011]
]

validated_batch = EmbeddingBatchAdapter.validate_python(raw_embeddings)
print(f"Validated batch containing {len(validated_batch)} embedding vectors.")

# Invalid data (strings that cannot be coerced):
try:
    EmbeddingBatchAdapter.validate_python([["0.012", "invalid_vector_element"]])
except ValidationError as e:
    print(f"Batch embedding validation failed: {e.errors()[0]['msg']}")
```

#### Example 7.2: Validating Redis Cache DTO Dictionaries
Validates structured tenant session caches retrieved as raw JSON from Redis.
```python
class CachedTenantSession(BaseModel):
    tenant_id: str
    active_user_count: int
    is_mfa_enforced: bool

# Adapter for a dictionary mapping user UUIDs to their cached sessions
TenantCacheAdapter = TypeAdapter(dict[str, CachedTenantSession])

raw_redis_json = """
{
    "usr_01": {"tenant_id": "ten_apex", "active_user_count": 4, "is_mfa_enforced": true},
    "usr_02": {"tenant_id": "ten_globex", "active_user_count": 12, "is_mfa_enforced": false}
}
"""

cache_data = TenantCacheAdapter.validate_json(raw_redis_json)
print(f"User 1 Tenant: {cache_data['usr_01'].tenant_id}")
```

#### Example 7.3: Validating Keycloak Role Sets (`set[TenantRole]`)
Validates and deduplicates a collection of Keycloak role claims extracted from an incoming JWT.
```python
TenantRoleType = Literal["SuperAdmin", "TenantAdmin", "KnowledgeManager", "Auditor", "Member"]
RoleSetAdapter = TypeAdapter(set[TenantRoleType])

# Input with duplicates and lowercase strings that coerce to Literal if matching:
raw_roles = ["TenantAdmin", "Member", "TenantAdmin", "Auditor"]
validated_roles = RoleSetAdapter.validate_python(raw_roles)

print(f"Deduplicated Role Set: {validated_roles}")
assert len(validated_roles) == 3
```

#### Example 7.4: Validating Dynamic Tool Call Declarations
Validates tool arguments generated dynamically by Google Gemini Flash before executing local functions.
```python
class SearchKnowledgeBaseArgs(BaseModel):
    query: str
    max_results: int = 5

ToolCallAdapter = TypeAdapter(SearchKnowledgeBaseArgs)

# Validate raw JSON string directly from Gemini Flash output:
llm_tool_output = '{"query": "When was the tenant onboarded?", "max_results": 3}'
tool_args = ToolCallAdapter.validate_json(llm_tool_output)
print(f"Executing search with query: '{tool_args.query}', limit={tool_args.max_results}")
```

#### Example 7.5: Validating Untrusted LLM JSON Extraction Arrays
Parses an arbitrary JSON array of extracted medical entities without declaring an unnecessary container model.
```python
class ExtractedEntity(BaseModel):
    entity_name: str
    category: Literal["Diagnosis", "Medication", "Dosage", "Policy"]
    confidence: float

EntityListAdapter = TypeAdapter(list[ExtractedEntity])

untrusted_llm_json = """
[
    {"entity_name": "Amoxicillin", "category": "Medication", "confidence": 0.98},
    {"entity_name": "500mg", "category": "Dosage", "confidence": 0.95}
]
"""

entities = EntityListAdapter.validate_json(untrusted_llm_json)
print(f"Extracted {len(entities)} medical entities safely:")
for ent in entities:
    print(f" - [{ent.category}] {ent.entity_name} ({ent.confidence})")
```

---

## 8. Custom Types & Rust Core Schema (`__get_pydantic_core_schema__`)

Sometimes your domain demands custom types that do not inherit from `BaseModel`—such as **Strongly Typed IDs** (`TenantId`), specialized pgvector vector wrappers, or sanitized HTML types.

In Pydantic v2, custom types plug directly into the compiled Rust engine by implementing the class method:
```python
@classmethod
def __get_pydantic_core_schema__(cls, source_type, handler) -> core_schema.CoreSchema:
```

This grants you low-level access to Pydantic's Rust validation pipeline (`pydantic_core.core_schema`), enabling maximum performance and seamless integration with FastAPI OpenAPI generation.

---

### 5 Real-World Normora Examples

#### Example 8.1: Strongly Typed `TenantId` (Prefixed UUID)
In enterprise architectures, IDs should be strongly typed and self-describing (e.g., `ten_550e8400-e29b-41d4-a716-446655440000`) to prevent passing a `UserId` into a `TenantId` parameter.
```python
from uuid import UUID
from pydantic_core import core_schema

class TenantId(str):
    """Strongly typed Tenant identifier prefixed with 'ten_'."""
    
    @classmethod
    def __get_pydantic_core_schema__(cls, source_type, handler) -> core_schema.CoreSchema:
        def validate_from_str(val: str) -> "TenantId":
            val = val.strip()
            if not val.startswith("ten_"):
                raise ValueError("TenantId must start with the prefix 'ten_'")
            raw_uuid = val[4:]
            try:
                UUID(raw_uuid)  # Validate UUID format
            except ValueError:
                raise ValueError(f"Invalid UUID suffix in TenantId: '{raw_uuid}'")
            return cls(val)

        return core_schema.chain_schema([
            core_schema.str_schema(),
            core_schema.no_info_after_validator_function(validate_from_str)
        ])

class TenantOrganization(BaseModel):
    id: TenantId
    name: str

# Valid prefixed UUID:
org = TenantOrganization(
    id="ten_550e8400-e29b-41d4-a716-446655440000",
    name="Acme Health"
)
print(f"Strongly typed Tenant ID: {org.id}")

# Invalid prefix:
try:
    TenantOrganization(id="usr_550e8400-e29b-41d4-a716-446655440000", name="Fail")
except ValidationError as e:
    print(f"Custom type error: {e.errors()[0]['msg']}")
```

#### Example 8.2: 1536-Dimensional pgvector Embedding Type
Enforces that a vector matches the exact dimensionality expected by Google Gemini Flash before executing pgvector queries.
```python
class PgVector1536(list):
    """A list of exactly 1536 float values for pgvector embeddings."""

    @classmethod
    def __get_pydantic_core_schema__(cls, source_type, handler) -> core_schema.CoreSchema:
        def validate_vector(val: list) -> "PgVector1536":
            if len(val) != 1536:
                raise ValueError(f"Embedding must contain exactly 1536 dimensions, got {len(val)}")
            return cls(float(x) for x in val)

        return core_schema.chain_schema([
            core_schema.list_schema(items_schema=core_schema.float_schema()),
            core_schema.no_info_after_validator_function(validate_vector)
        ])

class ChunkVectorRecord(BaseModel):
    chunk_id: str
    vector: PgVector1536

# Valid vector of 1536 zeros:
valid_vec = ChunkVectorRecord(chunk_id="chk_01", vector=[0.0] * 1536)
print(f"Validated vector dimension: {len(valid_vec.vector)}")

# Invalid dimension:
try:
    ChunkVectorRecord(chunk_id="chk_02", vector=[0.1, 0.2, 0.3])
except ValidationError as e:
    print(f"Vector validation rejected: {e.errors()[0]['msg']}")
```

#### Example 8.3: Sanitized Markdown String Type
Automatically strips dangerous HTML script tags from markdown documents during ingestion.
```python
import re

class SanitizedMarkdown(str):
    """String type that strips HTML script tags and dangerous event handlers."""

    @classmethod
    def __get_pydantic_core_schema__(cls, source_type, handler) -> core_schema.CoreSchema:
        def sanitize_markdown(raw_str: str) -> "SanitizedMarkdown":
            # Remove <script> tags
            clean = re.sub(r"<script.*?>.*?</script>", "", raw_str, flags=re.DOTALL | re.IGNORECASE)
            # Remove javascript: directives
            clean = re.sub(r"javascript:", "", clean, flags=re.IGNORECASE)
            return cls(clean.strip())

        return core_schema.chain_schema([
            core_schema.str_schema(),
            core_schema.no_info_after_validator_function(sanitize_markdown)
        ])

class IngestedNote(BaseModel):
    title: str
    body: SanitizedMarkdown

malicious_note = IngestedNote(
    title="Executive Brief",
    body="Meeting notes: Budget approved.<script>alert('xss');</script>"
)
print(f"Sanitized body: '{malicious_note.body}'")
assert "<script>" not in malicious_note.body
```

#### Example 8.4: SemVer Version String Type
Enforces strict Semantic Versioning (`vMAJOR.MINOR.PATCH`) for document taxonomy versions.
```python
class SemVer(str):
    """Semantic version string format: v1.0.0"""
    SEMVER_PATTERN = re.compile(r"^v\d+\.\d+\.\d+$")

    @classmethod
    def __get_pydantic_core_schema__(cls, source_type, handler) -> core_schema.CoreSchema:
        def validate_semver(val: str) -> "SemVer":
            val = val.strip()
            if not cls.SEMVER_PATTERN.match(val):
                raise ValueError(f"Version '{val}' must match semantic format 'vMAJOR.MINOR.PATCH' (e.g. 'v1.4.2')")
            return cls(val)

        return core_schema.chain_schema([
            core_schema.str_schema(),
            core_schema.no_info_after_validator_function(validate_semver)
        ])

class KnowledgeBaseMetadata(BaseModel):
    kb_name: str
    schema_version: SemVer

kb = KnowledgeBaseMetadata(kb_name="HR Policies", schema_version="v2.1.0")
print(f"Knowledge Base Version: {kb.schema_version}")
```

#### Example 8.5: Keycloak Bearer Token Type
Performs structural sanity checks on JWT bearer tokens (ensuring three base64-encoded segments separated by dots) before passing them to the security layer.
```python
class BearerToken(str):
    """Validates structural JWT format (header.payload.signature)."""

    @classmethod
    def __get_pydantic_core_schema__(cls, source_type, handler) -> core_schema.CoreSchema:
        def validate_jwt_structure(token_str: str) -> "BearerToken":
            token_str = token_str.strip()
            parts = token_str.split(".")
            if len(parts) != 3 or not all(len(p) > 0 for p in parts):
                raise ValueError("Bearer token must be a well-formed JWT consisting of 3 dot-separated parts.")
            return cls(token_str)

        return core_schema.chain_schema([
            core_schema.str_schema(),
            core_schema.no_info_after_validator_function(validate_jwt_structure)
        ])

class AuthHeaderSchema(BaseModel):
    authorization: BearerToken

token_ok = AuthHeaderSchema(authorization="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIn0.dozG4B_DYOp8")
print("Bearer token structure validated.")
```

---

## 9. Enterprise Settings Management (`pydantic-settings`)

In production, connection strings, Keycloak secrets, and hosted AI provider keys must **never be hardcoded**.

`pydantic-settings` extends `BaseModel` to create **`BaseSettings`**:
1. Reads directly from environment variables, `.env` files, or Docker secrets.
2. Performs type validation and coercion on application startup (**fail-fast configuration**).
3. Provides **`SecretStr`** to mask sensitive credentials in logs and OpenTelemetry traces.

```mermaid
flowchart LR
    Env[".env File / OS Environment Variables"] --> PS["pydantic-settings (BaseSettings)"]
    PS --> Validate{"Startup Type Validation"}
    Validate -- Missing / Invalid --> Halt["Halt Deployment (Fast Fail)"]
    Validate -- Valid Config --> Singleton["AppSettings Singleton Instance"]
```

---

### 5 Real-World Normora Examples

#### Example 9.1: Master `NormoraAppSettings` with `SecretStr`
Centralized application configuration with safe secret masking for Google Gemini Flash.
```python
from pydantic_settings import BaseSettings, SettingsConfigDict
from pydantic import SecretStr, Field
from typing import Literal

class NormoraAppSettings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        case_sensitive=False,
        extra="ignore"
    )

    # Core Environment
    ENVIRONMENT: Literal["development", "staging", "production"] = "development"
    PORT: int = Field(default=8000, ge=1024, le=65535)
    API_V1_STR: str = "/api/v1"

    # Hosted AI Providers (Masked!)
    GEMINI_API_KEY: SecretStr = SecretStr("dummy_gemini_api_key_for_testing")
    OPENAI_API_KEY: SecretStr | None = None

    # Keycloak OIDC Authentication
    KEYCLOAK_SERVER_URL: str = "http://localhost:8080"
    KEYCLOAK_REALM: str = "normora"
    KEYCLOAK_CLIENT_ID: str = "normora-backend"
    KEYCLOAK_CLIENT_SECRET: SecretStr = SecretStr("mock_keycloak_secret")

settings = NormoraAppSettings()

# SecretStr automatically masks secrets when printed or logged:
print(f"Running in: {settings.ENVIRONMENT} on port {settings.PORT}")
print(f"Gemini Key Masked: {settings.GEMINI_API_KEY}")  # SecretStr('**********')

# Safely extract raw value only when establishing outbound HTTPS connections:
raw_key = settings.GEMINI_API_KEY.get_secret_value()
print(f"Key extracted safely for API client: {raw_key[:5]}...")
```

#### Example 9.2: `DatabaseSettings` with Computed Async Connection String
Generates an `asyncpg` connection string for SQLAlchemy / Tortoise ORM dynamically from its parts.
```python
class DatabaseSettings(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="DB_")

    USER: str = "normora_app"
    PASSWORD: SecretStr = SecretStr("strong_production_password_99!")
    HOST: str = "postgres.internal"
    PORT: int = 5432
    NAME: str = "normora_pgvector"
    POOL_SIZE: int = Field(default=20, ge=5, le=100)

    @property
    def async_connection_url(self) -> str:
        pwd = self.PASSWORD.get_secret_value()
        return f"postgresql+asyncpg://{self.USER}:{pwd}@{self.HOST}:{self.PORT}/{self.NAME}"

db_config = DatabaseSettings()
print(f"Connecting to database at: {db_config.HOST}:{db_config.PORT}")
# print(db_config.async_connection_url)  # Contains password, never log in prod!
```

#### Example 9.3: `MinioStorageSettings` for Document Object Storage
Configures MinIO S3 object storage for multi-tenant document uploads.
```python
class MinioStorageSettings(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="MINIO_")

    ENDPOINT: str = "minio:9000"
    ACCESS_KEY: SecretStr = SecretStr("minio_admin")
    SECRET_KEY: SecretStr = SecretStr("minio_super_secret")
    SECURE_SSL: bool = False
    BUCKET_DOCUMENTS: str = "normora-documents"
    BUCKET_EXPORTS: str = "normora-exports"

minio_cfg = MinioStorageSettings()
print(f"MinIO Document Bucket: {minio_cfg.BUCKET_DOCUMENTS} on {minio_cfg.ENDPOINT}")
```

#### Example 9.4: `RagPipelineSettings` Model Hyperparameters
Configures Google Gemini Flash inference parameters and chunking boundaries.
```python
class RagPipelineSettings(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="RAG_")

    MODEL_NAME: str = "gemini-1.5-flash"
    TEMPERATURE: float = Field(default=0.2, ge=0.0, le=2.0)
    MAX_OUTPUT_TOKENS: int = Field(default=2048, ge=256, le=8192)
    CHUNK_SIZE: int = Field(default=512, ge=128, le=2048)
    CHUNK_OVERLAP: int = Field(default=64, ge=0, le=256)
    TOP_K: int = Field(default=5, ge=1, le=20)

rag_cfg = RagPipelineSettings()
print(f"RAG Model: {rag_cfg.MODEL_NAME} | Chunk: {rag_cfg.CHUNK_SIZE} (Overlap: {rag_cfg.CHUNK_OVERLAP})")
```

#### Example 9.5: `OpenTelemetrySettings` for Distributed Tracing
Configures OpenTelemetry OTLP collector endpoints for monitoring multi-tenant latency.
```python
class OpenTelemetrySettings(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="OTEL_")

    SERVICE_NAME: str = "normora-ai-fastapi"
    EXPORTER_OTLP_ENDPOINT: str = "http://otel-collector:4317"
    SAMPLING_RATIO: float = Field(default=1.0, ge=0.0, le=1.0)
    ENABLE_CONSOLE_EXPORTER: bool = False

otel_cfg = OpenTelemetrySettings()
print(f"Tracing service '{otel_cfg.SERVICE_NAME}' targeting {otel_cfg.EXPORTER_OTLP_ENDPOINT}")
```

---

## 10. Pydantic vs C# / .NET Validation Comparison

For senior .NET developers transitioning to Python and FastAPI, here is how Pydantic v2 compares directly to the C# data ecosystem:

| Feature / Architectural Pattern | C# / .NET 10 Equivalent | Python / Pydantic v2 Equivalent | Key Architectural Shift |
| :--- | :--- | :--- | :--- |
| **Model Declaration** | `public record TenantDto(...)` | `class TenantDto(BaseModel):` | Pydantic models automatically validate and parse on instantiation. |
| **Field Constraints** | `[Required]`, `[StringLength]`, `[Range]` | `Field(min_length=3, ge=1)` | Pydantic `Field()` attributes integrate directly into OpenAPI/Swagger schemas. |
| **Custom Field Validation** | `RuleFor(x => x.Slug).Must(...)` (FluentValidation) | `@field_validator("slug", mode="after")` | Validators are attached directly to the model definition as class methods. |
| **Cross-Field Validation** | `RuleFor(x => x).Must(...)` | `@model_validator(mode="after")` | Validates entire typed instance `self` after field-level checks pass. |
| **Immutability** | `readonly record struct` or `{ get; init; }` | `model_config = ConfigDict(frozen=True)` | Immutable models in Pydantic are hashable and thread-safe. |
| **Strict Type Checking** | Strong typing enforced at compile time | `model_config = ConfigDict(strict=True)` | Disables dynamic coercion; strings are not coerced to numbers. |
| **Computed Properties** | `public int Count => Items.Count;` | `@computed_field` with `@property` | Computed fields are automatically serialized in JSON and OpenAPI specs. |
| **Polymorphism** | `[JsonPolymorphic]`, `[JsonDerivedType]` | `Annotated[Union[A, B], Field(discriminator="type")]` | Pydantic uses Rust-level $O(1)$ dictionary lookups on the discriminator. |
| **Arbitrary Type Validation** | `JsonSerializer.Deserialize<List<T>>(json)` | `TypeAdapter(list[T]).validate_json(json)` | Validates any standalone primitive, list, or union without boilerplate models. |
| **Configuration / Secrets** | `IOptions<T>`, `appsettings.json`, UserSecrets | `BaseSettings` + `SecretStr` (`pydantic-settings`) | Fails fast on startup if environment variables are invalid or missing. |
| **Error Handling** | `ValidationException` (`List<ValidationFailure>`) | `ValidationError` (`e.errors()`) | Structured error dictionaries with machine-readable `loc`, `msg`, and `type`. |

---

## 11. Common Mistakes & Anti-Patterns

> [!CAUTION]
> ### Anti-Pattern 1: Mutable Default Arguments
> **The Mistake:** Writing `tags: list[str] = []` or `metadata: dict = {}` in model declarations.
> ```python
> # WRONG: In standard Python and poorly written schemas, this can share state!
> class MalformedUpload(BaseModel):
>     tags: list[str] = []
> ```
> **The Normora Standard:** Always use `default_factory=list` or `default_factory=dict`.
> ```python
> # CORRECT:
> class SafeUpload(BaseModel):
>     tags: list[str] = Field(default_factory=list)
> ```

> [!CAUTION]
> ### Anti-Pattern 2: Over-Relying on Type Coercion for Security-Critical Fields
> **The Mistake:** Relying on default coercion when accepting tenant IDs, payment amounts, or RBAC flags. If a client sends `"true"` or `"1"`, it might coerce unpredictably.
> ```python
> # WRONG: Coercion might allow unexpected inputs
> class UnsafeAdminCheck(BaseModel):
>     is_admin: bool
> ```
> **The Normora Standard:** Enable `strict=True` for security tokens, financial calculations, and LLM structured outputs.
> ```python
> # CORRECT:
> class SafeAdminCheck(BaseModel):
>     model_config = ConfigDict(strict=True)
>     is_admin: bool
> ```

> [!CAUTION]
> ### Anti-Pattern 3: Exposing `SecretStr` in Logs or Telemetry
> **The Mistake:** Calling `.get_secret_value()` directly inside logging strings or error messages.
> ```python
> # WRONG: Logs your secret to Datadog / OpenTelemetry!
> logger.info(f"Connecting with API Key: {settings.GEMINI_API_KEY.get_secret_value()}")
> ```
> **The Normora Standard:** Keep secrets wrapped in `SecretStr`. Only call `.get_secret_value()` at the exact line where you instantiate the outbound HTTPS client.
> ```python
> # CORRECT:
> logger.info(f"Connecting to Gemini API using key: {settings.GEMINI_API_KEY}")  # Logs: '**********'
> client = genai.Client(api_key=settings.GEMINI_API_KEY.get_secret_value())
> ```

> [!CAUTION]
> ### Anti-Pattern 4: Mutating Frozen Models
> **The Mistake:** Trying to modify a field in-place on a model configured with `frozen=True`.
> ```python
> # WRONG:
> chunk = DocumentChunkEmbedding(chunk_id=uuid4(), ...)
> chunk.token_count = 500  # Raises ValidationError at runtime!
> ```
> **The Normora Standard:** Treat frozen models like C# records. Use `.model_copy(update={...})` to create a new instance with updated fields.
> ```python
> # CORRECT:
> updated_chunk = chunk.model_copy(update={"token_count": 500})
> ```

> [!CAUTION]
> ### Anti-Pattern 5: Untyped Unions Instead of Discriminated Unions
> **The Mistake:** Using `Union[TypeA, TypeB]` without a discriminator. Pydantic tests types sequentially. If `TypeA` matches partially due to loose coercion, it will incorrectly parse payloads meant for `TypeB`.
> ```python
> # WRONG: Fragile and slow
> Packet = Union[TextPacket, CitationPacket]
> ```
> **The Normora Standard:** Always provide a `Literal` tag and declare `Field(discriminator="tag_name")`.
> ```python
> # CORRECT:
> Packet = Annotated[Union[TextPacket, CitationPacket], Field(discriminator="type")]
> ```

> [!CAUTION]
> ### Anti-Pattern 6: Calling Legacy Pydantic v1 Methods
> **The Mistake:** Using `model.dict()` or `model.json()`. In Pydantic v2, these legacy methods trigger deprecation warnings and bypass the high-speed Rust serialization engine.
> ```python
> # WRONG (v1 Legacy):
> data = model.dict()
> raw_json = model.json()
> ```
> **The Normora Standard:** Always call `.model_dump()` and `.model_dump_json()`.
> ```python
> # CORRECT (v2 Rust Engine):
> data = model.model_dump()
> raw_json = model.model_dump_json()
> ```

> [!CAUTION]
> ### Anti-Pattern 7: Cross-Field Validation Inside `@field_validator`
> **The Mistake:** Attempting to validate field `A` based on field `B` inside `@field_validator("A")`. In Pydantic v2, the second field may not have been parsed yet!
> ```python
> # WRONG: Unreliable field ordering
> @field_validator("end_date")
> def check_dates(cls, v, info):
>     # info.data might not contain 'start_date' yet!
>     pass
> ```
> **The Normora Standard:** Always use `@model_validator(mode="after")` for rules involving two or more fields.
> ```python
> # CORRECT:
> @model_validator(mode="after")
> def check_date_bounds(self) -> "Model":
>     if self.end_date < self.start_date:
>         raise ValueError("end_date must be after start_date")
>     return self
> ```

---

## 12. Senior Architect Interview Questions & Answers

### Q1: How does Pydantic v2 achieve up to 20x performance improvement over Pydantic v1?
**Answer:**
Pydantic v1 was implemented primarily in pure Python. Every field validation, recursive dictionary traversal, and regex check incurred Python bytecode interpretation overhead.

Pydantic v2 moved the entire core validation and serialization engine into **Rust (`pydantic-core`)**. When a Python class subclassing `BaseModel` is defined, Pydantic compiles its field definitions into a Rust-level validation schema DAG (Directed Acyclic Graph). When data enters (via JSON or dict), it is passed directly across the C-FFI boundary into Rust, where type parsing, string sanitization, and boundary checks execute in compiled native code with zero GIL contention.

### Q2: What is the fundamental difference between `model_validator(mode='before')` and `model_validator(mode='after')`?
**Answer:**
- `mode='before'`: Executes **before** any field parsing or type coercion takes place. The input argument is a raw untrusted data structure (typically a Python `dict` or incoming JSON object). It is used for reshaping incoming payloads, renaming legacy keys (e.g., mapping Keycloak JWT claims), or normalizing structures before Pydantic validates fields.
- `mode='after'`: Executes **after** all individual field validators and type checks have succeeded. It receives `self` (the fully typed and instantiated Pydantic model). It is used for cross-field business logic (e.g., verifying that a tenant's requested seats do not exceed their subscription tier limit).

### Q3: Why are Discriminated Unions critical in high-throughput AI streaming pipelines?
**Answer:**
In a standard union (`Union[A, B, C]`), Pydantic must sequentially evaluate incoming JSON against each model until one parses without error ($O(N)$ trial-and-error). This introduces severe latency and often results in subtle bugs where a payload intended for model `C` is coerced into model `A` because it satisfies a subset of optional fields.

A **Discriminated Union** uses a tag field (e.g., `"type"`) with literal values. Pydantic inspects only that tag and immediately routes the payload to the corresponding Rust schema in $O(1)$ time. If the tag is invalid, it fails immediately with a descriptive error.

### Q4: How does `extra='forbid'` protect against Mass-Assignment (Over-Posting) attacks?
**Answer:**
In modern web APIs, clients send JSON payloads that map directly to internal database entities. In a Mass-Assignment attack, an attacker submits unadvertised properties (such as `"is_super_admin": true` or `"tenant_id": "victim_tenant"`).

By default, Pydantic uses `extra='ignore'`, which silently strips unknown fields. While safe, it can conceal malicious intent. Setting `extra='forbid'` causes Pydantic to raise an immediate `ValidationError` if any unexpected field is present, alerting security filters (BFLA/BOLA) and preventing attackers from probing for hidden model properties.

### Q5: When should an architect use `TypeAdapter` instead of `BaseModel`?
**Answer:**
Use `BaseModel` when defining structured domain objects with named fields (e.g., `TenantCreateRequest`, `DocumentMetadata`).

Use **`TypeAdapter`** when validating top-level types that are not models:
1. Primitive collections: `list[UUID]`, `set[str]`, `dict[str, float]`.
2. High-dimensional vector arrays: `list[list[float]]` for batch pgvector embeddings.
3. Standalone unions or literal types: `Literal["red", "green", "blue"]`.
4. Parsing untrusted JSON arrays returned from LLMs directly into domain lists without creating a synthetic wrapper object.

### Q6: How does `SecretStr` prevent credential leakage in distributed tracing and logs?
**Answer:**
Accidental credential logging is a major enterprise compliance violation (SOC2, HIPAA, GDPR). In standard Python or C#, printing an object with an API key prints the raw secret.

Pydantic's `SecretStr` overrides `__str__` and `__repr__` to output `SecretStr('**********')`. Serializing a model via `.model_dump()` also hides or excludes the secret. The only way to retrieve the plaintext string is by explicitly calling `.get_secret_value()`. This prevents developers, OpenTelemetry collectors, and exception loggers from exposing Gemini API keys or database passwords in plain text.

### Q7: How does Pydantic v2 compare to C# FluentValidation from an architectural perspective?
**Answer:**
- **FluentValidation** separates validation rules into external validator classes (`AbstractValidator<T>`). Models are typically standard C# POCOs/Records, and validation is an explicit, opt-in step performed by calling `validator.Validate(instance)`.
- **Pydantic v2** adopts an integrated parsing model: validation is intrinsic to model construction. An invalid Pydantic model can never exist in memory—instantiating `Model(**data)` either succeeds with guaranteed valid types or raises a `ValidationError`. Furthermore, Pydantic handles automatic type coercion and OpenAPI schema generation natively, whereas in .NET these require separate packages and configurations.

---

## Summary & Next Step

In this module, you mastered the data defense engine that underpins high-performance Python and FastAPI architectures:
1. Defining enterprise contracts with **`BaseModel`**, **`Field`**, and safe defaults via `default_factory`.
2. Hardening schemas with **`model_config`** options: `strict=True`, `frozen=True`, `str_strip_whitespace=True`, and `extra='forbid'`.
3. Writing deep validation pipelines using **`@field_validator`** and **`@model_validator`** in both `before` and `after` modes.
4. Exposing dynamic attributes with **`@computed_field`** for automatic JSON and Swagger documentation.
5. Customizing outbound payloads with **`@field_serializer`**, **`@model_serializer`**, and `exclude_unset=True`.
6. Handling polymorphic real-time streams with **Discriminated Unions**.
7. Validating arbitrary collections and LLM arrays with **`TypeAdapter`**.
8. Creating strongly typed domain primitives with **`__get_pydantic_core_schema__`**.
9. Building resilient, production-grade configurations using **`pydantic-settings`** and **`SecretStr`**.

In **[Module 13: FastAPI Core Architecture & Web Engineering](file:///c:/Users/Sanjeev%20Kumar/Codes/learning-docs/Python_FastAPI_AI_Architect/13_Module_13_FastAPI_Core_Architecture.md)**, we will combine Pydantic with our async runtime to construct Normora's modular API endpoints, dependency injection pipelines, and real-time streaming infrastructure!
