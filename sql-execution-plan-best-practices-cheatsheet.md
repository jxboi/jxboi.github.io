# Part 3: Best Practices and One-Page Cheatsheet

---

## 1. Best Practices for Reading Plans

1. **Always use the Actual Plan** (Ctrl + M) — estimated plans hide spills
   and wrong row counts.
2. **Compare Estimated vs. Actual rows first** — this finds the root cause
   faster than anything else.
3. **Record a baseline before changing anything** (`SET STATISTICS IO, TIME ON`).
   Without a "before" number you can't prove improvement.
4. **Save the plan** as a `.sqlplan` file — useful for before/after comparison.
5. **Don't trust cost % blindly** — it's a relative guess, not milliseconds.
6. **Check "Executions," not just rows** — a cheap step run 2M times is costly.
7. **Use Query Store** to spot when a query suddenly got a bad plan.
8. **Test on production-like data** — plans depend on data size and distribution.

## 2. Best Practices for Writing Queries

```sql
-- ✓ DO: bare column, range comparison (seekable)
WHERE OrderDate >= '2024-01-01'

-- ✘ DON'T: function on the column (forces a scan)
WHERE YEAR(OrderDate) = 2024
```

- Select **only the columns you need** (`SELECT col1, col2` — not `SELECT *`).
- **Match data types** between columns and parameters.
- Avoid **scalar functions** in queries — they run once per row.
- Prefer **set-based** logic over cursors and loops.
- Keep **transactions short** — a "slow query" might actually be waiting
  on a lock, not the plan.

## 3. Best Practices for Indexes

1. Index the columns in **WHERE, JOIN, and ORDER BY** (equality columns first).
2. Use **INCLUDE** for extra columns instead of adding them as keys.
3. **Fewer, better indexes beat many overlapping ones** — every index
   slows down inserts and updates.
4. Treat **Missing Index hints as suggestions**, not commands.
5. If the plan shows **Key Lookup × many executions** → create a covering index.

```sql
-- The classic covering index pattern:
CREATE NONCLUSTERED IX_Name
ON dbo.Table (FilterColumn, SortColumn)
INCLUDE (ColumnsYouSelect);
```

## 4. Best Practices for Statistics

- Keep **AUTO_UPDATE_STATISTICS** ON.
- For big, fast-changing tables, schedule
  `UPDATE STATISTICS ... WITH FULLSCAN`.
- When a plan guesses wrong, **check when statistics were last updated**
  (`SELECT STATS_DATE(object_id, stats_id) FROM sys.stats`).

## 5. Best Practices for Spills and Memory

- Treat **every spill warning as a bug to fix** — not noise.
- Spills are fixed by fixing the row guess (statistics, query filter).
- Many queries spilling? Check tempdb configuration and server memory.

## 6. Best Practices for Changing Things Safely

1. **Change one thing at a time** — so you know what worked.
2. **Fix the biggest offenders first** — a few queries usually cause most
   of the load (find them via Query Store or `sys.dm_exec_query_stats`).
3. **Hints are a last resort, not a solution** — they freeze the plan and
   age badly as data changes.
4. **Use Query Store plan forcing** as a fast emergency fix, then fix the
   real cause properly.
5. **Re-check after a few weeks** — plans can regress as data grows.

---

## 7. One-Page Cheatsheet

### The 6-Step Method

```
1. Biggest cost box?        → start there
2. Est vs Actual rows?      → 10× off = wrong guess 🚨
3. Arrow thickness?         → thick-then-thin = reading too much
4. Yellow ⚠ warnings?       → spills, conversions — never ignore
5. Execution counts?        → cheap × millions = expensive
6. Seek or Scan?            → scan on big table = index problem
```

### Symptoms → Causes → Fixes

| Symptom in plan | Root cause | Fix |
|---|---|---|
| Est ≪ Actual rows | Stale stats / sniffing | `UPDATE STATISTICS` |
| Scan on big table | No index / non-SARGable | Add index; rewrite WHERE |
| Key Lookup × many | Index not covering | `CREATE INDEX ... INCLUDE` |
| ⚠ Spill to tempdb | Memory sized on bad guess | Fix the guess |
| Great for one value, awful for another | Parameter sniffing | `RECOMPILE` / `OPTIMIZE FOR` / force plan |
| Implicit conversion ⚠ | Type mismatch | Align data types |
| Query suddenly slow (was fast) | Plan regression | Query Store: compare + force good plan |

### The Golden Rules

> 1. **Measure first.** No baseline, no proof.
> 2. **Row guesses explain almost everything.** Check them first.
> 3. **Every spill is a bug.**
> 4. **Cheap and repeated = expensive.**
> 5. **Fix root causes, not symptoms** (hints are band-aids).
> 6. **One change at a time.**

### Handy Commands

```sql
SET STATISTICS IO, TIME ON;                 -- baseline reads & time
UPDATE STATISTICS dbo.Orders WITH FULLSCAN; -- refresh row guesses
SELECT * FROM sys.dm_exec_query_stats;      -- most expensive cached queries
-- Query Store: force a known-good plan after regression
```

---

*End of guide. Back to [01-basics-and-reading-plans.md](01-basics-and-reading-plans.md)*
