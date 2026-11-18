# Module 3: Core Data Structures (Lists, Tuples, Dictionaries, Sets)

Welcome to **Module 3** of the Normora Python & AI Masterclass.

In production AI systems, data is structured into collections:
* Document ingestion processes an ordered **list of text chunks**.
* Database queries return fixed **tuples of record attributes**.
* API requests and tool declarations are represented as nested **dictionaries**.
* Citations and permission roles are deduplicated using **sets**.

In this module, you will master Python's four foundational collection types. Every single concept includes **at least 5 real-world examples from the Normora platform**.

---

## 1. Lists (`list`): Ordered, Mutable Sequences

A **list** is an ordered, mutable collection defined with square brackets `[]`. You can modify, append, insert, and delete items.

### 5 Real-World Normora Examples:

#### Example 1.1: Appending Incoming Conversation Messages
```python
conversation_history = [
    {"role": "user", "content": "What is our company maternity leave policy?"}
]

# When Google Gemini responds, append the assistant's turn:
assistant_message = {
    "role": "assistant",
    "content": "According to the Normora Employee Handbook, employees receive 16 weeks of paid leave."
}
conversation_history.append(assistant_message)
print(f"Total turns in conversation: {len(conversation_history)}")  # 2
```

#### Example 1.2: Slicing the Top-3 Most Relevant Document Chunks
Take only the top 3 highest-ranked chunks to prevent stuffing too many tokens into the LLM context window:
```python
ranked_chunks = ["chunk-101", "chunk-104", "chunk-108", "chunk-112", "chunk-119"]

# Slice the first 3 items (indices 0, 1, 2):
top_chunks = ranked_chunks[:3]
print(f"Injected context chunks: {top_chunks}")  # ['chunk-101', 'chunk-104', 'chunk-108']
```

#### Example 1.3: Accessing the Most Recent Turn with Negative Indexing (`[-1]`)
```python
chat_log = ["Hello", "Can you help me?", "Where is the security policy?"]

# Read the last user prompt:
last_prompt = chat_log[-1]
print(f"Querying vector database with prompt: '{last_prompt}'")
```

#### Example 1.4: Managing Departmental Allowed File Types
```python
allowed_mime_types = ["application/pdf", "text/markdown"]

# Add Microsoft Word support to the allowed list:
allowed_mime_types.append("application/vnd.openxmlformats-officedocument.wordprocessingml.document")

# Check if an uploaded file type is supported:
incoming_upload = "application/pdf"
if incoming_upload in allowed_mime_types:
    print("MIME type authorized for document ingestion.")
```

#### Example 1.5: Sorting Document Chunks by Similarity Score
```python
chunks = [
    {"id": "c1", "similarity": 0.74},
    {"id": "c2", "similarity": 0.93},
    {"id": "c3", "similarity": 0.81}
]

# Sort in-place by similarity in descending order:
chunks.sort(key=lambda x: x["similarity"], reverse=True)
print(f"Top chunk: {chunks[0]['id']} with score {chunks[0]['similarity']}")  # c2 (0.93)
```

---

## 2. Tuples (`tuple`): Ordered, Immutable Records

A **tuple** is an ordered collection defined with parentheses `()`. Unlike lists, tuples are **immutable**—they cannot be changed after creation.

### 5 Real-World Normora Examples:

#### Example 2.1: Returning Multiple Values from a Vector Search
A vector search function returns the matching chunk ID, its cosine score, and the source page as a single tuple:
```python
search_result = ("chk-881", 0.942, 14)

# Unpack the tuple directly into named variables:
chunk_id, similarity_score, page_number = search_result
print(f"Matched chunk {chunk_id} on page {page_number} with score {similarity_score}")
```

#### Example 2.2: Fixed System Dimensions for Vector Embeddings
Model vector dimensions never change at runtime and should be protected from accidental mutation:
```python
# Embedding dimension configurations: (Provider Name, Dimensions)
GEMINI_EMBEDDING_CONFIG = ("text-embedding-004", 768)
OPENAI_EMBEDDING_CONFIG = ("text-embedding-3-small", 1536)

print(f"Configured {GEMINI_EMBEDDING_CONFIG[0]} with {GEMINI_EMBEDDING_CONFIG[1]} dimensions.")
```

#### Example 2.3: Database Composite Primary Keys
In Normora's multi-tenant database, a membership is uniquely identified by a composite key: `(tenant_id, user_id)`:
```python
membership_key = ("TENANT-01", "USER-55")
print(f"Checking membership for Tenant: {membership_key[0]}, User: {membership_key[1]}")
```

