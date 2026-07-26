# Chapter 6: Advanced Object-Oriented and Functional Paradigms

## 1. Learning Objectives
By the end of this chapter, you will be able to:
- Explain the mechanics of Polymorphism, Virtual Method Dispatch, and VTables in RyuJIT.
- Differentiate between `class`, `struct`, and the modern `record` type, particularly regarding Value Equality.
- Utilize advanced Pattern Matching to write declarative, functional-style code.
- Architect Domain-Driven Design (DDD) entities for enterprise software.

## 2. Introduction

Object-Oriented Programming (OOP) allows us to encapsulate state and behavior, as we saw in Chapter 2. However, encapsulation alone is not enough to build complex, extensible systems. We need mechanisms to abstract behavior (Polymorphism) so that different components can interact without knowing the exact concrete types of their dependencies.

Furthermore, as C# has evolved, Microsoft recognized that strict, mutable OOP is not always the best approach for concurrent, cloud-native systems. Starting heavily with C# 8.0, the language has absorbed numerous concepts from functional programming: immutability by default (Records) and declarative logic (Pattern Matching).

An enterprise architect must know when to use deep inheritance hierarchies and when to favor flat, immutable data structures. More importantly, they must understand the CPU cost of these abstractions.

## 3. Polymorphism, Virtual Dispatch, and VTables

Polymorphism ("many shapes") allows a method to accept an abstract base type, while executing the overridden logic of a concrete derived type at runtime.

### The C# Syntax
```csharp
public abstract class EvCharger
{
    // The 'virtual' or 'abstract' keyword tells the compiler 
    // this method can be overridden.
    public abstract void StartCharge();
}

public class FastDcCharger : EvCharger
{
    public override void StartCharge()
    {
        Console.WriteLine("Initiating 150kW DC Fast Charge...");
    }
}

public class SlowAcCharger : EvCharger
{
    public override void StartCharge()
    {
        Console.WriteLine("Initiating 7kW AC Charge...");
    }
}
```

When we iterate over a list of `EvCharger` objects, we don't need to check their specific types:

```csharp
EvCharger[] chargers = { new FastDcCharger(), new SlowAcCharger() };

foreach (var charger in chargers)
{
    charger.StartCharge(); // How does the CPU know which method to call?
}
```

### Compiler and Runtime Internals: The VTable

If a method is NOT virtual (a standard instance method), the Roslyn compiler emits a `call` IL instruction. The JIT compiler hardcodes the exact memory address of that method into the machine code. This is called **Static Dispatch**, and it is extremely fast.

If a method IS `virtual` or `abstract`, the compiler emits a `callvirt` (Call Virtual) IL instruction. This triggers **Dynamic Dispatch**.

Every type in the CLR has a **MethodTable**. If a class contains virtual methods, its MethodTable includes a **Virtual Method Table (VTable)**—an array of pointers to the actual memory addresses of the overridden methods.

When `charger.StartCharge()` is executed:
1. The CPU reads the hidden MethodTable pointer on the `charger` object instance in the Managed Heap.
2. It navigates to the VTable for that type.
3. It looks up the specific slot for `StartCharge`.
4. It jumps to the memory address stored in that slot.

**Performance Impact:**
Virtual dispatch requires memory indirection (pointer chasing). This breaks CPU instruction pipelining and prevents RyuJIT from inlining the method. While the overhead is measured in nanoseconds, in a tight loop executing millions of times, polymorphism can become a bottleneck. 

*Enterprise Rule:* Do not make methods `virtual` by default unless you explicitly intend for them to be overridden. C# defaults to non-virtual methods for exactly this reason (unlike Java, where methods are virtual by default).

## 4. Records, Primary Constructors, and Value Equality

In distributed systems (like microservices passing JSON over RabbitMQ), data often represents a snapshot in time. This data should be **immutable** (unchangeable after creation). 

Historically, creating immutable classes in C# required massive boilerplate:

```csharp
// Legacy C# Immutable Class
public class TelemetryData
{
    public string ChargerId { get; }
    public decimal Voltage { get; }

    public TelemetryData(string chargerId, decimal voltage)
    {
        ChargerId = chargerId;
        Voltage = voltage;
    }

    // You also had to manually override Equals(), GetHashCode(), and == operator!
}
```

### Enter the `record` Type (C# 9+)

A `record` is a reference type (like a `class`), but the compiler automatically generates the boilerplate for immutability and **Value Equality**.

```csharp
// Modern C# with Primary Constructors
public record TelemetryData(string ChargerId, decimal Voltage);
```

That single line of code instructs Roslyn to generate:
1. Public init-only properties for `ChargerId` and `Voltage`.
2. A constructor taking both parameters.
3. Overrides for `Equals()` and `GetHashCode()` that compare the *values* of the properties, rather than the memory address of the object.
4. A custom `ToString()` implementation.
5. Support for non-destructive mutation via the `with` expression.

