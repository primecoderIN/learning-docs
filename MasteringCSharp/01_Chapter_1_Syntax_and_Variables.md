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

## 9. Interview Questions

### Beginner Tier (Syntax and Basics)

**1. What is the difference between declaring a variable with a specific type versus using the `var` keyword?**
*Answer:* Declaring a specific type (e.g., `int x = 5;`) is explicit. Using `var` allows the compiler to infer the type based on the right-hand side of the assignment. `var` is statically typed; it is resolved at compile-time and cannot change type later.
*Example:*
```csharp
int explicitInt = 5; 
var inferredInt = 5; // Compiler knows this is an int
// inferredInt = "Hello"; // Compiler Error!
```

**2. What is a Top-Level Statement in C# 9+?**
*Answer:* It is a feature that allows you to write executable code without explicitly wrapping it in a `namespace`, `class`, and `static void Main` method. The compiler generates this boilerplate behind the scenes.
*Example:*
```csharp
// Just this one line is a valid C# program now:
Console.WriteLine("Hello, World!");
```

**3. Explain the difference between `==` and `=` in C#.**
*Answer:* `=` is the assignment operator (assigning a value to a variable). `==` is the equality operator (checking if two values are equal, returning a boolean).
*Example:*
```csharp
int x = 10; // Assignment
bool isTen = (x == 10); // Equality check (returns true)
```

**4. What is the purpose of the `char` data type?**
*Answer:* `char` represents a single 16-bit Unicode character. It is enclosed in single quotes, unlike a `string` which uses double quotes.
*Example:*
```csharp
char grade = 'A';
string name = "Alice";
```

**5. How do you write a single-line comment and a multi-line comment?**
*Answer:* Single-line uses `//`. Multi-line uses `/*` to start and `*/` to end.
*Example:*
```csharp
// This is a single line comment
/* 
   This is a 
   multi-line comment 
*/
```

**6. What is a `string` in C#?**
*Answer:* A `string` is a sequence of characters used to represent text. In C#, `string` is a Reference Type (allocated on the heap) and is immutable.
*Example:*
```csharp
string greeting = "Hello";
greeting += " World"; // Creates a brand new string object in memory
```

**7. How does a `while` loop differ from a `do-while` loop?**
*Answer:* A `while` loop checks its condition *before* executing the block. A `do-while` loop checks its condition *after* executing the block, guaranteeing the code runs at least once.
*Example:*
```csharp
int count = 0;
do 
{
    Console.WriteLine("Runs at least once!");
} while (count > 0);
```

### Intermediate Tier (Operators and Flow Control)

**8. Explain short-circuit evaluation in C#.**
*Answer:* In logical operations like `&&` or `||`, if the first operand determines the final result, the runtime skips evaluating the second operand.
*Example:*
```csharp
string user = null;
// The second part is NEVER executed because user != null is false.
// This prevents a NullReferenceException!
if (user != null && user.Length > 0) { }
```

**9. What is the difference between `break` and `continue`?**
*Answer:* `break` terminates the loop entirely. `continue` skips the rest of the current iteration and jumps to the next loop evaluation.
*Example:*
```csharp
for (int i = 0; i < 5; i++) {
    if (i == 2) continue; // Skips printing 2
    if (i == 4) break;    // Stops the loop completely
    Console.WriteLine(i); // Prints 0, 1, 3
}
```

**10. Why should you use `decimal` instead of `double` for financial calculations?**
*Answer:* `double` uses Base-2 binary math, which cannot accurately represent certain fractions (like 0.1), leading to rounding errors. `decimal` uses Base-10 math with 128-bit precision, making it perfectly accurate for money.
*Example:*
```csharp
double d1 = 0.1, d2 = 0.2;
Console.WriteLine(d1 + d2 == 0.3); // Prints FALSE!

decimal m1 = 0.1m, m2 = 0.2m;
Console.WriteLine(m1 + m2 == 0.3m); // Prints TRUE!
```

