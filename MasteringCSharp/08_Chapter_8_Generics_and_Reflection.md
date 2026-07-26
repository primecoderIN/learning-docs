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

## 8. Interview Questions

### Beginner Tier (Generics and Reflection Basics)

**1. What are Generics in C#, and why are they useful?**
*Answer:* Generics allow you to define classes, methods, or interfaces with a placeholder for the type (e.g., `<T>`). They provide type safety at compile time, prevent runtime casting errors, and eliminate the need for Boxing value types into `object`.

**2. Give an example of a Generic Collection.**
*Answer:* `List<T>` is the most common generic collection.
*Example:*
```csharp
List<int> numbers = new List<int>(); // 'int' replaces 'T'
numbers.Add(5); // Strongly typed
```

**3. What is Reflection?**
*Answer:* Reflection is the ability of managed code to read its own metadata (types, methods, properties) and invoke them dynamically at runtime. It is heavily used in older frameworks for Dependency Injection (scanning for constructors) and Serialization (JSON/XML).

**4. How do you get the `Type` of an object at runtime?**
*Answer:* You can use the `.GetType()` method on an instance, or the `typeof()` operator on a class name.
*Example:*
```csharp
Type t1 = myObj.GetType();
Type t2 = typeof(string);
```

**5. What is an Attribute in C#?**
*Answer:* An attribute is a declarative tag used to convey metadata to the runtime or compiler. Attributes inherit from `System.Attribute` and are enclosed in square brackets.
*Example:*
```csharp
[Obsolete("Use NewMethod instead")]
public void OldMethod() { }
```

**6. Does applying an Attribute change the behavior of the code?**
*Answer:* No. Attributes are completely passive metadata stored in the assembly. They only affect behavior if another piece of code (like an ORM, a testing framework, or a Source Generator) explicitly searches for them and takes action based on their presence.

**7. Can a method be generic?**
*Answer:* Yes, a single method can be generic even if the class it belongs to is not.
*Example:*
```csharp
public T DoSomething<T>(T item) { return item; }
```

### Intermediate Tier (Generic Constraints and Reification)

**8. Explain Generic Constraints.**
*Answer:* Generic constraints restrict the types that can be substituted for a generic parameter `T`. They are applied using the `where` keyword.
*Example:*
```csharp
// T must be a reference type and have a parameterless constructor
class Factory<T> where T : class, new() { } 
```

**9. What does the `where T : struct` constraint do?**
*Answer:* It ensures that `T` must be a Value Type (like `int`, `DateTime`, or a custom `struct`). It completely prevents the generic class from being instantiated with a Reference Type (like `string` or `object`).

**10. What does the `where T : class` constraint do?**
*Answer:* It ensures that `T` must be a Reference Type (a class, interface, delegate, or array).

**11. Explain why Java Generics (Type Erasure) differ fundamentally from C# Generics (Reified Generics).**
*Answer:* In Java, generics only exist at compile time; at runtime, all generic types are erased and boxed to `Object`, requiring implicit casting. In C#, the runtime is fully aware of generics. The CLR JIT-compiles unique native machine code for every specific Value Type used in a generic, completely eliminating Boxing.

**12. Can you use `typeof(T)` inside a generic method?**
*Answer:* Yes. Because C# Generics are reified, `T` is known at runtime. `typeof(T)` will correctly return the actual runtime type used to instantiate the generic method.

**13. What is `Activator.CreateInstance`?**
*Answer:* A Reflection method used to instantiate an object at runtime when the specific type is only known at runtime (e.g., passed in as a string or a `Type` variable).
*Example:*
```csharp
object myObj = Activator.CreateInstance(typeof(Customer));
```

**14. Why is Reflection considered slow?**
*Answer:* Reflection must parse string-based metadata tables at runtime, bypassing compile-time static analysis. Invoking methods via reflection bypasses JIT inlining, performs heavy security/visibility checks, and often forces value types to be Boxed onto the heap.

### Senior Tier (Advanced Metaprogramming)

