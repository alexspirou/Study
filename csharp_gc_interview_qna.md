# C# Garbage Collector — Interview Q&A (from Questpond video)

This document summarizes the video’s questions and answers, with grammar polished and wording tightened.

---

## 1) What is the Garbage Collector (GC)?
A background process that runs **non-deterministically** and reclaims **unreferenced managed objects** from memory.

## 2) How does the GC know when to clean objects?
When an object is **out of scope** and **no longer referenced** by any GC root (e.g., from the stack), it becomes eligible for collection.

## 3) Can we inspect the heap / GC activity?
Yes. Use **Windows Performance Monitor (perfmon)** and Visual Studio’s **Performance Profiler** (e.g., **.NET CLR Memory**, **GC Heap Size**) to visualize allocations, collections, and GC activity.

## 4) Does the GC clean primitive (value) types like `int`, `double`?
Not when they live on the **stack**. Stack values are not GC-managed heap objects, so GC heap size won’t change because of them.

## 5) Managed vs. unmanaged resources/objects/code — what’s the difference?
- **Managed:** Pure .NET objects under the **CLR’s control**.  
- **Unmanaged:** Things **outside** CLR control (file handles, DB connections, COM objects, native/C++ allocations, pinned memory).

## 6) Can the GC clean unmanaged resources?
**No.** Unmanaged resources must be released by your code or the owning API (e.g., `Close`, `Dispose`, native `free`).

## 7) What are GC generations?
**Logical buckets by object age**: the GC groups objects into generations to optimize collection.

## 8) What are Gen 0, Gen 1, and Gen 2?
- **Gen 0:** Short-lived objects (created and discarded quickly).  
- **Gen 1:** “In-between” objects; acts as a buffer.  
- **Gen 2:** Long-lived objects.

## 9) Why do we need generations?
**Performance.** Most collections focus on **Gen 0** (where churn is highest), and visit **Gen 1/2** less often, reducing scanning work.

## 10) Where should you clean unmanaged resources?
A **finalizer (destructor)** can clean them, but that **delays collection**. The **best practice** is the **Dispose pattern** (`IDisposable`) and calling `Dispose()` deterministically.

## 11) How does GC behave with a destructor present?
Finalizable objects typically require **at least two GC cycles** (promotion) before reclamation, which **slows reclamation** and increases Gen1/Gen2 pressure.

## 12) Is an empty destructor OK?
**Bad practice.** It needlessly puts the object on the finalization path, **degrading performance** with no benefit.

## 13) Explain the Dispose pattern.
Implement **`IDisposable.Dispose`** to release unmanaged resources, then call **`GC.SuppressFinalize(this)`** to **skip the finalizer**, allowing immediate reclamation after `Dispose()`.

## 14) Finalize vs. destructor — difference?
In C#, the **destructor syntax** compiles to a **`Finalize`** method. They refer to the **same mechanism** (finalization by the GC).

## 15) What does the `using` statement do (resource `using`, not namespaces)?
Defines a **scope**; when the scope ends, **`Dispose()` is called automatically**, ensuring timely cleanup of unmanaged resources.

## 16) Can you force the GC to run?
Yes — `GC.Collect()` (optionally targeting a specific generation).

## 17) Should you force the GC to run?
Generally **no**. The GC is **adaptive and smart**; manual collection usually **hurts performance**. Common legitimate call: `GC.SuppressFinalize`.

## 18) How do you detect memory issues?
Use Visual Studio’s **Performance Profiler** (e.g., **.NET Object Allocation Tracking**) and watch:  
- **Live objects / memory trend** (should fluctuate, not grow linearly).  
- **Allocations vs. deallocations** (green vs. red).

## 19) How do you find the exact source of memory pressure?
Inspect **allocation lists** and **hot types** in the profiler (types with the largest allocation sizes/counts), then review the related code.

## 20) What is a memory leak?
A condition where memory consumed by the application is **not released back** to the OS when it should be (e.g., app exit or after objects are no longer needed).

## 21) Can a .NET app leak memory even with a GC?
**Yes.** The app’s total memory = **managed** + **unmanaged**. The GC only reclaims **managed** objects. **Unmanaged** memory (or rooted managed objects) can still leak.

## 22) How do you detect unmanaged leaks in .NET?
Compare **Working Set (total memory)** vs **GC Heap Size (managed)**. If working set grows **linearly** while GC heap **fluctuates normally**, suspect an **unmanaged leak**. Use allocation tracking to narrow sources.

## 23) Weak vs. strong references — explain.
- **Strong reference:** Keeps the object **alive**.  
- **Weak reference:** Allows the GC to **collect** the object; you can query `IsAlive`/`TryGetTarget` to access it **only while it still exists**.

## 24) When should you use weak references?
Use sparingly, mainly for **caching/pooling** of **expensive-to-construct** objects where it’s acceptable for the cache to **evict** under memory pressure. Be careful: GC timing is **non-deterministic**.
