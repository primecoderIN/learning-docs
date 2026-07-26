# Chapter 7: Delegates, Events, and LINQ Internals

## 1. Learning Objectives
By the end of this chapter, you will be able to:
- Understand the internal structure of a `Delegate` and `MulticastDelegate`.
- Identify and resolve memory leaks caused by C# `event` subscriptions.
- Analyze the Roslyn-generated state machines behind `IEnumerable` and `yield return`.
- Optimize LINQ queries by understanding Deferred Execution vs. Materialization.

## 2. Introduction

One of the most powerful features of C# is its ability to treat functions as first-class citizens. You can assign a method to a variable, pass it as a parameter, and execute it asynchronously. This is the foundation of event-driven programming, callbacks, and Language Integrated Query (LINQ).

However, delegates and LINQ are heavily abstracted by the compiler. A simple lambda expression `x => x > 5` or a LINQ `.Where()` clause hides significant memory allocations and complex state machines. To write high-performance enterprise applications, you must understand exactly what the compiler is doing behind the scenes.

## 3. Anatomy of a Delegate

In C/C++, you can pass a "function pointer" to another method. It is simply a memory address. It is fast, but it is not type-safe, and it has no concept of object state.

In C#, a **Delegate** is an object-oriented, type-safe wrapper around a function pointer.

When you declare a delegate:
```csharp
public delegate void TelemetryReceivedHandler(string chargerId, decimal voltage);
```

The Roslyn compiler actually generates a completely new class that inherits from `System.MulticastDelegate`.

### Inside `MulticastDelegate`
An instance of a delegate holds two critical pieces of information:
1. `MethodPtr`: A pointer to the JIT-compiled native code of the method to execute.
2. `Target`: A reference to the object instance on which the method should be invoked (if it is an instance method). If the method is `static`, `Target` is null.

Because it holds a `Target` reference, **a delegate acts as a GC Root for the object it points to.** This is the source of many memory leaks in .NET.

### Action and Func
To avoid declaring custom delegates for every signature, .NET provides generic, built-in delegates:
- `Action<T1, T2...>`: Represents a method that returns `void`.
- `Func<T1, T2..., TResult>`: Represents a method that returns a value of type `TResult`.

## 4. Events and Memory Leaks

An `event` in C# is a syntactic wrapper over a `MulticastDelegate`. A `MulticastDelegate` maintains a linked list of delegates (an invocation list). When you trigger the event, the runtime iterates through the list, calling each method sequentially.

```csharp
public class EvNetwork
{
    // The compiler generates a private MulticastDelegate field, 
    // and public add/remove methods.
    public event Action<string> OnChargerDisconnected;

    public void TriggerDisconnect(string id)
    {
        OnChargerDisconnected?.Invoke(id);
    }
}
```

### The Event Memory Leak
Consider this scenario: A long-lived singleton `EvNetwork` exposes an event. A short-lived `DashboardUI` subscribes to it.

```csharp
public class DashboardUI
{
    public DashboardUI(EvNetwork network)
    {
        // Subscription: DashboardUI's method is added to EvNetwork's invocation list.
        network.OnChargerDisconnected += HandleDisconnect;
    }

    private void HandleDisconnect(string id) { /* update UI */ }
}
```

When the user closes the Dashboard, the `DashboardUI` object should be garbage collected. **It will not be.** 

Because `EvNetwork` (a singleton) holds a `MulticastDelegate`, and that delegate holds the `Target` reference to `DashboardUI`, the GC considers `DashboardUI` to be alive. The UI is leaked. If the user opens and closes the dashboard 100 times, 100 UI objects remain in memory forever.

**The Fix:** Always unsubscribe (`-=`) when an object is disposed, or use the Weak Event Pattern (using `WeakReference`).

## 5. Iterators and the State Machine (`yield return`)

In C#, `IEnumerable<T>` is the interface that enables the `foreach` loop. But how do you write a method that returns a sequence of data without loading the entire sequence into memory at once?

C# provides the `yield return` statement.

```csharp
public IEnumerable<int> GeneratePowerSequence()
{
    Console.WriteLine("Generating 10...");
    yield return 10;
    
    Console.WriteLine("Generating 20...");
    yield return 20;
}
```

**Compiler Internals:**
When Roslyn sees `yield return`, it completely rewrites your method. It generates a hidden, private class that implements `IEnumerator<int>`. This class is a **State Machine**.

1. When you call `GeneratePowerSequence()`, *none of your code executes*. It simply instantiates and returns the State Machine object.
2. When the caller calls `.MoveNext()` (e.g., inside a `foreach` loop), the state machine resumes execution from where it left off, runs until the next `yield return`, updates its internal `_state` integer, and pauses again.

