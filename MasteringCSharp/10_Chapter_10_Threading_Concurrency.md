# Chapter 10: Threading, Concurrency, and Synchronization

## 1. Learning Objectives
By the end of this chapter, you will be able to:
- Explain the .NET ThreadPool and Work Stealing algorithms.
- Calculate the physical overhead of an OS Thread (Context Switching and Memory).
- Implement Thread Synchronization safely using `lock`, `Monitor`, and `SemaphoreSlim`.
- Use modern concurrent data structures (`ConcurrentDictionary`).
- Architect high-throughput producer-consumer pipelines using `System.Threading.Channels`.

## 2. Introduction

In Chapter 9, we learned how `async`/`await` allows us to scale systems by releasing threads during I/O operations (network/disk access). But what happens when you have intense, purely mathematical calculations? Parsing a 10GB CSV file in memory, rendering graphics, or hashing passwords using BCrypt requires raw CPU cycles. 

This is **CPU-Bound** work. To scale CPU-bound work, we must utilize multiple CPU cores concurrently. 

However, multi-threading introduces chaos. When two threads access the exact same memory address at the exact same time, you encounter **Race Conditions**, data corruption, and deadlocks. As a Systems Architect, you must protect shared state using Synchronization Primitives, while ensuring you do not lock the system so heavily that you destroy its performance.

## 3. The Physical Cost of a Thread

An OS Thread is not a lightweight abstraction. It is a heavy, physical OS resource.
When you create `new Thread(DoWork).Start();`, the following happens:
1. The OS allocates ~1MB of continuous memory for the Thread Stack.
2. The OS creates kernel objects to track the thread.
3. The OS Scheduler must now allocate time slices to this thread.

### Context Switching
A CPU core can only execute one thread at a time. If you have 4 cores, you can execute exactly 4 threads simultaneously. 

If your application spawns 1,000 threads, the OS must rapidly pause one thread, save its CPU registers to memory, load the registers for the next thread, and resume it. This is a **Context Switch**. 
Context switching takes a few microseconds, but if you do it thousands of times a second, your CPU spends 90% of its time switching threads and only 10% of its time executing your code. This is known as **Thread Thrashing**.

## 4. The .NET ThreadPool and Work Stealing

To avoid thread thrashing, .NET uses a **ThreadPool**.
Instead of creating a new thread for every task, the CLR maintains a pool of worker threads (roughly equal to the number of CPU cores).

When you call `Task.Run(() => DoWork())`, you are pushing a work item onto the **Global Queue**. ThreadPool threads constantly pull work from this queue. 

**Work Stealing:**
To reduce contention on the Global Queue, each ThreadPool thread has its own **Local Queue**. If a thread finishes its work and its local queue is empty, it will look at another thread's local queue and "steal" work from the back of it. This highly optimized algorithm minimizes lock contention inside the runtime itself.

## 5. Synchronization Primitives

When multiple threads must mutate shared state, you must synchronize access.

### The `lock` Statement and `Monitor`
The easiest way to synchronize is the `lock` keyword. It compiles down to `Monitor.Enter()` and `Monitor.Exit()` inside a `try/finally` block.

```csharp
public class SessionCounter
{
    private int _count = 0;
    private readonly object _syncRoot = new object();

    public void Increment()
    {
        // Only one thread can enter this block at a time.
        // Other threads will be blocked by the OS until the lock is released.
        lock (_syncRoot)
        {
            _count++;
        }
    }
}
```
**Internals:** Every reference type object on the Managed Heap has a SyncBlock Index. `Monitor` utilizes this SyncBlock to manage the lock. Do not lock on `this` or strings, as they can be accessed globally, causing unexpected deadlocks. Always create a private `readonly object _syncRoot`.

### `SemaphoreSlim` (Async Locks)
You **cannot** use the `lock` keyword inside an `async` method if there is an `await` inside the lock block. The thread that enters the lock might yield, and a completely different thread might try to resume it, violating thread affinity.

Instead, use `SemaphoreSlim`.

