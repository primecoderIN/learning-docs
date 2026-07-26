# Chapter 2: Object-Oriented Foundations

## 1. Learning Objectives
By the end of this chapter, you will be able to:
- Define Classes and instantiate Objects.
- Understand the difference between Fields and Properties, and how properties compile to hidden methods.
- Pass parameters to methods and understand passing by value.
- Apply Encapsulation using access modifiers (`public`, `private`, `internal`).
- Organize code using Namespaces.

## 2. Introduction

In Chapter 1, we wrote imperative code: a series of statements executing top-to-bottom. While this is fine for a simple script, enterprise software is vastly more complex. A system like our EV Charging Platform must manage Tenants, Users, Chargers, Sessions, and Invoices. 

To manage this complexity, C# heavily utilizes **Object-Oriented Programming (OOP)**. OOP allows us to model real-world concepts as code constructs. It groups related data (state) and behavior (logic) into a single, cohesive unit called a **Class**.

However, we will not just learn the syntax. We will examine how the Roslyn compiler transforms these high-level architectural concepts into lower-level instructions.

## 3. Classes and Objects

A **Class** is a blueprint. It defines what data an entity holds and what actions it can perform.
An **Object** (or instance) is a specific realization of that blueprint, occupying physical memory in the application.

```csharp
// The Blueprint (Class)
public class EvCharger
{
    // Fields (State)
    public string ChargerId;
    public bool IsOnline;

    // Method (Behavior)
    public void Reboot()
    {
        IsOnline = false;
        Console.WriteLine($"Charger {ChargerId} is rebooting...");
        IsOnline = true;
    }
}
```

To use this blueprint, we must instantiate an object using the `new` keyword.

```csharp
// Instantiating the Object
EvCharger chargerA = new EvCharger();
chargerA.ChargerId = "CHG-001";
chargerA.IsOnline = true;

// Executing behavior
chargerA.Reboot(); 
```

**Compiler Insight:** When you call `new EvCharger()`, the CLR allocates a block of memory on the Managed Heap, initializes all fields to their default values (e.g., `false` for booleans, `null` for strings), and returns a memory address (a reference) pointing to that object.

## 4. Constructors

When an object is created, you often need to ensure it starts in a valid state. A **Constructor** is a special method that is called exactly once, immediately upon object creation.

```csharp
public class EvCharger
{
    public string ChargerId;
    public decimal MaxKw;

    // Constructor: Must have the exact same name as the class
    public EvCharger(string id, decimal maxKw)
    {
        ChargerId = id;
        MaxKw = maxKw;
    }
}

// Now you MUST provide the data when creating the object
EvCharger myCharger = new EvCharger("CHG-002", 150m);
```

## 5. Encapsulation and Access Modifiers

A core pillar of OOP is **Encapsulation**: the practice of hiding the internal state and requiring all interaction to be performed through explicitly defined methods. 

If anyone can modify `chargerA.IsOnline = true;` directly, the object cannot guarantee its own validity. We enforce encapsulation using **Access Modifiers**.

| Modifier | Visibility |
|----------|------------|
| `public` | Code anywhere in the application can access this. |
| `private`| Code *only* inside the same class can access this. |
| `internal`| Code anywhere within the same Assembly (.dll/.exe) can access this. |
| `protected`| Used in inheritance (covered later). |

Let's secure our class:

```csharp
public class EvCharger
{
    // The state is now hidden (Encapsulated)
    private string _chargerId;
    private bool _isOnline;

    public EvCharger(string id)
    {
        _chargerId = id;
        _isOnline = false; // Default state
    }

    // Controlled access method
    public void BringOnline()
    {
        // Business logic can be enforced here
        Console.WriteLine($"Connecting {_chargerId} to network...");
        _isOnline = true;
    }
}
```

## 6. Fields vs. Properties

In C#, a **Field** is a raw variable declared inside a class (like `private bool _isOnline;`). 

