# Module 2: Conditionals, Booleans & Decision Logic

Welcome to **Module 2** of the Normora Python & AI Masterclass.

In this module, you will learn how Python makes decisions using boolean expressions, comparison operators, conditional branching (`if`, `elif`, `else`), logical chaining (`and`, `or`, `not`), truthiness evaluation, and modern pattern matching (`match`/`case`).

Every concept in this module is accompanied by **at least 5 real-world examples from the Normora platform**—covering tenant subscriptions, departmental access control, document ingestion statuses, vector search similarity thresholds, and AI token quotas.

---

## 1. Boolean Expressions & Comparison Operators

A **Boolean expression** evaluates to either `True` or `False`. 

Python provides standard comparison operators:
* `==` (Equal to)
* `!=` (Not equal to)
* `>` (Greater than)
* `<` (Less than)
* `>=` (Greater than or equal to)
* `<=` (Less than or equal to)

### 5 Real-World Normora Examples:

#### Example 1.1: Monthly Token Quota Enforcement
Check if a tenant has exceeded their monthly allocated quota of 1,000,000 tokens:
```python
tokens_used = 1_045_000
monthly_quota = 1_000_000
has_exceeded_quota = tokens_used >= monthly_quota
print(f"Has exceeded quota: {has_exceeded_quota}")  # True
```

#### Example 1.2: Minimum Vector Similarity Gate for RAG Citations
Normora only cites document chunks if their cosine similarity score meets or exceeds 0.78:
```python
chunk_similarity = 0.845
MINIMUM_SIMILARITY_THRESHOLD = 0.78
is_qualified_citation = chunk_similarity >= MINIMUM_SIMILARITY_THRESHOLD
print(f"Qualified for citation: {is_qualified_citation}")  # True
```

#### Example 1.3: Document Upload Size Validation
Ensure an uploaded company policy does not exceed the 25 MB limit:
```python
uploaded_file_size_bytes = 28_400_000  # 28.4 MB
MAX_ALLOWED_SIZE_BYTES = 25_000_000    # 25 MB
is_valid_size = uploaded_file_size_bytes <= MAX_ALLOWED_SIZE_BYTES
print(f"File size valid: {is_valid_size}")  # False
```

#### Example 1.4: Departmental BFLA (Function-Level Access) Check
Verify whether a user belongs to the required "Legal" department:
```python
user_department = "Human Resources"
is_legal_dept = (user_department == "Legal")
print(f"User is in Legal: {is_legal_dept}")  # False
```

#### Example 1.5: Tenant Workspace Deletion Guard
Verify that a tenant's active user count is strictly zero before allowing permanent workspace deletion:
```python
current_active_members = 0
can_safely_delete_workspace = (current_active_members == 0)
print(f"Safe to delete workspace: {can_safely_delete_workspace}")  # True
```

---

## 2. Equality (`==`) vs. Identity (`is`)

* **`==`** checks **value equality** (do the two objects hold the same content?).
* **`is`** checks **object identity** (do the two variables point to the exact same location in memory?).

In Python, checking against singletons like `None`, `True`, or `False` must always use `is` or `is not`.

### 5 Real-World Normora Examples:

#### Example 2.1: Checking for Unset Custom Tenant Branding Logo
```python
tenant_custom_logo_url = None

# ✅ Pythonic: Check if the logo is absent using 'is'
if tenant_custom_logo_url is None:
    active_logo = "https://cdn.normora.ai/assets/default-logo.svg"
    print(f"Using fallback: {active_logo}")
```

#### Example 2.2: Distinguishing Two Independent Document Extraction Tasks
Two separate document jobs may have identical status text, but they are distinct memory instances:
```python
job_alpha_tags = ["policy", "hr"]
job_beta_tags = ["policy", "hr"]

print(job_alpha_tags == job_beta_tags)  # True: Their tag values are identical
print(job_alpha_tags is job_beta_tags)  # False: Distinct heap objects in memory
```