**15. How does the JIT compiler optimize Generics for Reference Types?**
*Answer:* While the JIT compiler generates entirely distinct native machine code for each Value Type used in a generic (e.g., `List<int>` vs `List<double>`), it shares the exact same native machine code for ALL Reference Types (e.g., `List<string>` and `List<Customer>`). It does this because all reference types are identical at the machine level (just memory pointers).

**16. What is `MakeGenericType` and why is it dangerous?**
*Answer:* It is a Reflection method used to create a generic type dynamically at runtime.
*Example:* `typeof(List<>).MakeGenericType(typeof(int));`
It is dangerous because it forces the CLR to validate constraints and trigger JIT compilation at runtime. It breaks Ahead-Of-Time (AOT) compilation because the compiler cannot predict which generic types will be dynamically constructed.

**17. How do you invoke a private method using Reflection?**
*Answer:* You use `GetMethod` with specific `BindingFlags`.
*Example:*
```csharp
var method = typeof(MyClass).GetMethod("PrivateMethod", BindingFlags.NonPublic | BindingFlags.Instance);
method.Invoke(myObj, null);
```

**18. What is the Curious Recurring Template Pattern (CRTP) in C#?**
*Answer:* An architectural pattern where a class inherits from a generic base class, passing itself as the generic parameter. It allows the base class to enforce constraints or return strongly-typed instances of the derived class.
*Example:*
```csharp
public abstract class Entity<T> where T : Entity<T> { }
public class User : Entity<User> { }
```

**19. How do you retrieve custom attributes from a class?**
*Answer:* You use the `Attribute.GetCustomAttribute` or `MemberInfo.GetCustomAttributes` methods via Reflection.
*Example:*
```csharp
var attributes = typeof(User).GetCustomAttributes(typeof(TableAttribute), false);
```

**20. What is `Reflection.Emit`?**
*Answer:* A highly advanced namespace that allows you to dynamically generate raw Intermediate Language (IL) opcodes and compile new assemblies, classes, and methods directly into memory at runtime. It is used by dynamic proxies (like Moq or Entity Framework lazy loading proxies).

