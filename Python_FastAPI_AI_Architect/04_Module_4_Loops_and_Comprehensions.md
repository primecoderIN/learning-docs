# Module 4: Loops, Iteration & Comprehensions

Welcome to **Module 4** of the Normora Python & AI Masterclass.

In AI engineering, iteration is everywhere:
* Iterating over retrieved document chunks to rank them by semantic relevance.
* Looping over conversation turns to calculate cumulative token consumption.
* Streaming incoming AI tokens from Google Gemini as they arrive chunk by chunk.
* Transforming raw database rows into clean JSON responses for our Angular client.

In this module, you will master `for` loops, `while` loops, iteration utilities (`enumerate()`, `zip()`), loop controls (`break`, `continue`), and Python's signature feature: **Comprehensions**.

---

## 1. The `for` Loop: Iterating Over Sequences

In Python, a `for` loop does not require manual index management (`int i = 0; i < n; i++`). Instead, it acts as a "for-each" iterator that directly pulls items from any iterable (lists, tuples, strings, dictionaries):

```python
departments = ["Engineering", "Legal", "HR", "Finance"]

for dept in departments:
    print(f"Syncing permissions for {dept}...")
```

### Iterating Over Strings
Strings are sequences of characters, so you can iterate over them character by character:
```python
tenant_code = "NORM-88"
for char in tenant_code:
    print(char)
```

---

## 2. Generating Sequences with `range()`

The built-in `range()` function generates an immutable sequence of numbers on demand without allocating them all into memory upfront.

**Syntax:** `range(start, stop, step)` (The `stop` value is exclusive).

```python
# 1. Single argument: range(stop) -> 0 to stop-1
for attempt in range(3):
    print(f"Retrying Gemini API call (Attempt {attempt + 1})...")

# 2. Start and Stop: range(start, stop)
for page in range(1, 5):
    print(f"Processing PDF Page {page}")  # Pages 1, 2, 3, 4

# 3. With Step: range(start, stop, step)
for chunk_offset in range(0, 1000, 250):
    print(f"Extracting tokens from offset {chunk_offset}")  # 0, 250, 500, 750
```

---

## 3. Essential Iteration Power Tools: `enumerate()` and `zip()`

### 1. `enumerate()`: Tracking the Index and Item Together
Never manually create an index counter (`i = 0; i += 1`). Use `enumerate()`:

```python
citations = [
    "Handbook: Page 12",
    "Security Policy: Page 3",
    "Remote Work Guidelines: Page 1"
]

# enumerate(iterable, start=1)
for index, citation in enumerate(citations, start=1):
    print(f"[{index}] {citation}")

# Output:
# [1] Handbook: Page 12
# [2] Security Policy: Page 3
# [3] Remote Work Guidelines: Page 1
```

### 2. `zip()`: Looping Over Multiple Collections in Parallel
When you have two or more related lists of the same length, `zip()` pairs their corresponding elements together into tuples:

```python
chunk_ids = ["chk-1", "chk-2", "chk-3"]
similarity_scores = [0.94, 0.89, 0.76]
departments = ["Legal", "HR", "Engineering"]

for c_id, score, dept in zip(chunk_ids, similarity_scores, departments):
    print(f"Chunk {c_id} ({dept}) has relevance: {score:.1%}")
```

---

## 4. The `while` Loop & Loop Controls

A `while` loop continues executing as long as its boolean condition remains `True`:

```python
retry_count = 0
MAX_RETRIES = 3

while retry_count < MAX_RETRIES:
    print(f"Checking document extraction worker status (Check {retry_count + 1})...")
    retry_count += 1
```

### `break` and `continue`
* **`break`**: Exits the loop immediately, skipping any remaining iterations.
* **`continue`**: Skips the rest of the *current* iteration and jumps immediately to the next one.

```python
# Stop searching when total tokens exceed the context window limit
chunks = [
    {"id": "c1", "tokens": 400},
    {"id": "c2", "tokens": 500},
    {"id": "c3", "tokens": 800},
    {"id": "c4", "tokens": 300}
]

MAX_PROMPT_TOKENS = 1200
accumulated_tokens = 0
selected_chunks = []

for chunk in chunks:
    # Skip chunks that are abnormally small/empty
    if chunk["tokens"] < 50:
        continue

    # Stop if adding this chunk would exceed our prompt token budget
    if accumulated_tokens + chunk["tokens"] > MAX_PROMPT_TOKENS:
        print(f"Budget limit reached. Stopping chunk inclusion at chunk {chunk['id']}.")
        break

    selected_chunks.append(chunk["id"])
    accumulated_tokens += chunk["tokens"]

print(f"Selected Chunks: {selected_chunks} (Total Tokens: {accumulated_tokens})")
```

---

## 5. List, Dictionary, and Set Comprehensions

Comprehensions are one of Python's most powerful, idiomatic features. They provide a concise, readable syntax for transforming, filtering, and mapping collections.

### 1. List Comprehensions
**Syntax:** `[expression for item in iterable if condition]`