#### Example 2.3: Checking if a Parent Department Exists
Root departments in Normora have no parent (`parent_department_id = None`):
```python
parent_department_id = None

is_root_department = (parent_department_id is None)
print(f"Is top-level root department: {is_root_department}")  # True
```

#### Example 2.4: Validating an Expired Invitation Token
When a tenant invitation is revoked, its accepted timestamp is cleared:
```python
invitation_accepted_at = None

if invitation_accepted_at is not None:
    print("Invitation already used.")
else:
    print("Invitation is still pending.")
```

#### Example 2.5: Identity Check on Tenant Singleton Services
Verifying that two module references point to the exact same shared database connection pool singleton:
```python
class DatabasePool:
    pass

pool_instance = DatabasePool()
tenants_module_pool = pool_instance
documents_module_pool = pool_instance

print(tenants_module_pool is documents_module_pool)  # True: Exactly identical singleton
```

---

## 3. Conditional Branching: `if`, `elif`, and `else`

Branching routes execution down different paths based on conditions. Python uses **4-space indentation** to define blocks.

### 5 Real-World Normora Examples:

#### Example 3.1: Document Ingestion Pipeline Router
```python
ingestion_state = "EXTRACTING_TEXT"

if ingestion_state == "PENDING":
    print("Step 1: Queued in MinIO storage outbox.")
elif ingestion_state == "EXTRACTING_TEXT":
    print("Step 2: Parsing PDF text and extracting tabular sections.")
elif ingestion_state == "GENERATING_EMBEDDINGS":
    print("Step 3: Calling Google Gemini embedding endpoint.")
elif ingestion_state == "INDEXED":
    print("Step 4: Vector stored in pgvector HNSW index. Ready for query!")
else:
    print("Error: Unknown ingestion state encountered.")
```

#### Example 3.2: Subscription Tier AI Rate Limiting
```python
subscription_tier = "Pro"

if subscription_tier == "Enterprise":
    requests_per_minute_limit = 1200
elif subscription_tier == "Pro":
    requests_per_minute_limit = 300
elif subscription_tier == "Starter":
    requests_per_minute_limit = 60
else:
    requests_per_minute_limit = 0  # Suspended or invalid
    print("Warning: Unrecognized tier. Blocking incoming API requests.")

print(f"Configured rate limit: {requests_per_minute_limit} RPM")
```

#### Example 3.3: Departmental Access Authorization
```python
requested_document_department = "Finance"
user_department = "Engineering"
user_role = "Member"

if user_role == "Owner":
    print("Access granted: Workspace Owner has global override privileges.")
elif user_department == requested_document_department:
    print(f"Access granted: User belongs to {requested_document_department}.")
else:
    print(f"Access denied: Document restricted to {requested_document_department}.")
```

#### Example 3.4: RAG Chunk Re-Ranking Thresholds
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

print(f"Citation Classification: {citation_tier}")
```

#### Example 3.5: Handling Conversation Session History Length
```python
total_history_turns = 28

if total_history_turns > 30:
    print("Session history exceeds limit. Triggering background conversation summary.")
elif total_history_turns > 20:
    print("Approaching context limit. Pruning oldest conversational turns.")
else:
    print("Session length within healthy limits. Appending turn directly.")
```

---

## 4. Logical Operators: `and`, `or`, and `not`

Logical operators combine conditions:
* **`and`**: `True` only if *all* conditions are `True`.
* **`or`**: `True` if *at least one* condition is `True`.
* **`not`**: Reverses the condition.

### 5 Real-World Normora Examples:

#### Example 4.1: BOLA Defense Check (Tenant Isolation + Ownership)
To view a document, the document must belong to the user's tenant AND the user must not be suspended:
```python
document_tenant_id = "TENANT-42"
user_tenant_id = "TENANT-42"
is_user_suspended = False

