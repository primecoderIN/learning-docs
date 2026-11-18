# Module 12: Data Validation with Pydantic v2 (The Rust Data Engine)

Welcome to **Module 12** of the Normora Python & AI Masterclass.

Generative AI applications are inherently non-deterministic: LLMs output unpredictable text, hallucinated JSON, or malformed tool arguments. If an AI backend sends invalid JSON directly to a database or frontend, your application crashes.

**Pydantic v2** is the defensive boundary wall between unpredictable external data (HTTP requests, LLM outputs, database rows) and your strictly typed application domain. Rewritten from scratch in **Rust (`pydantic-core`)**, Pydantic v2 parses and validates hundreds of thousands of complex nested payloads per second with sub-millisecond latency.

FastAPI uses Pydantic for:
1. Validating incoming HTTP JSON request bodies.
2. Generating OpenAPI / Swagger documentation automatically.
3. Serializing and filtering outbound API responses.
4. Enforcing strict schema contracts on LLM Tool Calling (Structured Outputs).

Every concept is followed by **at least 5 real-world Normora examples**.

---

## 1. `BaseModel`, Fields & Type Coercion

In Pydantic, data contracts are defined by subclassing **`BaseModel`**.  
Unlike standard Python classes or dataclasses, Pydantic performs **Data Parsing & Type Coercion**: if an API client sends `"50"` (a string) for an integer field, Pydantic automatically coerces it into `50` (an `int`), or raises a detailed `ValidationError` if the conversion is impossible.

### 5 Real-World Normora Examples:

#### Example 1.1: The `CreateTenantRequest` Schema
```python
from pydantic import BaseModel, Field

class CreateTenantRequest(BaseModel):
    name: str = Field(min_length=3, max_length=100, description="Organization legal name")
    subscription_plan: str = Field(default="Pro")
    active_seats: int = Field(ge=1, le=5000, description="Must have at least 1 seat")
    enforce_mfa: bool = False

# Validating valid data:
payload = {"name": "Acme Health", "active_seats": "45"}  # Notice: string "45" coerced to int!
request = CreateTenantRequest(**payload)
print(f"Validated: {request.name}, Seats: {request.active_seats} (Type: {type(request.active_seats)})")
```

#### Example 1.2: Catching `ValidationError` on Invalid Data
```python
from pydantic import ValidationError

invalid_payload = {"name": "A", "active_seats": 0}  # Name too short, seats < 1

try:
    CreateTenantRequest(**invalid_payload)
except ValidationError as e:
    print("--- Pydantic Validation Error Details ---")
    for error in e.errors():
        field_name = error["loc"][0]
        msg = error["msg"]
        print(f"Field '{field_name}': {msg}")
```

#### Example 1.3: Document Upload Metadata Schema
```python
class DocumentUploadMetadata(BaseModel):
    title: str = Field(min_length=1, max_length=255)
    department: str = Field(default="General")
    is_confidential: bool = False
    tags: list[str] = Field(default_factory=list)  # Safe default factory!

upload = DocumentUploadMetadata(title="Cybersecurity_Policy_2026.pdf", tags=["security", "compliance"])
print(f"Document: {upload.title}, Tags: {upload.tags}")
```

#### Example 1.4: RAG Semantic Search Query Request
```python
class SearchQueryRequest(BaseModel):
    query: str = Field(min_length=3, max_length=500, description="User search query")
    similarity_threshold: float = Field(default=0.75, ge=0.0, le=1.0)
    top_k: int = Field(default=5, ge=1, le=50)

search = SearchQueryRequest(query="What is our maternity leave policy?", top_k=3)
print(f"Query: '{search.query}', Threshold: {search.similarity_threshold}, Top K: {search.top_k}")
```

#### Example 1.5: Chat Message Block Schema
```python
from typing import Literal

class ChatMessageSchema(BaseModel):
    role: Literal["system", "user", "assistant", "tool"]
    content: str = Field(min_length=1)
    timestamp: float | None = None

msg = ChatMessageSchema(role="user", content="Where is our travel policy?")
print(f"Role: {msg.role}, Content: {msg.content}")
```

