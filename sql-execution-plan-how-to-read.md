# Part 1: SQL Server Execution Plan Basics — How to Read Them

*A simple, plain-language guide.*

---

## 1. What Is an Execution Plan?

When you send a query to SQL Server, it has to decide **how** to get the
data. There are usually many possible routes. SQL Server picks one and
writes down the steps. That list of steps is the **execution plan**.

Think of it like a GPS:

- Your query = "take me to the airport"
- The execution plan = the route the GPS picked
- A slow query = the GPS picked a route through rush-hour traffic

The plan tells you **why the trip was slow**.

## 2. Two Kinds of Plans

| Kind | When it's made | What it shows |
|---|---|---|
| **Estimated plan** | BEFORE running the query | SQL Server's guess |
| **Actual plan** | AFTER running the query | What really happened |

> **Rule of thumb:** Always use the **Actual Plan** when debugging a slow
> query. The estimated plan can't show real problems like running out of
> memory or wrong row counts.

## 3. How to Get a Plan (SSMS)

1. Highlight your query.
2. Press **Ctrl + M** (turns on "Include Actual Execution Plan").
3. Press **F5** to run.
4. Click the new **Execution Plan** tab.

Add this above your query to also see reads and time:

```sql
SET STATISTICS IO, TIME ON;
```

## 4. What a Plan Looks Like

A plan is a **tree of boxes** connected by arrows.

- Each **box** = one action (read a table, join, sort...)
- Each **arrow** = rows flowing between actions
- **Thick arrow** = many rows, **thin arrow** = few rows

```
        [SELECT]   ← final result comes out here
           ▲
        [JOIN]     ← combines two tables
         ▲   ▲
   [Sort]    [Scan]  ← reads the tables (start here)
```

> **Key rule:** Data flows **right to left**. Tables are on the right,
> the result is on the left. You read the plan right → left, but the
> cost percentages are easiest to scan left → right.

## 5. The 6-Step Reading Method

Follow these steps every time. They take 2 minutes and find most problems.

### Step 1: Find the Biggest Cost
Hover over each box. It shows a **cost %**. Start with the highest one.
(Cost % is a relative guess, not milliseconds — but it's a good start.)

### Step 2: Compare Estimated vs. Actual Rows
Right-click a box → **Properties** → find both numbers:

```
Estimated rows:      500
Actual rows:   2,000,000     ← 4000× bigger! 🚨 PROBLEM
```

If they differ a lot (10× or more), SQL Server guessed wrong.
**Wrong guesses cause most slow queries** (explained in Part 2).

### Step 3: Look at the Arrows

```
   ┌──────────┐        ┌─────────┐        ┌──────────┐
   │  result  │◄─ 100 ─│  Join   │◄─ 2M ──│   Scan   │
   └──────────┘        └─────────┘        └──────────┘
                        thin ✓              VERY THICK 🚨
```

A very thick arrow that suddenly becomes thin means SQL Server read
millions of rows just to keep a few. That's usually fixable with an index.

### Step 4: Look for Warnings
Yellow **"!" marks** on boxes mean real problems: memory spills,
implicit conversions, missing indexes. Never ignore them.

### Step 5: Check the "Executions" Number
A cheap operation run **2 million times** becomes expensive.
Right-click boxes and check **"Number of Executions"**.

### Step 6: Check How Tables Were Read
- **Seek** = went straight to the needed rows ✓
- **Scan** = read the whole table ✘ (bad on big tables)
- **Key Lookup** = extra trip back to the table for each row ✘

| Operator on a big table | Verdict |
|---|---|
| Index Seek | ✓ Usually good |
| Index/Table Scan | ⚠ OK for small tables, bad for big ones |
| Key Lookup × many executions | ✘ Needs a covering index |

## 6. Key Properties to Check

Right-click any box → **Properties**:

| Property | Why it matters |
|---|---|
| Estimated vs. Actual Rows | Wrong guess = wrong plan |
| Number of Executions | Reveals hidden per-row work |
| Seek Predicate vs. Predicate | Only "Seek Predicate" uses the index well |
| Memory Grant | Too small = spills to disk |
| Parameter Compiled Value | Reveals parameter sniffing |

---

➡ Continue to **02-finding-and-fixing-problems.md**
