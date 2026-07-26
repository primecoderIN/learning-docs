# Chapter 1: C# Syntax, Variables, and Control Flow

## 1. Learning Objectives
By the end of this chapter, you will be able to:
- Structure a modern C# application using Top-Level Statements.
- Declare variables and understand the underlying type system.
- Understand exactly how the `var` keyword is processed by the Roslyn compiler.
- Write robust expressions and statements.
- Implement control flow and understand how `switch` statements compile into optimized IL jump tables.

## 2. Introduction

Every software architect must start at the foundation. While C# is an incredibly deep and powerful language, its syntax is designed to be accessible, sharing a lineage with C, C++, and Java. 

However, in this book, we do not settle for "just getting it to work." Even the most basic features of C#—like declaring a variable or writing a loop—have profound implications on how the compiler generates Intermediate Language (IL) and how the runtime executes your code.

In this chapter, we will build the first component of our **Multi-Tenant EV Charging Platform**: a Command Line Interface (CLI) tool used by administrators to configure new chargers. Through this practical example, we will master the foundational syntax of C#.

## 3. Hello World: Modern C# Structure

Historically, a C# application required a significant amount of boilerplate code: a namespace, a class, and a static `Main` method. 

```csharp
// Legacy C# Structure (C# 1.0 - 8.0)
using System;

namespace EVPlatform.Cli
{
    class Program
    {
        static void Main(string[] args)
        {
            Console.WriteLine("EV Platform Booting...");
        }
    }
}
```

Since C# 9.0, Microsoft introduced **Top-Level Statements**, eliminating the boilerplate. The compiler automatically generates the namespace, the `Program` class, and the `Main` method for you behind the scenes.

```csharp
// Modern C# Structure (C# 9.0+)
using System;

Console.WriteLine("EV Platform Booting...");
```

**What happens internally?**
When the Roslyn compiler processes a file with Top-Level Statements, it dynamically emits a `<Program>$` class with a `<Main>$` method. This means you still have full access to command-line arguments via the implicit `args` array, but your code remains remarkably clean.

## 4. Variables, Data Types, and the `var` Keyword

C# is a **statically-typed**, **strongly-typed** language. This means every variable must have a known type at compile-time, and you cannot accidentally assign a string to an integer variable.

### Basic Data Types

C# supports several primitive data types, which map directly to underlying .NET Base Class Library (BCL) types.

| C# Keyword | .NET BCL Type | Size | Example Use Case |
|------------|---------------|------|------------------|
| `int`      | `System.Int32` | 32-bit | Loop counters, IDs |
| `long`     | `System.Int64` | 64-bit | High-precision timestamps, large IDs |
| `double`   | `System.Double`| 64-bit | Scientific calculations |
| `decimal`  | `System.Decimal`| 128-bit| Financial calculations (Tariffs, Pricing) |
| `bool`     | `System.Boolean`| 8-bit | True/False flags |
| `char`     | `System.Char`  | 16-bit | Single Unicode character |
| `string`   | `System.String`| Ref Type| Text data (e.g., "Charger-X1") |

### Variable Declaration

You declare a variable by specifying its type, its name, and optionally its initial value.

```csharp
int maxCurrentAmp = 32;
decimal kwPrice = 0.45m; // The 'm' suffix specifies a decimal literal
string chargerModel = "EV-Max-Pro";
bool isOnline = true;
```

### The Power of `var` (Compiler Type Inference)

C# 3.0 introduced the `var` keyword. A common beginner mistake is assuming `var` makes C# dynamically typed like JavaScript or Python. **It does not.**

```csharp
var maxVoltage = 240; // The compiler infers 'int'
var firmwareVersion = "v1.2.4"; // The compiler infers 'string'
```

When you use `var`, the Roslyn compiler inspects the right side of the assignment during the **Semantic Analysis** phase. It binds the type permanently. If you try to write `maxVoltage = "High";` on the next line, the compiler will throw an error (`CS0029: Cannot implicitly convert type 'string' to 'int'`).

**Enterprise Best Practice:** Use `var` when the type is obvious from the right-hand side of the assignment (e.g., `var list = new List<string>();`). If the return type of a method is unclear, explicitly declare the type to improve code readability for your fellow engineers.

## 5. Operators, Expressions, and Statements

An **expression** is code that evaluates to a value (e.g., `5 + 5`). A **statement** is a complete instruction executed by the program, usually ending in a semicolon (`;`).

### Arithmetic and Relational Operators
C# supports standard arithmetic (`+`, `-`, `*`, `/`, `%`) and relational (`==`, `!=`, `<`, `>`, `<=`, `>=`) operators.

```csharp
int a = 10;
int b = 3;
int sum = a + b; // 13
bool isGreater = a > b; // true
```

### Logical Operators
Used for boolean logic: `&&` (AND), `||` (OR), `!` (NOT).

```csharp
bool hasPower = true;
bool isAuthorized = false;
bool canCharge = hasPower && isAuthorized; // false
```

