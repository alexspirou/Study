# Stack vs Heap in .NET

In .NET, memory is divided into two main regions for execution:  

- **Stack** → fast, structured memory for local execution context.  
- **Heap** → slower, flexible memory for objects with dynamic lifetimes.  

---

## 📍 Stack
- **What it is:** A *Last-In, First-Out (LIFO)* region of memory.  
- **Used for:**
  - Method call frames (activation records).
  - Local value types (`int`, `struct`).
  - References (pointers) to objects on the heap.
- **Allocation:** Very fast (just moves the stack pointer).  
- **Deallocation:** Automatic when method returns (stack pointer moves back).  
- **Lifetime:** Tied to the method scope.  
- **Thread-bound:** Each thread has its own stack.

✅ Example:
```csharp
void Foo()
{
    int x = 42; // Stored directly on the stack
    Bar(x);
}

void Bar(int y)
{
    int z = y + 1; // Also on the stack
}
```

📊 Memory layout:
```
Stack:
| z = 43 |
| y = 42 |
| return addr |
| x = 42 |
```

---

## 📍 Heap
- **What it is:** A large pool of memory managed by the **.NET Garbage Collector (GC)**.  
- **Used for:**
  - Reference types (`class`, arrays, strings, delegates, etc.).
  - Boxed value types (`object o = 42;`).
- **Allocation:** Slower than stack (needs GC bookkeeping).  
- **Deallocation:** Done automatically by GC (not tied to method exit).  
- **Lifetime:** Exists as long as references point to it.  

✅ Example:
```csharp
class Person 
{
    public string Name;
}

void Foo()
{
    var p = new Person { Name = "Alex" }; 
    // 'p' reference is on the stack
    // Person object + "Alex" string live on the heap
}
```

📊 Memory layout:
```
Stack:
| p -> 0x1000 |

Heap:
0x1000: Person object
   Name -> 0x2000

0x2000: "Alex" string
```

---

## 📍 Key Differences

| Feature               | Stack 🗂️ | Heap 📦 |
|------------------------|----------|---------|
| **Speed**             | Very fast | Slower |
| **Allocation**        | Simple (pointer move) | Complex (GC, fragmentation) |
| **Deallocation**      | Auto when method ends | Garbage Collector |
| **Lifetime**          | Method scope | Controlled by GC |
| **Used for**          | Value types, references, call frames | Reference types, boxed values |
| **Thread safety**     | Each thread has its own stack | Shared across threads |

---

## 📍 Special Notes in .NET
- **Value types in classes**  
  Even though `int` is a value type, if it’s inside a class, it’s stored on the heap (as part of the object).  

  ```csharp
  class Box { public int N; }
  var b = new Box { N = 5 }; // N stored inside heap object
  ```

- **Structs vs Classes**  
  - Structs (`struct`) go on the stack *if they’re locals*.  
  - Classes (`class`) always live on the heap.  

- **Escape analysis** (in newer .NET versions)  
  JIT may keep some objects on the stack if it can prove they never escape (stack allocation optimization).

- **Strings**  
  Strings are reference types → always on the heap (immutable).

---

## 📍 Simple Analogy
- **Stack** = a stack of plates 🍽️ → easy to put on top, easy to take off top, but plates vanish when you leave the kitchen (method ends).  
- **Heap** = a warehouse 📦 → you put boxes anywhere, but you need someone (GC) to clean up later.  

---

# Value Types vs Reference Types in .NET

## 📌 Definition

### 🔹 Value Types
- Stored **directly** (the variable holds the actual data).  
- Typically allocated on the **stack** (unless part of a class or boxed).  
- Examples: `int`, `double`, `bool`, `struct`, `enum`.  

### 🔹 Reference Types
- Variable holds a **reference (pointer)** to the actual object.  
- The actual object lives on the **heap**, while the reference is on the stack (or inside another object).  
- Examples: `class`, `string`, `array`, `delegate`, `object`.  

| Category        | Types                                                                 |
|-----------------|----------------------------------------------------------------------|
| **Value Types** | `struct`, `enum`, `record struct`, `int`, `bool`, `double`, `char`, `decimal`, `long`, `short`, `byte` |
| **Reference Types** | `class`, `record`, `interface`, `delegate`, `array`, `string` |

---

## ⚙️ Memory Behavior

