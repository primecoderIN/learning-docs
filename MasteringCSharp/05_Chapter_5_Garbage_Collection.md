# Chapter 5: Garbage Collection and Resource Management

## 1. Learning Objectives
By the end of this chapter, you will be able to:
- Explain the Mark-and-Sweep algorithm and the Generational Garbage Collection model.
- Identify the differences between the Small Object Heap (SOH) and the Large Object Heap (LOH).
- Analyze GC Roots and memory pinning.
- Implement the `IDisposable` and `IAsyncDisposable` interfaces correctly.
- Understand Finalizers and the Finalization Queue.
- Measure and reduce GC pressure in high-throughput enterprise systems.

## 2. Introduction

In C++, developers must manually call `delete` to free memory they allocated with `new`. A single missed `delete` causes a memory leak; a double `delete` causes memory corruption and application crashes.

To eliminate these entire classes of bugs, the .NET Common Language Runtime (CLR) abstracts memory management away from the developer using a **Garbage Collector (GC)**. However, as an enterprise software engineer, you cannot treat the GC as a magic box. In high-performance systems—such as our Multi-Tenant EV Charging Platform processing millions of telemetry packets—the GC is often the single largest source of unpredictable latency.

When the GC runs, it must ensure that memory is not mutated while it is being analyzed. To do this, it often triggers a **"Stop The World"** event, suspending all application threads. A GC pause of 500 milliseconds might be acceptable in a desktop app, but it is catastrophic for a high-frequency trading system or an IoT ingestion pipeline. 

You must learn to work *with* the Garbage Collector, not against it.

## 3. The Mark-and-Sweep Algorithm

The .NET GC is a **Tracing Garbage Collector** that uses a Mark-and-Sweep-and-Compact algorithm.

### Phase 1: Marking (Finding Live Objects)
The GC must determine which objects are still being used by your application. It does this by starting at **GC Roots**.
A GC Root is a fundamental reference that the runtime knows is active. Examples include:
- Local variables on a thread's active stack.
- Static variables (which live for the lifetime of the AppDomain/Process).
- CPU Registers currently pointing to objects.
- Objects on the Finalization Queue (awaiting cleanup).

The GC traverses the object graph starting from these roots. Every object it reaches is "marked" as alive. If an object `A` has a field referencing object `B`, and `A` is marked, `B` is also marked.

### Phase 2: Sweeping and Compacting
Any object on the managed heap that was *not* marked is considered dead (unreachable). The GC reclaims the memory occupied by these dead objects.

Because objects are allocated contiguously, deleting random objects leaves holes in memory (fragmentation). To solve this, the GC **Compacts** the heap. It physically moves the surviving objects down in memory to remove the gaps, and then updates all variables in your application (the roots) to point to the new memory addresses.

**Visual Diagram: Heap Compaction**
```text
Before GC:
[Obj A (Live)] [Obj B (Dead)] [Obj C (Dead)] [Obj D (Live)] [Free Space ...]

After Mark & Sweep:
[Obj A (Live)] [   Empty    ] [   Empty    ] [Obj D (Live)] [Free Space ...]

After Compaction (Pointers Updated!):
[Obj A (Live)] [Obj D (Live)] [Free Space .................................]
```

## 4. The Generational Model

Tracing the entire heap takes a long time. To optimize this, the .NET GC uses a heuristic: **"Newly created objects are likely to die quickly. Older objects are likely to survive for a long time."**

The Managed Heap is divided into three Generations:

- **Generation 0 (Gen 0):** This is where new objects are allocated. It is very small and fits entirely in the CPU L2/L3 cache. When Gen 0 fills up, a Gen 0 GC occurs. This is extremely fast (often < 1 millisecond). Surviving objects are promoted to Gen 1.
- **Generation 1 (Gen 1):** Acts as a buffer between short-lived and long-lived objects. If a Gen 1 GC occurs, survivors are promoted to Gen 2.
- **Generation 2 (Gen 2):** Contains long-lived objects (e.g., singletons, static caches). A Gen 2 GC (also known as a Full GC) traces the *entire* heap. This is the expensive "Stop The World" pause you must avoid.

## 5. The Large Object Heap (LOH)

The Generational model applies to the **Small Object Heap (SOH)**. 
However, copying a 10-megabyte array during compaction would ruin performance. Therefore, any object **85,000 bytes or larger** is allocated on a separate heap called the **Large Object Heap (LOH)**.

**LOH Rules:**
- LOH objects are never compacted by default (to avoid expensive memory copies), which leads to fragmentation.
- LOH allocations are collected *only* during a Full Gen 2 GC.
- Because arrays of `double` (8 bytes) hit the 85,000-byte limit at roughly 10,600 elements, it is surprisingly easy to accidentally allocate on the LOH.

## 6. Deterministic Cleanup: IDisposable and using

The GC manages *memory*, but it does not manage *unmanaged resources* like File Handles, Network Sockets, Database Connections, or native OS locks. If you don't release a file handle, the OS will keep the file locked even if the C# object holding it is garbage collected.

To release unmanaged resources deterministically (immediately when you are done), you implement the `IDisposable` interface.

