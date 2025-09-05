# 📚 MYQUESTIONS

---

## 1) What is the difference between `Task` and `ValueTask`?
**Answer:**  
- `Task<T>` is a **reference type** → always allocates on the heap.  
- `ValueTask<T>` is a **value type (struct)** → avoids allocation **only** when constructed from an already-available result (e.g., from cache).  
- Prefer `ValueTask` in **hot paths** where results are usually synchronous. Profile first.  
- Downsides: can only be awaited once, more complex to compose, and misuse can hurt performance.

**Example — cache vs DB:**
```csharp
private readonly Dictionary<int, UserProfile> _cache = new();

public ValueTask<UserProfile> GetUserProfileAsync(int userId)
{
    if (_cache.TryGetValue(userId, out var profile))
    {
        // ✅ No heap allocation — result is already available
        return new ValueTask<UserProfile>(profile);
    }

    // ⛔ Allocation has already happened inside the async method (state machine + Task)
    return new ValueTask<UserProfile>(LoadUserProfileFromDbAsync(userId));
}

private async Task<UserProfile> LoadUserProfileFromDbAsync(int userId)
{
    await Task.Delay(200); // ⛔ async state machine allocation
    return new UserProfile { Id = userId, Name = "John Doe" }; // ⛔ heap object allocation
}

public sealed class UserProfile
{
    public int Id { get; init; }
    public string Name { get; init; } = "";
}
```

---

## 2) Why can’t you await a `ValueTask` more than once?
**Answer:**  
- `ValueTask<T>` is a **struct**, and awaiting it **consumes internal state**.  
- Structs are copied by value; re-awaiting can access invalid/corrupted state → exceptions.  
- If you must await the same operation multiple times, use `.AsTask()` (allocates, but reusable).

**Example:**
```csharp
async ValueTask<int> GetNumberAsync()
{
    await Task.Delay(50);
    return 42;
}

async Task Demo()
{
    ValueTask<int> vt = GetNumberAsync();

    int a = await vt; // ✅ OK
    int b = await vt; // ❌ InvalidOperationException: ValueTask can only be awaited once

    // ✅ Safe pattern when you need multiple awaits:
    ValueTask<int> vt2 = GetNumberAsync();
    int x = await vt2.AsTask();
    int y = await vt2.AsTask();
}
```

---

## 3) Is `ContinueWith(...)` like an event that triggers when a Task finishes?
**Answer:**  
Yes. Both `ContinueWith` and `await` attach **continuations**.  
- `ContinueWith`: manual continuation, easy to misuse (scheduler, exceptions).  
- `await`: compiler sugar over `GetAwaiter().OnCompleted(...)` with **better exception propagation** and **context handling**.

**Example (idiomatic vs continuation):**
```csharp
// Idiomatic with await
public async Task<string> GetWeatherSummaryAsync(string city)
{
    int t = await _service.GetTemperatureAsync(city);
    return t >= 30 ? "Hot" : t >= 20 ? "Warm" : "Cold";
}

// Manual continuation version
public Task<string> GetWeatherSummary_ContinuationAsync(string city)
{
    var tcs = new TaskCompletionSource<string>();

    _service.GetTemperatureAsync(city).ContinueWith(task =>
    {
        if (task.IsFaulted)
        {
            tcs.SetException(task.Exception!);
            return;
        }

        int t = task.Result;
        string summary = t >= 30 ? "Hot" : t >= 20 ? "Warm" : "Cold";
        tcs.SetResult(summary);
    });

    return tcs.Task;
}
```

---

## 4) How does `await` work behind the scenes?
**Answer:**  
- The compiler transforms an `async` method into a **state machine** with a hidden `MoveNext()` method.  
- On `await`, it fetches an awaiter, checks completion, and if incomplete, **registers a continuation** via `OnCompleted`.  
- When the awaited Task completes, the continuation invokes `MoveNext()` to resume execution.