| Feature             | Value Type | Reference Type |
|---------------------|------------|----------------|
| **Storage**         | Data stored directly (stack or inline in object) | Reference stored on stack, object stored on heap |
| **Assignment**      | Creates a **copy** of the value | Copies the **reference**, not the object |
| **Nullability**     | Cannot be `null` (unless `Nullable<T>`) | Can be `null` |
| **Default value**   | Zeroed-out version (e.g., `0`, `false`) | Always `null` by default |
| **Garbage Collection** | Not tracked (stack frames auto-clean) | Managed by GC |
| **Performance**     | Faster (no indirection) | Slightly slower (extra pointer dereference, GC overhead) |

---

## 🧠 Example — Copy Semantics

```csharp
// Value type (struct)
int a = 10;
int b = a;   // copy of value
b = 20;
Console.WriteLine(a); // 10 (unchanged)

// Reference type (class)
class Person { public string Name; }

var p1 = new Person { Name = "Alice" };
var p2 = p1;      // copy of reference
p2.Name = "Bob";  
Console.WriteLine(p1.Name); // "Bob" (same object!)
```

✅ Value types = **independent copies**  
✅ Reference types = **shared object**  

---

## 📦 Boxing & Unboxing

Value types can be **boxed** (wrapped into an object on the heap).  

```csharp
int x = 42;
object obj = x;   // boxing (value → heap)
int y = (int)obj; // unboxing (heap → value)
```

⚠️ Boxing/unboxing is **expensive** (allocations + casting). Avoid in performance-critical code.

---

## 🏗 When to Use

- **Value Types (structs):**
  - Small, immutable data (e.g., `Point`, `DateTime`, `Guid`)
  - High-performance scenarios where avoiding heap allocations matters

- **Reference Types (classes):**
  - Complex objects with identity
  - Large or mutable state
  - Need for polymorphism/inheritance

---

## 🔍 Quick Visual (Stack vs Heap)

```
int a = 5;        // [Stack] a → 5
int b = a;        // [Stack] b → 5 (copy)

Person p1 = new Person("Alice");
// [Stack] p1 → REF123
// [Heap ] REF123 → { Name="Alice" }

Person p2 = p1;
// [Stack] p2 → REF123 (same reference)
// [Heap ] REF123 → { Name="Alice" }
```

---

# Finalizers & IDisposable in C#

## Finalizers (`~ClassName()`)

- Special method (`~ClassName`) called by the **GC** before reclaiming memory.
- **Non-deterministic**: You don’t know *when* it will run.
- Purpose: Cleanup **unmanaged resources** (file handles, sockets, native memory) if `Dispose()` wasn’t called.

### Example
```csharp
public class FileResource
{
    private FileStream _stream;

    public FileResource(string path)
    {
        _stream = new FileStream(path, FileMode.OpenOrCreate);
    }

    ~FileResource()
    {
        _stream?.Dispose();
    }
}
```

---

## IDisposable

- Provides **deterministic cleanup** with `Dispose()`.
- Implemented by classes that use unmanaged or scarce resources.

```csharp
public class FileResource : IDisposable
{
    private FileStream _stream;

    public FileResource(string path)
    {
        _stream = new FileStream(path, FileMode.OpenOrCreate);
    }

    public void Dispose()
    {
        _stream?.Dispose();
        GC.SuppressFinalize(this); // Prevents finalizer
    }
}
```

Usage:
```csharp
using (var res = new FileResource("data.txt"))
{
    // use resource
} // Dispose() called automatically
```

---

## Dispose Pattern (Finalizer + IDisposable)

```csharp
public class ResourceHolder : IDisposable
{
    private IntPtr _nativeHandle;
    private bool _disposed = false;

    public ResourceHolder(IntPtr handle)
    {
        _nativeHandle = handle;
    }

    ~ResourceHolder()
    {
        Dispose(false); // GC calls finalizer
    }

    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this);
    }

    protected virtual void Dispose(bool disposing)
    {
        if (!_disposed)
        {
            if (disposing)
            {
                // free managed resources
            }

            if (_nativeHandle != IntPtr.Zero)
            {
                // free unmanaged resource
                _nativeHandle = IntPtr.Zero;
            }

            _disposed = true;
        }
    }
}
```

---

## Key Differences

| Aspect               | Finalizer (~ClassName) | IDisposable (Dispose) |
|----------------------|------------------------|-----------------------|
| **Who calls it**     | Garbage Collector      | Developer / `using`   |
| **When runs**        | Non-deterministic      | Deterministic         |
| **Performance**      | Expensive (extra GC)   | Cheap                 |
| **Use case**         | Safety net             | Primary cleanup       |