if (document_tenant_id == user_tenant_id) and (not is_user_suspended):
    print("BOLA Check Passed: Access granted to document metadata.")
else:
    print("BOLA Check Failed: 404 Not Found returned to prevent enumeration.")
```

#### Example 4.2: Enterprise SLA Guarantee Gate
Check if a tenant qualifies for dedicated 99.99% uptime guarantees:
```python
plan = "Enterprise"
has_custom_contract = True
has_outstanding_invoice = False

if (plan == "Enterprise" or has_custom_contract) and not has_outstanding_invoice:
    print("Routing query to dedicated high-availability Gemini infrastructure.")
```

#### Example 4.3: Validating User Role for White-Label Branding Changes
Only Workspace Owners or Administrators can update tenant logos and theme colors:
```python
user_role = "Admin"  # Can be "Owner", "Admin", "Member", "Guest"

if user_role == "Owner" or user_role == "Admin":
    print("Permission granted: Updating tenant custom CSS and logos.")
else:
    print("BFLA Error (403 Forbidden): Only Admins and Owners can modify branding.")
```

#### Example 4.4: Document Search Filter Validation
Check if the user provided search criteria before initiating an expensive database query:
```python
has_text_query = True
has_filter_department = False
has_date_range = False

# Search query is valid if at least one filter was supplied:
if has_text_query or has_filter_department or has_date_range:
    print("Dispatching search query to Documents module.")
else:
    print("Aborting search: User did not specify any query parameters.")
```

#### Example 4.5: Outbound Webhook Delivery Retry Condition
A webhook retry should occur only if attempts remain AND the failure was not a client-side 400 Bad Request:
```python
retry_attempts_left = 2
last_http_status = 503  # Service Unavailable (Transient server failure)

if retry_attempts_left > 0 and not (400 <= last_http_status < 500):
    print(f"Retrying webhook delivery. Status {last_http_status} is retryable.")
else:
    print("Discontinuing webhook retries.")
```

---

## 5. Truthiness: What Python Evaluates as `True` and `False`

In Python, every object has an inherent truth value.
* **Falsy values:** `None`, `False`, `0`, `0.0`, `""` (empty string), `[]` (empty list), `{}` (empty dict), `set()`.
* **Truthy values:** Any non-zero number, any non-empty string, any non-empty collection.

### 5 Real-World Normora Examples:

#### Example 5.1: Checking if Retrieved Document Chunks are Empty
```python
retrieved_chunks = []  # pgvector search returned zero matches

if not retrieved_chunks:
    print("No relevant knowledge found. AI will reply: 'I cannot find that in your company policies.'")
```

#### Example 5.2: Checking if User Supplied an Optional Search Query
```python
search_input = "   ".strip()  # User entered spaces, which strips to ""

if not search_input:
    print("Displaying all default tenant documents (no search query supplied).")
```

#### Example 5.3: Validating Tenant Department Memberships
```python
user_groups = ["Legal-Reviewers", "Compliance-Auditors"]

if user_groups:
    print(f"User belongs to {len(user_groups)} groups. Applying group query filters.")
```

#### Example 5.4: Checking for Remaining Token Balance
```python
tokens_remaining = 0

if not tokens_remaining:
    print("Token balance depleted. Halting generative streaming.")
```

#### Example 5.5: Checking if a Document Summary Exists
```python
document_cached_summary = "This document details Normora's employee travel expense reimbursement limits."

if document_cached_summary:
    print(f"Returning cached summary: {document_cached_summary}")
else:
    print("No summary cached. Invoking summarization agent.")