---

## 2. Deep Validation: `@field_validator` & `@model_validator`

* **`@field_validator`**: Validates or cleans a single specific field. Can run in `mode='before'` (before Pydantic parses types) or `mode='after'` (after standard type checking).
* **`@model_validator(mode='after')`**: Validates cross-field dependencies (where validation logic depends on multiple fields together).

### 5 Real-World Normora Examples:

#### Example 2.1: Field Validator for Slug Sanitization
Ensure tenant slugs are lowercase and contain only alphanumeric characters and hyphens:
```python
from pydantic import field_validator

class TenantSlugRequest(BaseModel):
    company_name: str
    slug: str

    @field_validator("slug")
    @classmethod
    def validate_slug(cls, v: str) -> str:
        clean_slug = v.strip().lower()
        if not clean_slug.replace("-", "").isalnum():
            raise ValueError("Slug must contain only alphanumeric characters and hyphens.")
        return clean_slug

req = TenantSlugRequest(company_name="Acme Corp", slug="  ACME-GLOBAL  ")
print(f"Cleaned slug: '{req.slug}'")  # 'acme-global'
```

#### Example 2.2: Field Validator for File Extension Whitelisting
```python
class IngestionRequest(BaseModel):
    filename: str

    @field_validator("filename")
    @classmethod
    def check_extension(cls, v: str) -> str:
        allowed = (".pdf", ".docx", ".txt", ".md")
        if not v.lower().endswith(allowed):
            raise ValueError(f"File must end with one of {allowed}")
        return v

doc = IngestionRequest(filename="Security_Checklist.pdf")
print(f"Valid document: {doc.filename}")
```

#### Example 2.3: Model Validator for Cross-Field Token Limit Logic
Cross-field check: If plan is "Free", `max_tokens` cannot exceed 100,000:
```python
from pydantic import model_validator

class WorkspaceQuotaConfig(BaseModel):
    plan: Literal["Free", "Pro", "Enterprise"]
    max_tokens: int

    @model_validator(mode="after")
    def verify_plan_quota_alignment(self) -> "WorkspaceQuotaConfig":
        if self.plan == "Free" and self.max_tokens > 100_000:
            raise ValueError("Free tier cannot be allocated more than 100,000 tokens.")
        return self

# Valid Pro tier:
quota = WorkspaceQuotaConfig(plan="Pro", max_tokens=1_000_000)

try:
    # Invalid Free tier:
    WorkspaceQuotaConfig(plan="Free", max_tokens=500_000)
except ValidationError as e:
    print(f"Cross-field error: {e.errors()[0]['msg']}")
```

#### Example 2.4: Model Validator for Date Range Validity
```python
from datetime import datetime

class AuditLogQuery(BaseModel):
    start_date: datetime
    end_date: datetime

    @model_validator(mode="after")
    def validate_date_range(self) -> "AuditLogQuery":
        if self.end_date < self.start_date:
            raise ValueError("end_date cannot be earlier than start_date.")
        return self

query = AuditLogQuery(
    start_date=datetime(2026, 1, 1),
    end_date=datetime(2026, 1, 31)
)
print("Audit query dates valid.")
```

#### Example 2.5: Sanitizing Search Queries (Stripping Injection Noise)
```python
class CleanSearchQuery(BaseModel):
    text: str

    @field_validator("text", mode="before")
    @classmethod
    def strip_and_sanitize(cls, v: str) -> str:
        if isinstance(v, str):
            return v.strip().replace("\x00", "")  # Strip null bytes
        return v

sanitized = CleanSearchQuery(text="   What is the vacation policy?   ")
print(f"Sanitized: '{sanitized.text}'")
```

---

## 3. Serialization, Aliases & camelCase Formatting

In Python backends, naming convention is `snake_case` (e.g., `tenant_id`).  
In frontend frameworks like Angular 22, JSON convention is `camelCase` (e.g., `tenantId`).

