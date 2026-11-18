# Module 6: Object-Oriented Programming (OOP) from Scratch

Welcome to **Module 6** of the Normora Python & AI Masterclass.

In enterprise software engineering, raw scripts and loose functions quickly collapse under the weight of growing complexity. Data structures and their associated business rules must be bundled into cohesive, maintainable, and type-safe domain models. In **Normora**—our enterprise multi-tenant knowledge management and conversational AI platform—entities such as `Tenant`, `DocumentChunk`, `UserMembership`, `GeminiEmbeddingEngine`, and `RagPipelineService` are designed using **Object-Oriented Programming (OOP)**.

If you come from a **C# / .NET 10** background, Python's OOP system will feel both refreshingly flexible and occasionally surprising. Python does not have a separate `interface` keyword, uses explicit `self` in instance methods, supports multiple inheritance with C3 linearization, and treats classes themselves as first-class runtime objects.

In this module, you will master Python OOP from the absolute ground up:
* The core mechanics: classes, instances, the `__init__` constructor, and what `self` physically represents.
* Instance attributes vs. class attributes (and the dreaded mutable class attribute pitfall).
* Instance methods, `@classmethod` (factory constructors), and `@staticmethod` utilities.
* Pythonic encapsulation with the `@property` decorator (getters, setters, and deleters).
* Inheritance, `super()`, and runtime polymorphism.
* Multiple inheritance, mixins, and Method Resolution Order (MRO) via C3 linearization.
* Abstract Base Classes (`abc.ABC` and `@abstractmethod`) for strict contract enforcement.
* The modern `__init_subclass__` hook for metaprogramming and plugin registration.
* High-performance domain modeling with `@dataclass` (`slots=True`, `frozen=True`, `field()`, `__post_init__`).
* Python's rich Data Model: special "dunder" methods (`__str__`, `__repr__`, `__eq__`, `__hash__`, `__lt__`, `__len__`, `__getitem__`, `__iter__`, `__enter__`/`__exit__`).
* The Descriptors protocol (`__get__`, `__set__`, `__delete__`) that powers modern Python frameworks.
* Composition vs. Inheritance: architectural decision rules for production AI pipelines.
* A brief demystification of Metaclasses.
* Normora's complete Python class architecture visualized with Mermaid diagrams.
* A comprehensive **Python OOP vs. C# OOP** reference table.
* Common Mistakes & Anti-Patterns highlighted with `> [!CAUTION]` alerts.
* Senior interview questions with model answers.

Every concept is backed by **at least 5 real-world Normora domain examples**.

---

## 1. Classes, Instances & The `__init__` Constructor

### What is a Class? What is an Instance?
A **Class** is an architectural blueprint. It defines the state (attributes) and behavior (methods) that objects constructed from it will possess. An **Instance** (or object) is a concrete, individual allocation created in memory from that blueprint.

```text
Class Blueprint:  Tenant(tenant_id, name, plan)
                         │
         ┌───────────────┴───────────────┐
         ▼                               ▼
Instance 1:                     Instance 2:
tenant_id: "T-101"              tenant_id: "T-102"
name: "Acme Health"             name: "Fintech Labs"
plan: "Enterprise"              plan: "Pro"
```

In Python, you define a class using the `class` keyword followed by `PascalCase` naming (per PEP 8). When you invoke the class name with parentheses—such as `tenant = Tenant("T-101", "Acme Health", "Enterprise")`—Python allocates the object in memory and automatically calls the special initializer method named `__init__`.

### What is `self`?
In C#, the current instance is referenced implicitly using the `this` keyword. In Python, **`self` is explicit**. 

When you define any instance method (including `__init__`), the very first parameter **must** be `self`. When you call `tenant.suspend()`, Python translates that behind the scenes into:
```python
Tenant.suspend(tenant)
```
`self` is simply a reference to the specific instance currently being operated on. Inside `__init__`, assigning `self.tenant_id = tenant_id` binds the argument to that unique object's dictionary (`__dict__`).

> [!NOTE]
> `self` is not a reserved Python keyword—it is a universal convention. You *could* technically name it `this` or `me`, but doing so violates PEP 8 and will confuse any Python linter or teammate. Always name it `self`.

### C# vs. Python: Constructor Mechanics
| Feature | C# (.NET 10) | Python 3.12+ |
| :--- | :--- | :--- |
| **Constructor Name** | Same as class name (or Primary Constructor `class Tenant(string id)`) | `def __init__(self, ...):` |
| **Instance Reference** | Implicit `this` | Explicit `self` (first argument) |
| **Field Declaration** | Explicitly declared in class body before assignment | Dynamically created inside `__init__` via `self.attribute = ...` |
| **Return Type** | No return type allowed | `__init__` always returns `None` implicitly |
| **Multiple Constructors** | Method overloading (`Tenant()`, `Tenant(id)`) | Single `__init__`; multiple constructors handled via `@classmethod` factories |

---

### 5 Real-World Normora Examples

#### Example 1.1: The `Tenant` Domain Model
In Normora, every workspace operates under tenant isolation. The `Tenant` entity tracks workspace keys, licensing tiers, and active status:

```python
class Tenant:
    def __init__(self, tenant_id: str, name: str, plan: str):
        # Explicitly assign instance attributes to self
        self.tenant_id: str = tenant_id
        self.name: str = name
        self.plan: str = plan
        self.is_active: bool = True
        self.monthly_token_usage: int = 0

    def record_token_consumption(self, tokens_used: int) -> None:
        """Increment tenant token counter and verify status."""
        self.monthly_token_usage += tokens_used
        print(f"[{self.tenant_id}] Added {tokens_used:,} tokens. Total: {self.monthly_token_usage:,}")

# Instantiating two independent objects:
tenant_a = Tenant("T-101", "Acme Health", "Enterprise")
tenant_b = Tenant("T-102", "Fintech Labs", "Pro")

tenant_a.record_token_consumption(14_250)
tenant_b.record_token_consumption(3_100)

print(f"Tenant A usage: {tenant_a.monthly_token_usage:,}")  # 14,250
print(f"Tenant B usage: {tenant_b.monthly_token_usage:,}")  # 3,100
```

#### Example 1.2: The `DocumentChunk` Vector Model
When processing corporate PDFs, raw text is partitioned into semantically coherent chunks before embedding generation:

```python
class DocumentChunk:
    def __init__(
        self,
        chunk_id: str,
        document_id: str,
        content: str,
        page_number: int,
        token_count: int
    ):
        self.chunk_id: str = chunk_id
        self.document_id: str = document_id
        self.content: str = content
        self.page_number: int = page_number
        self.token_count: int = token_count
        self.embedding: list[float] = []  # Populated later by Google Gemini Flash

chunk = DocumentChunk(
    chunk_id="chk-089",
    document_id="doc-security-policy",
    content="All employees must rotate API credentials every 90 days.",
    page_number=14,
    token_count=11
)

print(f"Initialized chunk '{chunk.chunk_id}' for page {chunk.page_number} ({chunk.token_count} tokens)")
```

#### Example 1.3: The `TenantMembership` RBAC Entity
Normora enforces multi-tenant Role-Based Access Control (RBAC). A user can be an "Owner" in one tenant and a "Member" in another:

```python
class TenantMembership:
    def __init__(self, membership_id: str, user_id: str, tenant_id: str, role: str):
        self.membership_id: str = membership_id
        self.user_id: str = user_id
        self.tenant_id: str = tenant_id
        self.role: str = role  # "Owner", "Admin", "Member", "Viewer"
        self.is_suspended: bool = False

    def can_manage_users(self) -> bool:
        """Business rule: Only Owner and Admin can manage workspace users."""
        return not self.is_suspended and self.role in ("Owner", "Admin")

membership_admin = TenantMembership("m-1", "usr-sanjeev", "T-101", "Admin")
membership_viewer = TenantMembership("m-2", "usr-alex", "T-101", "Viewer")

print(f"Admin can manage users: {membership_admin.can_manage_users()}")    # True
print(f"Viewer can manage users: {membership_viewer.can_manage_users()}")  # False
```

#### Example 1.4: The `ChatMessage` Conversation Model
Conversational turns in Normora track the author role, response text, and grounding citations:

```python
class ChatMessage:
    def __init__(self, message_id: str, role: str, content: str):
        self.message_id: str = message_id
        self.role: str = role  # "system", "user", "assistant"
        self.content: str = content
        self.citations: list[str] = []
        self.latency_ms: float = 0.0

    def attach_citation(self, document_title: str, page_number: int) -> None:
        """Attach verified source document reference."""
        self.citations.append(f"{document_title} (p. {page_number})")

msg = ChatMessage("msg-501", "assistant", "The annual PTO entitlement is 25 days.")
msg.attach_citation("HR_Benefits_2026.pdf", 7)
msg.latency_ms = 432.5

print(f"[{msg.role.upper()}]: {msg.content}")
print(f"Citations: {msg.citations} | Latency: {msg.latency_ms}ms")
```

#### Example 1.5: The `Department` Entity
Enterprise tenants organize documents by organizational unit:

```python
class Department:
    def __init__(self, department_id: str, tenant_id: str, name: str, code: str):
        self.department_id: str = department_id
        self.tenant_id: str = tenant_id
        self.name: str = name
        self.code: str = code.upper()
        self.total_documents_ingested: int = 0

    def increment_document_count(self, count: int = 1) -> None:
        self.total_documents_ingested += count

dept = Department("d-01", "T-101", "Engineering & Architecture", "eng")
dept.increment_document_count(12)
print(f"Dept: {dept.name} [{dept.code}] has {dept.total_documents_ingested} indexed documents.")
```

---

## 2. Instance Attributes vs. Class Attributes

Understanding the difference between **Instance Attributes** and **Class Attributes** is essential for building bug-free multi-tenant systems.

* **Instance Attributes:** Declared inside methods (typically `__init__`) prefixed with `self.`. Every instance receives its own isolated memory slot. Modifying an instance attribute on `tenant_1` has zero effect on `tenant_2`.
* **Class Attributes:** Declared directly inside the `class` body, outside any methods. They are attached to the class object itself and **shared across every single instance of that class**.

```python
class EmbeddingConfig:
    DIMENSIONS = 768  # Class Attribute: Shared across all instances
    
    def __init__(self, api_key: str):
        self.api_key = api_key  # Instance Attribute: Unique per instance
```

> [!WARNING]
> If you assign a value to a class attribute via an instance (`instance.DIMENSIONS = 1536`), Python does **not** update the class attribute! Instead, it silently creates a *new instance attribute* on that specific instance, shadowing the class attribute. To update a class attribute, always access it through the class name: `EmbeddingConfig.DIMENSIONS = 1536`.

### C# vs. Python: Static vs. Class Attributes
| C# (.NET 10) | Python 3.12+ | Purpose |
| :--- | :--- | :--- |
| `public static int Dimension = 768;` | `DIMENSION: int = 768` (in class body) | Shared across all objects |
| `public string ApiKey;` | `self.api_key = api_key` (in `__init__`) | Unique per object instance |
| `EmbeddingConfig.Dimension` | `EmbeddingConfig.DIMENSION` or `self.DIMENSION` | Direct access |

---

### 5 Real-World Normora Examples

#### Example 2.1: Default Embedding Model Configuration
The vector dimension and embedding model name are constant across all instances, but each tenant uses their own secure API key:

```python
class GoogleGeminiEmbeddingClient:
    # Class Attributes (Shared by all instances):
    PROVIDER_NAME: str = "Google Cloud Vertex AI"
    MODEL_IDENTIFIER: str = "text-embedding-004"
    EMBEDDING_DIMENSIONS: int = 768

    def __init__(self, tenant_id: str, api_key: str):
        # Instance Attributes (Unique per tenant connection):
        self.tenant_id: str = tenant_id
        self.api_key: str = api_key

client_acme = GoogleGeminiEmbeddingClient("T-101", "key_acme_sec_991")
client_fintech = GoogleGeminiEmbeddingClient("T-102", "key_fin_sec_224")

print(f"Acme Model: {client_acme.MODEL_IDENTIFIER}")        # text-embedding-004
print(f"Fintech Model: {client_fintech.MODEL_IDENTIFIER}")  # text-embedding-004
print(f"Vector Dimensions: {GoogleGeminiEmbeddingClient.EMBEDDING_DIMENSIONS}")  # 768
```

#### Example 2.2: Global Document Size Limits and Ingestion Rules
Class attributes define global infrastructure constraints, while instance attributes track specific file uploads:

```python
class DocumentUploader:
    # Class Attributes: System-wide platform constraints
    MAX_FILE_SIZE_MB: int = 50
    ALLOWED_EXTENSIONS: tuple[str, ...] = (".pdf", ".docx", ".txt", ".md")

    def __init__(self, tenant_id: str, uploaded_by_user_id: str):
        # Instance Attributes
        self.tenant_id: str = tenant_id
        self.uploaded_by_user_id: str = uploaded_by_user_id

    def validate_file(self, filename: str, file_size_mb: float) -> bool:
        if file_size_mb > DocumentUploader.MAX_FILE_SIZE_MB:
            print(f"Rejected: {file_size_mb}MB exceeds limit of {DocumentUploader.MAX_FILE_SIZE_MB}MB.")
            return False
        if not filename.lower().endswith(DocumentUploader.ALLOWED_EXTENSIONS):
            print(f"Rejected: Extension not permitted in {DocumentUploader.ALLOWED_EXTENSIONS}")
            return False
        return True

uploader = DocumentUploader("T-101", "usr-sanjeev")
print(f"Is valid: {uploader.validate_file('architecture.pdf', 12.4)}")  # True
print(f"Is valid: {uploader.validate_file('dump.exe', 2.1)}")          # False
```

