# Comprehensive Interview Guide: Beginner to Architect

To master C#, you must be able to articulate your knowledge under pressure. This guide contains 50 highly technical interview questions across five tiers of seniority, complete with detailed architectural answers.

---

## Tier 1: Beginner (Language Syntax & Basics)

**1. What is the difference between a `class` and a `struct`?**
*Answer:* A `class` is a reference type allocated on the Managed Heap, subject to Garbage Collection, and passed by reference (memory pointer). A `struct` is a value type allocated wherever it is declared (often on the Stack or inline inside an array/class) and is passed by value (copied) unless `ref` or `in` modifiers are used. 

**2. Explain what the `var` keyword does in C#.**
*Answer:* `var` is syntactical sugar for compile-time type inference. It does not make C# dynamically typed. The Roslyn compiler inspects the right side of the assignment and permanently binds the variable to that specific type.

**3. What is the difference between `==` and `.Equals()`?**
*Answer:* For reference types, `==` typically checks for reference equality (do they point to the same memory address?), while `.Equals()` can be overridden to check for value equality (do they contain the same data?). For value types and strings, `==` acts as a value equality check.

**4. What is a NullReferenceException?**
*Answer:* It occurs when you attempt to access a member (property, method) on a reference type variable that currently points to `null` (no memory address).

**5. What is the purpose of the `using` statement (not the directive)?**
*Answer:* The `using` statement ensures that an object implementing `IDisposable` has its `.Dispose()` method called deterministically when the code block exits, even if an exception is thrown. It compiles into a `try/finally` block.

**6. Explain the difference between `break` and `continue` in a loop.**
*Answer:* `break` immediately terminates the entire loop. `continue` skips the remaining code in the current iteration and jumps to the next evaluation of the loop condition.

**7. What is an `enum`?**
*Answer:* An `enum` (enumeration) is a distinct value type consisting of a set of named constants. Under the hood, it is backed by an integral type (usually `int`), making it memory-efficient and highly performant in `switch` statements.

**8. What is the difference between an Array and a `List<T>`?**
*Answer:* An Array has a fixed size defined at creation; it cannot grow or shrink. A `List<T>` is a dynamic collection backed by an array. When a `List<T>` reaches capacity, it allocates a new, larger array on the heap and copies the elements over.

**9. What does the `static` keyword mean?**
*Answer:* A `static` member belongs to the type itself rather than a specific object instance. There is only one copy of a static field in memory for the lifetime of the Application Domain, making it a GC Root.