However, writing explicit `GetStatus()` and `SetStatus()` methods for every piece of data is tedious. To solve this, C# introduced **Properties**. A Property looks like a field when you use it, but acts like a method under the hood.

```csharp
public class EvCharger
{
    private decimal _maxKw; // Backing field

    // Property
    public decimal MaxKw
    {
        get { return _maxKw; }
        set 
        { 
            if (value < 0) 
                throw new ArgumentException("Power cannot be negative.");
            _maxKw = value; 
        }
    }
}
```

**Compiler Internals:** 
The CLR has no concept of a "Property". When the Roslyn compiler sees the `MaxKw` property, it generates two hidden methods in the IL:
- `decimal get_MaxKw()`
- `void set_MaxKw(decimal value)`

When you write `charger.MaxKw = 150;`, the compiler quietly replaces it with `charger.set_MaxKw(150);`.

### Auto-Implemented Properties
If your property doesn't require custom validation logic, you can use Auto-Properties. The compiler will automatically generate the hidden backing field for you.

```csharp
public class Tenant
{
    public Guid Id { get; set; }
    public string Name { get; set; }
}
```

## 7. Methods and Parameters

Methods define behavior. When you pass a variable to a method, you are passing it as a **parameter**.

By default, all parameters in C# are passed **by value**.
- For primitive types (`int`, `bool`), a complete copy of the value is made.
- For reference types (objects), a copy of the *memory address* (the reference) is made.

```csharp
public void UpdateFirmware(string version, int retries)
{
    // Modifying 'retries' here does NOT affect the variable passed by the caller,
    // because 'retries' is a copy.
    retries = 0; 
}
```

## 8. Namespaces

As your application grows to hundreds of classes, name collisions become inevitable. You might have an `Invoice` class for billing and an `Invoice` class for reporting. 

**Namespaces** act as logical folders to organize code.

```csharp
namespace EVPlatform.Billing
{
    public class Invoice { /* ... */ }
}

namespace EVPlatform.Reporting
{
    public class Invoice { /* ... */ }
}
```

To use a class from another namespace, you must import it using the `using` directive at the top of your file.

```csharp
using EVPlatform.Billing;

// Now the compiler knows which Invoice you mean
Invoice bill = new Invoice(); 
```

## 9. Real Production Case Study: Domain Modeling

Let's model the core entities of our Multi-Tenant EV Platform using proper Encapsulation and Auto-Properties.

```csharp
using System;

namespace EVPlatform.Domain
{
    public class Organization
    {
        // Auto-properties with private setters.
        // They can be read publicly, but only modified within this class.
        public Guid Id { get; private set; }
        public string Name { get; private set; }

        public Organization(string name)
        {
            Id = Guid.NewGuid();
            Name = name;
        }

        public void Rename(string newName)
        {
            if (string.IsNullOrWhiteSpace(newName))
                throw new ArgumentException("Name cannot be empty.");
                
            Name = newName;
        }
    }

    public class Session
    {
        public Guid SessionId { get; private set; }
        public string ChargerId { get; private set; }
        public decimal TotalKwDelivered { get; private set; }
        public bool IsActive { get; private set; }

        public Session(string chargerId)
        {
            SessionId = Guid.NewGuid();
            ChargerId = chargerId;
            TotalKwDelivered = 0;
            IsActive = true;
        }

        // Encapsulated behavior
        public void AddEnergy(decimal kw)
        {
            if (!IsActive)
                throw new InvalidOperationException("Cannot add energy to a closed session.");
                
            TotalKwDelivered += kw;
        }

        public void StopSession()
        {
            IsActive = false;
        }
    }
}
```

## 10. Common Mistakes