**21. What happens if you use `Reflection.Emit` in an iOS app or NativeAOT application?**
*Answer:* The application will crash. iOS (due to Apple's security rules) and NativeAOT explicitly forbid JIT compilation and dynamic code generation at runtime. All executable code must exist ahead of time.

### Staff Engineer Tier (Source Generators and AOT)

**22. You are upgrading a core .NET serialization library. How do you eliminate its heavy Reflection overhead?**
*Answer:* I would migrate the library to use C# Source Generators. Instead of parsing the object's properties at runtime via `Type.GetProperties()`, the Source Generator analyzes the Roslyn Syntax Tree during the build process and emits hardcoded, statically typed C# serialization logic. This reduces runtime memory allocation to zero and avoids JIT warmup penalties.

**23. What is a C# Source Generator?**
*Answer:* A piece of code that runs inside the Roslyn compiler during the compilation phase. It inspects the user's syntax trees (code) and generates new C# files that are automatically added to the compilation. It is compile-time metaprogramming.

**24. Can a Source Generator modify existing code?**
*Answer:* No. Source Generators are strictly additive. They can only generate new files or new `partial` classes/methods. They cannot rewrite or delete existing code written by the developer (unlike IL Weaving tools like Fody).

**25. How do Source Generators solve the AOT (Ahead-Of-Time) problem?**
*Answer:* AOT compilers need to know exactly what code to compile before the app runs. If an app uses Reflection to figure out how to serialize a `Customer` object, the AOT compiler doesn't know it needs to generate serialization code for `Customer`. Source Generators emit literal C# code (`writer.WriteString(customer.Name)`) during the build, allowing the AOT compiler to see it and compile it statically.

**26. How do you cache Reflection lookups for performance?**
*Answer:* If you absolutely must use Reflection, never call `GetProperty` in a loop. You should reflect once during startup, build a delegate using `Expression.Lambda.Compile()` or `Reflection.Emit`, and cache that compiled Delegate in a concurrent dictionary. The subsequent calls invoke the delegate at near-native speeds.

**27. Explain the performance impact of `typeof(T)` vs `obj.GetType()`.**
*Answer:* `typeof(T)` is resolved at compile time (or JIT time for generics) and translates to a very fast, direct metadata token lookup. `obj.GetType()` requires reading the MethodTable pointer off the object instance on the heap at runtime. `typeof(T)` is significantly faster and should be preferred when the type is known statically.

### Architect Tier (Framework Design and IL Weaving)

**28. An architect mandates the removal of all `Reflection` in a high-frequency trading application. How do you handle mapping JSON to DTOs without Reflection?**
*Answer:* I would use the new `System.Text.Json` Source Generators. By decorating a `partial` `JsonSerializerContext` class with `[JsonSerializable(typeof(MyDto))]`, Roslyn generates a fully statically-typed JSON parser at compile time, bypassing reflection entirely and achieving zero-allocation parsing using `Utf8JsonReader`.

**29. What is IL Weaving (e.g., PostSharp, Fody)?**
*Answer:* Unlike Source Generators that operate on C# syntax trees, IL Weaving modifies the compiled Intermediate Language (.dll or .exe) *after* compilation but *before* deployment. It is used for Aspect-Oriented Programming (AOP), like automatically injecting logging or `INotifyPropertyChanged` into every property setter. It is powerful but brittle and hard to debug.

**30. Why does heavily using Generics increase the memory footprint of an application?**
*Answer:* Because C# Generics are Reified, the JIT compiler generates unique native machine code for every Value Type permutation. If you create a massive generic math library and instantiate it for `int`, `uint`, `short`, `byte`, `float`, and `double`, the JIT generates six distinct copies of the entire library's machine code, significantly increasing the instruction cache footprint and base memory usage.

**31. How do you design an Extensibility/Plugin system in modern .NET without breaking AOT constraints?**
*Answer:* You cannot use `Assembly.Load` and dynamic Reflection to discover plugins at runtime in NativeAOT. The architect must design a build-time discovery mechanism. A Source Generator can scan the solution for assemblies implementing `IPlugin` and generate a static Registry class that hardcodes the instantiation of those plugins.

**32. Explain the purpose of `ref struct` generic constraints (C# 13+).**
*Answer:* Historically, generics could not accept `ref struct` types (like `Span<T>`) because `ref structs` must live strictly on the stack, and generic variables might be boxed or stored on the heap. C# 13 introduces `allows ref struct` anti-constraints, enabling developers to write high-performance generic parsers that safely operate entirely on `Span<T>` without heap allocation.

**33. How does generic covariance (`out T`) affect internal runtime type checking?**
*Answer:* When you assign an `IEnumerable<Dog>` to an `IEnumerable<Animal>` (covariance), the compiler allows it. However, if dealing with arrays (`Dog[]` to `Animal[]`), it is unsafe because a user might try to insert a `Cat` into the `Animal[]` (which is actually a `Dog[]`). The CLR must inject runtime type checks (ArrayTypeMismatchException checks) on every array insertion to maintain memory safety, degrading performance.

**34. In the context of micro-ORMs like Dapper, how did they achieve high performance before Source Generators existed?**
*Answer:* Dapper reads the SQL result set using DataReader and uses Reflection *exactly once* per type to map column names to properties. It then uses `System.Reflection.Emit` (IL Generation) to build a dynamic method that executes the mapping logic directly. It compiles this IL into a Delegate and caches it. Subsequent queries hit the cached Delegate, running at the speed of hand-written C# mapping code.

## 9. Summary
Generics allow us to write flexible, reusable code that the JIT compiler optimizes perfectly for both Reference and Value types. While Reflection allows for powerful metaprogramming, its runtime cost is too high for the critical hot-paths of enterprise systems. By mastering Source Generators, modern C# architects can push the burden of metaprogramming to the compiler, achieving the flexibility of dynamic code with the raw speed of native C++.

In the next chapter, we will confront the most transformative—and most misunderstood—feature of modern C#: The Asynchronous State Machine (`async`/`await`).