```csharp
var t1 = new TelemetryData("CHG-1", 240m);
var t2 = new TelemetryData("CHG-1", 240m);

// True! Because Records use Value Equality. 
// If these were standard Classes, this would be False.
bool areEqual = (t1 == t2); 

// Non-destructive mutation (creates a new object)
var t3 = t1 with { Voltage = 230m };
```

*Note: C# 10 introduced `record struct`, combining the heap-allocation-free nature of structs with the synthesized equality and `with` expressions of records.*

## 5. Pattern Matching

Functional programming relies heavily on declarative data transformations. C# has rapidly expanded its `switch` expressions and pattern matching capabilities.

Instead of writing deep `if-else` blocks or using polymorphic visitors, we can use pattern matching to inspect object shapes and properties.

```csharp
public record PaymentRequest(decimal Amount, string Currency);
public record CreditCardPayment(decimal Amount, string Currency, string Token) : PaymentRequest(Amount, Currency);
public record RfidPayment(decimal Amount, string Currency, string RfidTag) : PaymentRequest(Amount, Currency);

public class PaymentProcessor
{
    // C# 8+ Switch Expression with Pattern Matching
    public string Process(PaymentRequest request) => request switch
    {
        CreditCardPayment { Amount: > 1000 } => "Transaction too large for card.",
        CreditCardPayment card => $"Processing card token: {card.Token}",
        RfidPayment rfid when rfid.Currency == "USD" => $"Processing RFID {rfid.RfidTag}",
        null => throw new ArgumentNullException(nameof(request)),
        _ => "Unknown payment method" // Discard pattern
    };
}
```

**Compiler Internals:** 
The Roslyn compiler is incredibly smart about pattern matching. It will analyze the type hierarchy and generate an optimized IL jump table or a sequence of `isinst` (Is Instance) IL checks. Furthermore, if you use enums or closed type hierarchies, the compiler can issue a warning if your switch expression is not exhaustive (meaning you missed a possible state).

## 6. Real Production Case Study: Domain-Driven Design (DDD)

In our EV Platform, we need to model a `ChargingSession`. In a highly transactional enterprise system, we use **Domain-Driven Design (DDD)**. 

Entities must protect their invariants. A session cannot be stopped if it was never started. It cannot exceed the max kW of the charger. We will use advanced OOP encapsulation, and Records for immutable Value Objects.

```csharp
using System;
using System.Collections.Generic;

namespace EVPlatform.Domain
{
    // Value Object: Identifiable only by its data, completely immutable.
    public record MeterValue(decimal KwDelivered, DateTime Timestamp);

    // Domain Entity: Has a unique identity, mutable state, but protects its invariants.
    public class ChargingSession
    {
        public Guid SessionId { get; }
        public string ChargerId { get; }
        
        // Private state
        private readonly List<MeterValue> _meterValues = new();
        private bool _isCompleted;

        // Expose a read-only view to prevent external modification
        public IReadOnlyList<MeterValue> MeterValues => _meterValues.AsReadOnly();

        public decimal TotalKw => _meterValues.Sum(m => m.KwDelivered);

        public ChargingSession(string chargerId)
        {
            SessionId = Guid.NewGuid();
            ChargerId = chargerId ?? throw new ArgumentNullException(nameof(chargerId));
            _isCompleted = false;
        }

        // Domain Behavior
        public void RecordMeter(MeterValue value)
        {
            if (_isCompleted)
                throw new InvalidOperationException("Session is already completed.");

            if (value.Timestamp > DateTime.UtcNow)
                throw new ArgumentException("Meter timestamp cannot be in the future.");

            _meterValues.Add(value);
        }

        public void CompleteSession()
        {
            _isCompleted = true;
        }
    }
}
```

## 7. Common Mistakes

| Level | Mistake | Impact | How to Avoid |
|-------|---------|--------|--------------|
| Beginner | Misunderstanding `class` equality. | `obj1 == obj2` returns false even if their properties are identical, leading to bugs in HashSets and Dictionaries. | Use `record` for data-transfer objects where value equality is expected. |
| Intermediate | Using `record` for EF Core Entities. | EF Core Change Tracking relies on reference mutability. Records cause massive headaches with ORMs. | Use standard `class` types for EF Core Entities. Use `record` for DTOs and Value Objects. |
| Senior | Deep Inheritance Hierarchies (>3 levels). | Brittle architecture, impossible to unit test effectively. | Favor Composition over Inheritance. Extract shared behavior into interfaces or composed dependencies. |
| Architect | Virtual Dispatch in high-throughput parsing loops. | CPU stall and cache misses. | Use sealed classes, static methods, or generic constraints (`where T : IParser`) which the JIT can devirtualize. |

## 8. Summary
Modern C# is a hybrid language. It retains the deep architectural capabilities of Object-Oriented Programming (Classes, Polymorphism, Encapsulation) while embracing the safety and predictability of Functional Programming (Records, Immutability, Pattern Matching). 

As an architect, your job is to apply these tools correctly: use standard classes for rich domain entities that mutate over time, and use records/pattern matching for the data passing between the boundaries of your system. In the next chapter, we will explore Delegates, Events, and the internal state machines that power LINQ.
