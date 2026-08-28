# Silent Bugs in C# — Part 1: The Simple Things That Kill You Quietly

## What is a "silent bug"?

A silent bug is a bug that:

- Compiles fine ✅
- Doesn't throw an exception ✅
- Passes your tests ✅
- **But does the wrong thing anyway** ❌

Nobody sees an error. Nobody gets an alert. The app "works." The wrong thing just quietly happens — maybe for weeks — until someone notices money is missing, data is wrong, or customers are angry.

This part covers the *simple* code patterns that cause silent bugs. No advanced stuff here — these are things every developer writes every day.

---

## 1. The `if` without an `else`

This is the #1 silent killer, because it looks completely harmless.

```csharp
if (user.IsActive)
    report.Include(user);
```

This code says: *"If the user is active, include them."* But it **also** says something you didn't write down: *"If the user is NOT active... do nothing and keep going."*

That hidden "do nothing" part is called the **implicit else**. It's real behavior. It's just not visible in the code.

### How it goes wrong

**Example 1: The hidden else changes meaning during refactoring**

Before:

```csharp
public void ProcessRefund(Order order)
{
    if (order.IsPaid)
        _gateway.Refund(order);

    _repository.MarkProcessed(order);   // runs for ALL orders
}
```

Someone "cleans this up" into a guard clause:

```csharp
public void ProcessRefund(Order order)
{
    if (!order.IsPaid)
        return;                        // ← this return is a hidden else!

    _gateway.Refund(order);
    _repository.MarkProcessed(order);
}
```

Looks cleaner, right? But the behavior changed:

- **Before:** unpaid orders skip the refund but *are still marked processed*.
- **After:** unpaid orders skip *everything*.

So now unpaid orders are never marked as processed. The background job picks them up again. Every cycle. Forever. No error anywhere — just a job that never finishes its work.

**Example 2: One branch gets fixed, the other doesn't**

```csharp
if (tier == CustomerTier.Gold)
{
    if (!IsEligible(tier))
        return price;          // Gold: eligibility checked ✅
    return price * 0.9m;
}

if (tier == CustomerTier.Premium)
    return price * 0.8m;       // Premium: eligibility never checked ❌
```

A new eligibility check was added to the Gold branch. The Premium branch silently skips it. Premium customers get discounts they shouldn't. No exception. No warning.

### The rule to remember

> **Every `if` is secretly an `if/else`. The else is "do nothing and keep going." If that's not what you want, write the else down.**

If a branch is intentionally empty, make it loud:

```csharp
if (user.IsActive)
    report.Include(user);
else
{
    // Intentionally do nothing: inactive users are just omitted.
}
```

Or better — use a `switch` expression, which forces you to handle every case:

```csharp
decimal discount = tier switch
{
    CustomerTier.Gold     => price * 0.9m,
    CustomerTier.Premium  => price * 0.8m,
    CustomerTier.Standard => price,   // explicit: no discount on purpose
    _ => throw new UnreachableException($"Unknown tier: {tier}")
};
```

Now there is no hidden path. Every case is written down, and anything unexpected throws instead of silently continuing.

---

## 2. The early `return` that skips more than you think

Guard clauses ("return early") are great — until they skip a line that used to run for everyone.

```csharp
public void SaveUser(User user)
{
    if (user.Email == null)
        return;              // skips everything below

    Validate(user);
    _db.Save(user);
    _auditLog.RecordSave(user);   // ← audit no longer records bad saves
}
```

The audit line used to run even for users with no email (because the old code used `if/else` around only part of the method). Now it doesn't. Auditing has a silent hole.

**The rule:** when you add an early `return` during refactoring, ask out loud: *"What lines below this used to run that now don't? Is that on purpose?"*

---

## 3. Fallback logic: "If A fails, use B" — the most dangerous pattern of all

This one deserves special attention, because it's the bug that **passes QA, passes tests, and passes production monitoring — all at the same time.**

### The innocent-looking code

```csharp
public decimal GetSurcharge(string region)
{
    try
    {
        return _pricingApi.GetSurcharge(region);   // A: the real source
    }
    catch
    {
        return _fallbackTable[region];              // B: fallback
    }
}
```

"This makes us resilient!" — and sometimes it does. But here's the trap.

### The trap: QA tests the fallback, not the real path

Think about what usually happens:

1. **In QA**, the pricing API (A) often isn't available or isn't configured. So **every call in QA actually runs the fallback (B)**. The fallback is not a rare edge case in QA — it's *the only code path being tested*.
2. QA checks the results. The numbers look right — because the fallback value happens to be correct *for QA*.
3. Tests are green. Release is approved. Code ships to production.
4. **In production**, the pricing API fails one day (expired certificate, network blip, whatever). The fallback kicks in. No error. Every request succeeds. Dashboards are green.
5. **But the fallback value is wrong for production.** It's:
   - a stale rate from last year,
   - a QA-specific setting,
   - a default that was never meant for real customers.