Pydantic v2 handles this seamlessly using **Field Aliases** or global **Alias Generators**.

* `.model_dump()`: Serializes model to a Python `dict`.
* `.model_dump_json()`: Serializes model directly to a JSON string.
* `by_alias=True`: Uses camelCase aliases on serialization.
* `exclude_unset=True`: Omits fields that were not explicitly provided in the request.

### 5 Real-World Normora Examples:

#### Example 3.1: camelCase Serialization for Angular 22 Client
```python
from pydantic import ConfigDict
from pydantic.alias_generators import to_camel

class CamelCaseModel(BaseModel):
    model_config = ConfigDict(
        alias_generator=to_camel,
        populate_by_name=True  # Allows instantiation using either snake_case or camelCase!
    )

class TenantResponseDto(CamelCaseModel):
    tenant_id: str
    company_name: str
    active_seat_count: int
    is_subscription_active: bool

dto = TenantResponseDto(
    tenant_id="T-99",
    company_name="Acme Health",
    active_seat_count=45,
    is_subscription_active=True
)

# Serialize to camelCase JSON for Angular:
json_output = dto.model_dump_json(by_alias=True, indent=2)
print("--- JSON Delivered to Angular SPA ---")
print(json_output)
```
**Output JSON:**
```json
{
  "tenantId": "T-99",
  "companyName": "Acme Health",
  "activeSeatCount": 45,
  "isSubscriptionActive": true
}
```

#### Example 3.2: Parsing Incoming camelCase JSON from Frontend
Because `populate_by_name=True` is enabled, Pydantic parses incoming Angular JSON effortlessly:
```python
incoming_angular_json = {
    "tenantId": "T-102",
    "companyName": "Fintech Global",
    "activeSeatCount": 120,
    "isSubscriptionActive": True
}

parsed_dto = TenantResponseDto(**incoming_angular_json)
print(f"Parsed in Python: {parsed_dto.tenant_id} -> {parsed_dto.company_name}")
```

#### Example 3.3: Excluding Unset Values on Partial Updates (HTTP PATCH)
```python
class UpdateTenantPatch(BaseModel):
    name: str | None = None
    plan: str | None = None
    is_active: bool | None = None

# User only wants to update 'plan':
patch = UpdateTenantPatch(plan="Enterprise")

# exclude_unset=True leaves out 'name' and 'is_active':
update_dict = patch.model_dump(exclude_unset=True)
print(f"SQL UPDATE payload: {update_dict}")  # {'plan': 'Enterprise'}
```

#### Example 3.4: Excluding Sensitive Passwords from Serialization
```python
class UserAccountDto(BaseModel):
    user_id: str
    email: str
    hashed_password: str = Field(exclude=True)  # NEVER serialized!

user = UserAccountDto(user_id="U-1", email="admin@normora.ai", hashed_password="$2b$12$eXampLe...")
print("Serialized user (password safely omitted):")
print(user.model_dump())  # {'user_id': 'U-1', 'email': 'admin@normora.ai'}
```

#### Example 3.5: Generating Standard JSON Schema for LLM Tool Declarations
Pydantic can generate standard JSON Schemas directly using `.model_json_schema()`:
```python
class SearchCompanyDocumentsParams(BaseModel):
    query: str = Field(description="The semantic search query keywords.")
    department: str = Field(default="General", description="Filter by department.")
    limit: int = Field(default=3, description="Maximum chunks to retrieve.")

json_schema = SearchCompanyDocumentsParams.model_json_schema()
print("Generated Tool Parameters Schema for Google Gemini:")
print(f"Properties: {list(json_schema['properties'].keys())}")
```

---

## 4. Polymorphic Schemas with Discriminated Unions

In AI applications, an incoming stream or tool output can be one of several different types:
* A `TextChunk`
* A `CitationFootnote`
* An `ActionRequest`