**10. What is an Interface?**
*Answer:* An Interface is a contract that defines a set of methods or properties without implementing them (prior to C# 8 default implementations). Any class or struct that implements the interface must provide the actual code for those members.

---

## Tier 2: Intermediate (OOP, LINQ, and Memory Basics)

**11. What is Boxing and Unboxing?**
*Answer:* Boxing is the process of converting a Value Type (like `int`) to a Reference Type (`object` or an interface). This causes a heap allocation and copies the value. Unboxing extracts the value type back out. Both are computationally expensive and trigger GC pressure.

**12. Explain Deferred Execution in LINQ.**
*Answer:* When you chain LINQ methods like `.Where()` and `.Select()`, the query is not executed immediately. Instead, an `IEnumerable` state machine is created. The actual data processing only occurs when the query is materialized (e.g., via `.ToList()`, or iterating with `foreach`).

**13. What is the difference between `IEnumerable` and `IQueryable`?**
*Answer:* `IEnumerable` executes queries in memory using delegates (LINQ to Objects). `IQueryable` translates the LINQ expression tree into a query language (like SQL) and executes it on the database engine (LINQ to SQL / EF Core).

**14. Why should you avoid `async void`?**
*Answer:* `async void` is a "fire-and-forget" mechanism. Because it doesn't return a `Task`, the calling method cannot `await` it or catch exceptions from it. If an exception is thrown inside an `async void` method, it crashes the entire application process.

**15. What is Dependency Injection (DI)?**
*Answer:* DI is the implementation of Inversion of Control. Instead of a class instantiating its own dependencies using `new`, the dependencies are passed in via the constructor. This decouples classes and makes them unit-testable via mocking.

**16. Describe the difference between Transient, Scoped, and Singleton DI lifetimes.**
*Answer:* Transient creates a new instance every time. Scoped creates one instance per HTTP request. Singleton creates one instance that is shared across the entire application lifetime.

**17. What is a Delegate?**
*Answer:* A delegate is a type-safe object-oriented function pointer. It holds a reference to a method and, if it's an instance method, a reference to the target object. `Action` and `Func` are built-in generic delegates.

**18. What is the N+1 Query Problem in Entity Framework?**
*Answer:* It occurs when you query a list of parent entities (1 query) and then loop through them, lazy-loading a child entity for each parent (N queries). This crushes database performance. Fix it using `.Include()` (eager loading) or `.Select()` (projection).

**19. How does Garbage Collection know what to delete?**
*Answer:* The GC uses a Mark-and-Sweep algorithm. It starts at GC Roots (active stack variables, static fields) and traverses the object graph, marking everything it touches. Any object on the heap that is not marked is considered dead and is swept away.

**20. What is a `record` in C#?**
*Answer:* A `record` is a reference type designed for immutable data models. The compiler automatically synthesizes properties, constructors, and value-based equality methods (`Equals`, `GetHashCode`), meaning two records with the exact same data are considered equal.

---

## Tier 3: Senior (Threading, Async Internals, and EF Core)

**21. Exactly what happens when you use the `await` keyword on an I/O operation?**
*Answer:* The thread does not block. The compiler generates an `IAsyncStateMachine`. The I/O request is registered with the OS I/O Completion Port (IOCP). The calling thread returns to the ThreadPool. When the hardware interrupt signals completion, the ThreadPool assigns a thread (often a different one) to resume the state machine where it left off.

**22. When should you use `ValueTask` instead of `Task`?**
*Answer:* Returning `Task` allocates a reference object on the managed heap. If an asynchronous method frequently completes *synchronously* (e.g., pulling data from an in-memory cache), `ValueTask` (a struct) avoids the heap allocation entirely.

**23. What is ThreadPool Work Stealing?**
*Answer:* To avoid lock contention on the global thread queue, each ThreadPool thread has a local queue. If a thread exhausts its local queue, it will look at another thread's local queue and "steal" work from the tail end of it.

**24. Explain the difference between `lock` and `SemaphoreSlim`.**
*Answer:* `lock` (which compiles to `Monitor`) is a thread-affine synchronization primitive; the thread that acquires the lock must be the one to release it. Therefore, you cannot `await` inside a `lock`. `SemaphoreSlim` is an asynchronous primitive that allows you to `await _semaphore.WaitAsync()`, allowing cross-thread locking.

**25. Why is `.AsNoTracking()` faster in EF Core?**
*Answer:* By default, EF Core stores a snapshot of every queried entity in the `DbContext` Change Tracker dictionary. `.AsNoTracking()` bypasses this, saving significant memory allocation and CPU cycles during materialization. It should be used for all read-only queries.

**26. What causes an Event Memory Leak in C#?**
*Answer:* Events are backed by `MulticastDelegate`, which holds a strong reference to the subscribing object (`Target`). If a short-lived object subscribes to a long-lived singleton's event and forgets to unsubscribe (`-=`), the GC will never collect the short-lived object because the delegate acts as a GC Root.

**27. Explain the difference between `Span<T>` and `Memory<T>`.**
*Answer:* `Span<T>` is a `ref struct`, meaning it can only live on the stack. It cannot be boxed or used in async state machines. `Memory<T>` is a standard struct that can be placed on the heap, making it safe to use across `await` boundaries.

**28. How does `yield return` work under the hood?**
*Answer:* The compiler generates a private state machine class implementing `IEnumerator<T>`. Calling the method returns this object without executing the code. Each call to `.MoveNext()` executes code up to the next `yield return`, updating an internal integer state, achieving deferred, zero-allocation iteration.

**29. What is a Captive Dependency?**
*Answer:* It occurs when a long-lived DI service (Singleton) injects a short-lived service (Scoped). The Scoped service becomes trapped inside the Singleton, keeping it alive permanently and causing cross-request data corruption (e.g., trapping a `DbContext`).

**30. Explain Virtual Method Dispatch (VTables).**
*Answer:* For `virtual` methods, the compiler emits a `callvirt` IL instruction. The JIT cannot hardcode the memory address. At runtime, the CPU reads the object's MethodTable pointer, navigates to the Virtual Method Table (VTable), looks up the overridden method's address, and jumps to it.

---

## Tier 4: Staff Engineer (Architecture, JIT, and Low-Level Memory)

**31. Compare JIT, ReadyToRun, and NativeAOT compilation.**
*Answer:* 
- **JIT (RyuJIT):** Compiles IL to machine code at runtime. Slow startup, but highly optimized for the specific host CPU (e.g., AVX instructions).
- **ReadyToRun (R2R):** Pre-compiles generic machine code during build, reducing JIT warmup, but retains IL as a fallback.
- **NativeAOT:** Completely removes the JIT and compiles directly to native OS binaries. Instant startup, tiny memory footprint, but breaks dynamic reflection and runtime code generation.

**32. How do C# Generics differ from Java Generics, and why is it faster?**
*Answer:* Java uses Type Erasure; generics only exist at compile time, and are boxed to `Object` at runtime. C# uses Reified Generics. For Value Types (`int`, `struct`), the RyuJIT generates highly specialized, unique native machine code, completely eliminating Boxing overhead.

**33. What is the Large Object Heap (LOH) and how does it cause fragmentation?**
*Answer:* Any object >= 85,000 bytes (or double arrays > 1000 elements) goes to the LOH. Because copying large objects is expensive, the GC does not compact the LOH by default. Over time, allocating and freeing large objects leaves gaps in memory, eventually leading to `OutOfMemoryException` even if total memory is low.

**34. Explain the Outbox Pattern and why it is critical for Microservices.**
*Answer:* Dual-write scenarios (saving to SQL and publishing to RabbitMQ) are prone to distributed inconsistency if the message broker fails. The Outbox Pattern writes the domain event to a SQL `Outbox` table in the *same transaction* as the entity update. A background worker reliably polls and forwards the messages.

**35. What is CQRS, and why is it used?**
*Answer:* Command Query Responsibility Segregation separates Write models (Commands: complex validation, domain logic, ORMs) from Read models (Queries: fast, flat DTOs, Dapper, NoSQL). It prevents complex Domain models from being bloated by reporting requirements and allows independent scaling.

**36. Explain how `System.Threading.Channels` improves upon `BlockingCollection`.**
*Answer:* `Channels` provide a modern, heavily optimized, `async`-first Producer/Consumer pipeline. `BlockingCollection` blocks the physical OS thread when waiting for items. `Channels` use `await foreach`, yielding the thread back to the ThreadPool, allowing massive scale with very few threads.

**37. Why are Source Generators replacing Reflection?**
*Answer:* Reflection requires parsing metadata strings and boxing values at runtime, causing severe CPU latency and GC pressure. It also breaks NativeAOT. Source Generators inspect the syntax tree at *compile time* and emit hardcoded, statically typed C# files, achieving the flexibility of Reflection with bare-metal speed.

**38. What is false sharing in multi-threading, and how do you avoid it?**
*Answer:* False sharing occurs when two threads modify independent variables that happen to reside on the same 64-byte CPU cache line. The CPU hardware forces cache invalidation across cores, destroying performance. You avoid it by adding explicit memory padding (`[StructLayout(LayoutKind.Explicit)]`) between heavily contested variables.

**39. How does Kestrel handle massive concurrent connections without running out of memory?**
*Answer:* Kestrel uses `System.IO.Pipelines` rather than allocating `byte[]` arrays for every request. It reads socket data directly into pinned, reusable buffers managed by the `MemoryPool`, ensuring Gen 2 GC is not triggered under high load.

**40. Explain the implications of `ServerGarbageCollection` vs `WorkstationGarbageCollection`.**
*Answer:* Workstation GC favors low latency for UI threads (doing GC on the same thread). Server GC creates dedicated, high-priority GC threads and dedicated heaps for *every CPU core*. It maximizes throughput for ASP.NET servers but uses significantly more base memory.

---

## Tier 5: Solution Architect (Enterprise SaaS & Distributed Systems)

**41. Design a Multi-Tenant Data architecture for a SaaS application. Compare the trade-offs.**
*Answer:* 
1. Database-per-tenant: Maximum security, high cost, brutal schema migration operations.
2. Schema-per-tenant: Logical separation, but EF Core struggles with dynamic schema routing.
3. Row-level (TenantId column): Highly scalable, single migration. Requires EF Core Global Query Filters to guarantee developers don't accidentally leak cross-tenant data.

**42. How do you implement Idempotency in a distributed API?**
*Answer:* If a client sends a POST request, but the network drops the HTTP 200 OK response, the client will retry, potentially charging a credit card twice. Idempotency requires the client to send an `Idempotency-Key` header. The API checks Redis/SQL; if the key exists, it returns the cached HTTP response instead of re-executing the transaction.

**43. Explain the Circuit Breaker pattern. Why is it better than simple Retries?**
*Answer:* Retries (`Polly.WaitAndRetry`) are for transient faults (network blips). If a downstream microservice is completely dead, 1,000 requests retrying 3 times equals 3,000 blocked threads on your server, leading to cascading failure. A Circuit Breaker detects the failure threshold, "trips" open, and fast-fails all subsequent requests immediately, saving your ThreadPool and giving the downstream service time to recover.

**44. How do you design for Eventual Consistency across microservices?**
*Answer:* By utilizing Domain Events, the Outbox Pattern, and an Event Bus (Kafka/RabbitMQ). You must design your system to tolerate stale reads (e.g., a user updates their profile, but the reporting service isn't updated for 5 seconds). Compensation transactions (Sagas) must be designed to rollback state if a downstream step fails.

**45. Describe how you would instrument a .NET application for observability.**
*Answer:* Use OpenTelemetry. Inject standard structured logging (Serilog). Enable Distributed Tracing so `TraceId` propagates across HTTP headers (W3C standard) to correlate logs across microservices. Export metrics (CPU, GC allocations, custom domain metrics) via Prometheus/Grafana to configure automated alerting.

**46. How do you manage database schema migrations in a CI/CD pipeline with zero downtime?**
*Answer:* Never apply migrations automatically on app startup, as multiple pods will cause race conditions and lock the DB. Run migrations as a separate idempotent pipeline step. Migrations must be backwards-compatible (e.g., adding a column is safe; renaming a column requires a multi-step deployment: Add new column -> App writes to both -> Migrate data -> App reads from new -> Drop old column).

**47. What is Vertical Slice Architecture, and how does it compare to Clean/Onion Architecture?**
*Answer:* Clean Architecture organizes code by technical concern (Domain, Infrastructure, UI), which can lead to scattering files across projects for a single feature. Vertical Slice organizes code by feature (e.g., `StartChargingSession`), bundling the command, handler, and endpoint together. It optimizes developer velocity and cohesion while preventing premature abstractions.

**48. How do you secure internal microservice-to-microservice communication?**
*Answer:* Use mTLS (Mutual TLS) provided by a Service Mesh (like Istio/Linkerd) so services authenticate each other via certificates. Pass the original user's JWT token via header forwarding, and validate the token signature and Audience (aud) claim in every downstream service.

**49. How would you optimize a .NET background worker processing 10 million rows a night?**
*Answer:* Do not load 10M rows into EF Core (Memory exhaustion). Use `IAsyncEnumerable` or Dapper to stream rows. Use `System.Threading.Channels` to decouple reading from processing. Use `Parallel.ForEachAsync` to process items concurrently, bounding the parallelism to CPU limits. Use Bulk SQL inserts (`SqlBulkCopy`) instead of individual `UPDATE` statements.

**50. You inherited a legacy monolithic C# application that crashes randomly under load. What is your diagnostic approach?**
*Answer:* 
1. Attach `dotnet-counters` to monitor GC pressure (Gen 2 collections) and ThreadPool starvation.
2. Collect a memory dump (`dotnet-dump`) when RAM spikes and analyze the Large Object Heap for string/array leaks.
3. Review APM metrics (AppInsights) for slow SQL queries lacking indices, causing connection pool exhaustion.
4. Search the codebase for `Task.Result`, `.Wait()`, or `async void` causing deadlocks.
5. Identify the bottlenecks and extract them iteratively using the Strangler Fig pattern.