#### Example 2.3: Tracking Total Active Conversation Sessions
A class attribute can serve as an in-memory counter to track how many instances have been initialized:

```python
class ConversationSession:
    # Class Attribute: Shared counter across the application
    total_active_sessions: int = 0

    def __init__(self, conversation_id: str, tenant_id: str):
        self.conversation_id: str = conversation_id
        self.tenant_id: str = tenant_id
        # Increment shared class counter:
        ConversationSession.total_active_sessions += 1

    def close(self) -> None:
        """Decrement counter when session terminates."""
        ConversationSession.total_active_sessions -= 1
        print(f"Session {self.conversation_id} closed. Remaining: {ConversationSession.total_active_sessions}")

s1 = ConversationSession("conv-01", "T-101")
s2 = ConversationSession("conv-02", "T-101")
s3 = ConversationSession("conv-03", "T-102")

print(f"Total live sessions: {ConversationSession.total_active_sessions}")  # 3
s2.close()
print(f"Total live sessions: {ConversationSession.total_active_sessions}")  # 2
```

#### Example 2.4: Centralized API Security Messages (Preventing Enumeration)
Normora adheres to strict security rules: Broken Object-Level Authorization (BOLA) returns `404 Not Found` to prevent attackers from discovering valid resource IDs. Class attributes centralize these standard messages:

```python
class ApiMessages:
    # Centralized constants (replaces C# public static class ApiMessages)
    BOLA_RESOURCE_NOT_FOUND: str = "The requested resource could not be found."
    BFLA_FORBIDDEN: str = "Access denied: insufficient permissions for this operation."
    TOKEN_QUOTA_EXHAUSTED: str = "Tenant token quota exceeded. Please upgrade your subscription."
    INVALID_DOCUMENT_FORMAT: str = "Unsupported document format or corrupted file."

print(f"Security error response: {ApiMessages.BOLA_RESOURCE_NOT_FOUND}")
```

#### Example 2.5: MinIO S3 Object Storage Configuration
Default bucket names and AWS-compatible endpoint regions are shared across all storage client instances:

```python
class MinioStorageClient:
    DEFAULT_REGION: str = "us-east-1"
    DOCUMENTS_BUCKET: str = "normora-documents"
    PROCESSED_CHUNKS_BUCKET: str = "normora-chunks"

    def __init__(self, endpoint_url: str, access_key: str, secret_key: str):
        self.endpoint_url: str = endpoint_url
        self.access_key: str = access_key
        self._secret_key: str = secret_key  # Protected instance attribute

    def get_document_s3_uri(self, tenant_id: str, file_name: str) -> str:
        # Combines class bucket name with tenant-isolated path
        return f"s3://{MinioStorageClient.DOCUMENTS_BUCKET}/{tenant_id}/{file_name}"

storage = MinioStorageClient("http://localhost:9000", "minioadmin", "minioadmin")
print(storage.get_document_s3_uri("T-101", "Compliance_Checklist.pdf"))
# Output: s3://normora-documents/T-101/Compliance_Checklist.pdf
```

---

## 3. Instance Methods, `@classmethod`, and `@staticmethod`

Methods defined inside a Python class fall into three distinct categories depending on what context they receive:

1. **Instance Methods (`def method(self, ...)`):** The default method type. Receives the specific object instance as `self`. Can read and modify instance state (`self.x`) as well as class state (`self.__class__.y`).
2. **Class Methods (`@classmethod`, `def method(cls, ...)`):** Decorated with `@classmethod`. Receives the class itself as its first parameter (`cls`), not an instance. Most commonly used as **Factory Constructors** to build instances from alternate data formats (like raw JSON dictionaries, database rows, or environment settings).
3. **Static Methods (`@staticmethod`, `def method(...)`):** Decorated with `@staticmethod`. Receives neither `self` nor `cls`. It is a plain utility function logically scoped inside the class namespace.

```mermaid
flowchart TD
    Class[Python Class]
    Class --> IM["Instance Method<br>def fn(self) -> Reads/writes instance state"]
    Class --> CM["@classmethod<br>def fn(cls) -> Factory constructors, class-level actions"]
    Class --> SM["@staticmethod<br>def fn() -> Pure utility functions, no self or cls"]
```

### C# vs. Python Method Comparison
| Method Type | C# (.NET 10) | Python 3.12+ | Purpose |
| :--- | :--- | :--- | :--- |
| **Instance Method** | `public void DeductTokens(int n)` | `def deduct_tokens(self, n: int):` | Operates on this specific object |
| **Class Method** | `public static Tenant FromJson(string json)` | `@classmethod def from_json(cls, data):` | Factory constructor / polymorphic creation |
| **Static Method** | `public static string Clean(string text)` | `@staticmethod def clean(text):` | Pure utility, no instance or class state needed |

---

### 5 Real-World Normora Examples

#### Example 3.1: Instance Method Modifying Tenant Financial State
Instance methods have direct write access to `self`:

```python
class TenantAccount:
    def __init__(self, tenant_id: str, balance_usd: float):
        self.tenant_id: str = tenant_id
        self.balance_usd: float = balance_usd
        self.is_suspended: bool = False

    # Instance method:
    def deduct_ai_charges(self, amount: float) -> bool:
        """Deduct API usage cost and suspend account if balance is negative."""
        self.balance_usd -= amount
        if self.balance_usd < 0:
            self.is_suspended = True
            print(f"[SECURITY ALERT] Tenant {self.tenant_id} suspended. Balance: ${self.balance_usd:.2f}")
            return False
        return True

account = TenantAccount("T-99", 10.00)
account.deduct_ai_charges(4.50)   # Balance is $5.50
print(f"Suspended: {account.is_suspended}")  # False
account.deduct_ai_charges(8.00)   # Balance is -$2.50
print(f"Suspended: {account.is_suspended}")  # True
```

#### Example 3.2: `@classmethod` as a Factory Constructor (From REST API JSON)
In C#, you might write `public static DocumentMetadata FromJson(JsonElement elem)`. In Python, the idiomatic pattern is a `@classmethod` receiving `cls`:

```python
class DocumentMetadata:
    def __init__(self, document_id: str, title: str, department: str, file_size_kb: int):
        self.document_id: str = document_id
        self.title: str = title
        self.department: str = department
        self.file_size_kb: int = file_size_kb

    @classmethod
    def from_api_payload(cls, payload: dict) -> "DocumentMetadata":
        """Factory constructor instantiating directly from an untyped API dictionary."""
        # cls(...) invokes the constructor of the class (or subclass) cleanly!
        return cls(
            document_id=payload["id"],
            title=payload["name"].strip(),
            department=payload.get("dept", "General"),
            file_size_kb=int(payload.get("size_bytes", 0) / 1024)
        )

# Constructing from raw JSON dictionary:
raw_json = {"id": "doc-554", "name": "  IT_Governance_2026.pdf  ", "dept": "Security", "size_bytes": 2_048_000}
doc_meta = DocumentMetadata.from_api_payload(raw_json)

print(f"Document: '{doc_meta.title}' | Dept: {doc_meta.department} | Size: {doc_meta.file_size_kb} KB")
```

#### Example 3.3: `@classmethod` Factory Presets for Multi-Tenant Subscription Tiers
Preset tiers encapsulate complex construction logic behind clean factory methods:

```python
class TenantPlan:
    def __init__(self, plan_name: str, monthly_token_limit: int, max_documents: int, allow_custom_rag: bool):
        self.plan_name: str = plan_name
        self.monthly_token_limit: int = monthly_token_limit
        self.max_documents: int = max_documents
        self.allow_custom_rag: bool = allow_custom_rag

    @classmethod
    def create_free_tier(cls) -> "TenantPlan":
        """Factory method for self-service developer tier."""
        return cls(plan_name="Free", monthly_token_limit=100_000, max_documents=10, allow_custom_rag=False)

    @classmethod
    def create_enterprise_tier(cls) -> "TenantPlan":
        """Factory method for enterprise contract customers."""
        return cls(plan_name="Enterprise", monthly_token_limit=50_000_000, max_documents=50_000, allow_custom_rag=True)

starter = TenantPlan.create_free_tier()
enterprise = TenantPlan.create_enterprise_tier()

print(f"Starter limit: {starter.monthly_token_limit:,} tokens | Custom RAG: {starter.allow_custom_rag}")
print(f"Enterprise limit: {enterprise.monthly_token_limit:,} tokens | Custom RAG: {enterprise.allow_custom_rag}")
```

#### Example 3.4: `@staticmethod` for Ingestion Text Sanitization
A static method does not require access to instance or class state. It lives in the class namespace for logical cohesion:

```python
class TextCleaner:
    @staticmethod
    def normalize_chunk_whitespace(text: str) -> str:
        """Collapse redundant tabs, multiple newlines, and spaces into clean text."""
        return " ".join(text.split())

    @staticmethod
    def strip_html_tags(raw_html: str) -> str:
        """Minimal fast HTML tag stripper for ingested intranet pages."""
        import re
        return re.sub(r"<[^>]+>", "", raw_html)

raw_text = "Clause   4.1:\n\tAll employees   must log into Keycloak via OIDC.\n\n"
clean_text = TextCleaner.normalize_chunk_whitespace(raw_text)
print(f"Cleaned chunk: '{clean_text}'")
```

#### Example 3.5: `@staticmethod` for SHA-256 Document Content Checksumming
Normora prevents duplicate document ingestion across a tenant by calculating a cryptographic hash of the raw bytes:

```python
import hashlib

class ChecksumUtility:
    @staticmethod
    def compute_sha256(content: bytes) -> str:
        """Generate SHA-256 fingerprint for document deduplication."""
        hasher = hashlib.sha256()
        hasher.update(content)
        return hasher.hexdigest()

doc_bytes = b"Normora confidential architecture document. Module 6 OOP."
fingerprint = ChecksumUtility.compute_sha256(doc_bytes)
print(f"Document fingerprint: {fingerprint}")
```

---

## 4. Encapsulation & The `@property` Decorator

In languages like C# or Java, encapsulation is achieved through access modifiers (`private`, `protected`, `public`) combined with explicit getter and setter methods or properties:

```csharp
// C# Property:
private int _maxTokens;
public int MaxTokens {
    get => _maxTokens;
    set {
        if (value < 0) throw new ArgumentOutOfRangeException();
        _maxTokens = value;
    }
}
```

### Python's Approach to Privacy
Python has **no language-level keywords for `private` or `protected`**. Instead, Python relies on explicit naming conventions and mutual developer consent ("we are all consenting adults here"):
1. **Public:** `self.name` (free for anyone to read or modify).
2. **Protected Convention (`_single_underscore`):** `self._tenant_id`. Indicates an internal implementation detail. Linters and IDEs will warn if accessed outside the class, but the runtime does not block it.
3. **Private Name Mangling (`__double_underscore`):** `self.__secret_key`. Python mangles this attribute name at runtime to `_ClassName__secret_key` to prevent accidental name collisions in subclasses.

### The `@property` Decorator
Python provides the **`@property`** decorator to create managed properties. It allows a method to be called syntax-wise **like a normal attribute** (without parentheses `()`), while executing a function behind the scenes. This enables:
* **Computed Read-Only Attributes** (dynamic calculation on access).
* **Validation on Assignment** (via `@property_name.setter`).
* **Deletion Cleanup** (via `@property_name.deleter`).

---

### 5 Real-World Normora Examples

#### Example 4.1: Computed Read-Only Property for User Display Name
Instead of storing redundant strings that can get out of sync, compute the full display name dynamically:

```python
class UserProfile:
    def __init__(self, first_name: str, last_name: str, email: str):
        self.first_name: str = first_name
        self.last_name: str = last_name
        self.email: str = email

    @property
    def display_name(self) -> str:
        """Dynamically computed property (read-only by default)."""
        return f"{self.first_name.strip()} {self.last_name.strip()}"

user = UserProfile("Sanjeev", "Kumar", "sanjeev@normora.ai")
# Notice: No parentheses when accessing display_name!
print(f"User Display Name: {user.display_name}")

# Attempting user.display_name = "New Name" raises AttributeError: property 'display_name' has no setter
```

#### Example 4.2: Protected Attribute with Validation Setter (Token Budget Guard)
Setting a negative token quota would corrupt billing. A setter enforces validation:

```python
class TenantTokenBudget:
    def __init__(self, tenant_id: str, max_tokens: int):
        self.tenant_id: str = tenant_id
        self._max_tokens: int = 0  # Protected internal attribute
        self.max_tokens = max_tokens  # Triggers setter immediately to validate initial value!

    @property
    def max_tokens(self) -> int:
        """Getter for max_tokens."""
        return self._max_tokens

    @max_tokens.setter
    def max_tokens(self, new_limit: int) -> None:
        """Setter with business validation."""
        if not isinstance(new_limit, int):
            raise TypeError("Token limit must be an integer.")
        if new_limit < 0:
            raise ValueError(f"Invalid token limit: {new_limit}. Quota cannot be negative.")
        self._max_tokens = new_limit

budget = TenantTokenBudget("T-101", 500_000)
budget.max_tokens = 1_000_000  # Valid setter call
print(f"Updated limit: {budget.max_tokens:,}")

try:
    budget.max_tokens = -50  # Raises ValueError
except ValueError as ex:
    print(f"Validation Guard Blocked Action: {ex}")
```

#### Example 4.3: Real-Time Computed Boolean State (`is_over_budget`)
Check whether a tenant has exceeded their allocated AI budget without maintaining a separate flag:

```python
class BillingCycleStatus:
    def __init__(self, tenant_id: str, tokens_consumed: int, token_allowance: int):
        self.tenant_id: str = tenant_id
        self.tokens_consumed: int = tokens_consumed
        self.token_allowance: int = token_allowance

    @property
    def is_over_budget(self) -> bool:
        """Dynamically evaluates current usage against subscription allowance."""
        return self.tokens_consumed > self.token_allowance

    @property
    def utilization_percentage(self) -> float:
        if self.token_allowance == 0:
            return 100.0
        return round((self.tokens_consumed / self.token_allowance) * 100, 2)

status = BillingCycleStatus("T-101", tokens_consumed=125_000, token_allowance=100_000)
print(f"Over budget? {status.is_over_budget} ({status.utilization_percentage}% utilized)")
```

#### Example 4.4: Formatting Storage Bytes as Human-Readable Megabytes
Transform raw machine values into friendly representations on the fly:

```python
class DocumentStorageDescriptor:
    def __init__(self, document_id: str, size_in_bytes: int):
        self.document_id: str = document_id
        self._size_in_bytes: int = size_in_bytes

    @property
    def size_mb(self) -> float:
        """Convert raw bytes to megabytes with 2 decimal precision."""
        return round(self._size_in_bytes / (1024 * 1024), 2)

    @property
    def size_kb(self) -> float:
        return round(self._size_in_bytes / 1024, 2)

doc = DocumentStorageDescriptor("doc-77", 26_214_400)
print(f"File {doc.document_id}: {doc.size_mb} MB ({doc.size_kb} KB)")
```

#### Example 4.5: Read-Only Authorization Security Context
Protect sensitive RBAC decisions from accidental runtime tampering:

```python
class TenantSecurityContext:
    def __init__(self, tenant_id: str, user_id: str, role: str):
        self._tenant_id: str = tenant_id
        self._user_id: str = user_id
        self._role: str = role

    @property
    def tenant_id(self) -> str:
        return self._tenant_id

    @property
    def can_purge_database(self) -> bool:
        # Strictly read-only property: Only the workspace Owner can purge data
        return self._role == "Owner"

context = TenantSecurityContext("T-101", "usr-44", "Admin")
print(f"Tenant: {context.tenant_id}")
print(f"Can purge database: {context.can_purge_database}")  # False
```

---

## 5. Inheritance, `super()`, and Polymorphism

Inheritance allows a child class (subclass) to inherit attributes and methods from a parent class (base class), promoting code reuse and establishing an **is-a** relationship.

* **Base Class (Superclass):** Defines common foundational attributes and behavior.
* **Child Class (Subclass):** Inherits from the base class and can add new attributes or override existing methods.
* **`super()`:** Built-in function used to invoke the parent class's implementation of a method (most commonly used in `__init__` to initialize base state).
* **Polymorphism:** The ability to treat different subclasses through their common base class interface, where each subclass provides its own specialized behavior for an overridden method.

```mermaid
flowchart TD
    Base["BaseEntity<br>entity_id, created_at<br>get_audit_summary()"]
    T["Tenant<br>+ company_name, plan<br>get_audit_summary() [overridden]"]
    D["Document<br>+ title, byte_size"]
    C["Conversation<br>+ session_id, user_id"]
    Base --> T
    Base --> D
    Base --> C
```

### C# vs. Python: Inheritance Mechanics
| Concept | C# (.NET 10) | Python 3.12+ |
| :--- | :--- | :--- |
| **Inheritance Syntax** | `class Tenant : BaseEntity` | `class Tenant(BaseEntity):` |
| **Calling Base Constructor** | `: base(entityId)` | `super().__init__(entity_id)` |
| **Virtual Methods** | Requires `virtual` keyword in base class | All Python methods are **virtual by default** |
| **Method Overriding** | Requires `override` keyword in child class | Simply define a method with the same name in the child |
| **Sealed / Final Classes** | `sealed class Tenant` | `@typing.final` decorator (enforced by static type checkers) |

---

### 5 Real-World Normora Examples

#### Example 5.1: `BaseEntity` with Automatic Timestamp Tracking
All Normora database entities inherit from `BaseEntity` for consistent auditing:

```python
import time

class BaseEntity:
    def __init__(self, entity_id: str):
        self.entity_id: str = entity_id
        self.created_at_epoch: float = time.time()
        self.is_deleted: bool = False

    def get_audit_summary(self) -> str:
        return f"[Entity {self.entity_id}] Created at {self.created_at_epoch:.0f} (Deleted: {self.is_deleted})"

class Tenant(BaseEntity):
    def __init__(self, tenant_id: str, company_name: str, tier: str):
        # Call the parent BaseEntity constructor using super():
        super().__init__(entity_id=tenant_id)
        self.company_name: str = company_name
        self.tier: str = tier

    # Override base method to enrich audit information:
    def get_audit_summary(self) -> str:
        base_summary = super().get_audit_summary()
        return f"{base_summary} | Company: '{self.company_name}' ({self.tier})"

tenant = Tenant("T-500", "Acme Health Systems", "Enterprise")
print(tenant.get_audit_summary())
```

#### Example 5.2: Polymorphism with Vector Retrieval Engines
Different vector search strategies (Dense pgvector, Sparse BM25, and Hybrid RRF) inherit from a common base retriever:

```python
class BaseRetriever:
    def __init__(self, tenant_id: str):
        self.tenant_id: str = tenant_id

    def retrieve(self, query: str, top_k: int = 3) -> list[str]:
        """Base contract to be overridden by subclasses."""
        raise NotImplementedError("Subclasses must implement retrieve()")

class SemanticPgVectorRetriever(BaseRetriever):
    def retrieve(self, query: str, top_k: int = 3) -> list[str]:
        # Overrides with dense vector search (HNSW index on pgvector)
        return [f"[pgvector] Chunk {i} cosine-matched for '{query}'" for i in range(1, top_k + 1)]

class KeywordBM25Retriever(BaseRetriever):
    def retrieve(self, query: str, top_k: int = 3) -> list[str]:
        # Overrides with lexical keyword search (PostgreSQL tsvector)
        return [f"[BM25] Document match {i} for terms in '{query}'" for i in range(1, top_k + 1)]

# Polymorphic dispatch:
engines: list[BaseRetriever] = [
    SemanticPgVectorRetriever("T-101"),
    KeywordBM25Retriever("T-101")
]

for engine in engines:
    results = engine.retrieve("annual leave policy", top_k=2)
    print(results)
```

#### Example 5.3: Extending Document Parsers for Multi-Format Ingestion
Normora's document ingestion pipeline processes PDFs, Word documents, and Markdown files through polymorphic parsers:

```python
class BaseDocumentParser:
    def __init__(self, file_path: str):
        self.file_path: str = file_path

    def extract_plain_text(self) -> str:
        return ""

class PdfParser(BaseDocumentParser):
    def extract_plain_text(self) -> str:
        return f"[PDF Parser] Extracted raw text and OCR tables from '{self.file_path}'"

class DocxParser(BaseDocumentParser):
    def extract_plain_text(self) -> str:
        return f"[DOCX Parser] Extracted paragraph blocks and bullet lists from '{self.file_path}'"

class MarkdownParser(BaseDocumentParser):
    def extract_plain_text(self) -> str:
        return f"[Markdown Parser] Stripped frontmatter and extracted headers from '{self.file_path}'"

parsers: list[BaseDocumentParser] = [
    PdfParser("/storage/docs/handbook.pdf"),
    DocxParser("/storage/docs/benefits.docx"),
    MarkdownParser("/storage/docs/readme.md")
]

for p in parsers:
    print(p.extract_plain_text())
```

#### Example 5.4: Custom Exception Hierarchy for BOLA and BFLA Security
In Normora, security exceptions inherit from an application base exception and carry HTTP status codes:

```python
class NormoraException(Exception):
    """Base exception for all domain errors within Normora."""
    def __init__(self, message: str, status_code: int):
        super().__init__(message)
        self.status_code: int = status_code

class BolaException(NormoraException):
    """Raised when an object requested does not belong to the calling tenant."""
    def __init__(self, resource_name: str = "Resource"):
        # Map to 404 to prevent resource ID enumeration attacks
        super().__init__(message=f"{resource_name} not found.", status_code=404)

class BflaException(NormoraException):
    """Raised when user lacks required role for the operation."""
    def __init__(self, action_name: str):
        super().__init__(message=f"Forbidden: Cannot execute '{action_name}'.", status_code=403)

class QuotaExceededException(NormoraException):
    def __init__(self, current_usage: int, limit: int):
        super().__init__(
            message=f"Monthly token quota exceeded ({current_usage:,}/{limit:,}).",
            status_code=429
        )

# Simulating an API endpoint catching domain exceptions:
try:
    raise BolaException("Document 'doc-secret-payroll'")
except NormoraException as ex:
    print(f"API Error Caught -> HTTP {ex.status_code}: {ex}")
```

#### Example 5.5: Base AI Agent Tool Calling Class
Subclasses inherit configuration metadata and override the `execute` method:

```python
class BaseAiTool:
    name: str = "base_tool"
    description: str = "Base description"

    def execute(self, tenant_id: str, **kwargs) -> dict:
        raise NotImplementedError("Tool execution not implemented.")

class SearchDocumentsTool(BaseAiTool):
    name: str = "search_documents"
    description: str = "Query tenant vector store for grounded citations."

    def execute(self, tenant_id: str, **kwargs) -> dict:
        query = kwargs.get("query", "")
        return {
            "status": "success",
            "tool": self.name,
            "tenant_id": tenant_id,
            "results": [f"Matching chunk 1 for '{query}'", f"Matching chunk 2 for '{query}'"]
        }

tool = SearchDocumentsTool()
print(f"Executing AI Tool '{tool.name}':")
response = tool.execute(tenant_id="T-101", query="travel reimbursement")
print(response)
```

---

## 6. Multiple Inheritance and Method Resolution Order (MRO & C3 Linearization)

Unlike C#, Java, or TypeScript—which strictly restrict classes to single class inheritance (permitting only multiple interfaces)—**Python supports full Multiple Inheritance**. A class can inherit directly from two or more parent classes:

```python
class TenantService(AuditableMixin, CacheableMixin, BaseService):
    pass
```

### The "Diamond Problem"
When class `D` inherits from both `B` and `C`, and both `B` and `C` inherit from class `A`, which version of an overridden method does `D` execute?

```mermaid
graph TD
    A["BaseService (Parent)<br>def execute()"]
    B["AuthService<br>def execute()"]
    C["AuditService<br>def execute()"]
    D["TenantManagementService<br>Inherits from AuthService and AuditService"]
    A --> B
    A --> C
    B --> D
    C --> D
```

### C3 Linearization & `__mro__`
Python resolves this ambiguity using an algorithm called **C3 Linearization**. It creates a deterministic, strictly ordered search path for every class called the **Method Resolution Order (MRO)**.

You can inspect any class's MRO using:
* `ClassName.__mro__` (tuple of classes in resolution order)
* `ClassName.mro()` (list of classes in resolution order)

When you call `self.method()`, Python walks the MRO from left to right and stops at the very first class that implements that method. When you call `super().method()`, Python delegates to the **next class in the MRO sequence**, not necessarily the immediate syntactic parent!

### Mixins: The Pythonic Alternative to Multiple Interfaces
A **Mixin** is a lightweight, focused class designed to inject specific behavior (such as serialization, auditing, caching, or rate-limiting) into subclasses without serving as a standalone entity.

---

### 5 Real-World Normora Examples

#### Example 6.1: `AuditableMixin` and `SoftDeleteMixin` for Database Entities
Cross-cutting concerns composed into domain models without duplicating code:

```python
import time

class AuditableMixin:
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.created_at: float = time.time()
        self.last_modified_at: float = time.time()

    def touch(self) -> None:
        """Update last modified timestamp."""
        self.last_modified_at = time.time()

class SoftDeleteMixin:
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.is_deleted: bool = False
        self.deleted_at: float | None = None

    def soft_delete(self) -> None:
        self.is_deleted = True
        self.deleted_at = time.time()

class BaseDocument:
    def __init__(self, document_id: str, title: str):
        self.document_id: str = document_id
        self.title: str = title

# Multiple inheritance combining BaseDocument with both Mixins:
class ManagedDocument(AuditableMixin, SoftDeleteMixin, BaseDocument):
    def __init__(self, document_id: str, title: str):
        # Cooperatively forward arguments through the MRO:
        super().__init__(document_id=document_id, title=title)

doc = ManagedDocument("doc-101", "Normora_Security_Architecture.pdf")
print(f"Doc: {doc.title} | Created: {doc.created_at:.0f} | Deleted: {doc.is_deleted}")
doc.soft_delete()
print(f"After soft delete -> Is deleted: {doc.is_deleted} at {doc.deleted_at:.0f}")
```

#### Example 6.2: Inspecting C3 Linearization on `TenantManagementService`
Examining how Python deterministically traverses a diamond inheritance graph:

```python
class BasePlatformService:
    def execute(self) -> str:
        return "BasePlatformService"

class TenantAuthService(BasePlatformService):
    def execute(self) -> str:
        return f"TenantAuthService -> {super().execute()}"

class TenantAuditService(BasePlatformService):
    def execute(self) -> str:
        return f"TenantAuditService -> {super().execute()}"

class TenantManagementCoordinator(TenantAuthService, TenantAuditService):
    def execute(self) -> str:
        return f"Coordinator -> {super().execute()}"

coord = TenantManagementCoordinator()
print("Execution Chain:", coord.execute())
print("\nMRO Resolution Order:")
for idx, cls in enumerate(TenantManagementCoordinator.__mro__, 1):
    print(f"  {idx}. {cls.__name__}")

# Notice the resolution: Coordinator -> TenantAuthService -> TenantAuditService -> BasePlatformService -> object
```