A **Discriminated Union** uses a discriminator field (e.g. `"type"`) to let Pydantic instantly pick the exact model to parse without trial-and-error guessing.

### 5 Real-World Normora Examples:

#### Example 4.1: Polymorphic AI Stream Packets
```python
from typing import Union, Annotated

class TextStreamPacket(BaseModel):
    type: Literal["text"]
    token: str

class CitationStreamPacket(BaseModel):
    type: Literal["citation"]
    document_name: str
    page: int

class ToolCallStreamPacket(BaseModel):
    type: Literal["tool_call"]
    tool_name: str
    arguments: dict

# Define Discriminated Union:
StreamPacket = Annotated[
    Union[TextStreamPacket, CitationStreamPacket, ToolCallStreamPacket],
    Field(discriminator="type")
]

class EventEnvelope(BaseModel):
    packet: StreamPacket

p1 = EventEnvelope(packet={"type": "text", "token": "Hello"})
p2 = EventEnvelope(packet={"type": "citation", "document_name": "Handbook.pdf", "page": 4})

print(f"Packet 1 parsed as: {type(p1.packet).__name__}")  # TextStreamPacket
print(f"Packet 2 parsed as: {type(p2.packet).__name__}")  # CitationStreamPacket
```

#### Example 4.2: Departmental Notification Events
```python
class EmailNotification(BaseModel):
    channel: Literal["email"]
    recipient: str

class SlackNotification(BaseModel):
    channel: Literal["slack"]
    channel_id: str

NotificationChannel = Annotated[
    Union[EmailNotification, SlackNotification],
    Field(discriminator="channel")
]

class DispatchJob(BaseModel):
    target: NotificationChannel

job = DispatchJob(target={"channel": "email", "recipient": "hr@normora.ai"})
print(f"Channel target: {job.target.channel} -> {job.target.recipient}")
```

#### Example 4.3: RAG Retrieval Candidate Types (Dense vs Keyword)
```python
class DenseVectorCandidate(BaseModel):
    method: Literal["dense"]
    cosine_similarity: float

class SparseKeywordCandidate(BaseModel):
    method: Literal["sparse"]
    bm25_rank: int

Candidate = Annotated[
    Union[DenseVectorCandidate, SparseKeywordCandidate],
    Field(discriminator="method")
]

c = Candidate.model_validate({"method": "dense", "cosine_similarity": 0.92})
print(f"Parsed candidate: {c.method} (Score: {c.cosine_similarity})")
```

#### Example 4.4: Dynamic Generative UI Component Specs (A2UI)
```python
class CardComponent(BaseModel):
    widget: Literal["card"]
    title: str
    description: str

class TableComponent(BaseModel):
    widget: Literal["table"]
    headers: list[str]
    rows: list[list[str]]

UiComponent = Annotated[
    Union[CardComponent, TableComponent],
    Field(discriminator="widget")
]

ui = UiComponent.model_validate({"widget": "card", "title": "HR Summary", "description": "20 days PTO"})
print(f"Delivering UI Component: <{ui.widget}> with title '{ui.title}'")
```

#### Example 4.5: Polymorphic Document Extraction Results
```python
class SuccessExtraction(BaseModel):
    status: Literal["success"]
    extracted_text: str
    chunk_count: int

class FailureExtraction(BaseModel):
    status: Literal["failed"]
    error_code: str
    retryable: bool

ExtractionResult = Annotated[
    Union[SuccessExtraction, FailureExtraction],
    Field(discriminator="status")
]

res = ExtractionResult.model_validate({"status": "failed", "error_code": "CORRUPTED_PDF", "retryable": False})
print(f"Extraction result: {res.status} (Code: {res.error_code})")
```

---

## 5. Enterprise Settings Management (`pydantic-settings`)

In production, passwords, database connection strings, and cloud AI API keys must **never be hardcoded in code**.

`pydantic-settings` automatically reads environment variables or `.env` files, validates their types at startup, and provides **Secret Masking (`SecretStr`)** so secrets are never accidentally printed in logs!

