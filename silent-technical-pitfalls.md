# Silent Bugs in C# — Part 2: The Technical Pitfalls

*This part assumes you've read Part 1 (the simple stuff: if/else, fallbacks, swallowed exceptions). Here we go deeper into language-level traps: async, LINQ, equality, mutability, and the tooling that can catch these for you.*

---

## 1. Async traps: the fire-and-forget disaster

### 1.1 The unawaited Task

```csharp
public void HandleRequest(Request req)
{
    _emailService.SendAsync(req);   // Task is thrown away!
}
```

This compiles fine — and often without even a warning. What actually happens:

- The email send starts in the background.
- If it **fails**, the exception is captured inside the abandoned `Task` — and disappears.
- The caller thinks everything worked.

This is a **silent bug by design**: the failure literally has nowhere to go.

**Fix:**

```csharp
public async Task HandleRequestAsync(Request req)
{
    await _emailService.SendAsync(req);
}
```

**Tooling:** treat compiler warning `CS4014` as an error, and add the `Microsoft.VisualStudio.Threading.Analyzers` package (rule `VSTHRD110` catches unawaited tasks). If you *intentionally* want fire-and-forget, make it loud:

```csharp
_ = Task.Run(() => _emailService.SendAsync(req))
    .ContinueWith(t => _logger.LogError(t.Exception, "Email failed"),
        TaskContinuationOptions.OnlyOnFaulted);
```

### 1.2 `async void`

```csharp
public async void ProcessAsync()   // ← exceptions escape into the void
```

- A caller **cannot** catch exceptions from an `async void` method with try/catch.
- Depending on the host, the exception either crashes the process or vanishes.

**Rule:** `async void` is only acceptable for UI event handlers. Everything else returns `Task`. Enforce this with analyzer rule `VSTHRD100`.

### 1.3 The combo: async + fallback

Note how these traps stack. An unawaited async call **cannot have a working fallback**, because nobody is around to observe the failure and trigger it. If you need fallback logic, you must `await` — otherwise your fallback is dead code that runs for nobody.

---

## 2. LINQ: deferred execution surprises

LINQ queries don't run when you *write* them. They run when you *loop over* them.

### 2.1 The refactored field that re-queries forever

```csharp
// Someone "extracted" this query into a field during refactoring
private readonly IEnumerable<Order> _pendingOrders 
    = _orders.Where(o => o.IsPending);
```

Every time you loop over `_pendingOrders`, the query **runs again against the current contents of `_orders`**. If items were added or removed since, you get different results than the moment the query was defined. Silently.

### 2.2 The opposite problem: premature `.ToList()`

Adding `.ToList()` during a cleanup **snapshots** the data. Later changes to the collection are invisible to you. Also silently.

Both directions produce wrong data with zero exceptions. The fix is making execution semantics **explicit in your method signatures**:

- Return `IReadOnlyList<T>` when the data is a snapshot (already executed).
- Return `IEnumerable<T>` only when you *mean* "this may re-run every time — beware."

---

## 3. Numbers: silent truncation and overflow

### 3.1 The cast that wraps around

```csharp
// Before refactoring:
long totalBytes = ComputeTotalBytes();

// After "simplification":
int totalBytes = (int)ComputeTotalBytes();   // no exception — just wrong
```

If the value exceeds ~2.1 billion, it **wraps around to a negative number**. No exception. No warning. Just wrong math flowing downstream.

**Fix:** use `checked` blocks where overflow matters — they throw instead of wrapping:

```csharp
checked
{
    int total = (int)ComputeTotalBytes();   // throws OverflowException instead
}
```

### 3.2 Money in `double`

Refactoring "for performance" from `decimal` to `double` silently introduces precision errors:

```csharp
double price = 19.99;   // actually stored as 19.989999999999998...
```

One cent off per transaction, thousands of transactions a day — a silent bug that only accountants will find, months later. **Rule: money is always `decimal`.**