#### Example 2.4: Tuple as a Dictionary Key for Caching
Because lists are mutable, they cannot be dictionary keys. Tuples are immutable and hashable, making them perfect for cache keys:
```python
# Cache key composed of: (tenant_id, document_id, query_hash)
rag_cache = {}
cache_key = ("TENANT-42", "DOC-99", "hash_abc123")

rag_cache[cache_key] = "Cached answer about expense policies."
print(rag_cache[("TENANT-42", "DOC-99", "hash_abc123")])
```

#### Example 2.5: Representing Bounding Box Coordinates on Extracted PDF Pages
```python
# (x0, y0, x1, y1) bounding box of cited paragraph
citation_bbox = (54.0, 120.5, 480.0, 165.2)
x_start, y_start, x_end, y_end = citation_bbox
print(f"Highlighting PDF region from ({x_start}, {y_start}) to ({x_end}, {y_end})")
```

---

## 3. Dictionaries (`dict`): Key-Value Hash Maps

A **dictionary** is an associative store of key-value pairs defined with `{}`. Dictionaries provide $O(1)$ fast lookups.

### 5 Real-World Normora Examples:

#### Example 3.1: Representing a Tenant Profile
```python
tenant = {
    "tenant_id": "T-104",
    "name": "Global Health Partners",
    "plan": "Enterprise",
    "max_seats": 500,
    "is_active": True
}
print(f"Tenant: {tenant['name']}, Plan: {tenant['plan']}")
```

#### Example 3.2: Safely Reading Optional Configuration with `.get()`
Reading with `[]` crashes with `KeyError` if the key is missing; `.get()` returns a safe default:
```python
tenant_settings = {
    "ai_streaming_enabled": True
}

# Read optional custom domain:
custom_domain = tenant_settings.get("custom_domain", "app.normora.ai")
print(f"Routing traffic to: {custom_domain}")  # app.normora.ai
```

#### Example 3.3: Defining an AI Tool Call Schema (For Google Gemini & OpenAI)
Tools exposed to AI models are declared as nested dictionaries matching the JSON Schema standard:
```python
search_tool_declaration = {
    "name": "search_company_documents",
    "description": "Searches Normora company policies by semantic relevance.",
    "parameters": {
        "type": "object",
        "properties": {
            "query": {"type": "string", "description": "The search keywords or question."},
            "department": {"type": "string", "description": "Optional department filter."}
        },
        "required": ["query"]
    }
}
print(f"Tool declared: {search_tool_declaration['name']}")
```

#### Example 3.4: Tracking Real-Time Token Consumption per Tenant
```python
tenant_usage_counters = {
    "TENANT-1": 42000,
    "TENANT-2": 115000
}

# Add 800 tokens to TENANT-1 after a streaming completion:
tenant_usage_counters["TENANT-1"] += 800
print(f"Updated usage: {tenant_usage_counters['TENANT-1']}")  # 42800
```

#### Example 3.5: Iterating Over Dictionary Key-Value Pairs with `.items()`
```python
department_document_counts = {
    "Legal": 142,
    "Engineering": 310,
    "Human Resources": 89
}

print("--- Normora Departmental Document Inventory ---")
for dept, count in department_document_counts.items():
    print(f"Department: {dept.ljust(16)} | Indexed Docs: {count}")
```

---

## 4. Sets (`set`): Unordered Unique Collections

A **set** is an unordered collection of unique elements defined with `{}`. Sets automatically eliminate duplicate entries.

### 5 Real-World Normora Examples:

#### Example 4.1: Deduplicating Document Tags Extracted from Ingestion
```python
extracted_tags = ["policy", "2026", "hr", "benefits", "policy", "benefits", "employee"]

# Cast to set to eliminate duplicates:
unique_tags = set(extracted_tags)
print(f"Unique tags: {unique_tags}")  # {'benefits', 'employee', 'policy', 'hr', '2026'}
```

#### Example 4.2: Fast $O(1)$ Permission Check
Checking membership in a set of 1,000 permissions takes nanoseconds:
```python
user_permissions = {"documents:read", "documents:upload", "chat:interact"}

required_permission = "tenants:delete"
if required_permission not in user_permissions:
    print("403 Forbidden: Missing 'tenants:delete' permission.")
```

#### Example 4.3: Mathematical Set Intersection (`&`) to Verify Multi-Department Access
Verify which departments both User A and User B share in common:
```python
user_a_depts = {"Engineering", "DevOps", "Infrastructure"}
user_b_depts = {"Security", "Infrastructure", "Compliance"}

shared_departments = user_a_depts & user_b_depts
print(f"Shared collaboration departments: {shared_departments}")  # {'Infrastructure'}
```

