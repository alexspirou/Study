# Practical Guide to Choosing Between Class, Struct, and Record in Modern C

## 1) Core mental model

-   **class** → reference type (identity, shared via references,
    heap-allocated, GC-managed).
-   **struct** → value type (data copied by value, typically
    stack/inline, no GC unless boxed).
-   **record** → syntax + semantics for value-based equality:
    -   **record class** → reference type with value-based equality.
    -   **record struct** → value type with value-based equality (and
        deconstruction).
    -   **readonly record struct** → immutable value semantics by
        default.

## 2) When to pick what (decision checklist)

**Use a struct if all are true:** - Represents a single value (e.g.,
Point, Range, Angle). - Small (≈ ≤ 16 bytes; a few primitive fields). -
Immutable (no post-construction mutation). - Not boxed often (you stay
in generic, strongly-typed code).

**Use a class if any of these hold:** - The type has identity (two
instances can be distinct even if data matches). - It's large or
mutable. - It will be shared broadly and mutated through references. -
It will be boxed or used via object/non-generic APIs.

**Use a record when you want value-based equality and
with-expressions:** - **record class**: value-based equality +
reference-type behavior. - **record struct**: value-based equality +
value-type behavior. - Prefer **readonly record struct** for pure value
objects.

## 3) Equality, identity, and mutation

### Classes

Default equality = reference equality (`ReferenceEquals`).\
You can override `Equals`/`GetHashCode`, but it's manual.

``` csharp
class Customer
{
    public Guid Id { get; init; }
    public string Name { get; set; } = "";
    // Reference equality by default.
}
```

### Structs

Default equality = field-by-field value equality (unless you override).\
Copies happen on pass/return/assign---mutable structs are error-prone.
Prefer immutable.

``` csharp
public readonly struct Point(int x, int y)
{
    public int X { get; } = x;
    public int Y { get; } = y;
    // Value equality by default (all fields compared).
}
```

### Records

Equality = value-based by default over declared members.\
`with` expressions support non-destructive mutation.\
`record class` still lives on the heap; `record struct` is a value type.

``` csharp
public record Person(string FirstName, string LastName);
// Reference type, but value-based equality:
var p1 = new Person("Ada", "Lovelace");
var p2 = new Person("Ada", "Lovelace");
Console.WriteLine(p1 == p2); // True (value equality)

public readonly record struct Angle(double Degrees);
// Value type with value equality and deconstruction.
```

## 4) Performance & memory

**Struct:** - Stored inline in arrays/fields → great cache locality. -
No GC unless boxed. - Copy cost on pass/return/assign (keep small).

**Class:** - One heap allocation per instance. - Cheap to pass
(reference copy). - Adds GC pressure at scale.

**record:** - Same memory model as class/struct variant; the "record"
part is about equality/with-patterns.

## 5) Boxing & interfaces

Boxing occurs when a struct is converted to `object` or to a non-generic
interface it implements.\
Prefer generic interfaces/collections to avoid boxing.

``` csharp
public readonly struct Temperature : IComparable<Temperature> // ✅ generic
{
    public int C { get; init; }
    public int CompareTo(Temperature other) => C.CompareTo(other.C);
}

var list = new List<Temperature>(); // no boxing
```

## 6) Immutability recommendations

-   **Structs**: make them immutable (`readonly struct` or
    `readonly record struct`).
-   **Classes/record classes**: prefer init-only setters or constructor
    parameters for "logical immutability".

``` csharp
public readonly record struct Vec2(double X, double Y);

public record Money(decimal Amount, string Currency)  // record class
{
    public Money WithAmount(decimal a) => this with { Amount = a };
}
```

## 7) Practical examples (good vs bad)

**Good struct (small, single value, immutable):**

``` csharp
public readonly record struct Point2D(int X, int Y); // ~8 bytes, immutable, value equality
```

**Bad struct (too big / identity-like / mutable):**

