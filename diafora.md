## 30) What is `Span<T>` in .NET, and why is it fast?
**Answer:**  
- `Span<T>` is a **stack-only `ref struct`** that references a contiguous region of memory (pointer + length).  
- Zero-allocation views/slices, **bounds-checked** and **safe**.  
- Great for parsers/encoders/formatters to avoid copying.

**Examples:**
```csharp
// Stack-allocated buffer (no GC)
Span<int> buffer = stackalloc int[3];
buffer[0] = 42; buffer[1] = 100; buffer[2] = 7;

// View over an array (zero copy)
int[] arr = {1, 2, 3, 4};
Span<int> span = arr;
int v = span[2]; // bounds-checked

// View over string chars without allocation
string s = "hello";
ReadOnlySpan<char> chars = s.AsSpan(); // zero-copy slice
var slice = chars.Slice(1, 3); // "ell"
```





## 20) Outbox Pattern (reliable messaging with DB + broker)
**Answer:**  
Avoids losing messages if your service crashes **between** DB commit and message publish.

**How:**  
1) In the **same transaction** as your write, insert an **OutboxMessage** row.  
2) A background worker reads undispatched rows and publishes to the broker.  
3) After successful publish, mark the row as dispatched.

**Sketch with EF Core:**
```csharp
public class OutboxMessage
{
    public Guid Id { get; set; }
    public string Type { get; set; } = "";
    public string Payload { get; set; } = "";
    public DateTime OccurredOnUtc { get; set; }
    public DateTime? DispatchedOnUtc { get; set; }
}

public async Task AddModuleAsync(Module m, CancellationToken ct)
{
    using var tx = await _db.Database.BeginTransactionAsync(ct);

    _db.Modules.Add(m);
    _db.Outbox.Add(new OutboxMessage
    {
        Id = Guid.NewGuid(),
        Type = "ModuleAdded",
        Payload = JsonSerializer.Serialize(new { m.Id, m.Name }),
        OccurredOnUtc = DateTime.UtcNow
    });

    await _db.SaveChangesAsync(ct);
    await tx.CommitAsync(ct); // State + outbox committed atomically
}

// Hosted Service: poll and publish
protected override async Task ExecuteAsync(CancellationToken stoppingToken)
{
    while (!stoppingToken.IsCancellationRequested)
    {
        var batch = await _db.Outbox
            .Where(x => x.DispatchedOnUtc == null)
            .OrderBy(x => x.OccurredOnUtc)
            .Take(100)
            .ToListAsync(stoppingToken);

        foreach (var msg in batch)
        {
            await _messageBus.PublishAsync(msg.Type, msg.Payload, stoppingToken); // RabbitMQ/Kafka
            msg.DispatchedOnUtc = DateTime.UtcNow;
        }

        await _db.SaveChangesAsync(stoppingToken);
        await Task.Delay(1000, stoppingToken);
    }
}
```



## 22) Singleton service vs static class (e.g., error parser)
**Answer:**  
- **Singleton service**: DI-friendly, configurable, mockable, easier to test and evolve.  
- **Static class**: no DI, global state risk, hard to test/mock.

**Example (DI):**
```csharp
public interface IErrorParser
{
    string Parse(Exception ex);
}

public sealed class ErrorParser : IErrorParser
{
    private readonly ILogger<ErrorParser> _logger;
    public ErrorParser(ILogger<ErrorParser> logger) => _logger = logger;

    public string Parse(Exception ex)
    {
        _logger.LogError(ex, "Parsing error");
        return ex.Message;
    }
}

// Program.cs / Startup
services.AddSingleton<IErrorParser, ErrorParser>();

// Usage (constructor injection)
public class Foo
{
    private readonly IErrorParser _parser;
    public Foo(IErrorParser parser) => _parser = parser;

    public string Handle(Exception ex) => _parser.Parse(ex);
}
```
---

## 24) Kestrel/ASP.NET request threading timeline (mental model)
**Answer:**  
0️⃣ App starts (`Main`)  
1️⃣ Kestrel begins listening  
2️⃣ Socket I/O thread accepts a connection  
3️⃣ Work is queued to the ThreadPool  
4️⃣ Controller action executes on a ThreadPool thread  
5️⃣ At `await` (I/O), the thread is released (non-blocking)  
6️⃣ When I/O completes, continuation resumes on a ThreadPool thread  
7️⃣ Kestrel writes the response

**Key:** `await` frees threads; **no thread is held** during I/O waits.

---
---

## 29) SQL & Microservices — high-level principles (bonus)
- Prefer **idempotent** handlers for retries.  
- Consider **sagas**/process managers for long-running, multi-service workflows.  
- Use **DLQs** (dead-letter queues) and **retry policies** for resilience.  
- Use **observability**: structured logs, metrics, tracing (e.g., OpenTelemetry + Grafana).

---