**Example — idiomatic vs conceptual compiled form:**
```csharp
// Idiomatic
public async Task<string> GetWeatherAsync(string city)
{
    Console.WriteLine("[LOG] Fetching weather...");
    int temp = await _service.GetTemperatureAsync(city);
    return temp >= 30 ? "Hot" : temp >= 20 ? "Warm" : "Cold";
}

// Conceptual (not exact compiler output, but equivalent idea)
public Task<string> GetWeatherCompiledLikeAsync(string city)
{
    var tcs = new TaskCompletionSource<string>();
    Console.WriteLine("[LOG] Fetching weather...");

    var task = _service.GetTemperatureAsync(city);
    if (task.IsCompletedSuccessfully)
    {
        int temp = task.Result;
        tcs.SetResult(temp >= 30 ? "Hot" : temp >= 20 ? "Warm" : "Cold");
    }
    else
    {
        task.GetAwaiter().OnCompleted(() =>
        {
            try
            {
                int temp = task.GetAwaiter().GetResult();
                tcs.SetResult(temp >= 30 ? "Hot" : temp >= 20 ? "Warm" : "Cold");
            }
            catch (Exception ex)
            {
                tcs.SetException(ex);
            }
        });
    }

    return tcs.Task;
}
```

---

## 5) When is `ValueTask` preferred over `Task`?
**Answer:**  
- In high-performance code paths where the result is **usually available synchronously**, such as cache hits or precomputed values.  
- Avoid overuse; complexity can outweigh benefits unless profiling shows it helps.

**Example — fast-path cache hit:**
```csharp
public ValueTask<Product> GetProductAsync(int id)
{
    if (_cache.TryGetValue(id, out var product))
        return new ValueTask<Product>(product);   // ✅ zero allocation

    return new ValueTask<Product>(FetchFromDbAsync(id)); // wraps Task (alloc already happened)
}
```

---

## 6) How can you limit concurrent async operations in ASP.NET Core (without blocking threads)?
**Answer:**  
Use `SemaphoreSlim.WaitAsync()` to bound concurrency in an async-friendly way (unlike `lock`, which blocks threads).

**Example:**
```csharp
private static readonly SemaphoreSlim _semaphore = new(3, 3); // allow up to 3 concurrent ops

[HttpGet("weather")]
public async Task<IActionResult> GetWeather(string city)
{
    await _semaphore.WaitAsync(); // ✅ non-blocking wait
    try
    {
        string json = await _httpClient.GetStringAsync($"https://api.example.com?q={city}");
        return Content(json, "application/json");
    }
    finally
    {
        _semaphore.Release();
    }
}
```

---

## 7) Does an `async` method allocate on the heap?
**Answer:**  
Yes. Marking a method `async` typically creates:  
- A **state machine** (class) and a **Task** object (heap).  
- Any objects you `new` inside also allocate.

**Example:**
```csharp
private async Task<UserProfile> LoadAsync()
{
    await Task.Delay(200);                 // ⛔ async state machine + Task allocation
    return new UserProfile();              // ⛔ heap object
}
```

---

## 8) Does `ValueTask` allocate?
**Answer:**  
- `ValueTask<T>` itself is a **struct** → no allocation.  
- But if it **wraps a `Task<T>`**, that Task has already allocated.  
- Only `new ValueTask<T>(value)` truly avoids allocations.

---

## 9) Why SynchronizationContext Can Cause Deadlocks

If you block a thread (e.g., with `.Result` or `.Wait()`), and an async
method tries to resume on that same thread, a **deadlock** occurs.

------------------------------------------------------------------------

Example: UI Thread Deadlock

``` csharp
public string GetData()
{
    return GetDataAsync().Result; // ❌ blocks UI thread
}

public async Task<string> GetDataAsync()
{
    await Task.Delay(1000); // tries to resume on UI thread
    return "done"; 
}
```

------------------------------------------------------------------------

 Explanation

-   UI thread is **blocked** by `.Result`
-   Continuation tries to run on the **UI thread**
-   Both wait for each other → **deadlock**

✅ **Fix**: use `await` all the way, or `ConfigureAwait(false)` to avoid
capturing the context.

------------------------------------------------------------------------
 🧠 Analogy

Think of it like:

-   **UI data** = your house
-   **UI thread** = the street where your house is\
-   **SynchronizationContext** = the GPS instructions to get back to
    that street

When you `await`, the continuation saves the GPS.
When the awaited task finishes, it uses the GPS to get back to the right
street (thread) where your house (UI data) lives.