``` csharp
public struct Customer  // ❌ not a single value; mutable; large; reference fields
{
    public Guid Id;
    public string Name;     // reference
    public string Email;    // reference
}
```

**Make it a record class or class instead:**

``` csharp
public record Customer(Guid Id, string Name, string Email); // value-based equality (by data)
```

or

``` csharp
public class Customer   // identity semantics (by reference)
{
    public Guid Id { get; init; }
    public string Name { get; set; } = "";
    public string Email { get; set; } = "";
}
```

**Array locality win:**

``` csharp
public readonly record struct Pixel(byte R, byte G, byte B, byte A); // 4 bytes
var image = new Pixel[10_000_000]; // contiguous, cache-friendly
```

## 8) Records: features you get

**Value-based equality:**

``` csharp
var a = new Person("Ada", "Lovelace");
var b = new Person("Ada", "Lovelace");
Console.WriteLine(a == b); // True
```

**with expressions (non-destructive mutation):**

``` csharp
var a2 = a with { LastName = "Byron" };
```

**Deconstruction:**

``` csharp
var (first, last) = a;
```

**Inheritance:** - `record class` supports inheritance hierarchies. -
`record struct` does not support inheritance (structs never do).

## 9) Common pitfalls & gotchas

**Mutable struct in collections:**

``` csharp
struct Range { public int Start, End; } // ❌ mutable
var list = new List<Range> { new() { Start = 1, End = 3 } };
var r = list[0];   // copy!
r.Start = 10;      // modifies the copy, not the list element
```

Fix: make it `readonly` and replace instead of mutating in place.

**Accidental boxing:**

``` csharp
object o = new Angle(90); // boxing
```

Keep APIs generic when working with value types.

**Large structs:** Copying is expensive; use `in` for read-only by-ref:

``` csharp
public readonly struct BigVal { /* many fields */ }
int Compute(in BigVal v) => /* read-only access without copying */;
```

**Locking:** Don't `lock` on structs (they're boxed to lock), and never
`lock(this)`/`lock(typeof(T))`/`lock("str")`. Use a private object:

``` csharp
private readonly object _sync = new();
lock (_sync) { /* ... */ }
```

## 10) Cheatsheet table

  -----------------------------------------------------------------------------------------------
  Choose...    Use when                           Equality      Mutability      Notes
                                                  default       default         
  ------------ ---------------------------------- ------------- --------------- -----------------
  **class**    Identity, large, mutable, shared,  Reference     Mutable         Heap-allocated;
               reference semantics                equality                      cheap to pass; GC
                                                                                load

  **struct**   Small (≤ \~16B), single value,     Field-wise    Immutable       Inline/stack;
               immutable, high locality, no       value         (recommended)   copies on
               frequent boxing                    equality                      pass/assign

  **record     Reference type + value-based       Value-based   Usually         Great for
  class**      equality + with/deconstruct        (by data)     init-only       DTOs/domain data
                                                                                without identity
                                                                                semantics

  **record     Value type + value-based           Value-based   Prefer readonly Best for tiny
  struct**     equality + with/deconstruct        (by data)                     value objects;
                                                                                avoid boxing

  **readonly   Same as above, enforced            Value-based   Immutable       Safest for value
  rec.         immutability                       (by data)                     objects
  struct**                                                                      
  -----------------------------------------------------------------------------------------------

## 11) Quick templates

**Small immutable value (best: readonly record struct):**

``` csharp
public readonly record struct Angle(double Degrees)
{
    public double Radians => Degrees * Math.PI / 180.0;
}
```

**Domain data without identity semantics (best: record class):**

``` csharp
public record Address(string Street, string City, string Country);
```

**Entity with identity semantics (best: class):**

``` csharp
public class Order
{
    public Guid Id { get; init; }
    public DateTime CreatedAt { get; init; }
    public decimal Total { get; private set; }
    public void Add(decimal amount) => Total += amount;
}
```
