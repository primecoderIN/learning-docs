# Appendices

## Appendix A: Production Readiness Checklist

Before a C# enterprise application is deployed to production, the Software Architect must verify the following:

### 1. Code Quality & Performance
- [ ] Are all `HttpClient` instances managed by `IHttpClientFactory` to prevent socket exhaustion?
- [ ] Are all LINQ queries evaluated? (No accidental N+1 queries via Lazy Loading).
- [ ] Are all read-only EF Core queries using `.AsNoTracking()`?
- [ ] Are hot-path methods returning `ValueTask` instead of `Task` to reduce GC pressure?
- [ ] Are `async` methods awaiting all I/O? (No `async void` except for event handlers, no `.Result` blocking).
- [ ] Are all large string/byte manipulations utilizing `Span<T>` or `ArrayPool<T>`?

### 2. Architecture & Resiliency
- [ ] Is CQRS implemented to separate read models from write models?
- [ ] Are transient database failures wrapped in a Polly Retry Policy?
- [ ] Is third-party API integration protected by a Circuit Breaker?
- [ ] Is the Outbox Pattern used for publishing Domain Events to prevent distributed inconsistencies?
- [ ] Is the API completely stateless? (Are sessions/tokens stored in Redis/JWT rather than RAM?)

### 3. Observability & Security
- [ ] Is OpenTelemetry configured to export Traces and Metrics?
- [ ] Are secrets (connection strings, API keys) excluded from source control and loaded via Azure KeyVault or Kubernetes Secrets?
- [ ] Are EF Core Global Query Filters applied for Tenant Data Isolation?

---

## Appendix B: Architect-Level Interview Questions

### 1. Memory and GC
**Q: Explain the difference between the Small Object Heap (SOH) and the Large Object Heap (LOH). What triggers allocation on the LOH, and why is it dangerous?**
*Answer:* The SOH is generational and compacted by the GC. The LOH is for objects > 85,000 bytes (e.g., large arrays or massive strings). The LOH is not compacted by default (to save CPU cycles moving large memory blocks). This leads to memory fragmentation. Furthermore, the LOH is only collected during a Full Gen 2 GC, which causes severe "Stop the World" pauses. Architects must avoid LOH allocations by using `ArrayPool<T>` or streaming data in chunks.

### 2. Concurrency
**Q: What is a SynchronizationContext? Why was `ConfigureAwait(false)` heavily used in libraries, and why is it no longer necessary in modern ASP.NET Core?**
*Answer:* A `SynchronizationContext` dictates where an awaited task should resume execution (e.g., on a specific UI thread in WPF). Library authors used `.ConfigureAwait(false)` to prevent deadlocks when UI threads blocked synchronously on async library code. Modern ASP.NET Core has no `SynchronizationContext`; tasks resume on any available ThreadPool thread. Thus, `ConfigureAwait(false)` in an ASP.NET Core web app does nothing but add slight noise to the code.

### 3. Language Internals
**Q: Explain how the Roslyn compiler handles the `yield return` statement in C#.**
*Answer:* Roslyn performs a massive code rewrite. It generates a hidden, private class that implements `IEnumerator<T>` and acts as a State Machine. The original method's local variables are hoisted to fields in this class. Each time `.MoveNext()` is called, the state machine executes code until it hits the next `yield return`, updates its internal state integer, and pauses. This enables deferred execution and minimal memory usage.

### 4. Architecture
**Q: You have a microservice that updates a SQL database and then publishes an event to RabbitMQ. How do you guarantee the event is published if the server crashes exactly after the SQL commit but before the RabbitMQ publish?**
*Answer:* Implement the Outbox Pattern. Create an `OutboxMessage` table in the SQL database. In the exact same SQL transaction that updates the business entity, insert the serialized event into the Outbox table. Because it's a single transaction, it's atomic. A separate background worker (or CDC tool like Debezium) then reads the Outbox table and safely publishes the event to RabbitMQ, marking it as processed once the broker acknowledges receipt.

---

## Appendix C: Performance Benchmarking

To prove performance hypotheses, C# Architects use **BenchmarkDotNet**.

*Rule: Never use `Stopwatch` in Debug mode to test performance. Always use BenchmarkDotNet in Release mode.*

```csharp
using BenchmarkDotNet.Attributes;
using BenchmarkDotNet.Running;
using System.Text.Json;
using System.Text.Json.Serialization;

[MemoryDiagnoser] // Tracks Gen 0, Gen 1, Gen 2 allocations
public class SerializationBenchmarks
{
    private readonly TelemetryPacket _packet = new("CHG-100", 240.0, 32.0);

    [Benchmark(Baseline = true)]
    public string LegacyReflectionSerialization()
    {
        return JsonSerializer.Serialize(_packet);
    }

    [Benchmark]
    public string ModernSourceGeneratedSerialization()
    {
        return JsonSerializer.Serialize(_packet, TelemetryJsonContext.Default.TelemetryPacket);
    }
}

// Run via: dotnet run -c Release
public class Program
{
    public static void Main() => BenchmarkRunner.Run<SerializationBenchmarks>();
}
```

*Expected Result:* The Source Generated benchmark will execute significantly faster and allocate fewer bytes on the managed heap because it completely bypasses Reflection-based metadata discovery.