------------------------------------------------------------------------
ASP.NET Core Controller Example

``` csharp
using Microsoft.AspNetCore.Mvc;
using System;
using System.Threading.Tasks;

[ApiController]
[Route("api/[controller]")]
public class DeadlockController : ControllerBase
{
    // ❌ Deadlock example
    [HttpGet("deadlock")]
    public string GetDeadlock()
    {
        // Blocking on async call — captures context by default
        var s = GetStringAsyncCaptureContext("https://example.com").Result;
        return s;
    }

    // ✅ Fixed by going fully async
    [HttpGet("fixed")]
    public async Task<string> GetFixed()
    {
        var s = await GetStringAsyncCaptureContext("https://example.com");
        return s;
    }

    // ✅ Also avoids deadlock even if blocking, because ConfigureAwait(false) skips context capture
    [HttpGet("no-deadlock-even-if-blocking")]
    public string NoDeadlockEvenIfBlocking()
    {
        var s = GetStringAsyncNoCapture("https://example.com").GetAwaiter().GetResult();
        return s;
    }

    private async Task<string> GetStringAsyncCaptureContext(string url)
    {
        await Task.Delay(100); // captures SynchronizationContext (default)
        return $"[OK] {url} @ {DateTime.UtcNow:O}";
    }

    private async Task<string> GetStringAsyncNoCapture(string url)
    {
        await Task.Delay(100).ConfigureAwait(false); // does not capture SynchronizationContext
        return $"[OK] {url} @ {DateTime.UtcNow:O}";
    }
}
```


---

## 10) Common blocking calls vs async alternatives
| ❌ Blocking call                       | ✅ Async alternative                          | Why async is better                         |
|---------------------------------------|-----------------------------------------------|---------------------------------------------|
| `Thread.Sleep(ms)`                    | `await Task.Delay(ms)`                        | Frees thread while waiting                  |
| `task.Wait()`                         | `await task`                                  | Avoids deadlocks + frees thread             |
| `task.Result`                         | `var r = await task`                          | Same                                        |
| `SemaphoreSlim.Wait()`                | `await SemaphoreSlim.WaitAsync()`             | Non-blocking coordination                   |
| `ManualResetEvent.WaitOne()`          | `TaskCompletionSource` + `await`              | Async signalling                            |
| `lock(obj) { ... }`                   | `await semaphore.WaitAsync(); ... Release()`  | Limits concurrency without blocking threads |

---

## 11) What are the main benefits of asynchronous programming?
**Answer:**  
1. **Responsive GUI apps** — UI thread stays free.  
2. **Scalable servers** — threads aren’t held while waiting on I/O, so you can handle more concurrent requests.

---

## 12) When do you need synchronization to protect shared data?
**Answer:** Only when **all three** are true:  
1) Multiple pieces of code run **concurrently**.  
2) They **access the same data**.  
3) At least one **writes**.

Use `SemaphoreSlim`, channels, immutable data, or message-passing patterns.

## 13) When Task.Run runs ? in which thread?

 🕗 When does `Task.Run` run?

-   `Task.Run` **schedules** the work to run **immediately** (as soon as
    possible).
-   It puts the work in the **.NET ThreadPool**, not in a brand-new
    thread.
-   The **ThreadPool** decides which available thread executes it.

------------------------------------------------------------------------

 🧵 Which thread executes it?

-   **Not** the thread that called `Task.Run` (unless it was already
    free and reused).
-   Instead, it's picked up by an **idle worker thread** from the
    ThreadPool.
-   The ThreadPool has a pool of threads managed by the runtime:
    -   They're reused across tasks.
    -   Their number auto-adjusts depending on workload.

------------------------------------------------------------------------

 ✅ Example

``` csharp
Console.WriteLine($"Main thread: {Thread.CurrentThread.ManagedThreadId}");

await Task.Run(() =>
{
    Console.WriteLine($"Task is running on thread: {Thread.CurrentThread.ManagedThreadId}");
});
```

 Possible output:

    Main thread: 1
    Task is running on thread: 6

-   Thread `1` is your **main thread**.\
-   Thread `6` is a **ThreadPool worker thread**.

------------------------------------------------------------------------

 🔑 Key Points

-   `Task.Run` ≠ new thread\
    (It **uses** a ThreadPool thread, not a fresh one each time.)
