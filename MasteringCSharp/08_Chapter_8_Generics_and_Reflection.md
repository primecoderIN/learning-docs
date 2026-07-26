# Chapter 8: Generics, Reflection, and Metaprogramming

## 1. Learning Objectives
By the end of this chapter, you will be able to:
- Explain the concept of Reified Generics and how the CLR handles Generic Type specialization.
- Understand the deep performance penalties associated with Reflection.
- Extract metadata using Custom Attributes.
- Architect high-performance, compile-time metaprogramming using C# Source Generators.

## 2. Introduction

Enterprise software relies heavily on reusable infrastructure: Dependency Injection containers, Object-Relational Mappers (ORMs), and JSON Serializers. These libraries cannot be hardcoded to know about your specific `Tenant` or `EvCharger` classes. They must operate on code they have never seen before.

To achieve this, C# provides two primary mechanisms: **Generics** (compile-time/JIT-time polymorphism) and **Reflection** (runtime type inspection). While both are incredibly powerful, they have vastly different performance profiles. 

In recent years, the C# ecosystem has aggressively moved away from Reflection in favor of **Source Generators**—shifting the heavy lifting of metaprogramming from runtime to compile-time.

## 3. Generics and Reification

Generics were introduced in C# 2.0. Unlike Java, where Generics are implemented via **Type Erasure** (the compiler strips out generic types and treats everything as `Object` at runtime), C# Generics are **Reified**.

Reification means the CLR knows exactly what type `T` is at runtime. 

```csharp
public class Cache<T>
{
    private Dictionary<string, T> _store = new();
    
    public void Add(string key, T value) => _store[key] = value;
}
```

### JIT Compilation of Generics

When you instantiate a generic type, RyuJIT handles it based on whether `T` is a Reference Type or a Value Type.

1. **Value Types (`int`, `struct`):**
   If you create a `Cache<int>`, the JIT compiler generates a *completely distinct* set of native machine code specifically for `int`. If you create a `Cache<double>`, it generates another set of native code for `double`. This ensures maximum performance and exact memory alignment, completely avoiding Boxing.

2. **Reference Types (`string`, `EvCharger`):**
   Because all Reference Types are essentially just 8-byte pointers (on a 64-bit OS), the JIT compiler is smart. It compiles the machine code for `Cache<object>` exactly once, and shares that exact same native code for `Cache<string>`, `Cache<EvCharger>`, and any other reference type. 

**Architectural Rule:** Generics are the highest-performing way to write reusable code in C#. Prefer Generics with constraints (`where T : class`) over interfaces or base classes whenever possible to allow the JIT to optimize the dispatch.

## 4. Reflection and Runtime Metaprogramming

Reflection allows your code to inspect its own metadata, instantiate objects, and invoke methods dynamically at runtime.

```csharp
public class ChargerConfig
{
    public int MaxKw { get; set; } = 150;
    public void Connect() { Console.WriteLine("Connected"); }
}

// Reflection Example
Type type = typeof(ChargerConfig);
object instance = Activator.CreateInstance(type); // Slower than 'new'

PropertyInfo prop = type.GetProperty("MaxKw");
int maxKw = (int)prop.GetValue(instance); // Very slow!

MethodInfo method = type.GetMethod("Connect");
method.Invoke(instance, null); // Extremely slow!
```

### The Cost of Reflection
Reflection is notoriously slow for three reasons:
1. **Metadata Parsing:** To find a property, the CLR must parse the string `"MaxKw"`, iterate through the assembly's metadata tables, and resolve the member.
2. **Boxing:** Methods like `GetValue` return `object`. If the property is an `int`, the CLR must allocate memory on the heap to Box the value before returning it.
3. **Security and Visibility Checks:** Every time you invoke a method via reflection, the CLR must verify that the caller has permission to invoke it, bypassing JIT inlining completely.

While caching `PropertyInfo` objects helps, invoking them remains significantly slower than direct code execution. 