Now the system is **confidently wrong**. And here's the worst part: **because the fallback "worked," no alert fired.** The outage that would have been caught in 5 minutes becomes three weeks of quietly wrong prices.

### The timeline of a real-world silent disaster

| When | What happens | What anyone sees |
|---|---|---|
| QA deploy | A is down in QA, so B is used; B's values are correct for QA | Green tests ✅ |
| Prod deploy | A works; fallback never runs | Green dashboards ✅ |
| Prod, week 2 | A's certificate expires; **every** call falls back to B | **Still green dashboards ✅** |
| Month 2 | Finance notices profit margins are shrinking | Nobody connects it to code |

The fallback *succeeded* every single time. That's exactly why nobody noticed.

### How to make fallbacks safe

A fallback is only okay if you can answer **yes** to all of these:

1. **"Is B's value actually correct when A fails?"** — Same meaning, same freshness? If B is a cached or old value, say so.
2. **"Would we know if the fallback ran?"** — Every fallback must fire a **metric and an alert**, not just a log line nobody reads.
3. **"Did we test B's *values* in a production-like environment?"** — Not just "does B exist," but "is B's data right?"

**Improved version:**

```csharp
public async Task<SurchargeResult> GetSurchargeAsync(string region)
{
    try
    {
        var value = await _pricingApi.GetSurchargeAsync(region);
        return SurchargeResult.Authoritative(value);
    }
    catch (HttpRequestException)   // only network failures fall back
    {
        _metrics.Increment("pricing.fallback.used");  // ← always visible
        var stale = await _fallbackTable.GetSurchargeAsync(region);
        return SurchargeResult.Fallback(stale);       // ← caller KNOWS it's a fallback
    }
}
```

Now the fallback is a **visible decision**, not a hidden substitution. The caller can choose to reject stale data instead of silently accepting a wrong number.

**Fallback rules to live by:**

- 🚨 **Alert on fallback usage**, not fallback errors. Fallbacks by definition produce no errors — that's the whole problem.
- 🎯 **Catch narrow exceptions only.** `catch (Exception)` means even business errors ("region not supported") trigger the fallback and return a made-up price.
- ⏳ **Put an expiry on stale data.** A fallback should refuse to serve data that's too old, not serve it quietly.
- 🧪 **Test the fallback by breaking A on purpose** and asserting B returns the *correct* value — with production configuration, not QA configuration.

---

## 4. Swallowed exceptions: the `catch` that eats your failure

```csharp
try
{
    _repository.Save(order);
}
catch (Exception)
{
    // "I'll handle this later"
}
```

This is the same disease as the fallback — just lazier. The save failed. Nobody knows. The order is gone. The method *looked* like it succeeded.

A fallback at least pretends to handle things. An empty catch doesn't even pretend. But both do the same thing: **they turn a failure into a fake success.**

**The fix is always the same minimum:**

```csharp
catch (Exception ex)
{
    _logger.LogError(ex, "Failed to save order {OrderId}", order.Id);
    throw;   // don't swallow it — let it bubble up
}
```

If you truly must swallow an exception, write a comment explaining *why* — so the next person knows it was a decision, not an accident.

---

## 5. "It works on my machine" — environment settings baked into code

A cousin of the fallback bug. During refactoring, someone hardcodes or leaves behind a value that was only correct in one environment:

```csharp
// Written while debugging locally, never removed
var timeout = 5000; // was: config.ReadTimeout — the config line got "simplified" away
```

Or the opposite:

```csharp
var timeout = config.Timeout ?? 30;   // WRONG: ?? checks null, not zero
```

That second one looks so clean. But the original code said *"if timeout is zero, use 30."* The new code says *"if timeout is null, use 30."* A zero now flows through — and in many libraries, **zero timeout means "wait forever."** Everything compiles, everything runs, nothing times out. Ever.

**The rule:** `??` (null-coalescing) is a fallback for *null*, not for *wrong values*. If a "bad" value is zero, empty, or negative, `??` will not save you.

---

## 6. The quick checklist (Part 1)

Before you merge a refactoring PR, ask:

- [ ] Did I add or change an `if`? **Have I tested BOTH branches — including the hidden else?**
- [ ] Did I add an early `return`? **What code below it no longer runs? On purpose?**
- [ ] Did I add a try/catch or fallback? **Is the fallback value correct in PRODUCTION, not just QA?**
- [ ] Does the fallback fire an **alert** when it runs, not just a log line?
- [ ] Did I catch a **specific** exception, or am I swallowing everything?
- [ ] Did I replace an explicit check with `??` or a default value? **Does null mean the same thing as zero/empty here?**
- [ ] Is there any value hardcoded "temporarily" that came from my dev or QA environment?

---

*→ Continue to Part 2 for the more technical pitfalls: async/await, LINQ, equality, structs vs classes, and the tools that catch this stuff for you.*
