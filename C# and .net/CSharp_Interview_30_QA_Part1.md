# C#/.NET Interview — 30 Questions (Part 1)

Concise answers based on the provided transcript.

---

## 1) Difference between .NET and C#
- **C#** is a programming language (syntax, keywords, control flow).
- **.NET** is the framework/runtime: libraries (e.g., `System.Collections`, `System.Xml`) plus the execution environment (CLR, GC, CTS, CLS) that compiles and runs your app.

## 2) .NET Framework vs .NET Core vs .NET 5
- **.NET Framework**: older, Windows-only.
- **.NET Core**: cross-platform, modular (smaller assemblies), better performance, strong CLI support (`dotnet build`, `dotnet run`).
- **.NET 5**: unified platform that brings Framework/Core/Mono, etc., into a single experience.

## 3) What is IL (Intermediate Language) code?
- **Partially compiled** intermediate code produced by .NET language compilers before it becomes machine code.

## 4) Why do we need JIT?
- The **Just-In-Time (JIT) compiler** converts IL to **native machine code** at runtime for the current OS/CPU.

## 5) Can we view IL code?
- Yes, with **disassemblers** such as **ILDASM** or **ILSpy**.

## 6) Why compile to IL first?
- **Dev and runtime environments differ** (OS/CPU). Compiling to IL lets the JIT generate **optimal native code** for the actual target environment at execution time.

## 7) Does .NET support multiple languages?
- Yes: **C#, VB.NET, F#, C++/CLI**, etc. All of them compile to **IL**.

## 8) What is the CLR (Common Language Runtime)?
- The **execution environment** for .NET. It:
  1. Converts IL to native code.
  2. Runs the app and manages memory (including invoking the **Garbage Collector**).

## 9) Managed vs unmanaged code
- **Managed code** runs under the CLR (compiled to IL, GC-managed).
- **Unmanaged code** runs outside the CLR (e.g., native C/C++ DLLs).

## 10) What does the Garbage Collector (GC) do?
- A background process that reclaims **unused managed memory** automatically during program execution.

## 11) Can the GC collect unmanaged objects?
- **No.** GC only collects **managed** objects. Unmanaged resources require explicit handling.

## 12) What is CTS (Common Type System)?
- Defines **common data types** so multiple .NET languages map their types (e.g., C# `int`, VB `Integer`) to a shared type (e.g., **Int32**).

## 13) What is CLS (Common Language Specification)?
- A **set of guidelines** for language features/behavior (e.g., case sensitivity, pointers) to ensure **cross-language interoperability**.

## 14) Stack vs Heap
- **Stack**: stores **value types** with their values in place; LIFO layout.
- **Heap**: stores **objects**; the stack holds a **reference** (pointer) to data on the heap.

## 15) Value types vs Reference types
- **Value types**: stored on the **stack**; variable and value live together.
- **Reference types**: stack stores a **reference**; actual object lives on the **heap**.

## 16) Boxing and Unboxing
- **Boxing**: moving a value type into a reference type (e.g., `int` → `object`).
- **Unboxing**: extracting the value type back from the reference.

## 17) Consequences of Boxing/Unboxing
- **Performance cost**, due to moving data between stack (value type) and heap (reference type).

## 18) Casting (implicit vs explicit)
- **Casting**: converting one type to another.
- **Implicit**: automatic when moving to a **wider** type (e.g., `int` → `double`).
- **Explicit**: required when moving to a **narrower** type (e.g., `double` → `int`).

## 19) What can happen during explicit casting?
- **Data loss** (e.g., truncating decimals when casting `double` to `int`).

## 20) Array vs ArrayList
- **Array**: fixed size, **strongly typed**.
- **ArrayList**: **resizable**, but **not strongly typed** (accepts mixed types).

## 21) Which performs better: Array or ArrayList?
- **Array** performs better because it’s **strongly typed** and avoids boxing/unboxing overhead.

## 22) What are Generic Collections?
- Collections like `List<T>` that are **strongly typed** (prevent boxing/unboxing) and **resizable**—combining array’s type safety with ArrayList’s flexibility.

## 23) What are Threads (Multithreading) in C#?
- Separate execution paths to run code in **parallel** (e.g., start two threads to run two methods concurrently).

## 24) Threads vs TPL (Task Parallel Library)
- **TPL/Task** abstracts over threads and leverages processor resources for **true parallelism**, **pooling**, **result return**, chaining, and **async/await** support. Threads are lower-level and have CPU affinity without these conveniences.

## 25) How are exceptions handled in C#?
- With **try/catch**. Wrap code that may fail in `try`; handle errors in `catch`.

## 26) Why use `finally`?
- To run **cleanup code** that must execute **whether or not** an exception occurs.

## 27) Why do we need the `out` keyword?
- To **return multiple values** from a method via additional `out` parameters.

## 28) What is the need for Delegates?
- A **pointer to a function** (callback), useful for communication patterns and callback scenarios (including multi-threading).

## 29) What are Events?
- **Encapsulation** over delegates that enforce a **publisher–subscriber** model and protect delegate invocation from external tampering.

## 30) Abstract Class vs Interface
- **Abstract class**: a **partially implemented** parent (use when you have shared base behavior).
- **Interface**: a **contract**—it defines required members and enforces structure without implementation.

---

**Tips from the transcript**
1. Answer to the point; avoid long-winded explanations.
2. Lead with the most important differences (e.g., cross-platform, performance, CLI).
3. Use precise technical vocabulary (e.g., “events are encapsulation over delegates,” “publisher–subscriber”).
