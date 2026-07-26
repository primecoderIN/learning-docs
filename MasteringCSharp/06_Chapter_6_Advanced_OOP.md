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

## 8. Interview Questions

### Beginner Tier (OOP Fundamentals)

**1. What is Polymorphism?**
*Answer:* Polymorphism means "many shapes." It allows a method to accept a base class (like `Animal`) or interface, but execute the specific, overridden behavior of the concrete derived class (like `Dog.Speak()`) at runtime.

**2. What is the difference between `abstract` and `virtual` methods?**
*Answer:* A `virtual` method provides a default implementation that a derived class *can* override. An `abstract` method has no implementation; it forces the derived class to provide the implementation. `abstract` methods can only exist in `abstract` classes.
*Example:*
```csharp
abstract class Base {
    public virtual void Opt() { /* Default code */ }
    public abstract void Req(); // Must be implemented
}
```

**3. Can a class inherit from multiple classes?**
*Answer:* No. C# does not support multiple inheritance for classes to avoid the "Diamond Problem" (ambiguity in which base method to call). However, a class can implement multiple interfaces.

**4. What does the `sealed` keyword do?**
*Answer:* When applied to a class, it prevents any other class from inheriting from it. When applied to an overridden method, it prevents further derived classes from overriding that specific method.
*Example:*
```csharp
sealed class SecuritySystem { } // Cannot be inherited
```

**5. What is the `base` keyword?**
*Answer:* The `base` keyword is used to access members (methods, properties, or constructors) of the base class from within a derived class.
*Example:*
```csharp
class Child : Parent {
    public Child() : base("Param") { } // Calls Parent constructor
}
```

**6. Explain method overloading vs method overriding.**
*Answer:* Overloading is compile-time polymorphism: having multiple methods with the same name but different parameters in the same class. Overriding is runtime polymorphism: changing the behavior of a base class's `virtual` method in a derived class using the `override` keyword.