---

## 4. Structs vs. classes: the copy that became an alias

```csharp
// Before: Settings was a struct → assignment makes a COPY
var backup = currentSettings;
currentSettings.Timeout = 30;
// backup.Timeout still has the old value ✅
```

Someone refactors `Settings` from a `struct` to a `class`. Now:

```csharp
var backup = currentSettings;      // both point to the SAME object
currentSettings.Timeout = 30;
// backup.Timeout is ALSO 30 now ❌ — silently
```

No exception. The "backup" silently mutates along with the original. Any invariant that depended on copy semantics is now broken.

**Fixes:**

- Prefer `readonly struct` for small value-like data — the compiler prevents mutation.
- Prefer `record` types for data carriers — they communicate "this is data, treat it carefully."
- Changing struct ↔ class should be treated as a **breaking behavioral change**, not a cleanup.

---

## 5. Equality: the dictionary that lies to you

### 5.1 The record that gained a field

```csharp
// Before
public record Customer(string Name);

// After a refactor adds a field
public record Customer(string Name, int Age);
```

Records auto-generate equality from **all** their fields. Any code doing `Distinct()`, dictionary lookups, or set operations now behaves differently:

- Two customers with the same name but different ages are suddenly "different" — duplicates appear where none existed before.
- Or the reverse: `list.Contains(x)` stops finding items it used to find.

Silently.

### 5.2 Equals without GetHashCode

If a class overrides `Equals` but not `GetHashCode` (or vice versa), hash-based collections break quietly:

```csharp
var set = new HashSet<Customer>();
set.Add(customer);
set.Contains(customer);   // can return FALSE — even though we just added it
```

**Fix:** always override both together, or better — use `record` and let the compiler do it consistently.

---

## 6. Polymorphism: `new` vs `override`

```csharp
public class Base { public virtual void Log() { /* A */ } }
public class Derived : Base { public new void Log() { /* B */ } }  // ← "new", not "override"

Base b = new Derived();
b.Log();   // runs A, not B — silently
```

Using `new` instead of `override` (or accidentally dropping `virtual` from the base during a refactor) means the method is *hidden*, not *overridden*. Polymorphism silently stops working. Refactoring tools like Rename can even cause this if applied carelessly across assemblies.

**Fix:** analyzer rule Sonar **S3442** flags misplaced `new` keywords. Prefer `sealed override` to make intent obvious.

---

## 7. Why the compiler doesn't save you

The C# compiler checks **structure** (types, signatures, access). It does not check **behavior**. That gap is exactly where silent bugs live:

| What goes wrong | Does the compiler catch it? |
|---|---|
| Missing `else` / hidden branch | ❌ |
| Swallowed exception | ❌ |
| Fallback returns the wrong value | ❌ |
| Unawaited Task | ⚠️ only a warning (often ignored) |
| Silent numeric truncation | ❌ (unless `checked`) |
| LINQ re-execution timing | ❌ |
| Struct → class copy semantics change | ❌ |
| Equality/hash mismatch | ❌ |
| `new` instead of `override` | ❌ |

> **A one-line diff is not a small change if it changes what a code path does.**

---

## 8. Tooling that catches silent bugs for you

### 8.1 Compiler + analyzers (do this first — it's nearly free)

In your `.csproj`:

```xml
<PropertyGroup>
  <Nullable>enable</Nullable>
  <AnalysisLevel>latest-all</AnalysisLevel>
  <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
</PropertyGroup>
```

Then add:

- **SonarAnalyzer.CSharp** — empty catches, broad catches, `new`-vs-`override`, dead code.
- **Microsoft.VisualStudio.Threading.Analyzers** — unawaited tasks, `async void`, sync-over-async.
- **AsyncFixer** — more async correctness rules.

### 8.2 Branch coverage, not line coverage