This allows you to iterate over massive datasets (like millions of rows from a database) using almost zero memory, because you only hold one item in memory at a time.

## 6. LINQ Internals and Performance

Language Integrated Query (LINQ) is built entirely on Extension Methods, Delegates, and `IEnumerable` State Machines.

```csharp
var chargers = new List<EvCharger> { /* ... */ };

// 1. Deferred Execution
var activeChargers = chargers.Where(c => c.IsOnline).Select(c => c.Id);
```

### Deferred Execution
The code above **does not execute the query**. `.Where()` and `.Select()` simply construct a chain of state machine objects. The iteration only occurs when you force **Materialization**.

### Materialization
You materialize a query by iterating over it (`foreach`) or by calling methods like `.ToList()`, `.ToArray()`, or `.Count()`.

```csharp
// 2. Materialization
List<string> ids = activeChargers.ToList(); 
```

**The Multiple Enumeration Trap:**
```csharp
var activeChargers = chargers.Where(c => c.IsOnline);

int count = activeChargers.Count(); // Executes the Where loop
var first = activeChargers.First(); // Executes the Where loop AGAIN!
```
If that `.Where()` clause involved an expensive database call or heavy computation, you just executed it twice. If you need to use the results multiple times, materialize it to a list first.

### Closure Allocations in LINQ
When you write a lambda expression that captures a local variable, the compiler generates a hidden closure class.

```csharp
string searchRegion = "US-West";

// 'searchRegion' is captured by the lambda!
var results = chargers.Where(c => c.Region == searchRegion);
```
**Internals:** Roslyn creates a new class on the heap to store `searchRegion`, and passes a method from that class to `.Where()`. This causes a heap allocation. In high-performance hot paths, avoid capturing external variables in lambdas.

## 7. Real Production Case Study: Safe Querying

In our EV Platform, we need an administrative endpoint that returns all chargers for a specific tenant, filtered by status. We must ensure we don't accidentally materialize millions of records into RAM.

```csharp
using System;
using System.Collections.Generic;
using System.Linq;

public class ChargerRepository
{
    private readonly List<EvCharger> _databaseMock = new();

    // Returns IEnumerable to allow the caller to compose further queries
    // WITHOUT allocating a new List immediately.
    public IEnumerable<EvCharger> GetChargers(Guid tenantId)
    {
        // Deferred execution. No data is filtered yet.
        return _databaseMock.Where(c => c.TenantId == tenantId);
    }
}

public class ReportingService
{
    private readonly ChargerRepository _repo;

    public ReportingService(ChargerRepository repo)
    {
        _repo = repo;
    }

    public List<string> GenerateFaultReport(Guid tenantId)
    {
        var chargers = _repo.GetChargers(tenantId);

        // We compose more LINQ on top of the original query.
        // It remains deferred until .ToList() is called.
        // When .ToList() executes, the pipeline processes each item one by one,
        // allocating ONLY the final strings to the heap.
        return chargers
            .Where(c => c.Status == ChargerStatus.Faulted)
            .Select(c => c.ChargerId)
            .ToList();
    }
}
```

## 8. Common Mistakes

| Level | Mistake | Impact | How to Avoid |
|-------|---------|--------|--------------|
| Beginner | Modifying a collection inside a `foreach` loop. | `InvalidOperationException: Collection was modified.` | Iterate backwards with a `for` loop, or materialize the items to remove into a separate list first. |
| Intermediate| Multiple Enumeration of `IEnumerable`. | Executing expensive logic or database queries multiple times unnecessarily. | Inspect your code. If you use a LINQ result more than once, append `.ToList()` to cache the results in memory. |
| Senior | Memory leaks via long-lived Event subscriptions. | OutOfMemoryException (OOM) over time. | Implement `IDisposable` on the subscribing class and use `-=` to remove the event handler. |
| Architect | Overusing LINQ in extreme high-throughput parsers. | High GC pressure due to closure allocations and delegate invocations. | In hot paths (e.g., parsing 100k TCP packets/sec), replace LINQ with standard `for` loops and `Span<T>`. |

## 9. Summary
Delegates and Events are the glue that holds loosely-coupled systems together, but they require careful memory management to avoid permanent leaks. LINQ and `yield return` provide an elegant, functional approach to data manipulation, powered entirely by compiler-generated state machines and deferred execution. 

In the next chapter, we will push the compiler even further. We will explore Generics, the heavy runtime costs of Reflection, and how modern C# Source Generators are revolutionizing metaprogramming.