| Level | Mistake | Impact | How to Avoid |
|-------|---------|--------|--------------|
| Beginner | Making all fields `public`. | Total loss of encapsulation. Objects end up in invalid states. | Default to `private` fields. Expose data via Properties only if necessary. |
| Intermediate| Putting complex logic in Property Getters. | Properties look like field access, so developers expect them to be instant. | If retrieving the data requires a database call or heavy calculation, write a method instead (e.g., `CalculateTotal()`). |
| Senior | Over-engineering class hierarchies too early. | Brittle architecture that is hard to refactor. | Favor Composition over Inheritance. Keep classes focused on a single responsibility. |

## 11. Interview Questions

### Beginner Tier (Classes and Objects)

**1. What is the difference between a `class` and an `object`?**
*Answer:* A `class` is a blueprint or template that defines state and behavior (fields, properties, methods). An `object` is an instance of that class, occupying physical memory in RAM.
*Example:*
```csharp
class Car { public string Color; } // Blueprint
Car myCar = new Car { Color = "Red" }; // Object in memory
```

**2. What is a constructor in C#?**
*Answer:* A constructor is a special method invoked automatically when an object is instantiated. It has the same name as the class and no return type. It is used to initialize the object's state.
*Example:*
```csharp
class User 
{
    public string Name;
    // Constructor
    public User(string name) { Name = name; }
}
var u = new User("Alice");
```

**3. What does the `new` keyword do?**
*Answer:* The `new` keyword instructs the .NET runtime to allocate a block of memory for the object on the Managed Heap, call the class constructor to initialize it, and return a reference (pointer) to that memory location.
*Example:*
```csharp
// 'p' holds a memory address pointing to the new object
Person p = new Person(); 
```

**4. What is the difference between a Field and a Property?**
*Answer:* A Field is a raw variable declared directly in a class, representing internal state. A Property is a wrapper around a field that uses `get` and `set` accessors, allowing you to add validation or logic when reading/writing the value.
*Example:*
```csharp
class BankAccount {
    private decimal _balance; // Field
    public decimal Balance {  // Property
        get { return _balance; }
        set { if (value >= 0) _balance = value; }
    }
}
```

**5. How do you prevent a class from being instantiated?**
*Answer:* You can make the class `static` (meaning it can only contain static members), or you can make its constructor `private`.
*Example:*
```csharp
public static class MathUtils {
    public static int Add(int a, int b) => a + b;
}
// var m = new MathUtils(); // Compiler Error
```

**6. What is the `this` keyword used for?**
*Answer:* `this` refers to the current instance of the class. It is commonly used to distinguish between class fields and method parameters that share the same name.
*Example:*
```csharp
class Employee {
    private string name;
    public Employee(string name) {
        this.name = name; // Resolves naming collision
    }
}
```

**7. What is a Namespace?**
*Answer:* A Namespace is a logical container used to organize classes and prevent naming collisions. Two classes can have the same name if they are in different namespaces.
*Example:*
```csharp
namespace App.Billing { class Service { } }
namespace App.Shipping { class Service { } }
// Resolves to App.Billing.Service
```

### Intermediate Tier (Encapsulation and Properties)

**8. What happens internally when you define an auto-implemented property like `public string Name { get; set; }`?**
*Answer:* The C# runtime has no concept of a "Property". The Roslyn compiler automatically generates a hidden private backing field (e.g., `<Name>k__BackingField`), and two hidden methods: `get_Name()` and `set_Name()`.
*Example:*
```csharp
// You write:
public string Name { get; set; }
// Compiler generates:
// private string <Name>k__BackingField;
// public string get_Name() { return <Name>k__BackingField; }
// public void set_Name(string value) { <Name>k__BackingField = value; }
```

**9. What is Encapsulation, and why is it important?**
*Answer:* Encapsulation is the bundling of data and methods that operate on that data into a single unit (class), while hiding the internal state from the outside world using access modifiers (like `private`). It protects the object from being put into an invalid state.
*Example:*
```csharp
class Oven {
    private int _temperature;
    public void SetTemperature(int temp) {
        if (temp > 500) throw new Exception("Too hot!");
        _temperature = temp; // Protected state
    }
}
```