---

## When is `Dispose()` called automatically?

✅ Cases:
- `using` / `using var` statement  
- ASP.NET Core DI container (scoped/singleton disposal)  
- Framework classes disposing children (e.g., `StreamReader` → `FileStream`)  
- `await using` with `IAsyncDisposable`  

❌ Otherwise, you must call `Dispose()` manually.

---

## Best Practices
- Prefer `IDisposable` for deterministic cleanup  
- Use `using` / `using var` to avoid forgetting disposal  
- Only add a finalizer if you directly hold unmanaged resources  
- Always call `GC.SuppressFinalize(this)` inside `Dispose()`  

---



# Managed vs Unmanaged Resources in .NET

---

## ✅ Managed Resources
- **Definition:** Objects that the **.NET Garbage Collector (GC)** knows how to allocate and free.  
- **Examples:**  
  - Classes (`string`, `List<T>`, `FileStream`, `SqlConnection`)  
  - Arrays  
  - Delegates  
- **Lifetime:** Automatically handled by the GC — you don’t need to explicitly free memory.  
- **Disposal:** If a managed object *wraps unmanaged resources* (like `FileStream` wraps an OS file handle), you should still call `Dispose()` so the unmanaged part is released promptly.  

---

## ❌ Unmanaged Resources
- **Definition:** Resources that the **.NET GC cannot clean up automatically**.  
- These are usually **OS-level resources** outside the managed heap.  
- **Examples:**  
  - File handles  
  - Database connections  
  - Network sockets  
  - Window handles (HWND)  
  - Pointers to unmanaged memory (via `Marshal.AllocHGlobal`)  
- **Lifetime:** You must explicitly release them (e.g., via `Dispose()` or `SafeHandle`) or they leak until the process ends.  

---

## 🔑 Key Differences

| Aspect                  | Managed Resources                | Unmanaged Resources                  |
|--------------------------|----------------------------------|---------------------------------------|
| Who manages memory?      | .NET GC                         | You (explicitly)                      |
| Cleanup mechanism        | Automatic via GC                | Manual: `Dispose()`, `SafeHandle`, `Finalize()` |
| Examples                 | `List<int>`, `string`, `StreamReader` | File handles, sockets, GDI objects, unmanaged memory |
| Risk if not freed        | Temporary memory pressure       | Permanent leaks, OS instability       |

---

## ⚙️ How .NET Handles Both
- For **purely managed objects** → GC is enough.  
- For **managed wrappers around unmanaged resources** → implement **`IDisposable`** so you can free unmanaged resources *as soon as possible*.  
- If you forget `Dispose()`, the **finalizer** (`~ClassName()`) is a safety net, but runs late and non-deterministically.  

---

## ✅ Example: Managed + Unmanaged Together
```csharp
public class FileWriter : IDisposable
{
    private FileStream _file; // wraps an unmanaged file handle

    public FileWriter(string path)
    {
        _file = new FileStream(path, FileMode.Create);
    }

    public void Write(string text)
    {
        var bytes = Encoding.UTF8.GetBytes(text);
        _file.Write(bytes, 0, bytes.Length);
    }

    public void Dispose()
    {
        // Dispose releases the unmanaged file handle immediately
        _file.Dispose();
    }
}
```

- `FileStream` is a **managed class**,  
- but internally it holds an **unmanaged OS file handle**,  
- so calling `Dispose()` ensures the handle is closed right away (instead of waiting for GC).  

# Memory Leaks in C# (.NET)

Even though .NET uses **automatic garbage collection (GC)**, memory leaks can still happen when objects remain **reachable** but are no longer needed.  
The GC only frees memory for objects that are truly **unreachable** — if something still holds a reference, memory stays allocated.

---

## 🔑 Common Causes of Memory Leaks in C#

###  Event Handlers Not Unsubscribed
- If you subscribe to an event but never unsubscribe, the **publisher** holds a reference to the **subscriber**.  
- As long as the publisher is alive, the subscriber cannot be garbage collected.

```csharp
public class Publisher
{
    public event EventHandler? SomethingHappened;
    public void Raise() => SomethingHappened?.Invoke(this, EventArgs.Empty);
}

public class Subscriber
{
    public Subscriber(Publisher pub)
    {
        pub.SomethingHappened += Handle; // ❌ never unsubscribed
    }

    private void Handle(object? sender, EventArgs e) { }
}
```

