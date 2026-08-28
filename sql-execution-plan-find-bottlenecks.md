# Part 2: Finding Bottlenecks and Fixing Them

*The 5 problems you'll see most often — and how to fix each one.*

---

## Problem 1: Wrong Row Guess (Cardinality Misestimate)

**This is the #1 cause of slow queries.**

SQL Server guesses how many rows a step will produce. If the guess is
wrong, everything downstream goes wrong too.

**How one bad guess ruins a whole plan:**

```
   Guess was wrong (500 vs 2,000,000)
        │
        ├──► picked the WRONG JOIN type
        ├──► reserved too little MEMORY
        └──► memory too small = SPILL TO DISK
        
        Result: slow query
```

**How to spot it:** compare Estimated vs. Actual rows on each box.

**Common causes and fixes:**

| Cause | Fix |
|---|---|
| Old statistics | `UPDATE STATISTICS TableName WITH FULLSCAN;` |
| Parameter sniffing | See Problem 5 |
| Function on a column | See Problem 2 |
| Skewed data | Rewrite query or use `OPTION (RECOMPILE)` |

## Problem 2: Non-SARGable Queries

"SARGable" = written so SQL Server **can** use an index seek.

```sql
-- ❌ BAD: function on the column → SQL Server must read EVERYTHING
SELECT * FROM Orders WHERE YEAR(OrderDate) = 2024;

-- ✓ GOOD: range on the bare column → can seek the index
SELECT * FROM Orders
WHERE OrderDate >= '2024-01-01' AND OrderDate < '2025-01-01';
```

**Rules:**
- Never wrap columns in functions (`YEAR()`, `UPPER()`, math) inside WHERE.
- Avoid leading wildcards: `LIKE '%abc'` can't seek; `LIKE 'abc%'` can.
- Make sure data types match (a `VARCHAR` column compared to an `NVARCHAR`
  parameter forces a conversion → scan + a warning in the plan).

## Problem 3: Key Lookups (Missing Covering Index)

**What you see in the plan:** an Index Seek... plus a **Key Lookup** box
with a huge **execution count**.

```
   ┌──────────────┐
   │ Index Seek   │  fast... but for EVERY row it must:
   └──────┬───────┘  go back to the table to fetch 2 more columns
          │
          ▼
   ┌──────────────┐
   │ Key Lookup   │  ✘ executed 1,000,000 times!
   └──────────────┘
```

**Fix:** create a **covering index** so the lookup isn't needed:

```sql
CREATE NONCLUSTERED INDEX IX_Orders_Status
ON dbo.Orders (Status, OrderDate DESC)   -- keys: filter, then sort
INCLUDE (CustomerID, OrderID);           -- extra columns stored in the index
```

After this, the plan shows one clean Index Seek with no lookup. ✓

## Problem 4: Spills to Disk

**What you see in the plan:** a warning "⚠ Operator used tempdb to spill
data" on a **Sort** or **Hash Match** box.

**What happened:** these operations need memory. SQL Server reserved some
based on its row **guess**. The guess was too small, so the extra data
spilled onto disk. Disk = slow.

```
   ┌─────────────────────┐
   │  Hash table (RAM)   │  reserved for 500 rows...
   │  ██████████░░░░     │──────────────┐
   └─────────────────────┘              ▼
        but 2,000,000 arrived    ┌────────────────┐
                                 │ tempdb SPILL ⚠ │
                                 └────────────────┘
```

**Fix:** it's almost always a row-guess problem underneath.
Fix the statistics or the query filter (Problems 1–2), and the spill
disappears on its own.

## Problem 5: Parameter Sniffing

**The situation:** the same query works with different parameter values.

```sql
-- Works for 5 rows... and for 2,000,000 rows
SELECT * FROM Orders WHERE CustomerID = @CustomerID;
```

SQL Server compiles the plan **once**, based on the **first** value it saw.
If the first value was rare (5 rows), it picks a plan that's terrible
for the common case (2 million rows).

```
   Plan was built for: "rare value" (5 rows)
            │
            ▼  reused for EVERY value
   "common value" (2M rows) → uses the 5-row plan → DISASTER
```

**Fixes, in order of preference:**

1. Fix statistics and indexes first (often the real cause).
2. `OPTION (RECOMPILE)` — build a fresh plan each time (only if the query
   runs infrequently or is cheap to compile).
3. `OPTION (OPTIMIZE FOR UNKNOWN)` — plan for a "typical" value.
4. **Query Store plan forcing** — pin the known-good plan.

## Quick Diagnosis Table

| What you see in the plan | What it means | Fix |
|---|---|---|
| Estimated ≪ Actual rows | Wrong guess | Update statistics; check sniffing |
| Scan on a big table | No useful index / non-SARGable | Add index; rewrite predicate |
| Key Lookup × many executions | Index not covering | Index with `INCLUDE` |
| ⚠ Spill warning | Too little memory | Fix the row guess |
| Nested Loops on millions of rows | Wrong join type from bad guess | Fix guess; hint as last resort |
| Implicit conversion warning | Type mismatch | Match column/parameter types |

---

## A Real Example, Start to Finish

**The slow query:**

```sql
SELECT o.OrderID, o.OrderDate, c.CustomerName
FROM dbo.Orders o
JOIN dbo.Customers c ON o.CustomerID = c.CustomerID
WHERE o.Status = 'P'
ORDER BY o.OrderDate DESC;
```

**The actual plan (simplified):**

```
Sort
 ◄─ Hash Match Join                        ⚠ SPILL
     ◄─ Clustered Index Scan on Orders     Est: 500   Actual: 2,000,000 🚨
     ◄─ Index Seek + Key Lookup            Executions: 2,000,000        🚨
```

**Diagnosis using the 6 steps:**

1. Biggest cost = the Hash Match join.
2. Est 500 vs Actual 2,000,000 → wrong guess 🚨 (Problem 1)
3. Spill warning → memory was sized for 500 rows (Problem 4)
4. Key Lookup × 2M → index isn't covering (Problem 3)
5. Scan on Orders → no index for `Status = 'P'`

**The fix (two simple statements):**

```sql
-- 1. Fix the wrong guess
UPDATE STATISTICS dbo.Orders WITH FULLSCAN;

-- 2. Covering index: filter + sort keys, include the rest
CREATE NONCLUSTERED INDEX IX_Orders_Status
ON dbo.Orders (Status, OrderDate DESC)
INCLUDE (CustomerID, OrderID);
```

**The new plan:**

```
SELECT
 ◄─ Nested Loops
     ◄─ Index Seek on IX_Orders_Status  ✓ guess now correct, no spill
     ◄─ Clustered Index Seek on Customers ✓ one seek per result row
```

Result: reads dropped from millions to a few hundred.

---

➡ Continue to **03-best-practices-and-cheatsheet.md**