**10. What is an Object Initializer?**
*Answer:* It is syntactical sugar that allows you to assign values to accessible properties or fields of an object at the time of creation without calling a parameterized constructor.
*Example:*
```csharp
var user = new User { 
    FirstName = "John", 
    LastName = "Doe" 
};
```

**11. Explain the `init` accessor introduced in C# 9.**
*Answer:* The `init` accessor replaces `set`. It allows a property to be assigned a value *only* during object initialization (via an object initializer). After the object is created, the property becomes read-only and immutable.
*Example:*
```csharp
class LogEntry {
    public string Message { get; init; }
}
var log = new LogEntry { Message = "Error!" };
// log.Message = "Changed"; // Compiler Error!
```

**12. What are the access modifiers available in C#?**
*Answer:* `public` (accessible anywhere), `private` (accessible only within the same class), `protected` (accessible within the class and derived classes), `internal` (accessible only within the same assembly/dll), and `protected internal` (assembly or derived classes).

**13. What is an Extension Method?**
*Answer:* Extension methods allow you to "add" methods to existing types without creating a new derived type, recompiling, or modifying the original type. They are static methods in a static class, using the `this` keyword on the first parameter.
*Example:*
```csharp
public static class StringExtensions {
    public static bool IsCapitalized(this string str) {
        return !string.IsNullOrEmpty(str) && char.IsUpper(str[0]);
    }
}
// Usage: bool result = "Hello".IsCapitalized();
```

**14. What is a Partial Class?**
*Answer:* The `partial` keyword allows a single class to be split across multiple physical `.cs` files. During compilation, Roslyn merges them into a single class. It is often used to separate auto-generated code (like Entity Framework models) from human-written custom logic.
*Example:*
```csharp
// File1.cs
public partial class Employee { public void DoWork() {} }
// File2.cs
public partial class Employee { public void TakeBreak() {} }
```

### Senior Tier (Memory and Value Passing)

**15. Explain the implications of passing a Reference Type by value versus passing a Value Type by value.**
*Answer:* Passing a Value Type by value copies the entire payload (e.g., all 64 bytes of a struct) onto the stack. Passing a Reference Type by value copies only the *reference* (the 8-byte pointer). Modifying properties of the Reference Type inside the method will affect the original object because both pointers reference the exact same heap memory.
*Example:*
```csharp
void Modify(Car c) { c.Color = "Blue"; }
Car myCar = new Car { Color = "Red" };
Modify(myCar); 
// myCar.Color is now "Blue"!
```

**16. What happens if you pass a Reference Type using the `ref` keyword?**
*Answer:* Passing a Reference Type by `ref` passes a pointer *to the pointer*. This allows the method to completely reassign the caller's variable to a brand new object on the heap, destroying the original reference.
*Example:*
```csharp
void Swap(ref Car c) { c = new Car { Color = "Green" }; }
Car myCar = new Car { Color = "Red" };
Swap(ref myCar);
// myCar now points to the Green car. The Red car is orphaned for GC.
```

**17. What is the difference between a shallow copy and a deep copy of an object?**
*Answer:* A shallow copy (e.g., `Object.MemberwiseClone()`) copies the values of the fields. If a field is a reference type, the pointer is copied, so both objects share the same nested reference. A deep copy creates completely new instances of all nested reference types.
*Example:*
```csharp
// Shallow copy risk:
clone.Address.City = "New York"; 
// original.Address.City also becomes "New York"!
```

**18. Why is it an anti-pattern to call a virtual method from inside a constructor?**
*Answer:* In C#, constructors execute from the base class downwards to the derived class. If a base class constructor calls a virtual method that is overridden in a derived class, the derived override executes *before* the derived class's constructor has finished running, potentially accessing uninitialized fields and causing a NullReferenceException.

**19. What is a Static Constructor, and when is it called?**
*Answer:* A static constructor is used to initialize static data. It is parameterless and executes at most once per Application Domain, triggered automatically by the CLR immediately before the first instance is created or any static members are referenced.
*Example:*
```csharp
class Cache {
    public static readonly DateTime StartTime;
    static Cache() { StartTime = DateTime.Now; } // Runs once
}
```