## 5. Attributes: Decorating Metadata

Attributes allow you to attach custom metadata to classes, properties, or methods. They do nothing on their own; they must be read via Reflection (or Source Generators) to be useful.

```csharp
[AttributeUsage(AttributeTargets.Class)]
public class AuditableAttribute : Attribute { }

[Auditable]
public class PaymentTransaction
{
    public decimal Amount { get; set; }
}
```

An ORM or an Audit logger could use Reflection at startup to find all classes tagged with `[Auditable]` and automatically wire up database triggers.

## 6. Source Generators: The Death of Reflection

Because Reflection is slow and completely breaks Ahead-Of-Time (AOT) compilation (like NativeAOT), Microsoft introduced **Source Generators** in C# 9.0.

A Source Generator is a piece of code that runs *inside the Roslyn compiler during the build process*. It inspects your syntax trees and generates additional C# source code files, which are then compiled into your assembly.

Instead of an ORM using Reflection at runtime to map a database row to an object, a Source Generator writes the exact `reader.GetInt32(0)` mapping code at compile-time.

### Real Production Case Study: Compile-Time Serialization

In our EV Platform, we need to serialize telemetry to JSON. Using standard Reflection-based `JsonSerializer` can cause high CPU overhead on 10,000 requests per second. We will use the modern Source-Generated JSON Serialization.

```csharp
using System.Text.Json.Serialization;

// 1. Define our Domain Record
public record TelemetryPacket(string ChargerId, double Voltage, double Current);

// 2. Define a partial class deriving from JsonSerializerContext.
// The [JsonSerializable] attribute tells the Source Generator to inspect TelemetryPacket.
[JsonSourceGenerationOptions(WriteIndented = false)]
[JsonSerializable(typeof(TelemetryPacket))]
public partial class TelemetryJsonContext : JsonSerializerContext
{
}
```

**What Happens at Compile-Time:**
The Roslyn Source Generator detects the `JsonSerializable` attribute. It writes a hidden C# file containing highly optimized, reflection-free mapping logic specific to `TelemetryPacket`.

**Usage at Runtime:**
```csharp
var packet = new TelemetryPacket("CHG-100", 240.5, 32.1);

// We pass the Source Generated context to the serializer.
// ZERO Reflection is used. Memory allocation is minimized. Speed is equivalent to hardcoded parsing.
string json = JsonSerializer.Serialize(packet, TelemetryJsonContext.Default.TelemetryPacket);
```

## 7. Common Mistakes

| Level | Mistake | Impact | How to Avoid |
|-------|---------|--------|--------------|
| Beginner | Confusing Generics with `object` casting. | Boxing overhead and loss of compile-time type safety. | Use Generics (`List<T>`) instead of legacy collections (`ArrayList`). |
| Intermediate| Using `Activator.CreateInstance` in a loop. | Massive CPU degradation. | Use `new()`, Generics with `where T : new()`, or compiled Expression Trees. |
| Senior | Heavy reliance on Reflection for mapping (e.g., legacy AutoMapper). | High allocation rates and slow execution times. | Switch to Source Generator-based mappers like Mapperly, or write manual mapping methods. |
| Architect | Designing frameworks that rely on dynamic assembly loading (Reflection.Emit). | Incompatible with NativeAOT and iOS deployment. | Always provide a Source Generator alternative for AOT compatibility. |

## 8. Summary
Generics allow us to write flexible, reusable code that the JIT compiler optimizes perfectly for both Reference and Value types. While Reflection allows for powerful metaprogramming, its runtime cost is too high for the critical hot-paths of enterprise systems. By mastering Source Generators, modern C# architects can push the burden of metaprogramming to the compiler, achieving the flexibility of dynamic code with the raw speed of native C++.

In the next chapter, we will confront the most transformative—and most misunderstood—feature of modern C#: The Asynchronous State Machine (`async`/`await`).
