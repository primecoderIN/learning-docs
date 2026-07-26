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

## 11. Summary
We have transitioned from procedural scripts into Object-Oriented design. By mastering Classes, Constructors, Properties, and Encapsulation, we can build robust models that protect their own state. You now have the foundational syntax and structural knowledge of C#. 

In the next chapter, we will pull back the curtain. We will explore exactly what happens to these classes when we compile the application, diving deep into the Roslyn Compiler, Intermediate Language (IL), and the Common Language Runtime (CLR).