Line coverage can be **100%** while an `else` path was never executed. Branch coverage tells you which *decisions* were tested both ways. This is the single most direct defense against missing-`else` bugs. Use Coverlet + ReportGenerator and *look at branch coverage*.

### 8.3 Characterization tests: pin behavior BEFORE refactoring

Before touching a method, write tests that lock in what it *currently* does — including the boring paths:

```csharp
[Theory]
[InlineData(CustomerTier.Gold, 90)]
[InlineData(CustomerTier.Premium, 80)]
[InlineData(CustomerTier.Standard, 100)]   // ← the hidden else, now a real test
public void Discount_Pinned(CustomerTier tier, int expectedPercent)
    => Assert.Equal(expectedPercent, _pricing.ApplyDiscount(100m, tier));
```

**Rule: every `if` you're about to touch gets a test for BOTH branches first.**

### 8.4 Mutation testing with Stryker.NET

[Stryker.NET](https://stryker-mutator.io) automatically mutates your code — flips conditions, deletes `else` blocks, removes catch bodies — and checks whether your tests **fail**. If a mutant survives, no test watches that path. That's exactly where silent bugs live.

### 8.5 Failure-injection tests for fallbacks

Don't just test "the fallback path doesn't throw." Break the primary source on purpose and assert the fallback returns the **correct value**:

```csharp
[Fact]
public async Task When_PricingApi_IsDown_Fallback_Returns_Valid_Rate()
{
    _pricingApi.Setup(x => x.GetRateAsync("EUR"))
               .ThrowsAsync(new HttpRequestException("down"));

    var result = await _sut.GetSurchargeAsync("EUR");

    Assert.True(result.Value > 0);
    Assert.False(result.IsAuthoritative);       // caller must KNOW it's a fallback
    Assert.True(DateTimeOffset.UtcNow - result.AsOf < TimeSpan.FromHours(24)); // freshness bound
}
```

And add a **reconciliation job** in production: periodically compare what the fallback *would* return against the real source. Divergence = deploy-stopping finding.

### 8.6 Observability: alert on fallback usage

Because silent bugs produce no errors, your alerts must watch **behavior, not just errors**:

- Metric: `fallback.used` per dependency. Alert if it spikes above a small percentage.
- Metric: `staleness.seconds` on any cached/stale data.
- Business reconciliation: orders submitted vs. orders persisted; rates served vs. rates authoritative (sampled).

The alert that saves you is not "exception rate is up" — it's **"100% of pricing calls used the fallback since 02:00."**

---

## 9. Quick checklist (Part 2)

- [ ] Any `async` method called without `await`? (CS4014 as error, VSTHRD110 installed)
- [ ] Any `async void` that isn't an event handler?
- [ ] Any `IEnumerable<T>` returned from a method — is lazy execution actually intended?
- [ ] Any cast between number types without `checked`? Any money stored in `double`?
- [ ] Did a struct change to a class (or vice versa)? Copy semantics changed — did anyone check?
- [ ] Did a record/class gain or lose a field? Equality behavior changed — did `Distinct()`/dictionary code get rechecked?
- [ ] Any `new` on a method that should be `override`?
- [ ] Are analyzers on, with warnings-as-errors?
- [ ] Did you check **branch** coverage, not just line coverage?
- [ ] For every fallback: failure-injection test + production-value check + usage alert?

---

## References

1. Fowler, M. (2018). *Refactoring: Improving the Design of Existing Code* (2nd ed.). Addison-Wesley.
2. Feathers, M. (2004). *Working Effectively with Legacy Code*. Prentice Hall.
3. Nygard, M. (2018). *Release It!* (2nd ed.). Pragmatic Bookshelf.
4. Cleary, S. (2019). *Async and Await*. Nitify Press.
5. Stryker.NET — https://stryker-mutator.io
6. SonarSource C# Rules — https://rules.sonarsource.com/csharp
7. Microsoft .NET Analyzers — https://learn.microsoft.com/en-us/dotnet/fundamentals/code-analysis/overview