**7. What is an Interface?**
*Answer:* An interface is a contract. It defines a set of methods, properties, or events without any implementation (prior to C# 8 default implementations). Any class that implements the interface must provide the code for those members.

### Intermediate Tier (Records and Pattern Matching)

**8. Explain the difference between a `class` and a `record`.**
*Answer:* A `class` uses reference equality by default (two objects are only equal if they point to the exact same memory address). A `record` is a reference type where the compiler automatically generates value-based equality methods. Two records are equal if all their properties contain the same values.

**9. What is a Primary Constructor in a record?**
*Answer:* A concise syntax for declaring a record and its properties on a single line. The compiler automatically generates `init`-only properties for each parameter.
*Example:*
```csharp
public record User(string Name, int Age); 
```

**10. How do you mutate a `record`?**
*Answer:* You don't mutate the original record because its properties are usually `init`-only. You use the `with` expression to create a new, distinct copy of the record with specific properties altered.
*Example:*
```csharp
var u1 = new User("Alice", 30);
var u2 = u1 with { Age = 31 }; // u1 is unchanged
```

**11. What is the `is` operator used for in Pattern Matching?**
*Answer:* It checks if an object is compatible with a specific type. In modern C#, it can also assign the casted value to a new variable in one step.
*Example:*
```csharp
if (obj is string s) {
    Console.WriteLine(s.Length); // 's' is strongly typed
}
```

**12. Show a `switch` expression using Property Pattern Matching.**
*Answer:* You can inspect the properties of an object directly inside the switch arms.
*Example:*
```csharp
string status = user switch {
    { Age: < 18 } => "Minor",
    { IsAdmin: true } => "Admin",
    _ => "Standard"
};
```

**13. What is a Positional Pattern?**
*Answer:* If a type (like a tuple or a record) has a `Deconstruct` method, you can pattern match against the deconstructed values directly.
*Example:*
```csharp
string result = point switch {
    (0, 0) => "Origin",
    (var x, var y) when x == y => "Diagonal",
    _ => "Other"
};
```

**14. What does the `new()` constraint do in Generics?**
*Answer:* It forces the generic type parameter to have a public, parameterless constructor, allowing you to instantiate it inside the generic class.
*Example:*
```csharp
class Factory<T> where T : new() {
    public T Create() => new T();
}
```

### Senior Tier (VTable and Dispatch)

**15. How does Virtual Method Dispatch work under the hood?**
*Answer:* The compiler emits a `callvirt` IL instruction. At runtime, the JIT uses the object's hidden MethodTable pointer to find the Virtual Method Table (VTable). It looks up the specific slot for the overridden method and jumps to that memory address. This pointer indirection prevents inlining.

**16. Why does `callvirt` always include a Null Check?**
*Answer:* Because dynamic dispatch relies on reading the MethodTable pointer located on the object instance in the heap. If the object reference is `null`, there is no MethodTable to read, and the CLR must throw a `NullReferenceException`. Static `call` instructions do not need the object instance to figure out the method address.

**17. What is Interface Dispatch, and why is it slower than Virtual Dispatch?**
*Answer:* Virtual dispatch looks up a slot in a fixed, linear VTable. Interface dispatch is more complex because a class can implement multiple interfaces, and the method might not be in the same VTable slot for different classes implementing the same interface. The CLR must search an interface map (Virtual Stub Dispatch), which requires slightly more CPU overhead.

**18. What is Method Hiding (the `new` keyword on methods)?**
*Answer:* If a derived class declares a method with the same name as a base class method, but uses `new` instead of `override`, it hides the base method. It does *not* participate in polymorphism. The method executed depends on the *compile-time* type of the variable, not the runtime type.
*Example:*
```csharp
class A { public void Go() {} }
class B : A { public new void Go() {} }
A obj = new B();
obj.Go(); // Calls A.Go()!
```

**19. How do Default Interface Methods (C# 8) impact architecture?**
*Answer:* They allow you to add new methods with implementations to an existing interface without breaking backward compatibility for classes that already implement the interface. It enables trait-like behavior but complicates multiple inheritance scenarios.

**20. What is Covariance and Contravariance?**
*Answer:* They allow implicit reference conversions for generic interfaces. Covariance (`out T`) allows a method to return a more derived type (e.g., returning a `Dog` when an `IEnumerable<Animal>` is expected). Contravariance (`in T`) allows a method to accept a less derived type (e.g., passing an `IComparer<Animal>` to sort a list of `Dog`s).

**21. Explain the "Fragile Base Class" problem.**
*Answer:* In deep inheritance hierarchies, changing a seemingly isolated method in a base class can have catastrophic, unforeseen side effects in derived classes that rely on the base class's internal state. This is why architects prefer Composition over Inheritance.

### Staff Engineer Tier (Compiler Internals and DDD)

**22. How does C# pattern matching (e.g., `switch` expressions) compile down into IL?**
*Answer:* Depending on complexity, Roslyn may emit a series of `isinst` (Is Instance) checks followed by conditional branches. If matching against a known closed hierarchy or specific constants, it optimizes it into IL jump tables or binary searches. It is vastly more efficient than manual `if (obj.GetType() == typeof(X))` chains.

**23. What is an exhaustive `switch` expression?**
*Answer:* The compiler analyzes the type being switched on. If it's an `enum`, `bool`, or a closed inheritance hierarchy (like a discriminated union approximation), the compiler guarantees that every possible state is handled. If a state is missed, it throws a compile-time warning (CS8509).

**24. When applying Domain-Driven Design (DDD), how do you decide whether a concept should be an Entity or a Value Object?**
*Answer:* An Entity has a distinct identity that persists over time, even if its properties change (e.g., a `User` changing their name). A Value Object has no conceptual identity; it is defined entirely by its attributes (e.g., a `Money` amount). In C#, Entities are typically encapsulated `class` types, while Value Objects are perfectly modeled as immutable `record` types.

**25. Why are ORMs like EF Core problematic with C# `record` types?**
*Answer:* EF Core Change Tracking relies heavily on object identity (reference equality) and mutability to track which fields were updated. Because records use value equality and are usually immutable (requiring `with` expressions to generate new instances), the ORM loses track of the object instance, making updates difficult or impossible without complex workarounds.

**26. How do you enforce invariants in a DDD Entity without exposing setters?**
*Answer:* You make all property setters `private` or `init`. State changes are only permitted through explicitly named business methods (e.g., `order.ApplyDiscount(10)`) that validate the input and throw exceptions if the invariant is violated, ensuring the object is never put into an invalid state.

**27. Explain the performance impact of Devirtualization.**
*Answer:* Devirtualization is a JIT optimization. If the JIT proves that a class is `sealed`, or that an interface method only has one concrete implementer at runtime, it replaces the expensive `callvirt` (VTable lookup) with a direct `call`. This eliminates pointer indirection and allows the method to be completely inlined into the caller.

### Architect Tier (Extreme Architecture and JIT Tricks)

**28. Virtual Dispatch in high-throughput parsing loops causes CPU stalls. How do you architect around this?**
*Answer:* Use "Constrained Generics" for static polymorphism. Instead of `public void Parse(IParser p)`, use `public void Parse<T>(T p) where T : IParser`. The JIT compiles a specialized, separate native code path for every struct passed as `T`. This completely devirtualizes the interface call, eliminating the VTable lookup and allowing the JIT to inline the parsing logic, achieving C++ speeds.

**29. What is the difference between an Anemic Domain Model and a Rich Domain Model?**
*Answer:* An Anemic Domain Model is an anti-pattern where Entities are just bags of public getters and setters (data), and separate "Service" classes contain all the logic. This violates OOP encapsulation. A Rich Domain Model places the business logic directly inside the Entity itself, protecting its own invariants and state.

**30. How do you emulate Discriminated Unions in C# using Pattern Matching?**
*Answer:* C# does not have native F#-style Discriminated Unions yet. Architects emulate them using an `abstract record` base class with a `private protected` constructor, ensuring no external classes can inherit from it. Then, declare the derived `record` types nested inside or in the same file. This creates a "closed" hierarchy that `switch` expressions can exhaustively match against.

**31. Explain the architectural implications of the `InternalsVisibleTo` attribute.**
*Answer:* To build true microservice boundaries within a monolith (Modular Monolith), the Core Domain should be marked `internal` so the API layer cannot accidentally bypass Application Services and mutate entities directly. `InternalsVisibleTo` is then used exclusively to allow the Unit Test project to bypass this boundary for testing.

**32. Why would an architect forbid the use of `dynamic` in an enterprise C# application?**
*Answer:* The `dynamic` keyword bypasses compile-time type checking and relies on the DLR (Dynamic Language Runtime) to resolve method calls at runtime. This requires heavy reflection, boxing, and caching mechanisms. It breaks static analysis, prevents JIT optimizations, causes severe performance penalties, and masks runtime crashes that should have been caught at compile time.

**33. How does the Liskov Substitution Principle (LSP) constrain your inheritance architecture?**
*Answer:* LSP dictates that a derived class must be seamlessly substitutable for its base class. If a consumer calls `bird.Fly()`, and the injected object is a `Penguin` that throws a `NotSupportedException`, LSP is violated. The architect must restructure the taxonomy, extracting a specific `IFlyingBird` interface rather than forcing a monolithic base class.

**34. What is the Expression Problem, and how do OOP and FP address it differently in C#?**
*Answer:* The Expression Problem is the challenge of extending a system in two dimensions: adding new Data Types or adding new Operations. OOP makes it easy to add new Data Types (just create a new derived class) but hard to add new Operations (must modify the base class and all derived classes). FP (Pattern Matching) makes it easy to add Operations (write a new `switch` statement) but hard to add new Data Types (must update every `switch` statement). A C# architect must choose the right paradigm based on which dimension is more likely to change.

## 9. Summary
Modern C# is a hybrid language. It retains the deep architectural capabilities of Object-Oriented Programming (Classes, Polymorphism, Encapsulation) while embracing the safety and predictability of Functional Programming (Records, Immutability, Pattern Matching). 

As an architect, your job is to apply these tools correctly: use standard classes for rich domain entities that mutate over time, and use records/pattern matching for the data passing between the boundaries of your system. In the next chapter, we will explore Delegates, Events, and the internal state machines that power LINQ.