```csharp
public class RateLimiter
{
    // Allows only 5 concurrent executions.
    private readonly SemaphoreSlim _semaphore = new SemaphoreSlim(5, 5);

    public async Task ProcessAsync()
    {
        await _semaphore.WaitAsync(); // Suspends asynchronously without blocking the thread
        try
        {
            await Task.Delay(100); // Simulate work
        }
        finally
        {
            _semaphore.Release();
        }
    }
}
```

## 6. Concurrent Collections

Never use a standard `Dictionary<TKey, TValue>` across multiple threads. If a thread resizes the dictionary's internal array while another thread is reading it, you will get a runtime crash or an infinite loop.

Use `System.Collections.Concurrent`.

```csharp
private ConcurrentDictionary<string, ChargerState> _states = new();

public void UpdateState(string id, ChargerState state)
{
    // Safe for multi-threading. Does NOT lock the entire dictionary.
    // It uses granular lock striping on specific internal buckets.
    _states.AddOrUpdate(id, state, (key, oldValue) => state);
}
```

## 7. Real Production Case Study: Channels

In our EV Platform, we have an ingestion pipeline. 10,000 telemetry packets arrive per second. We must write them to a database. 

If we write them individually, the database will crash. We need a **Producer/Consumer** pattern to buffer the packets and write them in bulk (batching).

Historically, developers used `BlockingCollection`. Today, the standard for ultra-high-performance asynchronous queues is `System.Threading.Channels`.

```csharp
using System.Threading.Channels;

public class TelemetryPipeline
{
    // A bounded channel prevents OutOfMemoryExceptions if producers outpace consumers.
    private readonly Channel<TelemetryPacket> _channel = Channel.CreateBounded<TelemetryPacket>(
        new BoundedChannelOptions(100_000)
        {
            FullMode = BoundedChannelFullMode.Wait,
            SingleReader = true,  // Optimization: tells the compiler we only have 1 consumer
            SingleWriter = false  // Multiple web threads produce data
        });

    // PRODUCER: Called by Web API threads
    public async ValueTask EnqueueAsync(TelemetryPacket packet)
    {
        await _channel.Writer.WriteAsync(packet);
    }

    // CONSUMER: Runs as a long-running background service
    public async Task StartProcessingAsync(CancellationToken ct)
    {
        // ReadAllAsync yields asynchronously until data is available. 
        // No threads are blocked while waiting!
        await foreach (var packet in _channel.Reader.ReadAllAsync(ct))
        {
            await ProcessAndSaveToDbAsync(packet);
        }
    }
    
    private async Task ProcessAndSaveToDbAsync(TelemetryPacket packet) { /* DB Logic */ }
}
```

## 8. Common Mistakes

| Level | Mistake | Impact | How to Avoid |
|-------|---------|--------|--------------|
| Beginner | Spawning `new Thread()` in a loop. | Thread Thrashing, OOM, application crash. | Use `Task.Run` to let the ThreadPool manage thread allocation efficiently. |
| Intermediate | Using `Task.Run` for I/O operations (like HTTP calls). | Wastes ThreadPool threads. | `Task.Run` is ONLY for CPU-bound work. Use `await client.GetAsync()` for I/O. |
| Senior | Locking on `typeof(MyClass)` or `this`. | Unpredictable cross-component deadlocks. | Always instantiate a private, dedicated `object` for synchronization locking. |
| Architect | Unbounded Queues in a Producer/Consumer pattern. | Memory Leaks. If writing to the DB is slow, the queue will grow until the server hits an OutOfMemoryException. | Use Bounded Channels. Apply backpressure so producers slow down if the queue is full. |

## 9. Summary
Concurrency is dangerous. We have seen how raw OS threads consume physical resources and how the .NET ThreadPool abstracts this away using Work Stealing. By leveraging `SemaphoreSlim` for asynchronous synchronization and `System.Threading.Channels` for lock-free producer/consumer pipelines, we can architect systems that process millions of records without corrupting shared memory or blocking the CPU.

In the next chapter, we move into the final frontier: Enterprise Architecture. We will explore how ASP.NET Core wires all these concepts together to process HTTP requests using Kestrel and Dependency Injection.