✅ **Fix:** Unsubscribe when done:
```csharp
pub.SomethingHappened -= Handle;
```

Or use **weak events** (`WeakEventManager`).

---

### Static References
- Objects referenced by static fields live for the entire application lifetime.
- If large objects or collections are stored statically and never cleared → **permanent memory leak**.

```csharp
public static class Cache
{
    public static List<object> Data = new(); // ❌ grows forever
}
```

✅ **Fix:** Use `MemoryCache`, clear collections, or weak references.

---

### IDisposable Not Used
- Resources like streams, sockets, database connections, or UI handles hold **unmanaged memory**.
- If `Dispose()` isn’t called, unmanaged memory leaks even if the object is collected.

```csharp
var file = new FileStream("data.txt", FileMode.Open);
// ❌ no Dispose → file handle leak
```

✅ **Fix:** Always use `using`:
```csharp
using var file = new FileStream("data.txt", FileMode.Open);
```

---

### Long-Lived Collections
- If objects are added to a collection (e.g., `List`, `Dictionary`) and never removed, they remain in memory.

```csharp
private static List<User> _users = new(); // ❌ grows forever
```

✅ Remove items when not needed or use **weak references**.

---

### Closures Capturing Variables
- Lambdas or async callbacks can **capture local variables** and extend their lifetime unintentionally.

```csharp
void Register(Action action)
{
    _actions.Add(action);
}

void Example()
{
    var data = new object();
    Register(() => Console.WriteLine("Captured variable still alive")); // ❌ data kept alive indirectly
}
```

✅ Be careful what lambdas capture.

---

### Timer/Background Threads
- `System.Timers.Timer` or `Task.Run` can keep delegates and state alive.
- If timers are not stopped/disposed, they keep references forever.

```csharp
var timer = new Timer(_ => Console.WriteLine("tick"), null, 0, 1000);
// ❌ if not disposed, keeps delegate alive
```

✅ Dispose timers when no longer needed.

---

### Pinned Objects (GCHandle, fixed)
- If you **pin objects** (e.g., interop, unsafe code), they cannot be moved/collected.

```csharp
GCHandle handle = GCHandle.Alloc(obj, GCHandleType.Pinned);
// ❌ if never Free() → leak
```

---

## 🔍 How to Detect Memory Leaks
- **Profilers**: dotMemory, ANTS, Visual Studio Diagnostic Tools.
- **GC.GetTotalMemory(true)** to monitor usage.
- **Dump Analysis**: `dotnet-dump`, WinDbg + SOS.

---

##  ✅ Best Practices to Avoid Memory Leaks
1. Always `Dispose` (`using` blocks).
2. Unsubscribe from events.
3. Avoid unnecessary static references.
4. Manage collections properly (remove old items).
5. Be careful with lambdas/closures.
6. Dispose timers and background workers.
7. Use `WeakReference` when appropriate.

---

📌 **Key Takeaway:**  
A memory leak in .NET doesn’t mean memory “disappears” — it means objects are **still referenced** and thus not collectible.  
Most leaks come from **unintended object retention** (events, static references, collections), not from the GC itself.

# 📝 Span<T> and ByRef in .NET

## 1. What is `Span<T>`?
- A **stack-only struct (`ref struct`)** that represents a view over contiguous memory.  
- Internally it has **two fields**:

```csharp
public readonly ref struct Span<T>
{
    private readonly ByReference<T> _pointer; // managed pointer → first element
    private readonly int _length;             // number of elements in the span
}
```

So:  
- **Pointer (`ByReference<T>`)** → where the memory starts (address of the first element).  
- **Length (`int`)** → how many elements you can safely access.  

---

## 2. What problem does it solve?
- **Avoids copies**: no need to create substrings or new arrays.  
- **Unified access**: same API for arrays, strings, stackalloc, unmanaged buffers.  
- **High performance**: slicing is O(1) (just moves pointer + updates length).  

---

## 3. Limits of `Span<T>`
- Cannot be stored on the **heap** (class field, boxing).  
- Cannot cross **`await`/`yield`** boundaries.  
- Cannot outlive its underlying memory.  
- Lifetime is restricted → enforced by the compiler.  

---

## 4. What is ByRef?
- A **managed pointer** type (CLR concept).  
- Behaves like a pointer but is **GC-safe** (runtime tracks it).  
- Used in:
  - `ref`, `out`, `in` parameters  
  - `ref locals` & `ref returns`  
  - `Span<T>` internals  

---

## 5. Inside the fields