**20. What is an Indexer in C#?**
*Answer:* An indexer allows an object to be indexed like an array using `[]`. It is essentially a property that takes parameters.
*Example:*
```csharp
class StringCollection {
    private string[] _data = new string[10];
    public string this[int index] {
        get { return _data[index]; }
        set { _data[index] = value; }
    }
}
// var col = new StringCollection(); col[0] = "Hello";
```

**21. Explain the difference between `const` and `static readonly` for reference types.**
*Answer:* You cannot use `const` for reference types (other than strings) because object initialization (`new`) requires runtime execution, while `const` requires compile-time evaluation. For reference types, you must use `static readonly`, which is evaluated at runtime (usually in a static constructor) and cannot be changed thereafter.
*Example:*
```csharp
// public const List<int> Numbers = new List<int>(); // ERROR
public static readonly List<int> Numbers = new List<int>(); // OK
```

### Staff Engineer Tier (Architecture and Assembly Boundaries)

**22. How does C# achieve encapsulation at the assembly level, and why is `internal` often preferred over `public` in microservices?**
*Answer:* The `internal` access modifier restricts visibility to the current assembly (`.dll`). In Domain-Driven microservices, the Core Domain assembly should expose only the minimal necessary public API to the Application layer. Keeping internal state machines, EF Core entities, and helper classes `internal` prevents architectural leakage and stops other developers from tightly coupling to implementation details.
*Example:*
```csharp
// Only accessible within this project
internal class PaymentProcessor { } 
```

**23. What is the `InternalsVisibleTo` attribute?**
*Answer:* When you mark a class `internal`, it cannot be accessed by other projects—including your Unit Test project. By adding `[assembly: InternalsVisibleTo("MyProject.Tests")]` to your project file, you grant a specific external assembly permission to access your internal types, allowing for rigorous testing without exposing the API globally.

**24. Why should you favor Composition over Inheritance when modeling complex domains?**
*Answer:* Deep inheritance hierarchies create tight coupling (the Fragile Base Class problem); a change in the base class ripples through all derived classes. It also locks you into single-inheritance. Composition (injecting specific, isolated strategy classes via interfaces) allows swapping algorithms at runtime, improves unit testability, and flattens the Virtual Method Table (VTable), yielding better JIT performance.
*Example:*
```csharp
// Bad: class FlyingCar : Car, Plane { } // C# forbids multiple inheritance
// Good: Composition
class Vehicle {
    private IEngine _engine;
    public Vehicle(IEngine engine) { _engine = engine; }
}
```

**25. Explain the Liskov Substitution Principle (LSP) and how inappropriate inheritance violates it.**
*Answer:* LSP states that derived classes must be substitutable for their base classes without altering the correctness of the program. If `Ostrich` inherits from `Bird`, but overrides `Fly()` to throw a `NotSupportedException`, it violates LSP. A consumer calling `bird.Fly()` will crash. You should architect via capabilities (Interfaces like `IFlyable`) rather than broad taxonomies.