```

---

## 6. One-Line Conditional: Ternary Expressions

**Syntax:** `value_if_true if condition else value_if_false`

### 5 Real-World Normora Examples:

#### Example 6.1: Setting UI Badge Color in Angular Client
```python
is_verified = True
badge_color = "emerald-500" if is_verified else "amber-500"
print(f"Status Badge: {badge_color}")  # emerald-500
```

#### Example 6.2: Assigning Embedding Dimension based on Provider
```python
ai_provider = "gemini"
vector_dimension = 768 if ai_provider == "gemini" else 1536
print(f"Target pgvector column: Vector({vector_dimension})")
```

#### Example 6.3: Defaulting Missing Tenant Branding
```python
custom_theme = None
active_theme = custom_theme if custom_theme else "normora-dark"
print(f"Theme: {active_theme}")  # normora-dark
```

#### Example 6.4: Setting Document Classification
```python
is_confidential = True
access_level = "RESTRICTED" if is_confidential else "PUBLIC"
print(f"Document Classification: {access_level}")
```

#### Example 6.5: Dynamic Message Suffix for AI Responses
```python
has_citations = True
disclaimer = "\n\n[Verified with internal documents]" if has_citations else "\n\n[General AI response]"
print(f"Footer: {disclaimer.strip()}")
```

---

## 7. Modern Pattern Matching (`match` / `case`)

Introduced in Python 3.10+, `match` / `case` provides clean, expressive branching for state machines, tool invocations, and HTTP error code routing.

### 5 Real-World Normora Examples:

#### Example 7.1: Conversational AI Tool Execution Router
When the LLM decides to call a tool, we match on the requested tool name:
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

#### Example 7.2: Document Ingestion File Type Handler
```python
file_extension = "pdf"

match file_extension:
    case "pdf":
        print("Extracting text via PyPDF and analyzing table geometries.")
    case "docx":
        print("Parsing Microsoft Word document XML and heading hierarchies.")
    case "md" | "txt":  # Matching multiple alternatives with pipe (|)
        print("Reading raw UTF-8 text directly without conversion.")
    case "csv" | "xlsx":
        print("Parsing tabular dataset into structured row embeddings.")
    case _:
        print(f"Unsupported format: .{file_extension}. Ingestion rejected.")
```

#### Example 7.3: HTTP Status Code to API Message Mapper
Mapping HTTP status codes to standardized Normora `ApiMessages`:
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

#### Example 7.4: Tenant Plan Feature Activation
```python
plan = "Enterprise"

match plan:
    case "Free":
        features = ["3 Documents", "Basic Search"]
    case "Starter":
        features = ["50 Documents", "Conversational RAG", "Email Support"]
    case "Pro":
        features = ["500 Documents", "Hybrid pgvector Search", "CopilotKit Integration"]
    case "Enterprise":
        features = ["Unlimited Documents", "Custom Branding", "A2UI Streaming", "Dedicated SLA"]
    case _:
        features = []

print(f"Active Features for {plan}: {features}")
```

#### Example 7.5: Chat Message Role Handler
Processing incoming chat turns for the Vercel AI SDK or Google Gemini API:
```python
message_role = "assistant"

match message_role:
    case "system":
        print("Injecting system prompt instructions to guide AI personality.")
    case "user":
        print("Appending user query and triggering RAG retrieval flow.")
    case "assistant":
        print("Rendering historical model response in conversation thread.")
    case "tool":
        print("Injecting JSON tool output back into conversational context.")
    case _:
        print(f"Invalid message role: {message_role}")
```

---

## Summary & Next Step

In this module, you mastered:
1. Comparison operators (`==`, `!=`, `<`, `>`, `<=`, `>=`).
2. Value equality (`==`) vs. identity (`is`), especially for checking `None`.
3. Indented conditional blocks (`if`, `elif`, `else`).
4. Logical operations (`and`, `or`, `not`) with short-circuit evaluation.
5. Python's truthiness rules (falsy vs truthy values).
6. One-line ternary conditions.
7. Structural pattern matching (`match` / `case`).

Now, let's proceed to **Module 5: Functions, Parameters (*args, **kwargs) & Variable Scope**, where we will package our logic into reusable, modular building blocks!