**Short-Circuit Evaluation:**
If you write `if (condition1 && condition2)`, and `condition1` is false, the CLR will *never* evaluate `condition2`. This is critical for performance and null-checking: `if (user != null && user.IsAdmin)`.

## 6. Control Flow

Control flow dictates the order in which statements are executed. 

### The `if` Statement
```csharp
int batteryPercentage = 85;

if (batteryPercentage >= 100)
{
    Console.WriteLine("Fully charged.");
}
else if (batteryPercentage > 20)
{
    Console.WriteLine("Charging in progress.");
}
else
{
    Console.WriteLine("Low battery warning!");
}
```

### The `switch` Statement and Jump Tables

The `switch` statement is used when you have multiple discrete paths based on a single variable.

```csharp
string command = "start";

switch (command)
{
    case "start":
        Console.WriteLine("Initiating charge session.");
        break;
    case "stop":
        Console.WriteLine("Ending charge session.");
        break;
    case "reboot":
        Console.WriteLine("Restarting charger.");
        break;
    default:
        Console.WriteLine("Unknown command.");
        break;
}
```

**Compiler Internals:** Why use `switch` instead of a long chain of `if-else`? 
If you have a large `switch` statement (e.g., on an `enum` or integer with many contiguous cases), the Roslyn compiler translates it into a **Jump Table** (using the `switch` IL instruction). 

Instead of evaluating every condition one by one (O(n) time), the CPU calculates an memory offset and jumps directly to the correct case (O(1) time). This makes `switch` statements extremely fast for state machines and parsers.

### Loops (`while`, `for`, `foreach`)

Loops allow you to execute a block of code multiple times.

**`while` Loop:** Executes as long as a condition is true.
```csharp
int retries = 0;
while (retries < 3)
{
    Console.WriteLine($"Attempt {retries + 1} to connect...");
    retries++;
}
```

**`for` Loop:** Best when you know exactly how many iterations you need.
```csharp
for (int i = 0; i < 5; i++)
{
    Console.WriteLine($"Pinging charger port {i}");
}
```

**`foreach` Loop:** The most common loop in enterprise C#, used to iterate over collections.
```csharp
string[] chargers = { "CHG-01", "CHG-02", "CHG-03" };

foreach (string charger in chargers)
{
    Console.WriteLine($"Checking status of {charger}");
}
```
*Note: Under the hood, `foreach` compiles into a state machine that calls `.GetEnumerator()` on the collection, allowing it to safely iterate through arrays, Lists, and database query results.*

## 7. Real Production Case Study: EV Platform CLI

Let's combine these fundamentals to build the mock initialization routine of our EV Charger Configuration CLI.

```csharp
using System;

Console.WriteLine("=== EV Charger Admin CLI ===");

// Variable Initialization
var maxSupportedKw = 150m;
var isConfigured = false;
var retryCount = 0;
var maxRetries = 3;

// Control flow: Loop until configured or max retries hit
while (!isConfigured && retryCount < maxRetries)
{
    Console.Write("Enter Command (init, status, quit): ");
    var input = Console.ReadLine();

    switch (input)
    {
        case "init":
            Console.WriteLine($"Initializing charger at {maxSupportedKw} kW...");
            isConfigured = true;
            Console.WriteLine("Success: Charger is ready.");
            break;

        case "status":
            Console.WriteLine("Charger is currently unconfigured.");
            break;

        case "quit":
            Console.WriteLine("Exiting CLI.");
            retryCount = maxRetries; // Force loop exit
            break;

        default:
            Console.WriteLine("Invalid command.");
            retryCount++;
            Console.WriteLine($"Retries left: {maxRetries - retryCount}");
            break;
    }
}

if (!isConfigured)
{
    Console.WriteLine("CRITICAL: Failed to configure charger.");
}
```

## 8. Common Mistakes

| Level | Mistake | Impact | How to Avoid |
|-------|---------|--------|--------------|
| Beginner | Confusing `=` (assignment) and `==` (equality). | Logic bugs, syntax errors. | Remember `=` assigns a value, `==` compares two values. |
| Beginner | Misunderstanding `var`. | Fear of dynamic typing in a static language. | Remember `var` is strictly resolved at compile-time by Roslyn. |
| Intermediate | Using `double` for money/pricing. | Precision loss due to floating-point math (e.g., `0.1 + 0.2 != 0.3`). | Always use `decimal` for financial calculations. |
| Senior | Huge `if/else` chains instead of `switch`. | Degraded CPU performance due to sequential branching. | Use `switch` to allow the compiler to generate O(1) jump tables. |

## 9. Summary
In this chapter, we covered the foundational syntax of C#. We explored Top-Level Statements, strongly-typed variables, operators, and control flow mechanisms. More importantly, we peeked behind the curtain to understand how the compiler handles type inference (`var`) and optimizes branching (`switch`). In the next chapter, we will introduce Object-Oriented Programming, moving beyond simple scripts to architectural building blocks: Classes, Objects, and Methods.