**11. What is a ternary operator?**
*Answer:* It is a shorthand `if-else` statement that returns a value based on a boolean condition. Syntax: `condition ? trueValue : falseValue`.
*Example:*
```csharp
int temp = 30;
string weather = temp > 25 ? "Hot" : "Cold";
```

**12. Explain the `switch` expression introduced in C# 8.**
*Answer:* It is a more concise, functional version of the `switch` statement that directly evaluates to a value, avoiding `break` statements and reducing boilerplate.
*Example:*
```csharp
int day = 3;
string dayName = day switch
{
    1 => "Monday",
    2 => "Tuesday",
    _ => "Unknown" // Default discard pattern
};
```

**13. What is the `const` keyword?**
*Answer:* It declares a constant field or local variable. The value must be known at compile-time and cannot be modified at runtime.
*Example:*
```csharp
const double Pi = 3.14159;
// Pi = 3.14; // Compiler Error!
```

**14. What is the null-coalescing operator (`??`)?**
*Answer:* It returns the left-hand operand if it is not null; otherwise, it evaluates and returns the right-hand operand.
*Example:*
```csharp
string input = null;
string result = input ?? "Default Name"; 
// result is "Default Name"
```

### Senior Tier (Compiler and Runtime Behaviors)

**15. How does the compiler handle String Interpolation (`$""`) internally?**
*Answer:* Prior to C# 10, it compiled to a `String.Format()` call, which caused boxing for value types. In C# 10+, it compiles into a `DefaultInterpolatedStringHandler` which uses `Span<char>` to construct the string with zero boxing allocations.
*Example:*
```csharp
int age = 30;
// C# 10+ compiles this into a highly optimized struct handler
string msg = $"I am {age} years old."; 
```

**16. What is the `readonly` keyword, and how does it differ from `const`?**
*Answer:* `const` is a compile-time constant baked directly into the IL of the calling assembly. `readonly` is a runtime constant; its value can be assigned in a constructor or inline, and the reference cannot be changed thereafter.
*Example:*
```csharp
public class Config {
    public const string Version = "1.0"; // Baked in
    public readonly DateTime StartTime = DateTime.Now; // Evaluated at runtime
}
```

**17. What happens if an arithmetic operation overflows an `int`?**
*Answer:* By default, C# silently overflows (wraps around to negative numbers) for performance reasons. You can catch this by wrapping the math in a `checked` block, which throws an `OverflowException`.
*Example:*
```csharp
int max = int.MaxValue;
int next = max + 1; // Evaluates to -2147483648
checked {
    // int safe = max + 1; // Throws OverflowException
}
```

**18. Explain how a `foreach` loop works under the hood.**
*Answer:* The compiler translates a `foreach` loop into a `while` loop that calls `.GetEnumerator()` on the collection, then calls `.MoveNext()` and accesses `.Current` on the returned state machine.
*Example:*
```csharp
// You write: foreach(var x in list) { }
// Compiler generates:
using (var e = list.GetEnumerator()) {
    while (e.MoveNext()) { var x = e.Current; }
}
```

**19. What is the `nameof` operator and why is it useful?**
*Answer:* It returns the string name of a variable, type, or member at compile time. It survives refactoring (if you rename the variable, the string updates), preventing magic string bugs.
*Example:*
```csharp
public void DoWork(string data) {
    if (data == null) 
        throw new ArgumentNullException(nameof(data)); // Better than "data"
}
```

**20. What is a Tuple in modern C#?**
*Answer:* Tuples (`ValueTuple`) are lightweight structures that allow you to group multiple data elements and return multiple values from a method without defining a formal `class`. They are value types stored on the stack.
*Example:*
```csharp
(string Name, int Age) GetPerson() => ("Alice", 30);

var person = GetPerson();
Console.WriteLine(person.Name);
```