```python
# The Traditional Way (4 lines):
scores = [0.95, 0.42, 0.88, 0.31, 0.91]
high_relevance = []
for score in scores:
    if score >= 0.80:
        high_relevance.append(score)

# The Pythonic List Comprehension Way (1 line):
high_relevance = [score for score in scores if score >= 0.80]
print(high_relevance)  # [0.95, 0.88, 0.91]
```

#### Transforming Elements:
```python
document_titles = ["  security_policy.pdf  ", "  employee_handbook.pdf  "]

# Strip whitespace and convert to uppercase
clean_titles = [title.strip().upper() for title in document_titles]
print(clean_titles)  # ['SECURITY_POLICY.PDF', 'EMPLOYEE_HANDBOOK.PDF']
```

### 2. Dictionary Comprehensions
**Syntax:** `{key_expression: value_expression for item in iterable if condition}`

```python
users = [
    {"user_id": "u-1", "name": "Alice", "role": "Admin"},
    {"user_id": "u-2", "name": "Bob", "role": "Member"},
    {"user_id": "u-3", "name": "Charlie", "role": "Admin"}
]

# Create a fast lookup map: user_id -> role
user_role_map = {user["user_id"]: user["role"] for user in users}
print(user_role_map)
# Output: {'u-1': 'Admin', 'u-2': 'Member', 'u-3': 'Admin'}
```

### 3. Set Comprehensions
**Syntax:** `{expression for item in iterable if condition}`

```python
chunks = [
    {"dept": "Legal", "score": 0.91},
    {"dept": "HR", "score": 0.85},
    {"dept": "Legal", "score": 0.89}
]

# Extract a unique set of departments that produced high-scoring chunks
active_depts = {c["dept"] for c in chunks if c["score"] > 0.80}
print(active_depts)  # {'Legal', 'HR'}
```

---

## 6. Normora Project: Grounded Context Builder for Gemini Flash

Let's build a real component for **Normora**: a RAG Context Builder that:
1. Iterates through retrieved document chunks.
2. Filters out low-similarity chunks (noise).
3. Formats valid chunks into a grounded markdown context block ready to be passed to Google Gemini Flash.
4. Generates citation footnotes.

Create `context_builder.py`:

```python
# ============================================================
# Normora RAG Engine: Grounded Context Builder
# ============================================================

retrieved_chunks = [
    {
        "chunk_id": "chk-101",
        "doc_name": "Compliance_Guide_2026.pdf",
        "page": 7,
        "department": "Legal",
        "text": "All employee confidential records must be encrypted using AES-256 at rest.",
        "similarity": 0.94
    },
    {
        "chunk_id": "chk-102",
        "doc_name": "Lunch_Menu.pdf",
        "page": 1,
        "department": "General",
        "text": "Taco Tuesday is held on the 2nd floor cafeteria at 12:30 PM.",
        "similarity": 0.35  # Irrelevant noise!
    },
    {
        "chunk_id": "chk-103",
        "doc_name": "Security_Standard.pdf",
        "page": 19,
        "department": "Engineering",
        "text": "Database credentials must be stored strictly in environment secrets and rotated every 90 days.",
        "similarity": 0.89
    }
]

SIMILARITY_THRESHOLD = 0.75

# Step 1: Filter relevant chunks using a list comprehension
qualified_chunks = [
    chunk for chunk in retrieved_chunks 
    if chunk["similarity"] >= SIMILARITY_THRESHOLD
]

print(f"Qualified chunks for context window: {len(qualified_chunks)} / {len(retrieved_chunks)}")

# Step 2: Build the grounded context block using enumerate()
context_blocks = []
citations = []

for index, chunk in enumerate(qualified_chunks, start=1):
    citation_label = f"[{index}] {chunk['doc_name']} (Page {chunk['page']})"
    citations.append(citation_label)
    
    formatted_block = f"""--- Document Chunk {index} ({citation_label}) ---
Department: {chunk['department']}
Content: {chunk['text']}"""
    context_blocks.append(formatted_block)

# Step 3: Combine into final Gemini prompt context
final_grounded_context = "\n\n".join(context_blocks)

print("\n" + "=" * 60)
print("FINAL CONTEXT INJECTED INTO GEMINI FLASH PROMPT:")
print("=" * 60)
print(final_grounded_context)

print("\n" + "=" * 60)
print("CITATIONS FOOTNOTE:")
print("=" * 60)
for citation in citations:
    print(f" * {citation}")
```

### Output:
```text
Qualified chunks for context window: 2 / 3

============================================================
FINAL CONTEXT INJECTED INTO GEMINI FLASH PROMPT:
============================================================
--- Document Chunk 1 ([1] Compliance_Guide_2026.pdf (Page 7)) ---
Department: Legal
Content: All employee confidential records must be encrypted using AES-256 at rest.

--- Document Chunk 2 ([2] Security_Standard.pdf (Page 19)) ---
Department: Engineering
Content: Database credentials must be stored strictly in environment secrets and rotated every 90 days.

============================================================
CITATIONS FOOTNOTE:
============================================================
 * [1] Compliance_Guide_2026.pdf (Page 7)
 * [2] Security_Standard.pdf (Page 19)
```

In **Module 5**, we will encapsulate reusable logic into **Functions, Parameters, Variable Scope, and Modules**!