```csharp
public class NetworkStreamReader : IDisposable
{
    private Socket _socket;

    public NetworkStreamReader()
    {
        _socket = new Socket(AddressFamily.InterNetwork, SocketType.Stream, ProtocolType.Tcp);
    }

    public void ReadData() { /* ... */ }

    // Required by IDisposable
    public void Dispose()
    {
        _socket?.Close();
        _socket?.Dispose();
    }
}
```

**The `using` Statement:**
To ensure `Dispose` is called even if an exception is thrown, C# provides the `using` keyword. It compiles into a `try/finally` block.

```csharp
// Modern C# 8+ using declaration
using var reader = new NetworkStreamReader();
reader.ReadData();
// reader.Dispose() is automatically called at the end of the method scope.
```

## 7. Finalizers and the Finalization Queue

What if a developer forgets to wrap `NetworkStreamReader` in a `using` statement? The socket handle would leak forever.
To provide a safety net, C# allows you to write a **Finalizer** (a method preceded by a tilde `~`).

```csharp
public class NetworkStreamReader : IDisposable
{
    private Socket _socket;
    
    // ... Constructor and Dispose ...

    // Finalizer (Destructor)
    ~NetworkStreamReader()
    {
        Dispose(false); // Clean up unmanaged resources
    }
}
```

**The Hidden Cost of Finalizers:**
If an object has a finalizer, the GC handles it very differently:
1. When the object is created, a reference is added to the **Finalization Queue**.
2. When the GC determines the object is dead, it *does not* reclaim its memory. Instead, it moves it to the **Freachable Queue**.
3. A dedicated, high-priority CLR thread reads the Freachable Queue and executes the `~Finalizer()` methods.
4. The object survives into the next GC generation (usually Gen 1 or Gen 2). Its memory is finally reclaimed on the *next* GC cycle.

**Architectural Rule:** Finalizers delay memory reclamation and promote short-lived objects to older generations. Implement them *only* as a fallback for objects holding raw OS handles (which is rare, as you should use `SafeHandle` classes from the BCL).

## 8. Real Production Case Study: Object Pooling

In our EV Platform, we parse incoming heartbeat messages. If we allocate a new `byte[]` buffer for every incoming TCP packet, we will generate massive Gen 0 pressure. If those packets arrive faster than the GC can collect them, they will be promoted to Gen 1 and Gen 2, eventually causing a Full GC pause that drops network connections.

**Solution: `ArrayPool<T>`**
Instead of allocating and discarding, we will rent arrays from a shared pool.

```csharp
using System.Buffers;
using System.Net.Sockets;

public class HeartbeatListener
{
    private readonly Socket _listenSocket;

    public async Task ListenAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            var clientSocket = await _listenSocket.AcceptAsync(ct);
            _ = ProcessClientAsync(clientSocket); // Fire and forget
        }
    }

    private async Task ProcessClientAsync(Socket client)
    {
        // RENT a buffer instead of allocating a new one
        // The array returned might be larger than 1024 bytes!
        byte[] buffer = ArrayPool<byte>.Shared.Rent(1024);
        
        try
        {
            int bytesRead = await client.ReceiveAsync(buffer, SocketFlags.None);
            
            // Slice the buffer to the actual bytes read using Span
            ReadOnlySpan<byte> actualData = buffer.AsSpan(0, bytesRead);
            ParseHeartbeat(actualData);
        }
        finally
        {
            // RETURN the buffer to the pool so another thread can use it.
            // If you forget this, the pool eventually allocates new arrays (memory leak).
            ArrayPool<byte>.Shared.Return(buffer);
            client.Dispose();
        }
    }

    private void ParseHeartbeat(ReadOnlySpan<byte> data) { /* ... */ }
}
```

## 9. Common Mistakes

| Level | Mistake | Impact | How to Avoid |
|-------|---------|--------|--------------|
| Beginner | Calling `GC.Collect()` manually. | Destroys GC heuristics. Pauses the application needlessly. | Never call `GC.Collect()` in production code unless you are building a memory profiler or testing framework. |
| Intermediate | Storing references to temporary objects in a `static` list. | Memory Leak. The `static` list is a GC Root that lives forever, preventing the objects from being collected. | Be extremely careful with `static` collections. Use `ConditionalWeakTable` or bounded caches. |
| Senior | Forgetting to `Dispose` HTTP clients or streams. | Port exhaustion or file locks. | Always use the `using` statement for anything implementing `IDisposable`. |
| Architect | High LOH Allocation rate (e.g. large JSON strings). | Frequent, slow Gen 2 collections. Heap fragmentation. | Stream JSON directly to objects. Use `ArrayPool<T>` for large buffers. |

## 10. Summary
The Garbage Collector is a marvel of software engineering, but it is not magic. By understanding how the Mark-and-Sweep algorithm traverses GC Roots, and how generations separate short-lived data from long-lived data, we can architect systems that are sympathetic to the runtime. We explored how to manage unmanaged resources with `IDisposable` and how to completely eliminate buffer allocation overhead using `ArrayPool<T>`. 

In the next chapter, we return to the C# language itself, diving deep into advanced Object-Oriented paradigms, Polymorphism, and how Virtual Method Tables (VTables) function under the hood.