#### Example 6.3: `OpenTelemetryTracerMixin` and `RateLimiterMixin` for AI Services
Injecting distributed tracing and rate limiting into Gemini API callers:

```python
class OpenTelemetryTracerMixin:
    def trace_operation(self, operation_name: str) -> None:
        print(f"[OpenTelemetry] Started span '{operation_name}' for instance {id(self)}")

class RateLimiterMixin:
    def __init__(self, *args, max_requests_per_minute: int = 60, **kwargs):
        super().__init__(*args, **kwargs)
        self.max_rpm: int = max_requests_per_minute

    def check_rate_limit(self) -> bool:
        print(f"[RateLimiter] Verified quota against limit of {self.max_rpm} RPM.")
        return True

class GeminiChatService(OpenTelemetryTracerMixin, RateLimiterMixin):
    def __init__(self, api_key: str, max_requests_per_minute: int = 120):
        super().__init__(max_requests_per_minute=max_requests_per_minute)
        self.api_key: str = api_key

    def send_prompt(self, prompt: str) -> str:
        self.trace_operation("gemini-1.5-flash:generate_content")
        self.check_rate_limit()
        return f"Gemini response for: '{prompt[:20]}...'"

service = GeminiChatService("gemini_live_secret", max_requests_per_minute=300)
print(service.send_prompt("What is vector cosine distance?"))
```

#### Example 6.4: `JsonSerializableMixin` for CQRS Command Payloads
Automatically adds dictionary and JSON serialization to any class:

```python
import json

class JsonSerializableMixin:
    def to_dict(self) -> dict:
        """Serializes public instance attributes into a clean dictionary."""
        return {
            key: value
            for key, value in self.__dict__.items()
            if not key.startswith("_")
        }

    def to_json(self) -> str:
        return json.dumps(self.to_dict(), indent=2)

class CreateTenantCommand(JsonSerializableMixin):
    def __init__(self, tenant_id: str, company_name: str, admin_email: str, plan_tier: str):
        self.tenant_id: str = tenant_id
        self.company_name: str = company_name
        self.admin_email: str = admin_email
        self.plan_tier: str = plan_tier

cmd = CreateTenantCommand("T-900", "HealthCorp Global", "admin@healthcorp.com", "Enterprise")
print(cmd.to_json())
```

#### Example 6.5: `TenantScopedMixin` and `EventPublisherMixin` for CQRS Handlers
Combining tenant boundary enforcement with MediatR-style domain event dispatch:

```python
class TenantScopedMixin:
    def __init__(self, *args, tenant_id: str, **kwargs):
        super().__init__(*args, **kwargs)
        self.tenant_id: str = tenant_id

    def validate_tenant_access(self, target_tenant_id: str) -> None:
        if self.tenant_id != target_tenant_id:
            raise PermissionError("BOLA Violation: Cross-tenant data access blocked.")

class EventPublisherMixin:
    def publish_domain_event(self, event_name: str, payload: dict) -> None:
        print(f"[DomainEvent] Dispatched '{event_name}' with payload keys: {list(payload.keys())}")

class IngestDocumentCommandHandler(TenantScopedMixin, EventPublisherMixin):
    def __init__(self, tenant_id: str):
        super().__init__(tenant_id=tenant_id)

    def handle(self, target_tenant_id: str, doc_name: str) -> None:
        self.validate_tenant_access(target_tenant_id)
        print(f"Ingesting '{doc_name}' into tenant {self.tenant_id}...")
        self.publish_domain_event("DocumentIngestedEvent", {"doc": doc_name, "tenant": self.tenant_id})

handler = IngestDocumentCommandHandler(tenant_id="T-101")
handler.handle("T-101", "Q3_Financials.pdf")
```

---

## 7. Abstract Base Classes (`abc.ABC` & `@abstractmethod`)

In C#, you define strict architectural contracts using `interface` or `abstract class`:
```csharp
// C# Interface
public interface IEmbeddingEngine {
    Task<float[]> GenerateEmbeddingAsync(string text);
}
```

In standard Python, duck typing is the default ("if it walks like a duck and quacks like a duck, it's a duck"). However, for enterprise systems like Normora, **runtime contract enforcement** is essential. You must guarantee that any developer writing an embedding engine or vector database adapter implements all required methods before their class can be instantiated.

### The `abc` Module
Python provides the `abc` (Abstract Base Classes) module:
* Inherit from `abc.ABC`.
* Decorate contract methods with `@abstractmethod`.
* If a subclass fails to implement any `@abstractmethod`, Python raises a **`TypeError` at instantiation time**, preventing corrupted objects from running in production.

```mermaid
flowchart TD
    ABC["BaseEmbeddingEngine (ABC)<br>@abstractmethod generate_embedding()<br>@abstractmethod batch_generate()"]
    Gemini["GoogleGeminiEmbeddingEngine<br>Implements generate_embedding()<br>Implements batch_generate()"]
    OpenAI["OpenAiEmbeddingEngine<br>Implements generate_embedding()<br>Implements batch_generate()"]
    Broken["IncompleteEngine<br>Missing batch_generate()"]
    ABC --> Gemini
    ABC --> OpenAI
    ABC -.->|TypeError on instantiation!| Broken
```

---

### 5 Real-World Normora Examples

#### Example 7.1: The `BaseEmbeddingEngine` Abstract Contract
Enforces that all cloud AI embedding adapters support single and batch generation:

```python
from abc import ABC, abstractmethod

class BaseEmbeddingEngine(ABC):
    @abstractmethod
    def generate_embedding(self, text: str) -> list[float]:
        """Generate a dense vector embedding for a single text chunk."""
        pass

    @abstractmethod
    def batch_generate(self, texts: list[str]) -> list[list[float]]:
        """Batch generate embeddings for high-throughput ingestion."""
        pass

class GoogleGeminiEmbeddingEngine(BaseEmbeddingEngine):
    def __init__(self, api_key: str):
        self.api_key: str = api_key

    def generate_embedding(self, text: str) -> list[float]:
        # Google Gemini text-embedding-004 generates 768-dim vectors
        return [0.012, -0.045, 0.982] * 256  # 768 floats

    def batch_generate(self, texts: list[str]) -> list[list[float]]:
        return [self.generate_embedding(t) for t in texts]

# Valid instantiation:
gemini = GoogleGeminiEmbeddingEngine("sec_key_gemini")
print("Generated vector length:", len(gemini.generate_embedding("Normora RAG")))

# Testing invalid instantiation:
try:
    class IncompleteEngine(BaseEmbeddingEngine):
        def generate_embedding(self, text: str) -> list[float]:
            return [0.1]
        # Forgot batch_generate!

    broken = IncompleteEngine()  # TypeError!
except TypeError as ex:
    print(f"Contract Enforcement Guard: {ex}")
```

#### Example 7.2: The `BaseVectorRepository` Contract for pgvector / HNSW
Enforces standard CRUD operations for vector storage:

```python
from abc import ABC, abstractmethod

class BaseVectorRepository(ABC):
    @abstractmethod
    def upsert_chunk(self, chunk_id: str, embedding: list[float], metadata: dict) -> bool:
        pass

    @abstractmethod
    def search_similar(self, query_vector: list[float], top_k: int = 5) -> list[dict]:
        pass

class PgVectorHnswRepository(BaseVectorRepository):
    def __init__(self, connection_string: str):
        self.connection_string: str = connection_string

    def upsert_chunk(self, chunk_id: str, embedding: list[float], metadata: dict) -> bool:
        print(f"[pgvector] Stored chunk {chunk_id} with HNSW index in PostgreSQL.")
        return True

    def search_similar(self, query_vector: list[float], top_k: int = 5) -> list[dict]:
        print(f"[pgvector] Executed cosine distance search <=> returning top {top_k} nearest neighbors.")
        return [{"chunk_id": "chk-1", "score": 0.94}, {"chunk_id": "chk-2", "score": 0.88}]

repo = PgVectorHnswRepository("postgresql://normora:pass@localhost:5432/vectors")
repo.upsert_chunk("chk-99", [0.1] * 768, {"tenant_id": "T-101"})
print(repo.search_similar([0.1] * 768, top_k=2))
```

#### Example 7.3: The `BaseObjectStorage` Contract for MinIO
Ensures object storage operations can be swapped between MinIO, AWS S3, or local test mocks:

```python
from abc import ABC, abstractmethod

class BaseObjectStorage(ABC):
    @abstractmethod
    def upload_bytes(self, bucket: str, object_name: str, data: bytes) -> str:
        """Upload raw bytes and return the storage URI."""
        pass

    @abstractmethod
    def generate_presigned_download_url(self, bucket: str, object_name: str, expiry_seconds: int = 3600) -> str:
        """Generate a time-limited secure download URL."""
        pass

class MinioObjectStorage(BaseObjectStorage):
    def __init__(self, endpoint: str):
        self.endpoint: str = endpoint

    def upload_bytes(self, bucket: str, object_name: str, data: bytes) -> str:
        return f"s3://{bucket}/{object_name} ({len(data)} bytes uploaded to MinIO)"

    def generate_presigned_download_url(self, bucket: str, object_name: str, expiry_seconds: int = 3600) -> str:
        return f"https://{self.endpoint}/{bucket}/{object_name}?token=presigned_exp_{expiry_seconds}"

minio = MinioObjectStorage("minio.normora.internal:9000")
print(minio.upload_bytes("documents", "T-101/handbook.pdf", b"PDF content"))
print(minio.generate_presigned_download_url("documents", "T-101/handbook.pdf"))
```

#### Example 7.4: The `BaseDocumentChunker` Contract
Ensures chunking algorithms (Fixed Token Chunker vs. Semantic Heading Chunker) honor a uniform interface:

```python
from abc import ABC, abstractmethod

class BaseDocumentChunker(ABC):
    def __init__(self, chunk_size: int, chunk_overlap: int):
        self.chunk_size: int = chunk_size
        self.chunk_overlap: int = chunk_overlap

    @abstractmethod
    def split_text(self, document_text: str) -> list[str]:
        pass

class FixedTokenChunker(BaseDocumentChunker):
    def split_text(self, document_text: str) -> list[str]:
        words = document_text.split()
        chunks = []
        step = max(1, self.chunk_size - self.chunk_overlap)
        for i in range(0, len(words), step):
            chunk = " ".join(words[i:i + self.chunk_size])
            chunks.append(chunk)
        return chunks

chunker = FixedTokenChunker(chunk_size=5, chunk_overlap=2)
doc = "Normora uses Google Gemini Flash for blazing fast multi-tenant conversational retrieval."
print(chunker.split_text(doc))
```

#### Example 7.5: The `BaseChatStreamHandler` Contract for Real-Time SignalR Hubs
Enforces token-by-token streaming lifecycle callbacks:

```python
from abc import ABC, abstractmethod

class BaseChatStreamHandler(ABC):
    @abstractmethod
    def on_token_received(self, token: str) -> None:
        """Fires every time the LLM streams a new word or fragment."""
        pass

    @abstractmethod
    def on_citation_found(self, citation_metadata: dict) -> None:
        """Fires when grounding citations are identified."""
        pass

    @abstractmethod
    def on_stream_completed(self, total_tokens: int, duration_seconds: float) -> None:
        """Fires when the generation completes."""
        pass

class SignalRChatStreamHandler(BaseChatStreamHandler):
    def __init__(self, connection_id: str):
        self.connection_id: str = connection_id

    def on_token_received(self, token: str) -> None:
        print(f"[SignalR -> {self.connection_id}] Token: '{token}'")

    def on_citation_found(self, citation_metadata: dict) -> None:
        print(f"[SignalR -> {self.connection_id}] Citation: {citation_metadata['title']}")

    def on_stream_completed(self, total_tokens: int, duration_seconds: float) -> None:
        print(f"[SignalR -> {self.connection_id}] Stream Finished: {total_tokens} tokens in {duration_seconds:.2f}s")

handler = SignalRChatStreamHandler("conn-xyz-987")
handler.on_token_received("Normora")
handler.on_citation_found({"title": "Tenant_Policy.pdf"})
handler.on_stream_completed(184, 1.25)
```

---

## 8. The `__init_subclass__` Hook (Subclass Registration & Validation)

In traditional Python 2 and early Python 3, if you wanted a base class to automatically detect, register, or validate every class that inherited from it, you had to write a complex **Metaclass**.

Introduced in **Python 3.6**, the **`__init_subclass__`** hook solved this cleanly. When a subclass is defined, Python calls `__init_subclass__` on its parent class automatically:

```mermaid
sequenceDiagram
    autonumber
    actor Developer
    participant BaseClass as BaseNormoraTool
    participant Registry as TOOL_REGISTRY
    participant SubClass as SearchDocumentsTool

    Developer->>SubClass: Defines class SearchDocumentsTool(BaseNormoraTool)
    SubClass->>BaseClass: Triggers __init_subclass__(tool_name="search")
    BaseClass->>Registry: Registers SearchDocumentsTool into registry dict!
    Note over Registry: No reflection, no assembly scanning needed!
```

### Why .NET Developers Love `__init_subclass__`
In C# / .NET, if you want MediatR command handlers or plugins to register automatically into your Dependency Injection container, you rely on **Reflection and Assembly Scanning** at startup:
```csharp
services.AddMediatR(cfg => cfg.RegisterServicesFromAssembly(typeof(Program).Assembly));
```
In Python, `__init_subclass__` provides **zero-overhead, compile-free registration** right at class import time!

---

### 5 Real-World Normora Examples

#### Example 8.1: AI Agent Tool Auto-Registry
Automatically registers any tool created by developers into a central registry for Google Gemini Flash function calling:

```python
class BaseNormoraAiTool:
    # Centralized registry of all available AI tools:
    TOOL_REGISTRY: dict[str, type["BaseNormoraAiTool"]] = {}

    def __init_subclass__(cls, tool_identifier: str, **kwargs):
        super().__init_subclass__(**kwargs)
        # Automatically register the subclass into the registry dictionary:
        cls.TOOL_REGISTRY[tool_identifier] = cls
        cls.identifier = tool_identifier
        print(f"[ToolRegistry] Registered AI tool '{tool_identifier}' -> {cls.__name__}")

    def execute(self, **kwargs) -> dict:
        raise NotImplementedError()

# Subclasses register automatically upon definition!
class VectorSearchTool(BaseNormoraAiTool, tool_identifier="vector_search"):
    def execute(self, **kwargs) -> dict:
        return {"result": f"Vector search for {kwargs.get('query')}"}

class DocumentSummarizerTool(BaseNormoraAiTool, tool_identifier="doc_summarizer"):
    def execute(self, **kwargs) -> dict:
        return {"result": "Document summary complete."}

print("Active Tools in Registry:", list(BaseNormoraAiTool.TOOL_REGISTRY.keys()))
# Output: Active Tools in Registry: ['vector_search', 'doc_summarizer']
```

#### Example 8.2: CQRS Command Handler Auto-Registry
Maps CQRS commands to their corresponding execution handlers without reflection:

```python
class BaseCommandHandler:
    HANDLER_REGISTRY: dict[str, type["BaseCommandHandler"]] = {}

    def __init_subclass__(cls, command_type: str, **kwargs):
        super().__init_subclass__(**kwargs)
        cls.HANDLER_REGISTRY[command_type] = cls

    def handle(self, payload: dict) -> None:
        raise NotImplementedError()

class CreateTenantHandler(BaseCommandHandler, command_type="CreateTenantCommand"):
    def handle(self, payload: dict) -> None:
        print(f"Creating tenant '{payload.get('name')}' in PostgreSQL...")

class IngestDocumentHandler(BaseCommandHandler, command_type="IngestDocumentCommand"):
    def handle(self, payload: dict) -> None:
        print(f"Ingesting document '{payload.get('filename')}'...")

print("Registered Command Handlers:", BaseCommandHandler.HANDLER_REGISTRY)
```

#### Example 8.3: Document Format Parser Registry by MIME Type
Routes incoming uploaded files directly to the appropriate parser based on registered MIME types:

```python
class BaseDocumentParser:
    MIME_REGISTRY: dict[str, type["BaseDocumentParser"]] = {}

    def __init_subclass__(cls, mime_type: str, **kwargs):
        super().__init_subclass__(**kwargs)
        cls.MIME_REGISTRY[mime_type] = cls

    def parse(self, content_bytes: bytes) -> str:
        raise NotImplementedError()

class PdfMimeParser(BaseDocumentParser, mime_type="application/pdf"):
    def parse(self, content_bytes: bytes) -> str:
        return f"Parsed {len(content_bytes)} bytes as PDF document."

class MarkdownMimeParser(BaseDocumentParser, mime_type="text/markdown"):
    def parse(self, content_bytes: bytes) -> str:
        return f"Parsed {len(content_bytes)} bytes as Markdown text."

def parse_incoming_file(mime: str, data: bytes) -> str:
    parser_cls = BaseDocumentParser.MIME_REGISTRY.get(mime)
    if not parser_cls:
        raise ValueError(f"No parser registered for MIME type: {mime}")
    return parser_cls().parse(data)

print(parse_incoming_file("application/pdf", b"%PDF-1.4 sample content"))
```

#### Example 8.4: Multi-Tenant Storage Provider Registry
Allows pluggable storage engines (MinIO, Azure Blob, AWS S3) to declare their provider name:

```python
class BaseStorageProvider:
    PROVIDERS: dict[str, type["BaseStorageProvider"]] = {}

    def __init_subclass__(cls, provider_id: str, **kwargs):
        super().__init_subclass__(**kwargs)
        cls.PROVIDERS[provider_id] = cls

    def store(self, key: str, value: bytes) -> None:
        raise NotImplementedError()

class MinioProvider(BaseStorageProvider, provider_id="minio"):
    def store(self, key: str, value: bytes) -> None:
        print(f"[MinIO] Stored key '{key}'")

class AzureBlobProvider(BaseStorageProvider, provider_id="azure_blob"):
    def store(self, key: str, value: bytes) -> None:
        print(f"[Azure Blob] Stored key '{key}'")

print("Available Storage Backends:", list(BaseStorageProvider.PROVIDERS.keys()))
```

#### Example 8.5: Strict Subclass Validation Guard
Enforce that every database repository subclass defines a mandatory `TABLE_NAME` class attribute:

```python
class BaseDbRepository:
    def __init_subclass__(cls, **kwargs):
        super().__init_subclass__(**kwargs)
        if not hasattr(cls, "TABLE_NAME") or not isinstance(cls.TABLE_NAME, str):
            raise TypeError(f"Class '{cls.__name__}' must define a string class attribute 'TABLE_NAME'.")

class TenantRepository(BaseDbRepository):
    TABLE_NAME = "normora_tenants"  # Valid!

try:
    class DocumentRepository(BaseDbRepository):
        pass  # Forgot TABLE_NAME!
except TypeError as ex:
    print(f"Architecture Validation Error: {ex}")
```

---

## 9. Data Classes (`@dataclass`) — Modern Python Domain Modeling

Writing boilerplate `__init__`, `__repr__`, and `__eq__` methods for simple data-carrying models is tedious. In **Python 3.7+**, the standard library introduced the **`@dataclass`** decorator.

```python
from dataclasses import dataclass

@dataclass
class DocumentChunkDto:
    chunk_id: str
    content: str
    page_number: int
```

Python automatically generates:
* `__init__()` (initializes all declared fields with their specified types).
* `__repr__()` (clean, readable developer print string).
* `__eq__()` (structural equality: two objects with identical attributes compare equal `==`).

### Advanced `@dataclass` Capabilities
* **`frozen=True`:** Makes instances **immutable** (any attempt to assign to fields raises `FrozenInstanceError`). This is the exact Python equivalent of C# `record`.
* **`slots=True` (Python 3.10+):** Generates `__slots__`, eliminating the per-instance `__dict__`. This drastically reduces memory consumption (up to 40%) and speeds up attribute access—critical when holding 100,000 vector chunks in memory!
* **`field(default_factory=...)`:** Safely handles mutable defaults (like `list`, `dict`, `set`) without sharing state across instances.
* **`__post_init__(self)`:** Executes immediately after the generated `__init__` completes. Used for cross-field validation, calculating derived fields, or type coercion.

### C# `record` vs. Python `@dataclass`
| Feature | C# (.NET 10) `record` | Python 3.12+ `@dataclass` |
| :--- | :--- | :--- |
| **Syntax** | `public record TenantDto(string Id, string Name);` | `@dataclass class TenantDto: id: str; name: str` |
| **Immutability** | `init`-only properties by default | Enabled via `@dataclass(frozen=True)` |
| **Value Equality** | Built-in by value | Built-in by value (`__eq__`) |
| **Deconstruction** | Built-in positional deconstruction | Supported via `dataclasses.astuple()` |
| **Memory Optimization** | Value types via `record struct` | Enabled via `@dataclass(slots=True)` |

---

### 5 Real-World Normora Examples

#### Example 9.1: High-Performance `DocumentChunkDto` with `slots=True`
Memory-optimized chunk DTO suitable for storing tens of thousands of chunks in background ingestion pipelines:

```python
from dataclasses import dataclass, field

@dataclass(slots=True)
class DocumentChunkDto:
    chunk_id: str
    document_id: str
    content: str
    page_number: int
    score: float = 0.0
    embedding: list[float] = field(default_factory=list)

chunk1 = DocumentChunkDto("chk-1", "doc-99", "Encrypted data at rest.", page_number=2, score=0.92)
print(chunk1)
# Output: DocumentChunkDto(chunk_id='chk-1', document_id='doc-99', content='Encrypted data at rest.', page_number=2, score=0.92, embedding=[])
```

#### Example 9.2: Immutable `TenantContext` Security Record (`frozen=True`)
Security context passed down the call stack that must never be mutated in flight:

```python
from dataclasses import dataclass

@dataclass(frozen=True, slots=True)
class TenantContext:
    tenant_id: str
    user_id: str
    role: str
    is_authenticated: bool

ctx = TenantContext(tenant_id="T-101", user_id="usr-sanjeev", role="Admin", is_authenticated=True)
print(f"Context verified: {ctx.user_id} in {ctx.tenant_id}")

try:
    # Attempting to tamper with security context:
    ctx.role = "Owner"
except Exception as ex:
    print(f"Immutability Guard Blocked Tampering: {type(ex).__name__} -> {ex}")
```

#### Example 9.3: Conversation Citation with Validation in `__post_init__`
Validates that confidence scores are within `[0.0, 1.0]` and calculates formatted display text:

```python
from dataclasses import dataclass, field

@dataclass
class CitationDto:
    document_title: str
    page_number: int
    relevance_score: float
    formatted_reference: str = field(init=False)  # Derived in __post_init__

    def __post_init__(self):
        # Validate score bounds:
        if not (0.0 <= self.relevance_score <= 1.0):
            raise ValueError(f"Relevance score {self.relevance_score} must be between 0.0 and 1.0")
        # Compute derived formatted string:
        self.formatted_reference = f"[{self.document_title}, Page {self.page_number} (Conf: {self.relevance_score:.0%})]"

citation = CitationDto("Compliance_Handbook_2026.pdf", page_number=14, relevance_score=0.96)
print(citation.formatted_reference)
# Output: [Compliance_Handbook_2026.pdf, Page 14 (Conf: 96%)]
```

#### Example 9.4: AI Chat Request with `field(default_factory=...)`
Avoiding the dangerous mutable default argument pitfall by using `default_factory`:

```python
from dataclasses import dataclass, field

@dataclass
class GeminiChatRequest:
    prompt: str
    temperature: float = 0.2
    max_output_tokens: int = 1024
    # Safe mutable defaults:
    safety_settings: list[str] = field(default_factory=lambda: ["HARM_CATEGORY_HATE_SPEECH=BLOCK_LOW_AND_ABOVE"])
    metadata_filters: dict[str, str] = field(default_factory=dict)

req_a = GeminiChatRequest(prompt="Summarize HR policy")
req_b = GeminiChatRequest(prompt="Draft NDA agreement")

req_a.metadata_filters["department"] = "HR"

# req_b remains untouched because each object received its own dictionary!
print("Req A filters:", req_a.metadata_filters)  # {'department': 'HR'}
print("Req B filters:", req_b.metadata_filters)  # {}
```

#### Example 9.5: CQRS `CreateTenantCommand` DTO
Immutable command representation passed through MediatR-style dispatchers:

```python
from dataclasses import dataclass

@dataclass(frozen=True, slots=True)
class CreateTenantCommand:
    tenant_id: str
    company_name: str
    billing_email: str
    initial_tier: str = "Free"

    def __post_init__(self):
        if "@" not in self.billing_email:
            raise ValueError(f"Invalid email: '{self.billing_email}'")

cmd = CreateTenantCommand("T-999", "Acme Logistics", "billing@acmelogistics.com")
print(f"Command valid: Create tenant '{cmd.company_name}' ({cmd.initial_tier})")
```

---

## 10. Special "Dunder" (Magic) Methods for Rich Object Behavior

Methods surrounded by double underscores—affectionately known as **"dunder" methods**—form the backbone of Python's **Data Model**. 

By implementing dunder methods, your custom Normora classes seamlessly integrate with Python's language features: operators (`==`, `<`, `+`), built-in functions (`len()`, `str()`, `repr()`, `hash()`), iteration (`for item in obj:`), and resource management (`with` statements).

```mermaid
flowchart LR
    Dunder["Python Dunder Protocols"]
    Dunder --> StrRepr["__str__ & __repr__<br>User display & Dev logging"]
    Dunder --> EqHash["__eq__ & __hash__<br>Set deduplication & Dict keys"]
    Dunder --> Order["__lt__<br>list.sort() & min()/max()"]
    Dunder --> Container["__len__, __getitem__, __iter__<br>Indexing & for loops"]
    Dunder --> Context["__enter__ & __exit__<br>with session managers"]
```

### Key Dunder Methods Summary
| Dunder Method | Invoked By | C# (.NET Equivalent) |
| :--- | :--- | :--- |
| `__str__(self)` | `str(obj)`, `print(obj)` | `public override string ToString()` |
| `__repr__(self)` | `repr(obj)`, interactive debugger | DebuggerDisplay / developer-focused string |
| `__eq__(self, other)` | `obj1 == obj2` | `IEquatable<T>.Equals()` |
| `__hash__(self)` | `hash(obj)`, `set()`, `dict` keys | `public override int GetHashCode()` |
| `__lt__(self, other)` | `obj1 < obj2`, `list.sort()` | `IComparable<T>.CompareTo()` |
| `__len__(self)` | `len(obj)` | `ICollection.Count` |
| `__getitem__(self, key)` | `obj[key]` | Custom Indexer `public T this[int index]` |
| `__iter__(self)` | `for x in obj:` | `IEnumerable<T>.GetEnumerator()` |
| `__enter__` / `__exit__` | `with obj:` | `IDisposable` / `using (...) { }` |

---

### 5 Real-World Normora Examples

#### Example 10.1: `__str__` and `__repr__` for Clean Logging vs. Debugging
* `__str__` returns an end-user readable string.
* `__repr__` returns an unambiguous representation that ideally looks like the Python code used to construct the object:

```python
class Tenant:
    def __init__(self, tenant_id: str, name: str, plan: str):
        self.tenant_id: str = tenant_id
        self.name: str = name
        self.plan: str = plan

    def __str__(self) -> str:
        # What user or log output displays:
        return f"Tenant '{self.name}' ({self.tenant_id}) [{self.plan}]"

    def __repr__(self) -> str:
        # What developers see in tracebacks, logs, and debuggers:
        return f"Tenant(tenant_id={self.tenant_id!r}, name={self.name!r}, plan={self.plan!r})"

tenant = Tenant("T-101", "Acme Health", "Enterprise")
print(str(tenant))   # Tenant 'Acme Health' (T-101) [Enterprise]
print(repr(tenant))  # Tenant(tenant_id='T-101', name='Acme Health', plan='Enterprise')
```

#### Example 10.2: `__eq__` and `__hash__` for Deduplicating Chunks in a `set()`
In vector pipelines, duplicate chunks must be filtered out. For an object to be placed in a `set` or used as a `dict` key, it must implement both `__eq__` and `__hash__`:

```python
class DocumentChunkFingerprint:
    def __init__(self, document_id: str, content_hash: str):
        self.document_id: str = document_id
        self.content_hash: str = content_hash

    def __eq__(self, other: object) -> bool:
        if not isinstance(other, DocumentChunkFingerprint):
            return False
        return (self.document_id, self.content_hash) == (other.document_id, other.content_hash)

    def __hash__(self) -> int:
        return hash((self.document_id, self.content_hash))

chunk_1 = DocumentChunkFingerprint("doc-1", "hash_abc_123")
chunk_2 = DocumentChunkFingerprint("doc-1", "hash_abc_123")  # Identical content
chunk_3 = DocumentChunkFingerprint("doc-1", "hash_xyz_789")

chunk_set = {chunk_1, chunk_2, chunk_3}
print(f"Total unique chunks in set: {len(chunk_set)}")  # 2 (chunk_2 deduplicated!)
```

#### Example 10.3: `__lt__` for Sorting RAG Search Results
Implementing `__lt__` (less than) enables native sorting via Python's built-in `sorted()` or `.sort()`:

```python
class ScoredSearchResult:
    def __init__(self, chunk_id: str, cosine_similarity: float):
        self.chunk_id: str = chunk_id
        self.cosine_similarity: float = cosine_similarity

    def __lt__(self, other: "ScoredSearchResult") -> bool:
        # Higher similarity should rank first:
        return self.cosine_similarity < other.cosine_similarity

    def __repr__(self) -> str:
        return f"{self.chunk_id}({self.cosine_similarity:.2f})"

results = [
    ScoredSearchResult("chk-A", 0.72),
    ScoredSearchResult("chk-B", 0.95),
    ScoredSearchResult("chk-C", 0.84)
]

# Sort ascending:
sorted_asc = sorted(results)
print("Ascending:", sorted_asc)

# Sort descending (most relevant first for LLM prompt context):
sorted_desc = sorted(results, reverse=True)
print("Descending (Top Ranked):", sorted_desc)
```

#### Example 10.4: `__len__`, `__getitem__`, and `__iter__` for Custom Collections
Make your domain collection act like a native Python list:

```python
class ChunkBatchCollection:
    def __init__(self, chunks: list[str]):
        self._chunks: list[str] = chunks

    def __len__(self) -> int:
        return len(self._chunks)

    def __getitem__(self, index: int) -> str:
        return self._chunks[index]

    def __iter__(self):
        return iter(self._chunks)

batch = ChunkBatchCollection(["Chunk 1: Overview", "Chunk 2: Security", "Chunk 3: Billing"])

print(f"Batch contains {len(batch)} chunks")  # Uses __len__
print(f"First chunk: '{batch[0]}'")           # Uses __getitem__

print("Iterating through batch:")            # Uses __iter__
for chunk in batch:
    print(f"  -> {chunk}")
```

#### Example 10.5: `__enter__` and `__exit__` for Multi-Tenant DbContext Context Managers
Just like `using (var session = dbContext.BeginTenantScope())` in C#, Python's context manager protocol guarantees resource cleanup even if exceptions occur:

```python
class TenantDbContextSession:
    def __init__(self, tenant_id: str):
        self.tenant_id: str = tenant_id
        self.connection_open: bool = False

    def __enter__(self) -> "TenantDbContextSession":
        self.connection_open = True
        print(f"[PostgreSQL] Opened tenant-isolated session for '{self.tenant_id}'. RLS policy activated.")
        return self

    def __exit__(self, exc_type, exc_val, exc_tb) -> bool:
        self.connection_open = False
        if exc_type is not None:
            print(f"[PostgreSQL] Rolled back transaction due to error: {exc_val}")
            return False  # Do not suppress exception, propagate upward!
        print(f"[PostgreSQL] Committed and closed session for '{self.tenant_id}'.")
        return True

# Using with statement:
with TenantDbContextSession("T-101") as session:
    print(f"  Executing vector queries inside scope. Connection open: {session.connection_open}")
print("Session scope exited.")
```

---

## 11. The Descriptors Protocol (`__get__`, `__set__`, `__delete__`)

Have you ever wondered how `@property`, `@classmethod`, SQLAlchemy ORM columns, and Pydantic fields work under the hood? They are all powered by Python's **Descriptor Protocol**.

A **Descriptor** is an object attribute with "binding behavior"—one whose attribute access is overridden by methods in the descriptor protocol:
* `__get__(self, instance, owner)`: Called when the attribute is accessed.
* `__set__(self, instance, value)`: Called when the attribute is assigned.
* `__delete__(self, instance)`: Called when the attribute is deleted via `del`.

```mermaid
sequenceDiagram
    autonumber
    actor Client
    participant TenantEntity as tenant
    participant Descriptor as NonNegativeTokenLimit

    Client->>TenantEntity: tenant.max_tokens = 500000
    TenantEntity->>Descriptor: __set__(tenant, 500000)
    Note over Descriptor: Validates >= 0 and stores into instance dict!
    Client->>TenantEntity: tenant.max_tokens
    TenantEntity->>Descriptor: __get__(tenant, Tenant)
    Descriptor-->>Client: Returns validated integer
```

### Why Use Descriptors?
Descriptors allow you to write reusable validation or transformation logic **once** and apply it across dozens of different attributes and classes without repeating `@property` getters and setters.

---

### 5 Real-World Normora Examples

#### Example 11.1: `NonNegativeInteger` Descriptor for Token Limits and Page Counts
Enforces non-negative integer validation across any attribute on any entity:

```python
class NonNegativeInteger:
    def __set_name__(self, owner, name):
        # Automatically captures the name of the attribute on the owner class
        self.private_name = f"_{name}"

    def __get__(self, instance, owner):
        if instance is None:
            return self
        return getattr(instance, self.private_name, 0)

    def __set__(self, instance, value):
        if not isinstance(value, int):
            raise TypeError(f"{self.private_name[1:]} must be an integer.")
        if value < 0:
            raise ValueError(f"{self.private_name[1:]} cannot be negative. Received: {value}")
        setattr(instance, self.private_name, value)

class TenantSubscription:
    token_limit = NonNegativeInteger()  # Reusable descriptor!
    max_documents = NonNegativeInteger()

    def __init__(self, tenant_id: str, token_limit: int, max_documents: int):
        self.tenant_id: str = tenant_id
        self.token_limit = token_limit
        self.max_documents = max_documents

sub = TenantSubscription("T-101", token_limit=500_000, max_documents=100)
print(f"Subscription limits: {sub.token_limit:,} tokens, {sub.max_documents} docs")

try:
    sub.token_limit = -10  # Raises ValueError
except ValueError as ex:
    print(f"Descriptor Validation Blocked Assignment: {ex}")
```

#### Example 11.2: `ValidTenantId` Descriptor Enforcing Formatting Rules
Guarantees tenant keys always follow the `T-[0-9]+` pattern:

```python
import re

class ValidTenantId:
    PATTERN = re.compile(r"^T-\d{3,6}$")

    def __set_name__(self, owner, name):
        self.name = f"_{name}"

    def __get__(self, instance, owner):
        if instance is None:
            return self
        return getattr(instance, self.name, "")

    def __set__(self, instance, value: str):
        if not isinstance(value, str) or not self.PATTERN.match(value):
            raise ValueError(f"Invalid tenant key format: '{value}'. Expected format: 'T-101' to 'T-999999'.")
        setattr(instance, self.name, value)

class Workspace:
    tenant_id = ValidTenantId()

    def __init__(self, tenant_id: str):
        self.tenant_id = tenant_id

ws = Workspace("T-501")
print(f"Valid workspace registered: {ws.tenant_id}")

try:
    bad_ws = Workspace("INVALID_KEY")
except ValueError as ex:
    print(f"Tenant Key Guard: {ex}")
```

#### Example 11.3: `CachedProperty` Descriptor for Costly Embedding Calculations
Computes an expensive property once, caches the result on the instance, and never re-calculates:

```python
class CachedVectorNorm:
    def __set_name__(self, owner, name):
        self.name = name

    def __get__(self, instance, owner):
        if instance is None:
            return self
        print(f"[Calculation] Computing Euclidean vector norm for chunk {instance.chunk_id}...")
        import math
        # Calculate norm: sqrt(sum(x^2))
        norm_val = math.sqrt(sum(x * x for x in instance.embedding))
        # Store directly in instance dict to cache:
        instance.__dict__[self.name] = norm_val
        return norm_val

class VectorChunk:
    norm = CachedVectorNorm()

    def __init__(self, chunk_id: str, embedding: list[float]):
        self.chunk_id: str = chunk_id
        self.embedding: list[float] = embedding

chunk = VectorChunk("chk-1", [0.3, 0.4])
print("First access (computes):", chunk.norm)  # 0.5 (prints computation message)
print("Second access (cached):", chunk.norm)    # 0.5 (returns from __dict__ immediately!)
```

#### Example 11.4: `AuditLoggedField` Descriptor for Sensitive Attributes
Logs whenever critical business fields are inspected or updated:

```python
class AuditLoggedField:
    def __set_name__(self, owner, name):
        self.storage_name = f"_{name}"
        self.field_name = name

    def __get__(self, instance, owner):
        if instance is None:
            return self
        val = getattr(instance, self.storage_name, None)
        print(f"[AUDIT LOG] Field '{self.field_name}' read by process. Value: {val}")
        return val

    def __set__(self, instance, value):
        print(f"[AUDIT LOG] Field '{self.field_name}' updated to: {value}")
        setattr(instance, self.storage_name, value)

class TenantSecurityProfile:
    mfa_enforced = AuditLoggedField()

    def __init__(self, tenant_id: str, mfa_enforced: bool):
        self.tenant_id: str = tenant_id
        self.mfa_enforced = mfa_enforced

profile = TenantSecurityProfile("T-101", mfa_enforced=True)
profile.mfa_enforced = False
current = profile.mfa_enforced
```

#### Example 11.5: `EncryptedStringField` Descriptor for Cloud API Keys
Masks confidential keys from plain-text exposure in memory:

```python
import base64

class EncryptedStringField:
    def __set_name__(self, owner, name):
        self.internal_name = f"_{name}"

    def __get__(self, instance, owner):
        if instance is None:
            return self
        encrypted = getattr(instance, self.internal_name, None)
        if not encrypted:
            return ""
        # Decodes on the fly:
        return base64.b64decode(encrypted.encode()).decode()

    def __set__(self, instance, value: str):
        # Obfuscates on storage:
        encoded = base64.b64encode(value.encode()).decode()
        setattr(instance, self.internal_name, encoded)

class GeminiApiKeyCredential:
    api_key = EncryptedStringField()

    def __init__(self, key: str):
        self.api_key = key

cred = GeminiApiKeyCredential("AIzaSyD-CONFIDENTIAL-GEMINI-KEY")
print("Internal storage attribute:", cred._api_key)  # Obfuscated base64
print("Decrypted via descriptor:", cred.api_key)      # Clean plain key
```

---

## 12. Composition vs. Inheritance (Architecture Best Practices)

One of the most famous principles in modern software engineering is:
> **"Favor object composition over class inheritance."** (Gang of Four)

* **Inheritance (is-a):** Establishes tight coupling. Subclasses inherit all public, protected, and internal methods of the parent. Changes to the base class risk breaking subtle invariants across subclasses (the **Fragile Base Class Problem**).
* **Composition (has-a):** Assembles complex functionality by combining separate, independent components as instance attributes. Components can be swapped dynamically at runtime, tested in isolation with mocks, and have no hidden coupling.

```mermaid
flowchart TD
    subgraph Inheritance["Inheritance (Tight Coupling - Anti-Pattern here)"]
        BadPipeline["RagPipelineService<br>Inherits from EmbeddingEngine, VectorDb, and GeminiClient!"]
    end

    subgraph Composition["Composition (Flexible & Testable - Normora Standard)"]
        GoodPipeline["RagPipelineService<br>Has-a: BaseEmbeddingEngine<br>Has-a: BaseVectorRepository<br>Has-a: GoogleGeminiClient"]
        EE["BaseEmbeddingEngine"]
        VR["BaseVectorRepository"]
        GC["GoogleGeminiClient"]
        GoodPipeline --> EE
        GoodPipeline --> VR
        GoodPipeline --> GC
    end
```

### When to Use Which in Normora?
* Use **Inheritance** when: There is a genuine **is-a** relationship, polymorphic substitution is required, and the base class provides concrete behavioral invariants (e.g. `BolaException` *is a* `NormoraException`).
* Use **Composition** when: Building services, pipelines, orchestrators, and controllers that coordinate multiple distinct infrastructure concerns (e.g. `RagPipelineService` *has a* retriever and *has a* generative model).