-   It runs **as soon as a worker is available**.
-   Useful for **CPU-bound work** to avoid blocking the calling thread
    (e.g., UI thread).


## 14) `await a Task` vs `Task.Run`

------------------------------------------------------------------------

| Aspect              | `await task` |
|---------------------|--------------|
| Where it runs       | Same thread / I/O system (does not switch thread) |
| Best for            | **I/O-bound work** (web requests, DB calls, file reads) |
| Return value        | Returns the awaited result when the Task completes |
| Risk                | ❌ None — just waiting for an already running Task |
``` csharp
public async Task<string> GetDataAsync()
{
    // I/O bound, no extra thread used
    string data = await new HttpClient().GetStringAsync("https://example.com");
    return data;
}
```

✅ Best for: **I/O-bound work** (database calls, file reads, web
requests).

------------------------------------------------------------------------

| Aspect              | `Task.Run` |
|---------------------|------------|
| Where it runs       | On a **ThreadPool worker thread** immediately |
| Best for            | **CPU-bound work** (loops, calculations) |
| Return value        | Returns a `Task`, which you can `await` |
| Risk                | ⚠ If not awaited or returned → becomes fire-and-forget |

``` csharp
public async Task<long> CalculateAsync()
{
    // CPU bound, offloaded to ThreadPool
    long sum = await Task.Run(() =>
    {
        long total = 0;
        for (int i = 0; i < 10_000_000; i++) total += i;
        return total;
    });

    return sum;
}
```

✅ Best for: **CPU-bound work** (calculations, image processing,
parsing).

------------------------------------------------------------------------

 ✅ Comparison Table

  -----------------------------------------------------------------------
  Feature                            `await task`        `Task.Run`
  ---------------------------------- ------------------- ----------------
  Starts new work?                   ❌ No               ✅ Yes

  Where it runs                      Same thread / I/O   ThreadPool
                                     system              worker thread

  Purpose                            Asynchronously wait Offload
                                                         CPU-bound work

  Typical use case                   I/O-bound (web, DB, CPU-bound
                                     file)               (loops,
                                                         calculations)

Fire-and-forget risk ❌ None (task is ⚠ If you don’t already running) keep/await the Task

  -----------------------------------------------------------------------
 🧠 Rule of Thumb

-   Use **`await`** for async APIs (I/O).\
-   Use **`Task.Run`** to push CPU-heavy work off the main/request
    thread.\
-   Avoid mixing them unnecessarily:

``` csharp
// ❌ Anti-pattern
await Task.Run(() => httpClient.GetStringAsync("https://example.com"));
```

(You already had an async API --- no need to wrap it in Task.Run.)


## 23) When should you use parallel programming?
**Answer:**  
- Use when work is **CPU-bound** and **independent** (e.g., image processing, numeric computation).  
- Client apps benefit more; servers already scale via request concurrency, and adding CPU parallelism can reduce throughput under load.

**Example (CPU-bound partitioned work):**
```csharp
int[] data = Enumerable.Range(1, 5_000_000).ToArray();
long sum = 0;

Parallel.ForEach(Partitioner.Create(0, data.Length), range =>
{
    long local = 0;
    for (int i = range.Item1; i < range.Item2; i++)
        local += data[i];

    Interlocked.Add(ref sum, local);
});

Console.WriteLine(sum);
```


## 25) Simple console example showing thread interaction with `await`
**Answer:**  
- `await Task.Delay(...)` doesn’t block; it schedules a resume later.  
- Fire-and-forget calls won’t be awaited unless you explicitly do so.

**Example:**
```csharp
class Program
{
    static async Task Main()
    {
        await TestReturn();  // awaited
        TestAwait();         // fire-and-forget (not recommended in real apps)
        await Task.Delay(10000); // keep process alive to observe output
    }

    private static async Task TestAwait()
    {
        await Delay(5000, "TestAwait");
    }

    private static Task TestReturn()
    {
        return Delay(5000, "TestReturn");
    }

    private static async Task Delay(int delay, string name)
    {
        Console.WriteLine(name);
        await Task.Delay(delay); // thread is freed here; continuation later
        Console.WriteLine("Completed");
    }
}
```





