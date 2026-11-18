# Module 1: Python Basics, Variables & Primitive Types

Welcome to **Module 1**! In this module, we assume you have **zero prior experience** with Python. 

By the end of this module, you will understand how Python executes code, how to store and manipulate data using variables and primitive types, how to perform arithmetic and string formatting, and how to write clean, maintainable Python code following industry conventions.

Throughout this entire course, every single concept is taught through the real-world domain of **Normora**—our enterprise multi-tenant knowledge management and conversational AI platform.

---

## 1. How Python Runs Code

Python is an accessible, readable, and highly expressive programming language. Unlike languages that require a complex build-and-compile cycle before you can test a single line of code, Python can be executed in two primary ways:

### 1. The Interactive Shell (REPL)
If you open your terminal and type `python`, you enter the **Read-Eval-Print Loop (REPL)**:
```bash
python
```
You will see a prompt like `>>>`. You can type Python code line-by-line, press Enter, and see immediate feedback:
```python
>>> 10 + 5
15
>>> exit()
```

### 2. Python Script Files (`.py`)
In real-world applications like Normora, we write our code into text files with a `.py` extension (e.g., `app.py`, `billing.py`). 
To run a script, pass the file name to the `python` command:
```bash
python billing.py
```

---

## 2. Your First Line of Python: Output with `print()`

To display messages, variables, or data on the screen, use Python's built-in `print()` function:

```python
print("Welcome to Normora Enterprise Knowledge Management!")
```

### Line Breaks & Semicolons
*   **No Semicolons:** Unlike C#, Java, JavaScript, or C++, Python statements do **not** end with a semicolon (`;`). A statement simply ends when the line ends.
*   **Case Sensitivity:** Python is strictly case-sensitive. `print()` works, but `Print()` or `PRINT()` will throw a `NameError`.

### Comments in Python
Comments allow you to leave notes in your code that Python completely ignores during execution:

```python
# This is a single-line comment.
# In Normora, we use comments to explain the *intent* behind complex logic.

print("Starting Normora service...")  # You can also add comments after a statement
```

---

## 3. Variables & Dynamic Typing

A **variable** is a named reference used to store a value in memory. 

In Python, you do not declare the variable's type upfront (like `string name` or `int count`). You simply choose a variable name, use the assignment operator (`=`), and provide a value. Python figures out the type automatically (**Dynamic Typing**).

```python
# Creating variables for a Normora tenant
tenant_name = "Acme Global"
tenant_id = 101
monthly_active_users = 45
is_enterprise_plan = True
```