**26. How do `required` properties (C# 11) improve Object-Oriented initialization?**
*Answer:* Historically, to force initialization, you had to write boilerplate constructors. The `required` modifier forces the caller to initialize the property via an Object Initializer. If they fail to do so, the compiler throws an error, ensuring the object is never put into an invalid state without writing massive constructors.
*Example:*
```csharp
public class User {
    public required string Email { get; init; }
}
// var u = new User(); // Compiler Error: Email is required!
var u = new User { Email = "test@test.com" }; // OK
```

**27. What is the Open/Closed Principle (OCP) and how do Extension Methods relate to it?**
*Answer:* OCP states that classes should be open for extension but closed for modification. If you need to add functionality to a `String` or a third-party `SealedClass`, you cannot modify its source code. Extension methods allow you to add new behavior (extension) without altering the original class (modification).

### Architect Tier (Performance and IL Generation)

**28. How does the JIT compiler handle method inlining, and how does Object-Oriented design interfere with it?**
*Answer:* Method inlining is a critical JIT optimization where the body of a small method is copied directly into the caller, eliminating the CPU overhead of a function call. However, the JIT generally *cannot* inline `virtual` methods because the exact method address isn't known until runtime (due to polymorphism and VTable lookups). Highly abstracted OOP code with deep interface layers often suffers from "Virtual Call Overhead" and prevents inlining.

**29. You are designing a high-throughput parser. Why might you use `static` methods heavily instead of instance methods?**
*Answer:* Instance methods require an object reference (`this`) to be passed implicitly as the first argument via CPU registers. `static` methods do not require an object instance. By passing `ReadOnlySpan<byte>` into static methods rather than instantiating parsing objects, you completely eliminate heap allocations (Gen 0 GC pressure) and avoid virtual dispatch, operating entirely on the stack and L1 cache.
*Example:*
```csharp
// Zero allocation, highly inlineable
public static int ParseId(ReadOnlySpan<byte> data) { ... }
```

**30. Explain the architectural implications of the `sealed` keyword on classes.**
*Answer:* Marking a class `sealed` prevents inheritance. Architecturally, it prevents developers from altering core behavior via unwanted overrides. Performance-wise, it allows the JIT compiler to perform "devirtualization". If the JIT knows a class cannot be inherited, it knows exactly which virtual methods (like `.ToString()`) will execute, allowing it to bypass the VTable and potentially inline the method, increasing raw CPU throughput.

**31. What is the "Fragile Base Class" problem in enterprise libraries?**
*Answer:* In a distributed team or public NuGet package, if you expose a public base class, thousands of applications might inherit from it. If you later add a new virtual method to the base class, it might accidentally collide with a method signature that a derived class had already created independently. The derived class now unintentionally overrides your base logic, causing catastrophic runtime bugs. Architects prefer Interfaces and Composition to avoid this.

**32. How do Primary Constructors (C# 12) alter class architecture?**
*Answer:* Primary constructors allow parameters to be declared directly on the class declaration line. They reduce boilerplate for Dependency Injection. However, unlike records, parameters in primary constructors on standard classes do *not* automatically become public properties; they remain scoped to the class body, strictly enforcing encapsulation unless manually exposed.
*Example:*
```csharp
// C# 12 Primary Constructor
public class ChargerService(IChargerRepo repo) {
    public void Start() => repo.Init();
}
```

**33. What is the Anemic Domain Model anti-pattern?**
*Answer:* An Anemic Domain Model occurs when Domain Entities (like `User` or `Order`) contain only public get/set properties (data) but zero business logic, while separate "Service" classes contain all the logic. This violates OOP encapsulation because the Entity cannot protect its own state. A Rich Domain Model places the business logic inside the Entity itself (e.g., `order.ApplyDiscount()`) and makes setters private.

**34. In a microservices architecture, how do you handle cross-cutting concerns (like logging or validation) without cluttering your Domain Objects?**
*Answer:* Domain objects should remain pure. Cross-cutting concerns should be applied using the Decorator Pattern or Interceptors. In ASP.NET Core, this is achieved via Middleware or MediatR Behaviors (Pipelines). The request flows through a generic Logging pipeline, then a Validation pipeline, and finally reaches the pure Domain handler, ensuring the Domain objects have zero dependencies on Infrastructure like Serilog or ILogger.

## 12. Summary
We have transitioned from procedural scripts into Object-Oriented design. By mastering Classes, Constructors, Properties, and Encapsulation, we can build robust models that protect their own state. You now have the foundational syntax and structural knowledge of C#. 

In the next chapter, we will pull back the curtain. We will explore exactly what happens to these classes when we compile the application, diving deep into the Roslyn Compiler, Intermediate Language (IL), and the Common Language Runtime (CLR).