### 5 Real-World Normora Examples:

#### Example 5.1: The Production Normora `AppSettings` Model
```python
from pydantic_settings import BaseSettings, SettingsConfigDict
from pydantic import SecretStr

class AppSettings(BaseSettings):
    # App Environment:
    ENVIRONMENT: Literal["development", "staging", "production"] = "development"
    PORT: int = 8000
    
    # Hosted AI Provider API Keys (Masked!):
    GEMINI_API_KEY: SecretStr = SecretStr("mock_gemini_secret_key_12345")
    OPENAI_API_KEY: SecretStr | None = None
    
    # Database Settings:
    POSTGRES_HOST: str = "localhost"
    POSTGRES_PORT: int = 5432
    POSTGRES_DB: str = "normora_db"
    
    model_config = SettingsConfigDict(env_file=".env", env_file_encoding="utf-8")

settings = AppSettings()
print(f"Running on port: {settings.PORT} in {settings.ENVIRONMENT} mode.")

# SecretStr masks the secret in print/repr:
print(f"API Key masked: {settings.GEMINI_API_KEY}")  # SecretStr('**********')

# Extract raw secret safely when making outbound HTTPS requests:
raw_key = settings.GEMINI_API_KEY.get_secret_value()
print(f"Can extract raw value securely when needed: {raw_key[:4]}***")
```

#### Example 5.2: Computed Database Connection URL
```python
class DatabaseSettings(BaseSettings):
    DB_USER: str = "normora_app"
    DB_PASSWORD: SecretStr = SecretStr("super_secure_db_pass")
    DB_HOST: str = "postgres.internal"
    DB_PORT: int = 5432
    DB_NAME: str = "normora_prod"

    @property
    def async_connection_url(self) -> str:
        # Generates asyncpg connection string for SQLAlchemy:
        pwd = self.DB_PASSWORD.get_secret_value()
        return f"postgresql+asyncpg://{self.DB_USER}:{pwd}@{self.DB_HOST}:{self.DB_PORT}/{self.DB_NAME}"

db_cfg = DatabaseSettings()
print(f"Host: {db_cfg.DB_HOST}, User: {db_cfg.DB_USER}")
```

#### Example 5.3: MinIO S3 Object Storage Settings
```python
class StorageSettings(BaseSettings):
    MINIO_ENDPOINT: str = "minio:9000"
    MINIO_BUCKET_DOCUMENTS: str = "normora-documents"
    MINIO_SECURE: bool = False

storage = StorageSettings()
print(f"Target MinIO bucket: {storage.MINIO_BUCKET_DOCUMENTS}")
```

#### Example 5.4: AI Model Hyperparameter Settings
```python
class AiModelSettings(BaseSettings):
    DEFAULT_MODEL: str = "gemini-1.5-flash"
    DEFAULT_TEMPERATURE: float = Field(default=0.2, ge=0.0, le=2.0)
    DEFAULT_MAX_TOKENS: int = Field(default=2048, le=8192)

ai_cfg = AiModelSettings()
print(f"Default model: {ai_cfg.DEFAULT_MODEL} (Temperature: {ai_cfg.DEFAULT_TEMPERATURE})")
```

#### Example 5.5: Fail-Fast Startup Validation
If a mandatory environment variable is missing (e.g. `DATABASE_URL` is omitted with no default), Pydantic halts application startup immediately with a clear error message, preventing broken deployments in CI/CD or Docker!

---

## Summary & Next Step

You have mastered:
1. Defining strict data contracts using `BaseModel` and `Field`.
2. Validating complex logic with `@field_validator` and `@model_validator`.
3. Auto-generating camelCase JSON for Angular clients with `alias_generator=to_camel`.
4. Designing polymorphic schemas using Discriminated Unions.
5. Managing production configurations and secret keys with `pydantic-settings`.

In **Module 13**, we combine Pydantic with our async runtime to build **FastAPI Core Architecture & Web Engineering**!