### Python Naming Conventions (`snake_case`)
In Python, naming follows the official community style guide known as **PEP 8**:
*   **Variables & Functions:** Always use **`snake_case`** (all lowercase words separated by underscores):
    *   ✅ `tenant_name`, `user_count`, `monthly_token_budget`
    *   ❌ `tenantName` (camelCase is for JavaScript/C#)
    *   ❌ `TenantName` (PascalCase is reserved for Python Classes)
*   **Constants:** If a value should never change throughout the application, write it in **`ALL_CAPS_SNAKE_CASE`**:
    *   `MAX_DOCUMENTS_PER_TENANT = 5000`
    *   `DEFAULT_EMBEDDING_DIMENSION = 768`

---

## 4. The 5 Fundamental Primitive Data Types

Python comes with 5 core primitive types that you will use constantly:

| Type | Python Name | Description | Normora Real-World Example |
|---|---|---|---|
| **String** | `str` | Text enclosed in quotes | `"Normora Documents Module"` |
| **Integer** | `int` | Whole numbers (positive, negative, zero) | `42`, `-5`, `1000000` |
| **Float** | `float` | Numbers with decimal points | `19.99`, `0.00015` |
| **Boolean** | `bool` | Logical truth values (`True` or `False`) | `True`, `False` |
| **NoneType** | `NoneType` | Represents the absence of a value (`None`) | `None` (like `null` in other languages) |

Let's inspect each one in detail.

### 1. Strings (`str`)
Strings represent text. You can define strings using single quotes (`'...'`) or double quotes (`"..."`):

```python
workspace_name = "Engineering Department"
document_title = 'Q3 Security Policy.pdf'
```

If you need a multi-line string (common when writing system prompts for AI models in Normora), use triple quotes (`"""..."""` or `'''...'''`):

```python
system_prompt = """You are Normora's AI Document Assistant.
Your job is to answer questions strictly based on the provided company policies.
Never hallucinate or guess facts not present in the context."""

print(system_prompt)
```

### 2. Integers (`int`)
Integers are whole numbers without a decimal point. Python integers have **arbitrary precision**—meaning they can grow as large as your computer's memory allows without overflowing!

```python
stored_chunks = 14500
token_count = 850320

# Tip: You can use underscores (_) as thousand separators for readability
total_tokens_processed = 1_000_000_000  # Exactly 1 billion
```

### 3. Floating-Point Numbers (`float`)
Floats represent real numbers containing decimal points:

```python
cost_per_thousand_tokens = 0.0015  # Cost in USD
search_similarity_score = 0.8974   # Semantic search relevance
```

### 4. Booleans (`bool`)
Booleans represent truth values. Note that in Python, **`True` and `False` must be capitalized**:

```python
is_tenant_verified = True
has_document_access = False

# ❌ Incorrect: true or false (will cause NameError)
# ✅ Correct: True or False
```

### 5. `None` (The Absence of a Value)
Python uses `None` (capitalized) to represent the intentional absence of a value or an uninitialized state. It is identical in concept to `null` in C# or JavaScript.

```python
# When a tenant has not yet uploaded their custom branding logo
tenant_custom_logo = None
```

---

## 5. Type Inspection & Explicit Type Conversion (Casting)

### Checking Types with `type()`
You can verify the type of any variable at runtime using the built-in `type()` function:

```python
tenant = "Normora"
user_count = 50
plan_price = 199.95
is_active = True

print(type(tenant))      # <class 'str'>
print(type(user_count))  # <class 'int'>
print(type(plan_price))  # <class 'float'>
print(type(is_active))   # <class 'bool'>
```

### Type Conversion (Casting)
Often, you receive data as text (e.g. from an HTTP query parameter, a database row, or user input) and need to convert it into a number to do calculations:

```python
raw_input = "250"  # This is a string (str)
print(type(raw_input))  # <class 'str'>

# Convert string to integer
converted_tokens = int(raw_input)
print(type(converted_tokens))  # <class 'int'>

# Convert integer to float
price = float(15)
print(price)  # 15.0

# Convert numbers or booleans to string
status_text = str(is_active)
print(status_text)  # "True"
```

> [!CAUTION]
> If you try to convert an invalid string into a number—such as `int("ten")`—Python will raise a `ValueError`. We will master handling these errors in Module 7.

---

## 6. Arithmetic Operators

Python supports all standard mathematical operations:

| Operator | Name | Example | Result |
|---|---|---|---|
| `+` | Addition | `10 + 5` | `15` |
| `-` | Subtraction | `20 - 7` | `13` |
| `*` | Multiplication | `4 * 3` | `12` |
| `/` | **True Division** (always produces a `float`!) | `10 / 2` | `5.0` |
| `//` | **Floor Division** (discards decimal remainder) | `10 // 3` | `3` |
| `%` | **Modulo** (returns division remainder) | `10 % 3` | `1` |
| `**` | **Exponentiation** (power) | `2 ** 3` | `8` (2 to the power 3) |

### Important Architectural Detail: Division in Python
In languages like C# or Java, dividing two integers (`10 / 4`) drops the decimal and gives `2`.  
**In Python 3, `/` always performs true floating-point division:**
```python
print(10 / 4)   # 2.5 (float)
print(10 // 4)  # 2 (int - floor division)
```

---

## 7. String Operations & Formatted Strings (f-strings)

Strings are fundamental to building AI applications because LLM prompts, citations, responses, and document texts are all strings.

### 1. Basic String Operations
```python
tenant_prefix = "NORM-"
tenant_code = "1049"

# Concatenation (joining strings with +)
full_tenant_id = tenant_prefix + tenant_code
print(full_tenant_id)  # "NORM-1049"

# Repetition (repeating strings with *)
divider = "-" * 30
print(divider)  # "------------------------------"
```

### 2. Common String Methods
Strings come with powerful built-in methods:

```python
document_name = "   Normora_Employee_Handbook_2026.PDF   "

# Remove leading and trailing whitespace
clean_name = document_name.strip()
print(clean_name)  # "Normora_Employee_Handbook_2026.PDF"

# Convert to lowercase (essential for case-insensitive search in RAG)
lower_name = clean_name.lower()
print(lower_name)  # "normora_employee_handbook_2026.pdf"

# Replace substrings
updated_name = clean_name.replace(".PDF", ".docx")
print(updated_name)  # "Normora_Employee_Handbook_2026.docx"
```

### 3. The Gold Standard: Formatted String Literals (f-strings)
Before Python 3.6, developers used awkward concatenation like:
```python
# ❌ Clunky, hard to read, and requires manual str() conversions
message = "Tenant " + tenant_name + " has " + str(monthly_active_users) + " users."
```

Modern Python provides **f-strings**. By putting the letter `f` right before the opening quote, you can inject variables and Python expressions directly inside curly braces `{}`:

```python
# ✅ Clean, readable, and lightning-fast
tenant_name = "Acme Corp"
user_count = 120
price_per_user = 12.50

summary = f"Tenant '{tenant_name}' has {user_count} users. Total: ${user_count * price_per_user}"
print(summary)
# Output: Tenant 'Acme Corp' has 120 users. Total: $1500.0
```

### 4. Precision Formatting with f-strings
You can format numbers directly inside f-strings using the `:` specifier:

```python
token_cost = 0.0456789

# Format as currency with exactly 2 decimal places: :.2f
print(f"Estimated Cost: ${token_cost:.2f}")  # Estimated Cost: $0.05

# Format large numbers with commas: :,
total_tokens = 1542000
print(f"Tokens Processed: {total_tokens:,}")  # Tokens Processed: 1,542,000
```

---

## 8. Interactive Input with `input()`

If you want to create an interactive console script that accepts user input, use `input()`.  
`input()` displays a prompt to the user and **always returns the entered value as a string (`str`)**:

```python
# Prompt the user for input
user_name = input("Enter your username: ")
raw_tier = input("Enter your subscription tier (Starter / Enterprise): ")

print(f"Configuring workspace for {user_name} on {raw_tier} plan...")
```

> [!IMPORTANT]
> Because `input()` always returns a string, if you want a number, you **must explicitly convert it**:
> ```python
> raw_seats = input("How many seats do you need? ")
> seat_count = int(raw_seats)  # Convert "5" to 5
> ```

---

## 9. Hands-On Project: The Normora Billing & Token Cost Estimator

Let's synthesize everything we learned in Module 1 by building a practical script for **Normora**.

### Business Scenario
In Normora:
*   Each tenant pays a **base platform fee** per month (e.g., $199.00).
*   Each active user seat costs an additional **$15.00 per month**.
*   Tenants also pay for their AI usage: **$0.002 per 1,000 AI tokens** consumed by their document queries.

We want to build a script that:
1. Stores the tenant's details.
2. Calculates the seat cost and AI token usage cost.
3. Computes the total monthly invoice.
4. Prints a formatted billing breakdown to the terminal.

### The Complete Implementation

Create a file named `billing_estimator.py` and write the following code:

```python
# ============================================================
# Normora Multi-Tenant Platform: Billing & Usage Estimator
# ============================================================

# Step 1: Fixed Pricing Constants
BASE_PLATFORM_FEE = 199.00
PRICE_PER_USER_SEAT = 15.00
PRICE_PER_1K_TOKENS = 0.002

# Step 2: Tenant Configuration
tenant_id = "TENANT-8921"
tenant_name = "HealthCare Dynamics"
active_users = 45
tokens_consumed = 2_750_000
is_in_good_standing = True

# Step 3: Perform Cost Calculations
seats_total_cost = active_users * PRICE_PER_USER_SEAT

# Calculate token cost: (Total Tokens / 1000) * Price per 1K Tokens
token_total_cost = (tokens_consumed / 1000) * PRICE_PER_1K_TOKENS

# Calculate total invoice amount
total_monthly_invoice = BASE_PLATFORM_FEE + seats_total_cost + token_total_cost

# Step 4: Display Formatted Invoice Report
print("=" * 55)
print(f"       NORMORA ENTERPRISE BILLING INVOICE")
print("=" * 55)
print(f"Tenant ID:           {tenant_id}")
print(f"Organization:        {tenant_name}")
print(f"Account Active:      {is_in_good_standing}")
print("-" * 55)
print(f"1. Base Platform Fee:                ${BASE_PLATFORM_FEE:.2f}")
print(f"2. User Seats ({active_users} seats @ $15):     ${seats_total_cost:.2f}")
print(f"3. AI Token Usage ({tokens_consumed:,} tokens): ${token_total_cost:.2f}")
print("-" * 55)
print(f"TOTAL AMOUNT DUE:                    ${total_monthly_invoice:.2f}")
print("=" * 55)
```

### Running the Code & Expected Output

When executed via `python billing_estimator.py`, it outputs:

```text
=======================================================
       NORMORA ENTERPRISE BILLING INVOICE
=======================================================
Tenant ID:           TENANT-8921
Organization:        HealthCare Dynamics
Account Active:      True
-------------------------------------------------------
1. Base Platform Fee:                $199.00
2. User Seats (45 seats @ $15):      $675.00
3. AI Token Usage (2,750,000 tokens): $5.50
-------------------------------------------------------
TOTAL AMOUNT DUE:                    $879.50
=======================================================
```

---

## Module 1 Self-Check & Exercises

Before moving to Module 2, test your understanding by answering these questions:

1. **What is the difference between `/` and `//` in Python?**
2. **Why will `print("Cost: " + 50)` throw a `TypeError`, and how does an f-string solve it?**
3. **If `tokens = "5000"`, why does `tokens * 2` give `'50005000'` instead of `10000`, and how do you fix it?**

When you are comfortable with these concepts, we will move to **Module 2: Conditionals, Booleans & Decision Logic**, where we will teach Python how to make smart decisions (like subscription limits and access control)!