---

### 5 Real-World Normora Examples

#### Example 12.1: The `RagPipelineService` Orchestrator (Pure Composition)
Composes vector retrieval, embedding generation, and LLM generation:

```python
class MockEmbeddingEngine:
    def embed_query(self, query: str) -> list[float]:
        return [0.05, -0.12, 0.44]

class MockVectorRepository:
    def find_nearest_chunks(self, vector: list[float], top_k: int = 2) -> list[str]:
        return ["Chunk: Keycloak handles OIDC tokens.", "Chunk: BFF proxies requests."]

class MockGeminiClient:
    def generate_grounded_answer(self, prompt: str, context: list[str]) -> str:
        return f"Based on: {context}, the answer is verified."

# Composed Service:
class RagPipelineService:
    def __init__(
        self,
        embedding_engine: MockEmbeddingEngine,
        vector_repo: MockVectorRepository,
        gemini_client: MockGeminiClient
    ):
        # Composition: Holding references to injected dependencies
        self._embedding_engine = embedding_engine
        self._vector_repo = vector_repo
        self._gemini_client = gemini_client

    def answer_user_query(self, query: str) -> str:
        vector = self._embedding_engine.embed_query(query)
        chunks = self._vector_repo.find_nearest_chunks(vector, top_k=2)
        return self._gemini_client.generate_grounded_answer(query, chunks)

rag = RagPipelineService(MockEmbeddingEngine(), MockVectorRepository(), MockGeminiClient())
print(rag.answer_user_query("How does authentication work?"))
```

#### Example 12.2: Document Ingestion Pipeline via Composition
Coordinates parsing, chunking, and storage without inheritance:

```python
class FileDownloader:
    def download(self, uri: str) -> bytes:
        return b"Dummy PDF bytes"

class TextExtractor:
    def extract(self, data: bytes) -> str:
        return "Extracted document paragraphs."

class TextSplitter:
    def split(self, text: str) -> list[str]:
        return [text[:15], text[15:]]

class DocumentIngestionPipeline:
    def __init__(self, downloader: FileDownloader, extractor: TextExtractor, splitter: TextSplitter):
        self.downloader = downloader
        self.extractor = extractor
        self.splitter = splitter

    def process(self, s3_uri: str) -> list[str]:
        raw_bytes = self.downloader.download(s3_uri)
        plain_text = self.extractor.extract(raw_bytes)
        return self.splitter.split(plain_text)

pipeline = DocumentIngestionPipeline(FileDownloader(), TextExtractor(), TextSplitter())
print("Processed chunks:", pipeline.process("s3://normora-docs/handbook.pdf"))
```

#### Example 12.3: `TenantContext` Composed of Security & Billing Policies
Assembling modular security boundaries:

```python
class SubscriptionPlan:
    def __init__(self, plan_name: str, token_cap: int):
        self.plan_name: str = plan_name
        self.token_cap: int = token_cap

class SecurityPolicy:
    def __init__(self, enforce_mfa: bool, allowed_ips: list[str]):
        self.enforce_mfa: bool = enforce_mfa
        self.allowed_ips: list[str] = allowed_ips

class TenantContext:
    def __init__(self, tenant_id: str, plan: SubscriptionPlan, security: SecurityPolicy):
        self.tenant_id: str = tenant_id
        # Composed objects:
        self.plan = plan
        self.security = security

tenant_ctx = TenantContext(
    "T-101",
    SubscriptionPlan("Enterprise", 50_000_000),
    SecurityPolicy(enforce_mfa=True, allowed_ips=["10.0.0.1/24"])
)
print(f"Tenant {tenant_ctx.tenant_id} MFA: {tenant_ctx.security.enforce_mfa} | Cap: {tenant_ctx.plan.token_cap:,}")
```

#### Example 12.4: Conversational AI Agent Composed of Tool Registry and Memory
The conversational agent coordinates memory and tools through composition:

```python
class ConversationMemory:
    def __init__(self):
        self.history: list[dict] = []

    def add_turn(self, role: str, content: str):
        self.history.append({"role": role, "content": content})

class AgentToolExecutor:
    def run_tool(self, tool_name: str, args: dict) -> str:
        return f"Executed {tool_name} successfully."

class ConversationalAgent:
    def __init__(self, memory: ConversationMemory, tool_executor: AgentToolExecutor):
        self.memory = memory
        self.tool_executor = tool_executor

    def interact(self, user_prompt: str) -> str:
        self.memory.add_turn("user", user_prompt)
        tool_output = self.tool_executor.run_tool("search_docs", {"q": user_prompt})
        response = f"Answer using {tool_output}"
        self.memory.add_turn("assistant", response)
        return response

agent = ConversationalAgent(ConversationMemory(), AgentToolExecutor())
print(agent.interact("What is the BOLA defense policy?"))
print("History turns:", len(agent.memory.history))
```

#### Example 12.5: The Decorator Pattern for Audited Repositories
Wrapping a repository via composition to add telemetry without modifying the underlying class:

```python
class PlainTenantRepository:
    def get_by_id(self, tenant_id: str) -> dict:
        return {"id": tenant_id, "name": "Acme Corp"}

class AuditedTenantRepository:
    def __init__(self, inner_repo: PlainTenantRepository):
        # Composition wrapping:
        self._inner_repo = inner_repo

    def get_by_id(self, tenant_id: str) -> dict:
        print(f"[Telemetry] Audit: Querying tenant {tenant_id}...")
        result = self._inner_repo.get_by_id(tenant_id)
        print(f"[Telemetry] Audit: Query returned: {result['name']}")
        return result

audited_repo = AuditedTenantRepository(PlainTenantRepository())
print(audited_repo.get_by_id("T-101"))
```

---

## 13. Metaclasses in Python: A Brief Introduction

In Python, **classes themselves are objects**. When Python parses a `class` statement, it executes that body and constructs a class object of type `type`.

A **Metaclass** is simply the "class of a class". Just as an object is an instance of a class, a class is an instance of a metaclass:
```text
Instance  ──(is instance of)──>  Class  ──(is instance of)──>  Metaclass (type)
```

### When Should You Use Metaclasses?
In modern Python (3.6+), **95% of tasks previously requiring metaclasses are now handled more cleanly using `@dataclass` or `__init_subclass__`**. 

As Python creator Tim Peters famously stated:
> *"Metaclasses are deeper magic than 99% of users should ever worry about. If you wonder whether you need them, you don’t."*

However, understanding metaclasses is invaluable for understanding framework internals like Pydantic, SQLAlchemy, or Django ORM.

---

### 5 Real-World Normora Examples

#### Example 13.1: Singleton Metaclass for Gemini API Client Connection Pools
Ensures only one instance of an expensive cloud AI connection pool exists in memory:

```python
class SingletonMeta(type):
    _instances: dict[type, object] = {}

    def __call__(cls, *args, **kwargs):
        # Intercepts class instantiation
        if cls not in cls._instances:
            instance = super().__call__(*args, **kwargs)
            cls._instances[cls] = instance
        return cls._instances[cls]

class GeminiClientManager(metaclass=SingletonMeta):
    def __init__(self, cluster_url: str):
        self.cluster_url: str = cluster_url

client_1 = GeminiClientManager("https://gemini-proxy.normora.internal")
client_2 = GeminiClientManager("https://different-url-ignored.internal")

print("Both variables point to the exact same memory address:", client_1 is client_2)  # True
print("Cluster URL:", client_2.cluster_url)  # Initial URL retained!
```

#### Example 13.2: Domain Event Schema Validator Metaclass
Verifies at class creation time that every domain event declares an `EVENT_NAME` and `SCHEMA_VERSION`:

```python
class DomainEventMeta(type):
    def __new__(mcs, name, bases, namespace):
        # Do not enforce rules on the base abstract class itself:
        if name != "BaseDomainEvent":
            if "EVENT_NAME" not in namespace:
                raise TypeError(f"Domain Event class '{name}' must define 'EVENT_NAME'.")
            if "SCHEMA_VERSION" not in namespace:
                raise TypeError(f"Domain Event class '{name}' must define 'SCHEMA_VERSION'.")
        return super().__new__(mcs, name, bases, namespace)

class BaseDomainEvent(metaclass=DomainEventMeta):
    pass

class TenantCreatedEvent(BaseDomainEvent):
    EVENT_NAME = "tenant.created"
    SCHEMA_VERSION = 1

try:
    class BrokenEvent(BaseDomainEvent):
        EVENT_NAME = "broken.event"
        # Missing SCHEMA_VERSION!
except TypeError as ex:
    print(f"Metaclass Schema Enforcement: {ex}")
```

#### Example 13.3: Immutable Class Body Metaclass for Security Constants
Prevents any attributes on a class from being modified after class definition:

```python
class FrozenClassMeta(type):
    def __setattr__(cls, key, value):
        raise TypeError(f"Cannot modify immutable security class '{cls.__name__}'. Attribute '{key}' is locked.")

class SecurityConstants(metaclass=FrozenClassMeta):
    TOKEN_EXPIRY_HOURS = 8
    MAX_LOGIN_ATTEMPTS = 5

try:
    SecurityConstants.MAX_LOGIN_ATTEMPTS = 999  # Attempt to weaken security
except TypeError as ex:
    print(f"FrozenClassMeta Blocked Mutation: {ex}")
```

#### Example 13.4: Auto-Tracing Metaclass Wrapping All Methods in OpenTelemetry Spans
Automatically wraps every method of a service in profiling telemetry without manual decorators:

```python
import functools

class AutoTraceMeta(type):
    def __new__(mcs, name, bases, namespace):
        for attr_name, attr_value in namespace.items():
            if callable(attr_value) and not attr_name.startswith("__"):
                # Wrap each public method with a simulated OpenTelemetry span:
                namespace[attr_name] = mcs._wrap_with_telemetry(attr_name, attr_value)
        return super().__new__(mcs, name, bases, namespace)

    @staticmethod
    def _wrap_with_telemetry(method_name, func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            print(f"[Span Start] Executing method: '{method_name}'")
            res = func(*args, **kwargs)
            print(f"[Span End] Finished method: '{method_name}'")
            return res
        return wrapper

class TenantSyncService(metaclass=AutoTraceMeta):
    def sync_keycloak_users(self, tenant_id: str):
        return f"Synchronized users for {tenant_id}"

service = TenantSyncService()
print(service.sync_keycloak_users("T-101"))
```

#### Example 13.5: Attribute Name Uppercasing Metaclass for Environment Configurations
Ensures that all configuration keys declared on a class are converted to standardized uppercase names:

```python
class StandardizedEnvConfigMeta(type):
    def __new__(mcs, name, bases, namespace):
        new_namespace = {}
        for key, val in namespace.items():
            if not key.startswith("__") and not callable(val):
                new_namespace[key.upper()] = val
            else:
                new_namespace[key] = val
        return super().__new__(mcs, name, bases, new_namespace)

class AppSettings(metaclass=StandardizedEnvConfigMeta):
    database_url = "postgresql://localhost:5432/normora"
    minio_bucket = "documents"

print("Standardized AppSettings attributes:")
print("DATABASE_URL:", getattr(AppSettings, "DATABASE_URL"))
print("MINIO_BUCKET:", getattr(AppSettings, "MINIO_BUCKET"))
```

---

## 14. Normora Python Architecture & OOP vs. C# OOP Reference

### Normora Domain Hierarchy in Python
The following Mermaid class diagram illustrates how Normora's modular monolith entities, repositories, and AI services are structured in Python:

```mermaid
classDiagram
    class BaseEntity {
        +str entity_id
        +float created_at_epoch
        +bool is_deleted
        +get_audit_summary() str
    }

    class Tenant {
        +str company_name
        +str plan
        +int monthly_token_usage
        +record_token_consumption(tokens) void
    }

    class DocumentChunk {
        +str document_id
        +str content
        +int page_number
        +list[float] embedding
    }

    class BaseEmbeddingEngine {
        <<Abstract>>
        +generate_embedding(text)* list[float]
        +batch_generate(texts)* list[list[float]]
    }

    class GoogleGeminiEmbeddingEngine {
        +str api_key
        +generate_embedding(text) list[float]
        +batch_generate(texts) list[list[float]]
    }

    class BaseVectorRepository {
        <<Abstract>>
        +upsert_chunk(chunk_id, embedding, meta)* bool
        +search_similar(query_vector, top_k)* list[dict]
    }

    class PgVectorHnswRepository {
        +str connection_string
        +upsert_chunk(chunk_id, embedding, meta) bool
        +search_similar(query_vector, top_k) list[dict]
    }

    class RagPipelineService {
        -_embedding_engine: BaseEmbeddingEngine
        -_vector_repo: BaseVectorRepository
        -_gemini_client: GeminiChatService
        +answer_user_query(query) str
    }

    BaseEntity <|-- Tenant
    BaseEntity <|-- DocumentChunk
    BaseEmbeddingEngine <|-- GoogleGeminiEmbeddingEngine
    BaseVectorRepository <|-- PgVectorHnswRepository
    RagPipelineService o-- BaseEmbeddingEngine : has-a
    RagPipelineService o-- BaseVectorRepository : has-a
```

---

### Comprehensive Comparison: Python OOP vs. C# OOP