| Field    | Meaning | Example |
|----------|---------|---------|
| `_pointer` (ByReference<T>) | Managed pointer → address of the **first element** | If `Span<int>` is over `{10,20,30}`, `_pointer` → element `10` |
| `_length` (`int`) | Number of elements that can be accessed | For slice `[20,30]`, `_length = 2` |

Indexing (`span[i]`) = `*(pointer + i)` with bounds check (`i < length`).  

---

## 6. C++ Pointer vs .NET ByRef

| Feature                | C++ Pointer (`int*`) | C# ByRef (`ref int`) |
|------------------------|----------------------|-----------------------|
| Holds memory address   | ✅ Yes               | ✅ Yes                |
| Holds length info      | ❌ No                | ❌ No (but Span adds it) |
| GC safe                | ❌ No                | ✅ Yes                |
| Pointer arithmetic     | ✅ Allowed           | ❌ Not allowed        |
| Can dangle (invalid)   | ✅ Yes               | ❌ No (runtime safe)  |

---

## 7. Key Insight
- `Span<T>` = **(managed pointer → first element) + (int length)**.  
- Indexer does → **pointer + index**, then bounds-check against `length`.  
- Safe + allocation-free + high-performance.  

---

✅ **One-liner:**  
`Span<T>` is a `(pointer, length)` pair stored on the stack, giving you a GC-safe window into memory.




# ⚡ Fast Span vs 🐢 Slow Span (with `char`)

---

## 1. ⚡ Fast Span → `stackalloc char[...]`

```csharp
void Fast()
{
    Span<char> span = stackalloc char[4]; // buffer on stack
    span[0] = 'A';
    span[1] = 'B';
    span[2] = 'C';
    span[3] = 'D';

    foreach (char c in span)
        Console.WriteLine(c);
} // buffer disappears here
```

### Stack layout
```
stackalloc buffer (stack memory)
 ┌─────┬─────┬─────┬─────┐
 │ 'A' │ 'B' │ 'C' │ 'D' │
 └─────┴─────┴─────┴─────┘

Span<char> on stack
 ┌──────────────┬──────────────┐
 │ pointer ---> │ buffer[0]    │
 │ length = 4   │              │
 └──────────────┴──────────────┘
```

✅ Fast: the span points **directly** to stack memory.  
✅ No heap involved.  

---

## 2. 🐢 Slow Span → via `Memory<char>`

```csharp
void Slow()
{
    Memory<char> mem = new char[] { 'A', 'B', 'C', 'D' }; // array on heap
    Span<char> span = mem.Span; // Span built from Memory<char>

    foreach (char c in span)
        Console.WriteLine(c);
}
```

### Heap layout
```
char[] array (heap)
 ┌───────────┬───────────────┐
 │ Header    │ Length = 4    │
 ├───────────┼───────────────┤
 │ [0]='A'   │ [1]='B'       │
 │ [2]='C'   │ [3]='D'       │
 └───────────┴───────────────┘

Memory<char> (heap)
 ┌──────────────────────┐
 │ ref -> char[] array  │
 │ start = 0, len = 4   │
 └──────────────────────┘
```

### Stack layout
```
Span<char> on stack
 ┌─────────────┬─────────────┐
 │ pointer --->│ array[0]    │ (resolved through Memory<char>)
 │ length = 4  │             │
 └─────────────┴─────────────┘
```

❌ Slightly slower:  
- You first create a `Memory<char>` object on the **heap**.  
- Then `.Span` translates it into a `Span<char>` on the stack.  
- That extra heap indirection = **slow span** compared to stackalloc.

# Bonus: Why a Struct Must Be Boxed When Used as an Interface in .NET

- **Interface variables hold references**  
  An interface value is always a reference to an object that implements it.

- **Structs are not references**  
  A `struct` is raw bytes with no metadata.  

- **Interface dispatch needs object metadata**  
  Requires header, method table pointer, and interface map.  

- **Boxing provides missing pieces**  
  Wraps struct into heap object with metadata so interface calls work.  

### Example
```csharp
interface IShow { void Show(); }

struct S : IShow 
{
    public void Show() => Console.WriteLine("S");
}

S s = new();
s.Show();        // direct call, no boxing

IShow i = s;     // boxing happens here
i.Show();        // interface dispatch on boxed object
```

✅ Use generics with constraints to avoid boxing:
```csharp
void Call<T>(T x) where T : IShow
{
    x.Show(); // constrained call → no boxing
}
```
