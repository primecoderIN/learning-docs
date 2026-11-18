# Module 2: Conditionals, Decision Logic & Structural Pattern Matching

Welcome to **Module 2** of the **Normora Python, FastAPI & AI Systems Architecture** course.

In modern cloud platforms, decision logic is the nervous system of your software. In **Normora**—our enterprise multi-tenant knowledge management and conversational AI platform—decision logic governs everything from multi-tenant data isolation (BOLA defense) and Keycloak role-based authorization (BFLA defense) to document ingestion workflows, pgvector similarity gates, context-window pruning, and Google Gemini Flash tool orchestration.

If you are coming from **C# / .NET 10**, you already understand boolean logic and control flow. However, Python handles decisions with unique idioms: **structural truthiness**, **short-circuit evaluation that returns values rather than booleans**, **chained comparisons**, the **walrus operator (`:=`)**, and Python 3.10+ **structural pattern matching (`match`/`case`)** which far exceeds the capabilities of traditional switch statements.

This module will teach you Python decision logic from the ground up, assuming zero prior Python knowledge, while mapping concepts directly to your C# background and grounding every single lesson in production-grade Normora domain architecture.

---

## Table of Contents
1. [Architectural Orientation: Decision Logic in Normora](#1-architectural-orientation-decision-logic-in-normora)
2. [Boolean Expressions & Comparison Operators](#2-boolean-expressions--comparison-operators)
3. [Equality (`==`) vs. Identity (`is`): The CPython Memory Model](#3-equality--vs-identity-is-the-cpython-memory-model)
4. [Conditional Branching: `if`, `elif`, and `else`](#4-conditional-branching-if-elif-and-else)
5. [Nested Conditionals, Guard Clauses & Early Return](#5-nested-conditionals-guard-clauses--early-return)
6. [Logical Operators & Short-Circuit Evaluation (`and`, `or`, `not`)](#6-logical-operators--short-circuit-evaluation-and-or-not)
7. [Truthiness: What Python Evaluates as `True` and `False`](#7-truthiness-what-python-evaluates-as-true-and-false)
8. [One-Line Conditionals: The Ternary Operator](#8-one-line-conditionals-the-ternary-operator)
9. [The Walrus Operator (`:=` Assignment Expressions)](#9-the-walrus-operator--assignment-expressions)
10. [Modern Structural Pattern Matching (`match` / `case`)](#10-modern-structural-pattern-matching-match--case)
11. [Common Mistakes & Anti-Patterns](#11-common-mistakes--anti-patterns)
12. [C#/.NET vs. Python Decision Logic: Reference Matrix](#12-cnet-vs-python-decision-logic-reference-matrix)
13. [End-to-End Normora Project: The RAG Citation & Security Evaluation Engine](#13-end-to-end-normora-project-the-rag-citation--security-evaluation-engine)
14. [Technical Interview Questions & Deep Dives](#14-technical-interview-questions--deep-dives)
15. [Module Summary & Next Steps](#15-module-summary--next-steps)

---

## 1. Architectural Orientation: Decision Logic in Normora

Before diving into syntax, let's understand how decision logic operates inside the **Normora** modular monolith.

```mermaid
flowchart TD
    Req[Incoming HTTP Request / SignalR Message] --> AuthCheck{Valid Keycloak OIDC Token?}
    AuthCheck -- No --> Ret401[Return 401 Unauthorized]
    AuthCheck -- Yes --> BolaCheck{Tenant ID Matches Resource?}
    BolaCheck -- No --> Ret404["BOLA Defense: Return 404 Not Found (Prevent Enumeration)"]
    BolaCheck -- Yes --> BflaCheck{User Has Required Role?}
    BflaCheck -- No --> Ret403["BFLA Defense: Return 403 Forbidden"]
    BflaCheck -- Yes --> ActionRouter{Requested Operation}
    ActionRouter -- Ingest Document --> IngestFlow[Background Extraction & Embedding Pipeline]
    ActionRouter -- RAG Query --> RagFlow[Vector Similarity Gate & Google Gemini Flash Streaming]
    ActionRouter -- Tool Call --> MatchTool[match/case Tool Execution Router]
```

### The C# Developer Mindset Shift

In C#, you write decision logic inside curly braces `{}` and parentheses `()`. In Python:
1. **Significant Indentation**: Code blocks are delimited by **4 spaces**, not curly braces `{}`. Missing or inconsistent indentation raises an `IndentationError`.
2. **No Parentheses Required**: Conditions do not require surrounding parentheses (`if is_active:` rather than `if (isActive)`).
3. **Trailing Colons**: Every compound statement header (`if`, `elif`, `else`, `match`, `case`) ends with a colon `:`.
4. **Keywords & Casing**: Python keywords are strictly lowercase (`if`, `elif`, `else`, `and`, `or`, `not`, `is`), while boolean literals are TitleCase (`True`, `False`, `None`).

| Concept | C# (.NET 10) | Python (3.10+) |
| :--- | :--- | :--- |
| **Block delimiter** | Curly braces `{ ... }` | Indentation (standard 4 spaces) |
| **Conditional clause** | `if (condition) { }` | `if condition:` |
| **Else if clause** | `else if (condition) { }` | `elif condition:` |
| **Boolean literals** | `true`, `false` | `True`, `False` |
| **Null literal** | `null` | `None` |
| **Logical operators** | `&&`, `\|\|`, `!` | `and`, `or`, `not` |
| **Pattern matching** | `switch (x) { case ... => }` | `match x: case ...:` |

---

## 2. Boolean Expressions & Comparison Operators

A **Boolean expression** evaluates to either `True` or `False`. In Python, the `bool` type is actually a subclass of `int` (`True == 1` and `False == 0`).

Python provides six standard comparison operators:
* `==` Equal to (compares values)
* `!=` Not equal to (compares values)
* `>` Greater than
* `<` Less than
* `>=` Greater than or equal to
* `<=` Less than or equal to

### Chained Comparisons

One of Python's most elegant features is **chained comparisons**. In C#, checking if a value lies between two boundaries requires logical operators:
```csharp
// C# requires explicit logical combination:
if (similarity >= 0.75 && similarity <= 0.95) { ... }
```

In Python, you can chain comparisons naturally, exactly like mathematical notation:
```python
# Python chained comparison:
if 0.75 <= similarity <= 0.95:
    print("Qualified citation window.")
```
Python evaluates the middle term only once, making chained comparisons both faster and more readable.

### 5 Real-World Normora Examples

#### Example 2.1: Monthly Token Quota Enforcement
Check if a tenant has exceeded their monthly allocated quota of 1,000,000 tokens for Google Gemini Flash:
```python
tokens_used = 1_045_000
monthly_quota = 1_000_000

has_exceeded_quota = tokens_used >= monthly_quota
print(f"Has exceeded quota: {has_exceeded_quota}")  # True

# Calculate remaining tokens safely
remaining_tokens = monthly_quota - tokens_used
is_quota_depleted = remaining_tokens <= 0
print(f"Quota depleted: {is_quota_depleted}")  # True
```

#### Example 2.2: Minimum Vector Similarity Gate for RAG Citations
Normora only cites document chunks retrieved from PostgreSQL `pgvector` HNSW index if their cosine similarity meets or exceeds 0.78:
```python
chunk_similarity = 0.845
MINIMUM_SIMILARITY_THRESHOLD = 0.78

is_qualified_citation = chunk_similarity >= MINIMUM_SIMILARITY_THRESHOLD
print(f"Qualified for citation: {is_qualified_citation}")  # True
```

#### Example 2.3: Document Upload Size Validation
Validate that an uploaded company handbook does not exceed MinIO storage limits (25 MB):
```python
uploaded_file_size_bytes = 28_400_000  # 28.4 MB
MAX_ALLOWED_SIZE_BYTES = 25_000_000    # 25 MB

is_valid_size = uploaded_file_size_bytes <= MAX_ALLOWED_SIZE_BYTES
print(f"File size valid: {is_valid_size}")  # False
```

#### Example 2.4: Departmental BFLA (Function-Level Access) Check
Verify whether an incoming Keycloak JWT user claim belongs to the required "Legal" department:
```python
user_department = "Human Resources"
required_department = "Legal"

is_authorized_department = (user_department == required_department)
print(f"User is authorized for Legal actions: {is_authorized_department}")  # False
```

#### Example 2.5: Chained Comparison: Document Chunk Length Validation
Ensure a text chunk destined for Google Gemini embedding generation falls strictly within the optimal token boundary (between 128 and 512 tokens):
```python
chunk_token_count = 384
MIN_CHUNK_TOKENS = 128
MAX_CHUNK_TOKENS = 512

# Pythonic chained comparison (evaluates chunk_token_count once)
is_optimal_chunk_size = MIN_CHUNK_TOKENS <= chunk_token_count <= MAX_CHUNK_TOKENS
print(f"Chunk size is within optimal RAG bounds: {is_optimal_chunk_size}")  # True
```

---

## 3. Equality (`==`) vs. Identity (`is`): The CPython Memory Model

For developers coming from C#, understanding the difference between `==` and `is` is critical to avoiding subtle bugs.

* **`==`** checks **value equality** (calls the object's `__eq__()` dunder method). It asks: *Do these two objects represent the same data?*
* **`is`** checks **object identity** (compares memory addresses via CPython's `id()` function). It asks: *Do these two variables point to the exact same memory address on the heap?*

```mermaid
flowchart LR
    subgraph Heap Memory
        ObjA["['policy', 'hr'] (Address: 0x10A)"]
        ObjB["['policy', 'hr'] (Address: 0x20B)"]
        SingletonNone["None (Address: 0x7FFF - Singleton)"]
    end

    JobAlpha["job_alpha_tags"] --> ObjA
    JobBeta["job_beta_tags"] --> ObjB
    CustomLogo["custom_logo_url"] --> SingletonNone

    classDef obj fill:#1e293b,stroke:#3b82f6,stroke-width:2px,color:#f8fafc;
    class ObjA,ObjB,SingletonNone obj;
```

> [!IMPORTANT]
> In Python, `None`, `True`, and `False` are **guaranteed singletons**. You must **always** check for `None` using `is None` or `is not None`. Never use `== None`. Using `== None` triggers `__eq__()`, which can be overridden by custom classes to return unexpected results and is slower.

### Comparison Reference

| Operation | Python Syntax | C# Equivalent |
| :--- | :--- | :--- |
| **Value Equality** | `a == b` | `a.Equals(b)` or overloaded `a == b` |
| **Value Inequality** | `a != b` | `!a.Equals(b)` or overloaded `a != b` |
| **Reference Identity** | `a is b` | `object.ReferenceEquals(a, b)` |
| **Reference Non-Identity** | `a is not b` | `!object.ReferenceEquals(a, b)` |
| **Null Check** | `a is None` | `a is null` |
| **Not Null Check** | `a is not None` | `a is not null` |

### 5 Real-World Normora Examples

#### Example 3.1: Checking for Unset Custom Tenant Branding Logo
When a tenant has not configured custom white-label branding, `custom_logo_url` is `None`:
```python
tenant_custom_logo_url = None

# ✅ Pythonic: Check identity against singleton None
if tenant_custom_logo_url is None:
    active_logo = "https://cdn.normora.ai/assets/default-logo.svg"
    print(f"Using fallback logo: {active_logo}")
```

#### Example 3.2: Distinguishing Two Independent Document Extraction Tasks
Two independent background tasks extracting documents from MinIO may hold identical metadata, but they reside at distinct memory addresses:
```python
job_alpha_tags = ["policy", "hr"]
job_beta_tags = ["policy", "hr"]

# Value equality: Their contents match
print(job_alpha_tags == job_beta_tags)  # True

# Object identity: They are distinct heap allocations
print(job_alpha_tags is job_beta_tags)  # False
print(f"Address A: {id(job_alpha_tags)} vs Address B: {id(job_beta_tags)}")
```

#### Example 3.3: Root Department Detection in Hierarchical RBAC
In Normora's Tenants module, root organizational units have no parent department (`parent_department_id = None`):
```python
parent_department_id = None

is_root_department = (parent_department_id is None)
print(f"Is top-level root department: {is_root_department}")  # True
```

#### Example 3.4: Validating an Expired Invitation Token
When a tenant workspace invitation is created, its `accepted_at` timestamp is initialized to `None`:
```python
invitation_accepted_at = None

if invitation_accepted_at is not None:
    print("Invitation has already been claimed.")
else:
    print("Invitation is pending. Verification code can be processed.")
```

#### Example 3.5: Identity Check on Shared Database Connection Pool
Verifying that the Tenants module and the Documents module share the exact same database connection pool instance across MediatR boundaries:
```python
class DatabaseConnectionPool:
    def __init__(self, connection_string: str):
        self.connection_string = connection_string

# Singleton instance initialized in Normora IoC container
shared_pool = DatabaseConnectionPool("postgresql://app:secret@db.normora.internal/normora_core")

tenants_module_pool = shared_pool
documents_module_pool = shared_pool

# Verify both module references point to the exact same singleton instance
is_shared_singleton = (tenants_module_pool is documents_module_pool)
print(f"Modules share identical connection pool: {is_shared_singleton}")  # True
```

---

## 4. Conditional Branching: `if`, `elif`, and `else`

Conditional branching directs code execution through alternate paths. Python uses `if`, `elif` (short for "else if"), and `else`.

### Syntax Rules
1. The condition is evaluated.
2. If `True`, the indented block underneath runs, and the rest of the chain is skipped.
3. If `False`, Python evaluates each `elif` sequentially.
4. If all preceding conditions evaluate to `False`, the `else` block runs.

```mermaid
flowchart TD
    StartState([Document Uploaded]) --> CheckPending{State == 'PENDING'?}
    CheckPending -- Yes --> ActPending[Queue in MinIO storage outbox]
    CheckPending -- No --> CheckExtract{State == 'EXTRACTING_TEXT'?}
    CheckExtract -- Yes --> ActExtract[Parse PDF text & tables via PyPDF]
    CheckExtract -- No --> CheckEmbed{State == 'GENERATING_EMBEDDINGS'?}
    CheckEmbed -- Yes --> ActEmbed[Call Google Gemini Flash Embeddings API]
    CheckEmbed -- No --> CheckIndex{State == 'INDEXED'?}
    CheckIndex -- Yes --> ActIndex[Vector stored in pgvector HNSW index]
    CheckIndex -- No --> ActError[Raise Unknown Ingestion State Error]
```

### 5 Real-World Normora Examples

#### Example 4.1: Document Ingestion Pipeline Router
Route background processing jobs to the appropriate micro-operation:
```python
ingestion_state = "EXTRACTING_TEXT"

if ingestion_state == "PENDING":
    print("Step 1: Queued in MinIO storage outbox.")
elif ingestion_state == "EXTRACTING_TEXT":
    print("Step 2: Parsing PDF text and extracting tabular sections.")
elif ingestion_state == "GENERATING_EMBEDDINGS":
    print("Step 3: Calling Google Gemini Flash embedding endpoint (text-embedding-004).")
elif ingestion_state == "INDEXED":
    print("Step 4: Vector stored in pgvector HNSW index. Ready for RAG search!")
else:
    print(f"Error: Unknown ingestion state '{ingestion_state}' encountered.")
```

#### Example 4.2: Subscription Tier AI Rate Limiting
Enforce requests-per-minute (RPM) limits based on the tenant's commercial tier:
```python
subscription_tier = "Pro"

if subscription_tier == "Enterprise":
    requests_per_minute_limit = 1200
elif subscription_tier == "Pro":
    requests_per_minute_limit = 300
elif subscription_tier == "Starter":
    requests_per_minute_limit = 60
elif subscription_tier == "Free":
    requests_per_minute_limit = 10
else:
    requests_per_minute_limit = 0
    print("Warning: Unrecognized tier. Blocking incoming API requests.")

print(f"Tenant rate limit configured: {requests_per_minute_limit} RPM")
```

#### Example 4.3: Departmental Access Authorization & Role Hierarchy
Evaluate access rights when a member attempts to inspect confidential departmental records:
```python
requested_document_department = "Finance"
user_department = "Engineering"
user_role = "Member"

if user_role == "Owner":
    print("Access granted: Workspace Owner has global override privileges.")
elif user_role == "Admin" and user_department == requested_document_department:
    print(f"Access granted: Departmental Admin for {requested_document_department}.")
elif user_department == requested_document_department:
    print(f"Access granted: User is a member of {requested_document_department}.")
else:
    print(f"Access denied (BFLA): Document restricted to {requested_document_department}.")
```

#### Example 4.4: RAG Chunk Re-Ranking Thresholds
Classify retrieved document chunks after cross-encoder re-ranking:
```python
re_ranker_score = 0.88

if re_ranker_score >= 0.90:
    citation_tier = "Primary Source (High Confidence)"
elif re_ranker_score >= 0.75:
    citation_tier = "Secondary Supporting Context"
elif re_ranker_score >= 0.50:
    citation_tier = "Marginal Relevance (Supplementary)"
else:
    citation_tier = "Excluded (Irrelevant)"

print(f"Citation Classification: {citation_tier}")  # Secondary Supporting Context
```

#### Example 4.5: Conversation History Context Window Manager
Manage token limits for ongoing conversation threads before forwarding prompts to Google Gemini Flash:
```python
total_history_turns = 28

if total_history_turns > 30:
    print("Session exceeds 30 turns. Triggering background summarization worker.")
elif total_history_turns > 20:
    print("Approaching context limit. Pruning oldest conversational turns.")
else:
    print("Session length within healthy limits. Appending turn directly.")
```

---

## 5. Nested Conditionals, Guard Clauses & Early Return

When multiple levels of validation are required, developers often write **nested conditionals** (the "Arrow Anti-Pattern" or "Pyramid of Doom"):

```python
# ❌ ANTI-PATTERN: The Arrow Anti-Pattern (hard to read, test, and maintain)
def verify_document_access(user, doc):
    if user is not None:
        if user.is_authenticated:
            if user.tenant_id == doc.tenant_id:
                if not user.is_suspended:
                    if doc.status == "INDEXED":
                        return "Access Granted"
                    else:
                        return "Document Not Ready"
                else:
                    return "User Suspended"
            else:
                return "Not Found"  # BOLA
        else:
            return "Unauthenticated"
    return "Invalid User"
```

### The Guard Clause Pattern (Early Exit)

In high-reliability enterprise platforms like Normora, security checks (BOLA and BFLA) must be enforced using **guard clauses**. A guard clause tests for an invalid condition upfront and exits immediately, keeping the "happy path" flat and un-nested.

```mermaid
flowchart TD
    Start([Request Evaluation]) --> G1{User Authenticated?}
    G1 -- No --> R401[Return 401 Unauthorized]
    G1 -- Yes --> G2{Tenant Match? BOLA}
    G2 -- No --> R404[Return 404 Not Found]
    G2 -- Yes --> G3{User Suspended?}
    G3 -- Yes --> R403[Return 403 Forbidden]
    G3 -- No --> HappyPath[Proceed to RAG Execution]

    classDef guard fill:#ef4444,stroke:#991b1b,stroke-width:2px,color:#fff;
    classDef success fill:#10b981,stroke:#047857,stroke-width:2px,color:#fff;
    class G1,G2,G3 guard;
    class HappyPath success;
```

### 5 Real-World Normora Examples

#### Example 5.1: Tenant Document Decryption Key Access (Guard Pattern)
Enforce BOLA protection with flat, secure guard clauses:
```python
def get_document_decryption_key(user_tenant_id: str, doc_tenant_id: str, is_active_member: bool, doc_status: str):
    # Guard 1: BOLA check - prevent tenant enumeration by returning 404
    if user_tenant_id != doc_tenant_id:
        return {"status": 404, "message": "Document not found"}

    # Guard 2: Member status check
    if not is_active_member:
        return {"status": 403, "message": "Membership inactive or suspended"}

    # Guard 3: Document lifecycle check
    if doc_status != "INDEXED":
        return {"status": 409, "message": "Document is currently undergoing indexing"}

    # Happy path: Clean, flat execution
    return {"status": 200, "key": "sec_key_994827104_aes256"}

result = get_document_decryption_key("TENANT-A", "TENANT-A", True, "INDEXED")
print(f"Result: {result}")
```

#### Example 5.2: RAG Query Processing Pipeline
Validate incoming user prompts before calling Google Gemini Flash:
```python
def process_rag_query(query_text: str, tenant_tokens_remaining: int, is_maintenance_mode: bool):
    # Guard 1: System health
    if is_maintenance_mode:
        return "Service unavailable: Platform undergoing maintenance."

    # Guard 2: Empty query validation
    cleaned_query = query_text.strip()
    if not cleaned_query:
        return "Validation Error: Query cannot be empty."

    # Guard 3: Token budget
    if tenant_tokens_remaining <= 0:
        return "Quota Exceeded: Please upgrade your subscription tier."

    # Happy path
    return f"Dispatching '{cleaned_query}' to pgvector HNSW search and Gemini Flash."

print(process_rag_query("What is the Q3 travel budget?", 50000, False))
```

#### Example 5.3: Real-Time SignalR Streaming Handshake
Validate incoming SignalR websocket connection parameters:
```python
def validate_signalr_handshake(jwt_claims: dict, client_tenant_id: str):
    # Guard 1: Token presence
    if not jwt_claims:
        return False, "401 Unauthorized: Missing bearer token"

    # Guard 2: Token expiration
    if jwt_claims.get("exp_seconds", 0) <= 0:
        return False, "401 Unauthorized: Token has expired"

    # Guard 3: Tenant claim match
    if jwt_claims.get("tenant_id") != client_tenant_id:
        return False, "404 Not Found: Tenant scope mismatch"

    return True, "Handshake accepted. Connecting to /hubs/chat-stream."

is_ok, msg = validate_signalr_handshake({"tenant_id": "T-10", "exp_seconds": 3600}, "T-10")
print(f"Handshake status: {msg}")
```

#### Example 5.4: Background Worker Batch Document Chunking
A background worker processes PDF extraction chunks from MinIO:
```python
def process_chunk_batch(worker_healthy: bool, chunks: list, max_batch_size: int = 100):
    if not worker_healthy:
        return "Worker paused: Unhealthy status reported to OpenTelemetry."

    if not chunks:
        return "No chunks in batch queue. Worker sleeping."

    if len(chunks) > max_batch_size:
        return f"Batch size {len(chunks)} exceeds limit of {max_batch_size}. Rejecting."

    return f"Successfully queued {len(chunks)} chunks for pgvector embedding."

print(process_chunk_batch(True, ["chunk_1", "chunk_2", "chunk_3"]))
```

#### Example 5.5: API Key Revocation & Rate-Limiting Guard
Validate programmatic API keys used for enterprise automated document ingestion:
```python
def verify_api_key(key_record: dict | None, request_ip_whitelisted: bool):
    if key_record is None:
        return False, "Invalid API key"

    if key_record.get("is_revoked"):
        return False, "API key has been revoked by workspace admin"

    if not request_ip_whitelisted:
        return False, "Incoming IP address not authorized for this API key"

    return True, "API Key authenticated successfully"

record = {"key_id": "key_01", "is_revoked": False}
print(verify_api_key(record, True))
```

---

## 6. Logical Operators & Short-Circuit Evaluation (`and`, `or`, `not`)

Logical operators combine multiple boolean expressions:
* **`and`**: Returns `True` if **both** operands are truthy.
* **`or`**: Returns `True` if **at least one** operand is truthy.
* **`not`**: Inverts the truth value of the operand.

```mermaid
flowchart TD
    subgraph AND Short-Circuit
        A1[Evaluate Left Operand] --> A2{Left is Falsy?}
        A2 -- Yes --> A3[Return Left Immediately<br/>Right operand NEVER evaluated]
        A2 -- No --> A4[Evaluate & Return Right Operand]
    end

    subgraph OR Short-Circuit
        B1[Evaluate Left Operand] --> B2{Left is Truthy?}
        B2 -- Yes --> B3[Return Left Immediately<br/>Right operand NEVER evaluated]
        B2 -- No --> B4[Evaluate & Return Right Operand]
    end
```

### The Deep Truth: Python's `and` and `or` Return Operands, Not Booleans!

> [!NOTE]
> Unlike C#, where `&&` and `||` strictly return the `bool` values `true` or `false`, Python's `and` and `or` operators **return the actual value of the operand that decided the outcome**.

* `a or b`: If `a` is truthy, Python returns `a` immediately without evaluating `b`. If `a` is falsy, it evaluates and returns `b`.
* `a and b`: If `a` is falsy, Python returns `a` immediately without evaluating `b`. If `a` is truthy, it evaluates and returns `b`.

This behavior powers ubiquitous Python idioms:
```python
# Fallback idiom (similar to C# null-coalescing 'custom_theme ?? "normora-dark"'):
active_theme = custom_theme or "normora-dark"

# Safe navigation idiom:
tenant_name = tenant and tenant.get("name")
```

### Operator Precedence: `not` > `and` > `or`

Python evaluates logical operators in strict precedence order:
1. `not` (Highest)
2. `and` (Middle)
3. `or` (Lowest)

Always use parentheses `()` to clarify complex boolean expressions and eliminate ambiguity.

### 5 Real-World Normora Examples

#### Example 6.1: BOLA Defense Check (Tenant Isolation + Active Membership)
To view a document, the document's tenant ID must match the user's tenant ID, AND the user must not be suspended:
```python
doc_tenant_id = "TENANT-42"
user_tenant_id = "TENANT-42"
is_user_suspended = False

if (doc_tenant_id == user_tenant_id) and (not is_user_suspended):
    print("BOLA Check Passed: Access granted to document metadata.")
else:
    print("BOLA Check Failed: Returning 404 to prevent resource enumeration.")
```

#### Example 6.2: Enterprise SLA Guarantee Gate
A query qualifies for dedicated high-availability Google Gemini infrastructure if the tenant is on Enterprise OR holds a custom contract, AND has no overdue invoices:
```python
subscription_tier = "Enterprise"
has_custom_sla_contract = False
has_overdue_invoice = False

if (subscription_tier == "Enterprise" or has_custom_sla_contract) and not has_overdue_invoice:
    print("Routing query to dedicated high-availability Gemini infrastructure.")
```

#### Example 6.3: Validating User Role for White-Label Branding Changes
Only Workspace Owners or Administrators can update tenant logos, custom themes, and domain bindings:
```python
user_role = "Admin"  # Can be "Owner", "Admin", "Member", "Guest"

if user_role == "Owner" or user_role == "Admin":
    print("Permission granted: Updating tenant custom CSS and logos.")
else:
    print("BFLA Violation (403 Forbidden): Only Admins and Owners can modify branding.")
```

#### Example 6.4: Document Search Filter Validation with Fallback Idiom
Demonstrating both boolean validation and short-circuit operand returns:
```python
user_search_query = None
default_catalog_query = "status:INDEXED"

# Short-circuit fallback: Returns the first truthy value
effective_query = user_search_query or default_catalog_query
print(f"Executing search with query: '{effective_query}'")  # status:INDEXED

has_text = bool(user_search_query)
has_department_filter = True
has_date_range = False

if has_text or has_department_filter or has_date_range:
    print("Valid search request: At least one filter criterion is active.")
```

#### Example 6.5: Outbound Webhook Delivery Retry Condition
A webhook failure should be retried only if retry attempts remain AND the HTTP status is not a client-side 4xx error:
```python
retries_left = 3
http_status = 503  # Service Unavailable (transient error)

# Safe evaluation: Avoids retrying 400 Bad Request or 404 Not Found
if retries_left > 0 and not (400 <= http_status < 500):
    print(f"Scheduling retry for webhook. HTTP {http_status} is transient.")
else:
    print("Discontinuing webhook retry attempts.")
```

---

## 7. Truthiness: What Python Evaluates as `True` and `False`

In C#, an `if` statement **requires** an expression of type `bool`:
```csharp
// In C#, this will NOT compile:
// if (items) { ... }  // CS0029: Cannot implicitly convert type 'List' to 'bool'
if (items != null && items.Count > 0) { ... }
```

In Python, **any object** can be tested for truth value. This concept is called **truthiness**.

### Truthiness & Falsiness Reference Table

| Category | Type | Falsy Values (evaluates to `False`) | Truthy Values (evaluates to `True`) | Notes / C# Comparison |
| :--- | :--- | :--- | :--- | :--- |
| **Constants** | `NoneType`, `bool` | `None`, `False` | `True` | `None` is Python's `null` |
| **Integers** | `int` | `0` | Any non-zero (`1`, `-42`, `1000`) | C# requires `x != 0` |
| **Floating Point** | `float` | `0.0`, `-0.0` | Any non-zero (`0.001`, `-3.14`) | C# requires `f != 0.0` |
| **Strings** | `str` | `""` (empty string) | Any non-empty (`"Normora"`, `" "`) | Note: `" "` (whitespace) is truthy! |
| **Lists** | `list` | `[]` (empty list) | Any non-empty list (`["doc1"]`) | Replaces `list.Any()` |
| **Dictionaries**| `dict` | `{}` (empty dict) | Any non-empty dict (`{"k": "v"}`) | Replaces `dict.Count > 0` |
| **Sets** | `set` | `set()` (empty set) | Any non-empty set (`{1, 2}`) | Replaces `set.Count > 0` |
| **Tuples** | `tuple` | `()` (empty tuple) | Any non-empty tuple (`("a",)`) | Replaces tuple checks |
| **Custom Classes** | User-defined | If `__bool__()` returns `False` or `__len__()` returns `0` | All instances by default | Custom classes define truthiness |

> [!TIP]
> Under the hood, Python checks if the object's class defines `__bool__()`. If `__bool__()` is not defined, Python checks for `__len__()`. If `len(obj) == 0`, it is falsy; otherwise truthy. If neither exists, the object is always `True`.

### 5 Real-World Normora Examples

#### Example 7.1: Checking if Retrieved Document Chunks are Empty
Verify whether PostgreSQL pgvector returned any matching knowledge chunks:
```python
retrieved_chunks = []  # Search returned 0 rows above threshold

# ✅ Pythonic: Directly check truthiness (replaces 'if len(retrieved_chunks) == 0:')
if not retrieved_chunks:
    print("No relevant knowledge found. AI responds: 'I cannot find that in your company policies.'")
```

#### Example 7.2: Checking if User Supplied an Optional Search Query
Handle user query input after stripping leading/trailing whitespace:
```python
raw_input = "   "
cleaned_input = raw_input.strip()  # Strips to "" (empty string)

# Empty string evaluates to False
if not cleaned_input:
    print("Displaying all default tenant documents (no search query supplied).")
```

#### Example 7.3: Validating Tenant Department Memberships
Inspect whether a user belongs to any active security groups:
```python
user_groups = ["Legal-Reviewers", "Compliance-Auditors"]

# Non-empty list evaluates to True
if user_groups:
    print(f"User belongs to {len(user_groups)} groups. Applying group query filters.")
else:
    print("User has no group memberships. Applying public-only document filter.")
```

#### Example 7.4: Checking for Remaining Token Balance
Stop streaming tokens when the tenant's allocation drops to zero:
```python
tokens_remaining = 0

# Integer 0 evaluates to False
if not tokens_remaining:
    print("Token balance depleted. Halting generative streaming from Gemini Flash.")
```

#### Example 7.5: Detecting Cached Document Summary
Check if a document already has a precomputed summary stored in PostgreSQL:
```python
cached_summary = "This document details Normora's employee travel expense reimbursement limits."

if cached_summary:
    print(f"Cache Hit: {cached_summary}")
else:
    print("Cache Miss: Invoking Gemini Flash summarization pipeline.")
```

---

## 8. One-Line Conditionals: The Ternary Operator

Python provides a concise ternary expression for conditional assignments:

$$\text{result} = \text{value\_if\_true} \;\mathbf{if}\; \text{condition} \;\mathbf{else}\; \text{value\_if\_false}$$

### Syntax Comparison: C# vs Python

```csharp
// C# Ternary Operator:
string badge = isVerified ? "bg-emerald-500" : "bg-amber-500";
```

```python
# Python Conditional Expression:
badge = "bg-emerald-500" if is_verified else "bg-amber-500"
```

Notice the order: Python places the **true value first**, followed by the `if condition`, followed by the `else false value`.

### 5 Real-World Normora Examples

#### Example 8.1: Setting UI Badge Color in Angular 22 Client
Calculate the Tailwind CSS status badge class sent to the Angular frontend:
```python
is_verified = True
badge_color = "bg-emerald-500" if is_verified else "bg-amber-500"
print(f"Status Badge Class: {badge_color}")  # bg-emerald-500
```

#### Example 8.2: Assigning Embedding Dimension based on AI Provider
Google Gemini embeddings (`text-embedding-004`) produce 768 dimensions, while OpenAI embeddings produce 1536 dimensions:
```python
ai_provider = "gemini"
vector_dimension = 768 if ai_provider == "gemini" else 1536
print(f"Target pgvector column: Vector({vector_dimension})")
```

#### Example 8.3: Defaulting Missing Tenant Branding Theme
Assign a default theme if custom branding is not configured:
```python
custom_theme = None
active_theme = custom_theme if custom_theme else "normora-dark"
print(f"Active Theme: {active_theme}")  # normora-dark
```

#### Example 8.4: Setting Document Classification
Tag metadata during document ingestion:
```python
is_confidential = True
access_level = "RESTRICTED" if is_confidential else "PUBLIC"
print(f"Document Classification: {access_level}")
```

#### Example 8.5: Dynamic Citations Disclaimer Suffix for AI Streaming
Append an attribution footer when generating responses via SignalR:
```python
has_citations = True
disclaimer = "\n\n[Verified with internal documents]" if has_citations else "\n\n[General AI guidance]"
print(f"Stream Footer: {disclaimer.strip()}")
```

---

## 9. The Walrus Operator (`:=` Assignment Expressions)

Introduced in **Python 3.8 (PEP 572)**, the **walrus operator (`:=`)** assigns a value to a variable as part of an expression. It gets its nickname from its resemblance to the eyes and tusks of a walrus.

### The Problem It Solves

Without the walrus operator, you frequently evaluate an expression, assign it to a variable, and then test that variable:
```python
# Traditional approach (2 lines, separate assignment):
match_result = citation_pattern.search(text)
if match_result:
    citation_id = match_result.group(1)
```

With the walrus operator, you combine assignment and evaluation in a single line:
```python
# Pythonic approach with Walrus Operator:
if match_result := citation_pattern.search(text):
    citation_id = match_result.group(1)
```

### C# Developer Equivalent
In C#, you may be used to inline assignments or C# 9+ pattern matching:
```csharp
// C# pattern matching assignment:
if (citationPattern.Match(text) is { Success: true } match) {
    var citationId = match.Groups[1].Value;
}
```

### 5 Real-World Normora Examples

#### Example 9.1: Regex Extraction of Citation Metadata
Extract citation markers (e.g., `[Doc#42]`) from Google Gemini's raw streaming output:
```python
import re

ai_response_chunk = "According to our HR policy [Citation:DOC-8492], employees receive 25 vacation days."
citation_pattern = re.compile(r"\[Citation:([A-Z0-9\-]+)\]")

# Walrus assigns regex match and checks truthiness simultaneously
if match := citation_pattern.search(ai_response_chunk):
    cited_doc_id = match.group(1)
    print(f"Extracted Citation ID: {cited_doc_id}")  # DOC-8492
else:
    print("No inline citation found in this response chunk.")
```

#### Example 9.2: Document Chunk Buffer Size Evaluation
Ensure incoming chunk text does not overflow the maximum chunk size before storing:
```python
raw_extracted_text = "Normora enterprise platform guidelines..." * 50
MAX_CHUNK_CHARS = 1000

# Compute length, assign to chunk_size, and test limit in one line
if (chunk_size := len(raw_extracted_text)) > MAX_CHUNK_CHARS:
    print(f"Chunk size {chunk_size} exceeds {MAX_CHUNK_CHARS} limit. Splitting required.")
```

#### Example 9.3: Cached Embedding Lookup & Hit Validation
Check a local Redis cache for precomputed chunk vector embeddings:
```python
cache = {"chunk_hash_99a": [0.012, 0.842, -0.194]}
target_hash = "chunk_hash_99a"

# Assign cached_vector and verify it is not None
if (cached_vector := cache.get(target_hash)) is not None:
    print(f"Cache hit! Retrieved vector of length {len(cached_vector)} without calling API.")
```

#### Example 9.4: Extracting Rate-Limit Reset Header from Gemini API
Inspect response headers from Google Gemini Flash to handle rate-limiting backoff:
```python
response_headers = {"Content-Type": "application/json", "Retry-After": "45"}

# Extract 'Retry-After' and evaluate presence in one line
if retry_after_seconds := response_headers.get("Retry-After"):
    print(f"Rate limited by provider. Sleeping for {retry_after_seconds} seconds.")
```

#### Example 9.5: Parsing Background Extraction Error Details
Check background worker job status payload for error descriptions:
```python
job_record = {
    "job_id": "job_doc_884",
    "status": "FAILED",
    "error_details": "Corrupted PDF cross-reference table."
}

if error_msg := job_record.get("error_details"):
    print(f"Worker failure logged to OpenTelemetry: {error_msg}")
```

---

## 10. Modern Structural Pattern Matching (`match` / `case`)

Introduced in **Python 3.10 (PEP 634/635/636)**, **structural pattern matching** (`match`/`case`) is vastly more powerful than a simple switch statement. It matches on values, data shapes, sequence lengths, dictionary keys, and object types, while allowing variable destructuring and guard conditions.

```mermaid
flowchart TD
    ToolReq["Incoming LLM Tool Call Payload"] --> MatchBlock["match requested_tool:"]
    MatchBlock --> Case1["case 'search_documents':"]
    MatchBlock --> Case2["case 'lookup_members':"]
    MatchBlock --> Case3["case 'generate_pdf':"]
    MatchBlock --> CaseGuard["case {'tool': name, 'args': args} if not args:"]
    MatchBlock --> CaseWild["case _: (Default Catch-All)"]

    Case1 --> Exec1[Execute pgvector HNSW Hybrid Search]
    Case2 --> Exec2[Query Tenants Module for Active Users]
    Case3 --> Exec3[Compile Document Excerpt into PDF]
    CaseGuard --> ExecGuard[Reject: Missing Arguments]
    CaseWild --> ExecWild[Log Unknown Tool Invocation Error]
```

### Pattern Matching Capabilities
1. **Literal Patterns**: Match exact values (`case "pdf":`).
2. **Or-Patterns (`|`)**: Match multiple alternatives (`case "md" | "txt":`).
3. **Sequence Patterns**: Destructure lists or tuples (`case ["search", query, *filters]:`).
4. **Mapping Patterns**: Destructure dictionaries (`case {"role": "Admin", "dept": d}:`).
5. **Class Patterns**: Match custom object types and attributes (`case DocumentChunk(similarity=s) if s >= 0.8:`).
6. **Guards (`if`)**: Add boolean conditions to a case (`case int(val) if val > 100:`).
7. **Wildcard (`_`)**: Catch-all default case.

### 5 Real-World Normora Examples

#### Example 10.1: Conversational AI Tool Execution Router
When Google Gemini Flash requests a tool execution via function calling:
```python
requested_tool = "search_company_documents"

match requested_tool:
    case "search_company_documents":
        print("Executing hybrid pgvector search against Normora Documents.")
    case "lookup_department_members":
        print("Querying Tenants module for active departmental users.")
    case "get_subscription_usage":
        print("Calculating current billing cycle token and seat consumption.")
    case "generate_citation_pdf":
        print("Compiling verified document excerpt into downloadable PDF.")
    case _:
        print(f"Error: Unknown tool '{requested_tool}' requested by LLM.")
```

#### Example 10.2: Document Ingestion Multimodal Handler (Or-Patterns)
Handle different file extensions during document ingestion:
```python
file_extension = "pdf"

match file_extension:
    case "pdf":
        print("Extracting text via PyPDF and analyzing table geometries.")
    case "docx":
        print("Parsing Microsoft Word document XML and heading hierarchies.")
    case "md" | "txt":  # Pipe (|) matches multiple alternatives
        print("Reading raw UTF-8 text directly without conversion.")
    case "csv" | "xlsx":
        print("Parsing tabular dataset into structured row embeddings.")
    case _:
        print(f"Unsupported format: .{file_extension}. Ingestion rejected.")
```

#### Example 10.3: HTTP Status Code to Centralized `ApiMessages` Mapper
Map standard HTTP status codes to standardized Normora API response messages:
```python
status_code = 404

match status_code:
    case 200 | 201:
        response_message = "Operation completed successfully."
    case 400:
        response_message = "Invalid request payload or malformed syntax."
    case 403:
        response_message = "BFLA Violation: Insufficient permissions for this action."
    case 404:
        response_message = "BOLA Protection: Requested resource not found."
    case 429:
        response_message = "Rate limit exceeded. Please back off."
    case _:
        response_message = "An internal server error occurred."

print(f"[{status_code}] {response_message}")
```

#### Example 10.4: Complex Mapping Patterns with Guards (Tenant Plan Feature Matrix)
Inspect tenant plan configurations using dictionary pattern matching and guard clauses:
```python
tenant_context = {
    "tier": "Enterprise",
    "seat_count": 150,
    "has_dedicated_gemini_endpoint": True
}

match tenant_context:
    case {"tier": "Free"}:
        print("Features: 3 Documents, standard search.")
    case {"tier": "Starter", "seat_count": seats} if seats <= 10:
        print(f"Starter Plan: {seats} seats, 50 Documents, email support.")
    case {"tier": "Pro"}:
        print("Pro Plan: 500 Documents, hybrid pgvector search, CopilotKit UI.")
    case {"tier": "Enterprise", "has_dedicated_gemini_endpoint": True}:
        print("Enterprise Dedicated: Unlimited Documents, A2UI streaming, Custom SLA.")
    case {"tier": "Enterprise"}:
        print("Enterprise Standard: Shared cluster, unlimited documents.")
    case _:
        print("Invalid or suspended subscription profile.")
```

#### Example 10.5: Sequence Destructuring on Conversational AI Messages
Destructure structured chat command sequences:
```python
command_payload = ["search", "vacation policy", "--department", "HR"]

match command_payload:
    case ["search", str(query)]:
        print(f"Global document search for: '{query}'")
    case ["search", str(query), "--department", str(dept)]:
        print(f"Scoped search for: '{query}' restricted to department: '{dept}'")
    case ["summarize", int(doc_id)]:
        print(f"Triggering summarization on document ID: {doc_id}")
    case ["help"]:
        print("Listing available conversational assistant commands.")
    case _:
        print("Unrecognized command syntax. Type 'help' for instructions.")
```

---

## 11. Common Mistakes & Anti-Patterns

Below are the most critical mistakes developers make when writing conditional logic in Python.

### Anti-Pattern 1: Explicit Comparison to `True` or `False`
> [!CAUTION]
> **Never compare booleans to `True` or `False` using `== True` or `== False`.**
> ```python
> # ❌ WRONG: Redundant and un-pythonic
> if is_user_suspended == True:
>     return "Access Denied"
> 
> # ✅ CORRECT: Test truthiness directly
> if is_user_suspended:
>     return "Access Denied"
> ```
> Explicit comparison is redundant, slower, and breaks when testing objects that implement truthiness via `__bool__()`.

### Anti-Pattern 2: Using `== None` Instead of `is None`
> [!CAUTION]
> **Never use `== None`. Always use `is None` or `is not None`.**
> ```python
> # ❌ WRONG: Triggers __eq__() method, which can be overridden or slow
> if document_logo == None:
>     pass
> 
> # ✅ CORRECT: Verifies object identity against singleton None
> if document_logo is None:
>     pass
> ```

### Anti-Pattern 3: The Arrow Anti-Pattern (Excessive Nesting)
> [!CAUTION]
> **Do not write deep pyramids of nested `if` statements. Invert conditions and use Guard Clauses.**
> ```python
> # ❌ WRONG: Deep pyramid
> def evaluate(tenant):
>     if tenant:
>         if tenant.is_active:
>             if tenant.has_credits:
>                 return process(tenant)
> 
> # ✅ CORRECT: Clean guard clauses with early returns
> def evaluate(tenant):
>     if not tenant:
>         return "Missing tenant"
>     if not tenant.is_active:
>         return "Tenant inactive"
>     if not tenant.has_credits:
>         return "No credits"
>     return process(tenant)
> ```

### Anti-Pattern 4: Operator Precedence Traps with `not`, `and`, and `or`
> [!CAUTION]
> **Remember that `not` binds tighter than `and` and `or`.**
> ```python
> is_admin = False
> is_owner = True
> has_permission = True
> 
> # This evaluates as: (not is_admin) and is_owner or has_permission
> # Because 'not' has higher precedence than 'and'
> result = not is_admin and is_owner or has_permission
> 
> # ✅ ALWAYS use parentheses to make your intent crystal clear:
> safe_result = (not is_admin) and (is_owner or has_permission)
> ```

### Anti-Pattern 5: Redundant Evaluation Instead of Using the Walrus Operator
> [!CAUTION]
> **Avoid calling expensive functions twice (or doing lookup + check) when `:=` can capture and test in one step.**
> ```python
> # ❌ INEFFICIENT: Double dictionary lookup
> if cache.get(chunk_id) is not None:
>     vector = cache.get(chunk_id)  # Redundant lookup!
> 
> # ✅ EFFICIENT: Assign and test in one step
> if (vector := cache.get(chunk_id)) is not None:
>     use_vector(vector)
> ```

### Anti-Pattern 6: Misplacing the Wildcard `_` in `match`/`case`
> [!CAUTION]
> **The wildcard pattern `case _:` must always be the LAST case.**
> Any `case` statement placed after `case _:` is dead code and will never execute. Python will raise a `SyntaxError: wildcard makes remaining patterns unreachable`.

### Anti-Pattern 7: Confusing Bitwise Operators (`&`, `|`) with Logical Operators (`and`, `or`)
> [!CAUTION]
> In Python, `&` and `|` are bitwise operators (or set operators), while `and` and `or` are logical operators.
> ```python
> # ❌ DANGEROUS: Bitwise & does NOT short-circuit!
> # If user is None, user.is_active will raise an AttributeError!
> if (user is not None) & user.is_active:
>     pass
> 
> # ✅ CORRECT: Logical 'and' short-circuits safely
> if (user is not None) and user.is_active:
>     pass
> ```

---

## 12. C#/.NET vs. Python Decision Logic: Reference Matrix

Use this cheat sheet to quickly translate between C# 10/12 and modern Python 3.10+:

| Feature | C# (.NET 10) | Python (3.10+) |
| :--- | :--- | :--- |
| **Simple Condition** | `if (x > 0) { ... }` | `if x > 0:` |
| **Else If** | `else if (x == 0) { ... }` | `elif x == 0:` |
| **Else** | `else { ... }` | `else:` |
| **Logical AND** | `a && b` (short-circuiting) | `a and b` (returns operand) |
| **Logical OR** | `a \|\| b` (short-circuiting) | `a or b` (returns operand) |
| **Logical NOT** | `!a` | `not a` |
| **Null / None Check** | `x is null` / `x == null` | `x is None` |
| **Not Null / Not None** | `x is not null` / `x != null` | `x is not None` |
| **Reference Equality** | `object.ReferenceEquals(a, b)` | `a is b` |
| **Value Equality** | `a.Equals(b)` or `a == b` | `a == b` |
| **Ternary Operator** | `condition ? valTrue : valFalse` | `val_true if condition else val_false` |
| **Null Coalescing** | `theme ?? "dark"` | `theme or "dark"` (for truthy values) |
| **Chained Comparison** | `a <= x && x <= b` | `a <= x <= b` |
| **Inline Assignment** | `if (GetVal() is { } v)` | `if (v := get_val()):` |
| **Switch / Match** | `switch (x) { case ... => }` | `match x: case ...:` |
| **Switch Default** | `_ => ...` | `case _:` |

---

## 13. End-to-End Normora Project: The RAG Citation & Security Evaluation Engine

Below is a complete, runnable script demonstrating all decision-making constructs learned in this module. It implements Normora's **RAG Citation & Security Evaluation Engine**, combining Keycloak JWT extraction, BOLA defense, BFLA authorization, the walrus operator, short-circuit evaluation, and structural pattern matching.

```python
"""
Normora Multi-Tenant RAG Citation & Security Evaluation Engine
Demonstrating Module 2 Conditionals, Truthiness, Walrus, and Pattern Matching.
"""

from typing import Any, Dict, List, Optional
import re

# Centralized ApiMessages constants (matching Normora.Shared.Contracts)
class ApiMessages:
    SUCCESS = "Operation completed successfully."
    UNAUTHORIZED = "401 Unauthorized: Invalid or missing bearer token."
    BOLA_VIOLATION = "404 Not Found."  # Prevents tenant enumeration
    BFLA_VIOLATION = "403 Forbidden: Insufficient function privileges."
    QUOTA_EXCEEDED = "429 Too Many Requests: Monthly token quota depleted."


def evaluate_rag_request(
    user_context: Dict[str, Any],
    document_record: Optional[Dict[str, Any]],
    raw_query: str,
    retrieved_chunks: List[Dict[str, Any]],
    ai_response_text: str
) -> Dict[str, Any]:
    """
    Evaluates a RAG query turn through Normora's security and citation gates.
    """

    # 1. Guard Clause: Token Authentication
    if not user_context or not user_context.get("is_authenticated"):
        return {"status": 401, "message": ApiMessages.UNAUTHORIZED}

    # 2. Guard Clause: Document Existence & BOLA Defense
    # If the document does not exist OR belongs to another tenant, return 404
    user_tenant_id = user_context.get("tenant_id")
    if (document_record is None) or (document_record.get("tenant_id") != user_tenant_id):
        return {"status": 404, "message": ApiMessages.BOLA_VIOLATION}

    # 3. Guard Clause: BFLA Departmental Authorization
    user_role = user_context.get("role", "Guest")
    user_dept = user_context.get("department")
    doc_dept = document_record.get("department")

    is_workspace_owner = (user_role == "Owner")
    has_department_access = (user_dept == doc_dept)

    if not (is_workspace_owner or has_department_access):
        return {"status": 403, "message": ApiMessages.BFLA_VIOLATION}

    # 4. Guard Clause: Quota Check using Chained Comparison & Walrus Operator
    tokens_consumed = user_context.get("tokens_consumed", 0)
    monthly_limit = user_context.get("monthly_token_limit", 1_000_000)

    if tokens_consumed >= monthly_limit:
        return {"status": 429, "message": ApiMessages.QUOTA_EXCEEDED}

    # 5. Walrus Operator in Condition: Validate Clean Query Length
    if (query_length := len(clean_query := raw_query.strip())) == 0:
        return {"status": 400, "message": "Query cannot be empty."}

    # 6. Truthiness check on retrieved vector chunks
    if not retrieved_chunks:
        return {
            "status": 200,
            "response": "I could not find any relevant information in your tenant documents.",
            "citations": []
        }

    # 7. Ternary Operator: Configure citation classification based on top chunk similarity
    top_similarity = retrieved_chunks[0].get("similarity", 0.0)
    confidence_tier = "HIGH" if top_similarity >= 0.85 else "MEDIUM"

    # 8. Walrus Operator with Regex: Extract inline citation markers
    citation_regex = re.compile(r"\[DOC-(\d+)\]")
    found_citations = []
    if match := citation_regex.search(ai_response_text):
        found_citations.append(f"Document ID: {match.group(1)}")

    # 9. Structural Pattern Matching: Process action based on query intent
    intent_command = clean_query.split()[0].lower()
    
    match intent_command:
        case "summarize":
            action_performed = "Generated executive summary of document."
        case "audit" | "compliance":
            action_performed = "Dispatched query to Compliance Audit logging queue."
        case "search":
            action_performed = f"Hybrid pgvector search executed ({len(retrieved_chunks)} chunks evaluated)."
        case _:
            action_performed = "Standard conversational RAG streaming completed."

    return {
        "status": 200,
        "message": ApiMessages.SUCCESS,
        "confidence": confidence_tier,
        "top_similarity": top_similarity,
        "action": action_performed,
        "query_length": query_length,
        "citations": found_citations or ["Default Tenant Citation"]
    }


# =====================================================================
# Verification Runs
# =====================================================================

if __name__ == "__main__":
    current_user = {
        "is_authenticated": True,
        "tenant_id": "TENANT-ACME",
        "role": "Member",
        "department": "Engineering",
        "tokens_consumed": 450_000,
        "monthly_token_limit": 1_000_000
    }

    current_doc = {
        "doc_id": "DOC-991",
        "tenant_id": "TENANT-ACME",
        "department": "Engineering",
        "status": "INDEXED"
    }

    chunks = [
        {"chunk_id": 1, "similarity": 0.89, "text": "Normora deployment guidelines on Kubernetes."},
        {"chunk_id": 2, "similarity": 0.79, "text": "Microservice communication via MediatR."}
    ]

    llm_output = "Normora microservices communicate via MediatR [DOC-991] internally."

    result = evaluate_rag_request(
        user_context=current_user,
        document_record=current_doc,
        raw_query="search how services communicate",
        retrieved_chunks=chunks,
        ai_response_text=llm_output
    )

    print("--- Normora RAG Evaluation Result ---")
    for key, value in result.items():
        print(f"  {key}: {value}")
```

---

## 14. Technical Interview Questions & Deep Dives

Here are 8 common interview questions on Python decision logic, conditionals, and memory mechanics, tailored for experienced engineers transitioning from .NET:

### Q1: What is the exact difference between `==` and `is` in CPython? How does integer and string interning affect them?
**Answer:**
`==` evaluates **value equality** by invoking the `__eq__()` magic method on the left operand. `is` evaluates **object identity** by checking if both operands share the same memory pointer (`id(a) == id(b)`).

In CPython, small integers (typically between `-5` and `256`) and certain string literals are **interned** (cached in a global lookup table to save memory). Consequently, `x = 100; y = 100; x is y` evaluates to `True`. However, for numbers outside this range (e.g., `x = 1000; y = 1000`), `x is y` will often evaluate to `False`. Therefore, **never use `is` for value comparisons of numbers or strings**; reserve `is` exclusively for singletons like `None`, `True`, and `False`.

### Q2: How does Python evaluate truthiness under the hood? What is the lookup priority between `__bool__` and `__len__`?
**Answer:**
When an object is evaluated in a boolean context (`if obj:`), CPython first checks if the object's class implements the `__bool__()` method. If present, it must return `True` or `False`.
If `__bool__()` is not defined, CPython falls back to checking for the `__len__()` method. If `len(obj) == 0`, the object evaluates to `False`; otherwise `True`.
If neither method is defined on the class, all instances of the class evaluate to `True`.

### Q3: Why do Python's `and` and `or` operators return operand values rather than boolean `True`/`False`?
**Answer:**
Python's logical operators are designed for short-circuit expression composition.
* `a or b` evaluates `a`. If `a` is truthy, evaluation stops and `a` is returned. If `a` is falsy, `b` is evaluated and returned.
* `a and b` evaluates `a`. If `a` is falsy, evaluation stops and `a` is returned. If `a` is truthy, `b` is evaluated and returned.

This allows idiomatic fallback assignment (`theme = custom_theme or "normora-dark"`) and safe navigation without requiring separate null-coalescing operators like C#'s `??`.

### Q4: What is the Walrus Operator (`:=`), how does its variable scoping work, and what are its performance benefits?
**Answer:**
The walrus operator (assignment expression, PEP 572) assigns a value to a variable within an expression while returning that value.
Variables assigned with `:=` leak into the enclosing function or module scope (they are **not** scoped strictly to the `if` block).
Its performance benefit lies in eliminating duplicate computations: e.g., computing a regex search, finding string length, or querying a cache dictionary only once while simultaneously testing its truthiness in a conditional branch.

### Q5: How does Python 3.10's `match`/`case` differ from C#'s `switch` expressions?
**Answer:**
While both support pattern matching, Python's `match`/`case` is a structural pattern matching statement designed for dynamic, untyped, or semi-structured data:
1. It effortlessly destructures arbitrary nested dictionaries (`case {"tier": "Pro", "features": [first, *rest]}:`).
2. It destructures sequence lengths (`case [cmd, target]:`).
3. Captured variables are bound directly to local scope.
4. It does not fall through (no `break` statement needed).
5. It supports arbitrary boolean guard expressions using `if` clauses on cases.

### Q6: How do chained comparisons work in Python (e.g., `0 < score <= 1.0`), and does Python evaluate the middle expression twice?
**Answer:**
In Python, `a < b < c` is syntactically equivalent to `a < b and b < c`, with one crucial optimization: the middle expression `b` is **evaluated only once**. In C#, writing `score > 0 && score <= 1.0` requires referencing or computing `score` twice. If the middle expression is a function call (e.g., `0 < get_score() <= 1.0`), Python ensures the function executes exactly once.

### Q7: In an enterprise multi-tenant backend like Normora, why should BOLA checks return HTTP 404 instead of HTTP 403?
**Answer:**
Returning `403 Forbidden` confirms to an attacker that the targeted resource ID exists in the database, even though they lack permission to view it. This enables **ID enumeration attacks**, where an attacker scans IDs to map out competitor resource identifiers. Returning `404 Not Found` treats cross-tenant resources as non-existent, eliminating enumeration vectors entirely.

### Q8: What are the operator precedence rules between `not`, `and`, `or`, and comparison operators?
**Answer:**
Comparison operators (`==`, `!=`, `<`, `>`, `<=`, `>=`) have higher precedence than logical operators.
Among logical operators, the precedence is strictly:
1. `not` (Highest)
2. `and` (Middle)
3. `or` (Lowest)

Therefore, `not a == b and c or d` is evaluated as `((not (a == b)) and c) or d`. Misunderstanding this precedence is a frequent source of authorization bypass bugs, making parentheses essential for readability and safety.

---

## 15. Module Summary & Next Steps

In this module, you mastered:
1. **Comparison Operators & Chained Comparisons**: Writing clean, readable boundaries (`128 <= tokens <= 512`) evaluated with single-pass efficiency.
2. **Value Equality (`==`) vs. Object Identity (`is`)**: Understanding CPython's memory model and the singleton nature of `None`.
3. **Indented Branching (`if`, `elif`, `else`)**: Writing idiomatic, structured decision trees without parentheses or braces.
4. **Guard Clauses & Early Returns**: Eliminating the Arrow Anti-Pattern to build robust BOLA and BFLA security barriers.
5. **Logical Operators & Value-Returning Short-Circuiting**: Leveraging Python's `and`, `or`, and `not` operators for fallback assignments and guarded execution.
6. **Structural Truthiness**: Mastering Python's truth testing protocol across primitives, sequences, and custom classes.
7. **Ternary Expressions**: Writing clean, one-line conditional expressions (`val_true if cond else val_false`).
8. **The Walrus Operator (`:=`)**: Combining assignment and conditional testing to optimize performance and eliminate duplicate code.
9. **Modern Structural Pattern Matching (`match`/`case`)**: Utilizing sequence destructuring, mapping extraction, or-patterns, and guard expressions for complex state machines and tool calling.

You are now fully equipped to make architectural decisions in Python code. Proceed to **Module 3: Loops, Iterations & Comprehensions** to master high-performance data processing pipelines!