**21. Explain the `using` declaration (C# 8) vs the `using` block.**
*Answer:* The `using` block requires curly braces and indents the code. The `using` declaration is a local variable declaration that automatically calls `Dispose` when the variable goes out of scope at the end of the method, flattening the code.
*Example:*
```csharp
void ReadFile() {
    using var stream = new StreamReader("file.txt");
    // stream.Dispose() is automatically called at the closing bracket of this method
}
```

### Staff Engineer Tier (IL, Memory, and Architecture)

**22. Describe exactly how the Roslyn compiler lowers a large `switch` statement.**
*Answer:* For contiguous integer cases, Roslyn emits a `switch` IL instruction which is an O(1) jump table. It calculates the memory offset and jumps directly to the case block. For string switches, it emits a highly optimized Dictionary lookup or a binary search, avoiding O(n) `if/else` branching.
*Example:*
```csharp
// If day is between 1 and 100, the compiler calculates the 
// exact memory address to jump to instantly.
switch (day) {
    case 1: break; 
    case 50: break;
}
```

**23. What are the architectural consequences of standard string manipulation (`+`, `.Substring()`) in a high-throughput API?**
*Answer:* Strings are immutable reference types. Using `+` or `Substring` allocates a brand new object on the Managed Heap. At massive scale (10,000 req/sec), this triggers Gen 0 Garbage Collection constantly, starving the CPU. Staff Engineers mandate `ReadOnlySpan<char>` for slicing and `StringBuilder` for concatenation to eliminate heap allocation.
*Example:*
```csharp
string url = "https://api.com/v1/users";
// Bad: Allocates a new string
string id = url.Substring(21); 
// Good: Zero allocation window into existing memory
ReadOnlySpan<char> fastId = url.AsSpan().Slice(21); 
```

**24. How does `string.Intern()` work and when should you use it?**
*Answer:* The CLR maintains an Intern Pool (a global hash table of strings). `string.Intern()` checks if a string exists in the pool; if so, it returns the global reference. It is useful when parsing millions of JSON objects that share identical small strings (like "status", "active"), saving massive amounts of RAM. However, interned strings are *never* garbage collected.
*Example:*
```csharp
string a = new string(new char[] {'h','i'});
string b = new string(new char[] {'h','i'});
// a == b is true (value equality)
// ReferenceEquals(a, b) is false.
string c = string.Intern(a);
```

**25. Explain the `stackalloc` keyword.**
*Answer:* It allocates a block of memory directly on the execution stack rather than the heap. This memory is automatically discarded when the method returns, making it zero-cost for the Garbage Collector. It is heavily used in high-performance parsing.
*Example:*
```csharp
// Allocates 128 bytes on the stack instantly
Span<byte> buffer = stackalloc byte[128]; 
```

**26. What is a `BitConverter` and what architectural issues arise regarding Endianness?**
*Answer:* `BitConverter` converts base data types to byte arrays. x86/x64 CPUs are Little-Endian (least significant byte first), but Network protocols (TCP/IP) are Big-Endian. A Staff Engineer must use `BinaryPrimitives.ReadInt32BigEndian` when parsing network packets to avoid hardware-dependent bugs.
*Example:*
```csharp
byte[] packet = { 0x00, 0x00, 0x00, 0x01 };
// Safely reads as 1 regardless of CPU architecture
int value = BinaryPrimitives.ReadInt32BigEndian(packet);
```

**27. What are the performance implications of the `dynamic` keyword?**
*Answer:* `dynamic` bypasses compile-time type checking. At runtime, the DLR (Dynamic Language Runtime) uses Reflection and emit caching to resolve the method call. It is incredibly slow, causes boxing, and breaks NativeAOT compilation. It should be avoided in modern C# in favor of strongly-typed generics or `System.Text.Json` nodes.
*Example:*
```csharp
dynamic obj = GetJson();
// The compiler has no idea if 'Name' exists until runtime.
Console.WriteLine(obj.Name); 
```

### Architect Tier (Language Evolution and System Design)

**28. Why did Microsoft introduce Top-Level statements, and how does it affect the Startup lifecycle of an enterprise app?**
*Answer:* It lowers the barrier to entry for beginners and scripts. Architecturally, it streamlines the `Program.cs` file in ASP.NET Core microservices, shifting the focus entirely to Dependency Injection and Middleware configuration. The compiler still generates `<Program>.<Main>`, meaning lifecycle hooks (`AppDomain.CurrentDomain.ProcessExit`) still function identically.
*Example:*
```csharp
// Program.cs in modern ASP.NET Core
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();
app.Run();
```

**29. You are designing a custom rules engine. Why would you compile C# expression trees dynamically instead of using `switch` statements?**
*Answer:* A `switch` is static; it must be compiled into the binary. A rules engine (e.g., executing user-defined discounts) requires dynamic logic at runtime. By building `System.Linq.Expressions.Expression` trees and compiling them to `Func<T>`, the CLR JIT-compiles the dynamic rule into native machine code in memory, offering near-native execution speed without recompiling the application.
*Example:*
```csharp
ParameterExpression param = Expression.Parameter(typeof(int), "x");
BinaryExpression body = Expression.GreaterThan(param, Expression.Constant(10));
var isGreaterThanTen = Expression.Lambda<Func<int, bool>>(body, param).Compile();
```

**30. How do floating-point rounding errors impact distributed financial systems, and how do you architect around it?**
*Answer:* If Service A uses `double` and Service B uses `decimal`, an invoice total might serialize as `10.0000000000001` in JSON. When hashed for a digital signature or idempotency key, the hash changes, failing the transaction. Architects mandate `decimal` for all financial properties and strict JSON serialization converters to truncate/round to physical currency limits (e.g., 2 decimal places) before transit.

**31. Explain the architectural use case for `ref` locals and `ref` returns.**
*Answer:* In extreme high-performance scenarios (like game engines or Kestrel networking), copying large structs from arrays to local variables consumes CPU cache bandwidth. `ref` returns allow a method to return a pointer directly to an array element. Modifying the `ref` local modifies the underlying array instantly without copying data.
*Example:*
```csharp
public ref int GetReferenceToElement(int[] array, int index) {
    return ref array[index];
}
```

**32. What is the difference between `checked` and `unchecked` contexts globally vs locally?**
*Answer:* By default, C# compiles in an `unchecked` context for performance. If an integer overflows, it silently wraps. You can enable global checking via `<CheckForOverflowUnderflow>true</CheckForOverflowUnderflow>` in the `.csproj`, but this adds CPU instructions to every math operation. Architects prefer local `checked { }` blocks strictly around critical financial or security calculations to balance safety and performance.

**33. How does the choice between `struct` and `class` impact L1/L2 CPU Cache utilization?**
*Answer:* Classes are reference types. An array of classes (`Customer[]`) is actually an array of pointers pointing to scattered memory addresses on the heap. Iterating this array causes constant CPU cache misses. An array of structs (`Vector3[]`) stores the data sequentially in continuous memory. The CPU pre-fetcher loads the entire array into the L1 cache, making iteration orders of magnitude faster.

**34. Describe how `readonly struct` and `in` parameters solve defensive copying.**
*Answer:* When you pass a `struct` to a method using `in`, it is passed by readonly-reference (pointer), saving the copy overhead. However, if the struct is not marked `readonly`, the compiler fears the method might mutate it, so it makes a "defensive copy" anyway, ruining the optimization. Architects strictly enforce `readonly struct` for DTOs to guarantee zero-allocation, pointer-based parameter passing.
*Example:*
```csharp
public readonly struct Point { public int X { get; } }
// Passed by 8-byte pointer, guaranteed no defensive copy!
public void Calculate(in Point p) { } 
```

## 10. Summary
In this chapter, we covered the foundational syntax of C#. We explored Top-Level Statements, strongly-typed variables, operators, and control flow mechanisms. More importantly, we peeked behind the curtain to understand how the compiler handles type inference (`var`) and optimizes branching (`switch`). In the next chapter, we will introduce Object-Oriented Programming, moving beyond simple scripts to architectural building blocks: Classes, Objects, and Methods.