#### Example 4.4: Set Difference (`-`) to Find Unindexed Documents
Identify documents in MinIO storage that haven't been indexed into pgvector yet:
```python
uploaded_in_storage = {"doc-1", "doc-2", "doc-3", "doc-4"}
indexed_in_pgvector = {"doc-1", "doc-2"}

pending_indexing = uploaded_in_storage - indexed_in_pgvector
print(f"Documents queued for indexing: {pending_indexing}")  # {'doc-3', 'doc-4'}
```

#### Example 4.5: Filtering Unique Source Document Citations
Extracting unique document titles cited in a multi-turn AI response:
```python
cited_chunk_sources = [
    "Security_Standard.pdf",
    "Employee_Handbook.pdf",
    "Security_Standard.pdf",
    "Travel_Policy.pdf",
    "Employee_Handbook.pdf"
]

distinct_citations = set(cited_chunk_sources)
print(f"Total citations generated: {len(distinct_citations)}")  # 3 unique documents
```

---

## 5. Mutability vs. Immutability & Common Architectural Pitfalls

* **Mutable objects** (`list`, `dict`, `set`) can be modified in place.
* **Immutable objects** (`int`, `float`, `str`, `tuple`, `bool`, `None`) cannot be changed; modifying them creates a new object in memory.

### 5 Real-World Normora Examples:

#### Example 5.1: The Shallow Copy Reference Trap
Assigning a list with `=` does not copy it; it creates a second pointer to the exact same list:
```python
# ❌ Pitfall:
tenant_default_roles = ["Admin", "Member"]
new_workspace_roles = tenant_default_roles  # Both point to the exact same list!

new_workspace_roles.append("Auditor")

# Modifying new_workspace_roles accidentally modified tenant_default_roles!
print(tenant_default_roles)  # ['Admin', 'Member', 'Auditor'] (Bug!)

# ✅ Fix: Use .copy() or slice [:]
new_workspace_roles = tenant_default_roles.copy()
```

#### Example 5.2: Modifying Dictionaries In-Place
```python
active_session = {"session_id": "sess-1", "tokens": 100}

def record_token_usage(session: dict, additional_tokens: int):
    # In-place mutation affects the original dictionary:
    session["tokens"] += additional_tokens

record_token_usage(active_session, 50)
print(active_session["tokens"])  # 150
```

#### Example 5.3: Immutability of Strings (Safe from Mutation)
```python
base_endpoint = "https://api.normora.ai"
# Calling a string method returns a new string; it never changes the original:
upper_endpoint = base_endpoint.upper()

print(base_endpoint)   # "https://api.normora.ai" (Unchanged!)
print(upper_endpoint)  # "HTTPS://API.NORMORA.AI"
```

#### Example 5.4: The Mutable Default Argument Bug (Critical Python Trap!)
Never use a mutable object (`[]` or `{}`) as a default function argument:
```python
# ❌ Deadly Trap: The list is created once when Python loads the file!
def create_tenant(name: str, tags: list = []):
    tags.append("active")
    return {"name": name, "tags": tags}

t1 = create_tenant("Acme")
t2 = create_tenant("BetaCorp")

print(t2["tags"])  # ['active', 'active'] (t2 inherited t1's tag!)

# ✅ The Architect's Solution: Use None as default
def create_tenant_safe(name: str, tags: list | None = None):
    if tags is None:
        tags = []
    tags.append("active")
    return {"name": name, "tags": tags}
```

#### Example 5.5: Freezing User Roles with `frozenset`
When you need an immutable, tamper-proof set (e.g. system-level immutable roles):
```python
SYSTEM_PROTECTED_ROLES = frozenset(["SuperAdmin", "TenantOwner"])

# SYSTEM_PROTECTED_ROLES.add("Guest")  --> Raises AttributeError!
print("SuperAdmin" in SYSTEM_PROTECTED_ROLES)  # True
```

---

## Summary & Next Step

You have mastered:
1. **Lists:** Ordered, mutable sequences for queues, message histories, and chunk collections.
2. **Tuples:** Ordered, immutable records for coordinates, database keys, and multi-return functions.
3. **Dictionaries:** Key-value hash maps for tenant profiles, JSON payloads, and AI tool definitions.
4. **Sets:** Unique collections for fast membership tests and deduplicating document tags and citations.
5. **Mutability Realities:** Protecting your software from shallow copy references and mutable default arguments.

Now, let's proceed to **Module 5: Functions, Parameters (*args, **kwargs) & Variable Scope**!