| Feature | C# (.NET 10) | Python 3.12+ | Architectural Impact |
| :--- | :--- | :--- | :--- |
| **Class Declaration** | `public class Tenant { ... }` | `class Tenant:` | Python requires no visibility keyword on class definition |
| **Constructor** | `public Tenant(string id) { ... }` or Primary Constructor `class Tenant(string id)` | `def __init__(self, id: str):` | Explicit `self` parameter is mandatory in Python |
| **Instance Reference** | Implicit `this` | Explicit `self` | In Python, methods take `self` as their first parameter |
| **Static / Class Attributes** | `public static int Count = 0;` | `COUNT: int = 0` (defined in class body) | Python class attributes are shared across all instances |
| **Class Methods** | `public static Tenant Create() { ... }` | `@classmethod def create(cls): ...` | Python passes the class `cls` dynamically, enabling clean inheritance factories |
| **Static Utility Methods** | `public static string Clean(string s)` | `@staticmethod def clean(s: str) -> str:` | Plain functions placed inside the class namespace |
| **Encapsulation / Privacy** | `private`, `protected`, `public`, `internal` | Naming conventions (`_single`, `__mangled`) | Python uses convention and property descriptors instead of compiler access checks |
| **Properties** | `public int Limit { get; set; }` | `@property` and `@prop.setter` | Python properties look like regular attributes to the caller |
| **Interfaces / Contracts** | `interface IRetriever { ... }` | Abstract Base Classes (`abc.ABC` + `@abstractmethod`) | Enforced at runtime instantiation (`TypeError`) |
| **Immutable Records** | `public record TenantDto(string Id, string Plan);` | `@dataclass(frozen=True, slots=True)` | Generates value equality (`__eq__`) and immutability |
| **Multiple Inheritance** | Not allowed (Classes); Multiple Interfaces supported | Fully supported (Classes & Mixins) | Python resolves diamond ambiguities via C3 Linearization (`__mro__`) |
| **Operator Overloading** | Explicit operator methods (`public static bool operator ==(...)`) | Special "dunder" methods (`__eq__`, `__lt__`, `__len__`, `__iter__`) | Hooks directly into Python's core Data Model and syntax |
| **Resource Disposal** | `IDisposable` + `using (var s = ...)` | Context Managers (`__enter__` and `__exit__`) + `with` | Python `with` blocks guarantee cleanup regardless of exceptions |
| **Metaprogramming** | Roslyn Source Generators / Reflection Emit | `__init_subclass__` and Metaclasses | Dynamic registration and class transformations at import time |

---

## 15. Common Mistakes & Anti-Patterns

### Anti-Pattern 1: Mutable Default Arguments in `__init__`
> [!CAUTION]
> In Python, default arguments are evaluated **once at function definition time**, not each time the function is called! If you use a mutable object (like a `list` or `dict`) as a default argument, **every instance will share the exact same list**.
> 
> ```python
> # ❌ CRITICAL BUG: All chat messages share the exact same citations list!
> class ChatMessage:
>     def __init__(self, text: str, citations: list = []):
>         self.text = text
>         self.citations = citations
> 
> msg1 = ChatMessage("First response")
> msg1.citations.append("Doc1.pdf")
> 
> msg2 = ChatMessage("Second response")
> print(msg2.citations)  # 💥 PRINTS ['Doc1.pdf']! Cross-tenant data leak!
> 
> # ✅ CORRECT: Use None as default and initialize inside __init__:
> class ChatMessage:
>     def __init__(self, text: str, citations: list[str] | None = None):
>         self.text = text
>         self.citations = citations if citations is not None else []
> ```

---

### Anti-Pattern 2: Mutating Class Attributes via an Instance
> [!CAUTION]
> If you assign to a class attribute through an instance (`self.MAX_SIZE = 100`), Python does **not** update the shared class variable. Instead, it creates a new instance variable on `self`, leaving the class attribute unchanged and creating subtle state inconsistencies.
> 
> ```python
> class DocumentUploader:
>     MAX_SIZE_MB = 25  # Class attribute
> 
> uploader = DocumentUploader()
> # ❌ INCORRECT: Shadows class attribute on this instance only!
> uploader.MAX_SIZE_MB = 100
> print(DocumentUploader.MAX_SIZE_MB)  # Still 25!
> 
> # ✅ CORRECT: Update on the class explicitly if intended:
> DocumentUploader.MAX_SIZE_MB = 100
> ```

---

### Anti-Pattern 3: Forgetting `super().__init__()` in Subclasses
> [!CAUTION]
> In C#, the parameterless base constructor is invoked automatically. In Python, **if you define `__init__` in a child class, the parent's `__init__` is NOT called unless you call `super().__init__()` explicitly**. Forgetting this leaves inherited attributes uninitialized and causes runtime `AttributeError` crashes.
> 
> ```python
> class BaseEntity:
>     def __init__(self, entity_id: str):
>         self.entity_id = entity_id
>         self.created_at = 1700000000
> 
> # ❌ INCORRECT:
> class Tenant(BaseEntity):
>     def __init__(self, tenant_id: str, company: str):
>         self.company = company
>         # Forgot super().__init__(tenant_id)!
> 
> t = Tenant("T-1", "Acme")
> print(t.entity_id)  # 💥 AttributeError: 'Tenant' object has no attribute 'entity_id'
> 
> # ✅ CORRECT:
> class Tenant(BaseEntity):
>     def __init__(self, tenant_id: str, company: str):
>         super().__init__(entity_id=tenant_id)
>         self.company = company
> ```

---

### Anti-Pattern 4: Deep Inheritance Hierarchies in AI Services
> [!CAUTION]
> Creating deep inheritance trees like `BaseService -> TenantService -> DocumentService -> RagPipelineService -> GeminiRagService` leads to the **Fragile Base Class problem**. Modifying a base service method can unexpectedly break down-stream AI pipelines. Use **Composition** instead: assemble specialized components (`retriever`, `chunker`, `llm_client`) as injected dependencies.

---

### Anti-Pattern 5: Implementing `__eq__` Without `__hash__`
> [!CAUTION]
> When you implement a custom `__eq__` method on a class, Python automatically sets `__hash__ = None`. This makes instances **unhashable**, meaning they will crash with `TypeError: unhashable type` if added to a `set` or used as keys in a `dict`. If your class represents an entity with a unique identifier, always implement both `__eq__` and `__hash__`:
> 
> ```python
> class DocumentChunk:
>     def __init__(self, chunk_id: str):
>         self.chunk_id = chunk_id
> 
>     def __eq__(self, other):
>         return isinstance(other, DocumentChunk) and self.chunk_id == other.chunk_id
> 
>     def __hash__(self):
>         return hash(self.chunk_id)  # ✅ Required for set() and dict keys!
> ```

---

### Anti-Pattern 6: Overusing Metaclasses When Simpler Hooks Suffice
> [!CAUTION]
> Writing complex metaclasses when a simple `@dataclass` or `__init_subclass__` hook accomplishes the same task creates unnecessary cognitive overhead. Metaclasses interfere with multiple inheritance and reduce IDE autocompletion support. Reserve metaclasses exclusively for low-level architecture where class creation itself must be intercepted.

---

## 16. Interview Questions & Model Answers

### Question 1: What is the practical difference between `__str__` and `__repr__`, and why does it matter in enterprise logging?
**Answer:**
`__str__` is intended to return a friendly, human-readable string representation of the object suitable for end-user display and UI presentation. `__repr__` is intended to provide an unambiguous, developer-focused string that ideally mirrors the exact Python expression needed to recreate the object (or clearly identifies its state and memory type). 

In enterprise logging, if an unhandled exception occurs or structured JSON logs are emitted, `repr()` is invoked by formatters and debuggers. If `__repr__` is omitted, Python falls back to `<Tenant object at 0x7fa2...>`, obscuring critical diagnostic identifiers (like `tenant_id` or `user_id`) during incident triage.

---

### Question 2: How does Python's Method Resolution Order (MRO) work, and how does C3 Linearization solve the Diamond Problem?
**Answer:**
Python uses the C3 Linearization algorithm to construct a deterministic, monotonic ordering of classes for attribute and method lookup in multiple inheritance hierarchies. In the Diamond Problem (where class `D` inherits from `B` and `C`, and both inherit from `A`), C3 guarantees three invariants:
1. Subclasses appear before their parent classes (`D` before `B` and `C`).
2. Parent class order declared in the class definition is preserved (`B` before `C`).
3. Monotonicity: if class `X` precedes class `Y` in the MRO of any parent, `X` will precede `Y` in the MRO of all subclasses.

Developers can inspect this resolution path at any time via `ClassName.__mro__`. When `super()` is invoked, it delegates to the *next class in the MRO sequence*, avoiding duplicate or out-of-order execution.

---

### Question 3: What are the key differences between `@dataclass(frozen=True)` and a C# `record`, and how does `slots=True` optimize memory?
**Answer:**
Both `@dataclass(frozen=True)` and C# `record` provide immutable domain models with value-based structural equality (`__eq__` compares attributes, not memory references) and readable print strings. 

However, by default, Python instances store their attributes in a dynamic dictionary (`__dict__`), which incurs substantial memory overhead. Adding `slots=True` (in Python 3.10+) instructs Python to allocate a fixed-size array for attributes instead of a dictionary. This reduces memory footprint by 30–40% per instance and accelerates attribute access times—vital when holding hundreds of thousands of vector chunks in memory during high-throughput RAG ingestion.

---

### Question 4: Why should modern Python architectures prefer `__init_subclass__` over custom Metaclasses?
**Answer:**
Prior to Python 3.6, automatically registering subclasses or validating child class attributes required authoring a custom metaclass (inheriting from `type`). Metaclasses are notoriously difficult to maintain, conflict when classes inherit from multiple parents with different metaclasses, and add cognitive friction. 

`__init_subclass__` is a standard hook called automatically on the base class whenever a subclass is imported. It runs without custom metaclasses, accepts arbitrary keyword arguments directly in the class definition (e.g. `class Tool(Base, name="search")`), and allows clean plugin/handler registries without assembly scanning.

---

### Question 5: How does Python's Descriptor Protocol work, and how does it relate to the `@property` decorator?
**Answer:**
The Descriptor Protocol allows an object to customize what happens when it is accessed as an attribute on another class. A descriptor implements one or more of `__get__`, `__set__`, and `__delete__`. 

The built-in `@property` decorator is simply Python's standard library implementation of a data descriptor: when you write `@property def full_name(self): ...`, Python binds an instance of the `property` descriptor type to the class namespace. When someone evaluates `user.full_name`, the descriptor's `__get__` method is triggered, invoking your underlying getter function. Descriptors also power ORMs like SQLAlchemy and validation libraries like Pydantic.

---

### Question 6: What happens under the hood when a mutable default argument is passed into `__init__`, and how does `@dataclass` protect against it?
**Answer:**
In Python, default parameter expressions in function signatures are evaluated **once when the function definition is executed at module import**, not per function call. If a parameter defaults to a mutable structure (e.g. `def __init__(self, tags=[])`), that single list object is shared across every instance created without that parameter. Modifying `instance1.tags` corrupts `instance2.tags`. 

Python's `@dataclass` actively protects developers from this mistake: declaring `tags: list = []` in a dataclass raises a compile-time `ValueError: mutable default <class 'list'> is not allowed`. Instead, dataclasses force you to write `field(default_factory=list)`, guaranteeing a fresh list instance for every object.

---

### Question 7: When should an enterprise AI architecture choose Composition over Inheritance?
**Answer:**
Inheritance should be reserved for strict **is-a** relationships where polymorphic substitution is required (e.g., `GeminiEmbeddingEngine` is a `BaseEmbeddingEngine`). 

For services, pipelines, and controllers (such as Normora's `RagPipelineService`), **Composition (has-a)** is strictly superior. A RAG pipeline *has an* embedding model, *has a* vector store, and *has an* LLM generator. Using composition prevents the Fragile Base Class problem, eliminates multiple inheritance conflicts, allows components to be mocked independently in unit tests, and permits swapping out infrastructure adapters (e.g., switching from pgvector to Pinecone) at runtime via dependency injection without refactoring class hierarchies.

---

## Summary & Next Step

In this module, you mastered:
1. **Classes and Instances:** Blueprint mechanics, dynamic initialization with `__init__`, and why `self` is explicitly passed.
2. **Attribute Scoping:** Differentiating per-instance attributes from shared class-level constants and counters.
3. **Methods:** Instance methods for state changes, `@classmethod` factories for polymorphic instantiation from JSON/dictionaries, and `@staticmethod` utilities.
4. **Encapsulation:** Protecting attributes with conventions (`_`) and building validated properties with `@property`.
5. **Inheritance & Polymorphism:** Establishing base contracts with `super()` and overriding methods cleanly.
6. **Multiple Inheritance & MRO:** Managing mixins and resolving diamond graphs deterministically via C3 Linearization.
7. **Abstract Base Classes:** Enforcing strict architecture contracts at runtime using `abc.ABC` and `@abstractmethod`.
8. **Subclass Hooks:** Zero-overhead plugin and tool registration using `__init_subclass__`.
9. **Data Classes:** Modeling high-performance, immutable domain DTOs with `@dataclass(slots=True, frozen=True)`.
10. **The Dunder Protocol:** Customizing Python core behavior with `__str__`, `__repr__`, `__eq__`, `__hash__`, `__lt__`, `__len__`, and `__enter__`/`__exit__`.
11. **Descriptors:** Authoring reusable attribute validation guards with `__get__` and `__set__`.
12. **Composition over Inheritance:** Designing modular, testable enterprise AI pipelines.
13. **Metaclasses:** Understanding class creation mechanics and singletons.
14. **Defensive Programming:** Avoiding mutable defaults, shadowing bugs, and unhashable objects.

In **Module 7: Error Handling, Exception Hierarchies & File I/O**, you will learn how to design resilient multi-tenant exception middleware, stream large files safely from MinIO S3, and protect conversational AI endpoints against unexpected crashes!
