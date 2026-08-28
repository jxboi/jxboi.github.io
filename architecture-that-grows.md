# Building Simple Software That Can Grow
### A Complete Guide for C# / .NET Developers

---

## Abstract

.NET teams face a false choice: build something simple that can't grow, or build something "for scale" that drowns in complexity before the first customer arrives. This paper shows a third path — **scale-ready simplicity**. We cover:

1. The principles behind systems that start simple and grow easily
2. An honest pros/cons comparison of every mainstream architecture style
3. A concrete .NET structure (the modular monolith) with code examples
4. A full best-practices checklist
5. A decision guide for picking the right style

**The core idea:** no architecture is universally correct. The right one depends on *what* you're scaling (traffic? teams? features?) and how cheaply you can change your mind later.

---

## Table of Contents

1. [What Kind of "Growth" Do You Mean?](#1-what-kind-of-growth-do-you-mean)
2. [Three Rules to Remember](#2-three-rules-to-remember)
3. [Architecture Styles — Pros and Cons](#3-architecture-styles--pros-and-cons)
4. [The Recommended Core: The Modular Monolith](#4-the-recommended-core-the-modular-monolith)
5. [Growing When Traffic Increases](#5-growing-when-traffic-increases)
6. [Growing When the Team Grows](#6-growing-when-the-team-grows)
7. [Best Practices Checklist](#7-best-practices-checklist)
8. [Cheat Sheet: Which Architecture Should I Pick?](#8-cheat-sheet-which-architecture-should-i-pick)
9. [The Bottom Line](#9-the-bottom-line)
10. [Further Reading](#10-further-reading)

---

## 1. What Kind of "Growth" Do You Mean?

"Scaling" means different things. Know which one you care about:

| Type of growth | Example problem |
|---|---|
| 🚦 **More traffic** | Too many requests, slow database |
| 👥 **More developers** | People stepping on each other's code |
| 📦 **More features** | Every new feature breaks old ones |
| 🔄 **More change** | New products, new integrations |

**Surprise:** Traffic is usually the *least* common problem. Most teams suffer from **too many developers** or **too many features**. Slow code is fixable in a day. A messy codebase where 10 people can't work safely? That takes years to fix.

---

## 2. Three Rules to Remember

### Rule 1: Simplicity is not a phase — it's the goal

Every abstraction you add is a tax every future developer pays — forever.

**Tip:** Only create an interface or abstraction when you have a real reason: a second implementation, or a hard boundary. Don't create an `IRepository<T>` just because it "seems right."

### Rule 2: Make it cheap to be wrong

You *will* guess wrong about what needs to change. That's fine — as long as fixing it is easy:

- Keep business logic separate from the database and HTTP stuff.
- Don't guess the future. Just make sure you can change your mind without pain.

### Rule 3: Know which decisions are hard to undo

Some decisions are easy to reverse (add a NuGet package, rename a class). Some are almost impossible:

| Decision | Hard to undo? | How much care it needs |
|---|---|---|
| Add a NuGet package | 😊 Easy | Low |
| Extract a method to another class | 😊 Easy | Low |
| Your database tables and columns | 😱 Very hard | **Huge** |
| Your public API contract | 😱 Very hard | **Huge** |
| Programming language / cloud platform | 😱 Very hard | Huge |
| Monolith vs. services | 😊 Easy (if code is organized well) | Medium |

👉 **Spend 90% of your thinking on the database design and API contracts.** Everything else should stay easy to change.

---

## 3. Architecture Styles — Pros and Cons

Every style trades something:

| Style | What it trades away |
|---|---|
| Layered monolith | Module boundaries; independence between features |
| Clean/Hexagonal | Some simplicity; onboarding speed for juniors |
| Vertical slices | A shared "big picture" of the domain |
| Modular monolith | Nothing upfront — but requires boundary discipline |
| Microservices | Operational simplicity; local transactions; fast early delivery |
| Event-driven | Easy debugging; synchronous reasoning |
| Serverless | Control, portability, predictable performance |

There is no "winning" row. There is only *fit*. Here's each one in detail.

---

### 3.1 The Simple Layered Monolith (Classic N-Tier)

The default ASP.NET style: Controllers → Services → Repositories → Database. Usually a few projects.

```
MyApp.Web/        (controllers)
MyApp.Services/   (business logic)
MyApp.Data/       (EF Core, database)
```

**Pros**
- ✅ Everyone knows it. Fast to build.
- ✅ Easy to debug — one app, one stack trace.
- ✅ One thing to deploy. Easy CI/CD.
- ✅ Database transactions just work across all features.
- ✅ Easy to hire for.

**Cons**
- ❌ Every feature touches every layer, so everyone's changes conflict.
- ❌ The shared "Data" or "Entities" project becomes a dumping ground; coupling grows silently.
- ❌ No clear ownership — one team's change breaks another team's feature.
- ❌ Compiles/tests as one unit; builds get slower as code grows.
- ❌ Very hard to split into services later — nothing was designed to separate.

**Use it when:** prototypes, internal tools, MVPs, teams under ~5 developers, systems you expect to throw away. Fine to *start* here — just restructure before month 12 if the product succeeds.

---

### 3.2 Clean Architecture / Hexagonal

Business logic in the middle; infrastructure (database, HTTP, queues) at the edges behind interfaces. Dependencies point inward.

```
MyApp.Domain/          (business rules — zero dependencies)
MyApp.Application/     (use cases, interfaces)
MyApp.Infrastructure/  (EF Core, external services)
MyApp.Api/             (controllers, DI setup)
```

**Pros**
- ✅ Business rules are easy to test without a database — fast unit tests.
- ✅ Swap SQL Server for Postgres, or REST for gRPC, without touching business logic.
- ✅ The compiler prevents database code leaking into business rules.
- ✅ Great when your business rules are genuinely complicated (finance, insurance, logistics).

**Cons**
- ❌ Lots of ceremony for simple apps: mapping classes, interfaces with one implementation, DTO translation everywhere.
- ❌ New developers get confused ("why can't my controller just use DbContext?") — and sometimes they're right.
- ❌ Often becomes cargo-cult: layers and patterns that exist "because the architecture says so."
- ❌ The classic failure mode: `IRepository<T>` + `IUnitOfWork` + MediatR + AutoMapper, all added by reflex, slowing down an app that just reads and writes rows.

**Use it when:** your domain is genuinely complex and long-lived. Skip it for simple CRUD apps.

---

### 3.3 Vertical Slices

Organize by **feature**, not by technical layer. Everything for one feature lives together.

```
Features/
  Orders/
    PlaceOrder/    (endpoint + logic + database code, all in one folder)
    CancelOrder/
  Billing/
    CreateInvoice/
```

**Pros**
- ✅ Everything about a feature is in one place. Changes are easy and local.
- ✅ Each feature uses only the pattern it needs — no forced abstractions.
- ✅ Deleting a feature = deleting one folder.
- ✅ New developers find things fast: "find the feature, read the folder."

**Cons**
- ❌ Shared rules (like "all money uses our `Money` class") need deliberate extra care.
- ❌ Slices can copy-paste from each other and drift apart.
- ❌ Harder to see the "big picture" of your entities and their invariants.

**Use it when:** you build lots of features and mostly do CRUD. Great *inside* modules (see 3.4).

---

### 3.4 The Modular Monolith ⭐ (Recommended for most teams)

**One app to deploy, but internally split into clear "modules"** — like Billing, Orders, Customers. Each module has a public interface and owns its own data.

**Pros**
- ✅ As simple to run as a monolith: one deploy, one stack trace.
- ✅ But with clear boundaries — like services, but without the distributed-systems pain.
- ✅ The compiler enforces your boundaries (via project references) — free governance.
- ✅ Per-module tests and ownership → real team scaling.
- ✅ Later, you can pull a module out into its own service — without a rewrite.
- ✅ Per-module `DbContext`s make per-module databases trivial later.
- ✅ Database transactions still work within a module.

**Cons**
- ❌ Needs discipline — modules can cheat unless you automate enforcement (architecture tests).
- ❌ Everything deploys together; a bad module can take down the app (fix with feature flags and careful deploys).
- ❌ The whole app scales *somewhat* together until you extract modules (usually fine for years).
- ❌ One shared runtime means one noisy feature can slow down others.
- ❌ Less "résumé-attractive" — some teams jump to microservices for status, not need.

**Use it when:** basically always, as a starting point. The correct answer to "should we do microservices?" is "yes, eventually — for the modules that earn it."

---

### 3.5 Microservices

Many small, separately deployed services that talk over HTTP/gRPC or a message broker.

**Pros**
- ✅ Each service deploys and scales on its own — true technical elasticity.
- ✅ Each team owns their service and releases when they want.
- ✅ One service crashing doesn't take down the others.
- ✅ Different tech per service is possible (a Python ML service beside C# services is legitimate).
- ✅ Per-service databases and scaling strategies.

**Cons**
- ❌ All the pain of distributed systems **immediately**: network failures, retries, duplicate messages, "eventually consistent" data, sagas instead of transactions.
- ❌ Serious monitoring from day one: distributed tracing (OpenTelemetry), correlation IDs, log aggregation — real infrastructure, not a NuGet package.
- ❌ Wrong boundaries are catastrophic: a "distributed monolith" gives you *all* the costs of both worlds.
- ❌ Versioning, contract testing, and cross-service refactoring are slow.
- ❌ Local development and CI/CD get complicated fast.
- ❌ Under ~15–20 developers, the overhead usually outweighs the benefit.

**Use it when:** you're a large organization, or one module genuinely needs its own scaling, availability, or release cadence. As a *destination* for a few modules — not a *starting point*.

---

### 3.6 Event-Driven (Queues and Messages)

Parts of the system talk through a message broker (Azure Service Bus, RabbitMQ, Kafka) instead of calling each other directly.

**Pros**
- ✅ Loosely connected pieces — add new listeners without touching senders.
- ✅ Traffic spikes get absorbed by the queue instead of cascading into failures.
- ✅ Great for background work: emails, reports, notifications, projections.
- ✅ Scales horizontally very well.

**Cons**
- ❌ Much harder to debug — no single stack trace; you debug *traces*, not calls.
- ❌ Data is "eventually consistent" — UIs must handle "still processing..." states.
- ❌ Duplicate messages, ordering, and poison messages are now your problem.
- ❌ Event schema versioning is a discipline teams underestimate.
- ❌ Often overused — "make it async for scale" when one simple database transaction would do.

**Use it for:** background jobs and communication *between* modules. Not for everything — synchronous request/response stays correct for most user interactions.

---

### 3.7 Serverless (Azure Functions / AWS Lambda)

Small functions that run in the cloud, triggered by events, scaling automatically.

**Pros**
- ✅ No servers to manage. Scales to zero (no cost when idle). Pay per execution.
- ✅ Great for spiky, event-triggered jobs: file processing, webhooks, scheduled tasks.
- ✅ Very fast delivery of small units of functionality.

**Cons**
- ❌ Cold starts (better in .NET 8+ with AOT, but real); execution time limits.
- ❌ Tricky local development and testing; vendor lock-in on triggers/bindings.
- ❌ Costs at steady high load can *exceed* containers; unpredictable bills.
- ❌ Bad place for complex business logic — function-per-endpoint fragments your domain model.

**Use it for:** small helpers at the edges of your system (queue processors, blob triggers, timers). Not as the core.

---

### 3.8 Quick Comparison

| | Layered | Clean | Slices | Modular Monolith | Microservices | Event-Driven | Serverless |
|---|---|---|---|---|---|---|---|
| Fast to start | ★★★ | ★★ | ★★★ | ★★★ | ★ | ★★ | ★★★ |
| Simple | ★★★ | ★★ | ★★★ | ★★ | ★ | ★★ | ★★★ |
| Good for complex rules | ★★ | ★★★ | ★★ | ★★★ | ★★ | ★★ | ★ |
| Good for team growth | ★ | ★★ | ★★ | ★★★ | ★★★ | ★★ | ★★ |
| Good for traffic growth | ★★ | ★★ | ★★ | ★★★ | ★★★ | ★★★ | ★★★ |
| Easy to change/extract later | ★ | ★★★ | ★★★ | ★★★ | ★★ | ★★ | ★ |
| Easy to operate | ★★★ | ★★★ | ★★★ | ★★★ | ★ | ★★ | ★★ |
| Easy to debug | ★★★ | ★★★ | ★★★ | ★★★ | ★★ | ★ | ★★ |

**The winning path:**

```
Simple layered app (first few weeks, if even that)
   → Modular Monolith (with vertical slices inside modules)  ← stay here for years
      → Pull out one busy module as its own service          ← only when it earns it
         → Event-driven integration between extracted services
```

---

## 4. The Recommended Core: The Modular Monolith

### 4.1 The Solution Layout

```
MyApp.sln
├── src/
│   ├── MyApp.Api/                  ← the app that runs (Program.cs, endpoints, auth)
│   ├── MyApp.Workers/              ← background worker host (optional)
│   ├── Modules/
│   │   ├── Billing/                ← one module (public contract + implementation)
│   │   ├── Orders/                 ← another module
│   │   └── Customers/              ← another module
│   └── Shared/
│       └── Shared.Kernel/          ← truly shared small things (Money, Result<T>, Clock)
└── tests/
    ├── Orders.Tests/
    ├── Billing.Tests/
    └── MyApp.Architecture.Tests/   ← tests that protect your boundaries!
```

### 4.2 The Rules

**Rule 1 — Modules don't peek inside each other.** They only talk through a public interface:

```csharp
// The ONLY file in Billing that other modules can use
public interface IBillingModule
{
    Task<Result<InvoiceId>> CreateInvoiceAsync(
        Guid customerId, IReadOnlyList<InvoiceLine> lines, CancellationToken ct);
}
```

**Rule 2 — Each module registers itself**, and `Program.cs` stays tiny:

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services
    .AddCustomersModule(builder.Configuration)
    .AddOrdersModule(builder.Configuration)
    .AddBillingModule(builder.Configuration);

var app = builder.Build();
app.MapOrdersEndpoints();
app.MapCustomersEndpoints();
app.Run();
```

**Rule 3 — Each module owns its own data.** Its own `DbContext`, its own tables/schema.

- ❌ No joins across modules.
- ❌ No giant shared `DbContext`.
- ✅ Need customer info inside Orders? Call `ICustomersModule`, or keep a small local copy updated via events.

This rule feels annoying — and pays for itself the first time you extract a module, or the first time two teams fight over a migration conflict in CI.

**Rule 4 — Keep `Shared.Kernel` tiny.** Only put things there that truly belong everywhere (like a `Money` class). Don't let it become a junk drawer.

> **Why this matters:** When Billing gets big and busy, you can pull it out into its own service. You just swap one DI registration for an HTTP client that uses the same `IBillingModule`. **Nobody else's code changes.** That's the whole trick.

---

## 5. Growing When Traffic Increases

When load goes up, don't panic. Follow these steps **in order**:

1. **Make the server bigger** (more CPU/RAM). Boring, but it works longer than you think.

2. **Stay stateless from day one:**
   - Sessions → Redis (or JWTs), never in server memory
   - Distributed cache → `IDistributedCache` / `HybridCache`
   - Files → blob storage, never local disk
   - Data protection keys → persisted (Redis/Azure), or multi-instance auth breaks

   This one habit makes every later step a settings change instead of a refactor.

3. **Add read replicas** for the database (most apps read way more than they write).

4. **Add caching** for hot, frequently-read data (Redis, `HybridCache`) — with a clear invalidation plan per module.

5. **Move a busy module's tables to its own database** — easy, because it already owns its data.

6. **Use queues for background work** — emails, image processing, reports. If the user doesn't need the answer *right now*, put it on a queue:

```csharp
public sealed class InvoiceFinalizer(
    IQueue<InvoiceFinalized> queue,
    BillingDbContext db) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        await foreach (var msg in queue.ReadAllAsync(ct))
        {
            // idempotent handling — always assume duplicate delivery
        }
    }
}
```

   Two queue rules that save you pain:
   - **Assume every message arrives twice.** Make handlers safe to run twice (idempotent).
   - **Never "save to DB, then send message" as two separate steps** — one can fail. Use the **outbox pattern**: save the message in the same database transaction as your data, then a background job publishes it.

7. **When a module truly outgrows the app**, extract it as a service:
   1. Move its implementation to its own repo/deployment.
   2. Publish a client package (or generate one from OpenAPI/gRPC) implementing the same interface.
   3. Swap the DI registration in the composition root.
   4. Move its schema — it was already isolated.

   Because boundaries, contracts, and data ownership were enforced all along, this is weeks of work — not a two-year rewrite.

---

## 6. Growing When the Team Grows

Architecture is really about people. It works when:

- **Each module has a clear owner.** Changing a module's public interface needs the owner's approval (use a `CODEOWNERS` file).
- **Builds stay fast.** If tests take 40 minutes, everyone stops running them. Keep module tests separate and fast (aim: under 10 minutes for the whole build).
- **A new developer can ship something in week one** — inside one module — without understanding the whole system.
- **The module's public interface is its documentation.** Keep it small enough to read in two minutes.

Modules that mirror team ownership make both the org chart and the dependency graph tractable (Conway's Law works for you instead of against you).

---

## 7. Best Practices Checklist

### 🏗️ Structure
- ✅ One app to deploy, many projects. Project references ARE your architecture.
- ✅ Keep things `internal` by default. Only module contracts are `public`. `InternalsVisibleTo` only for the module's own tests.
- ✅ Organize by feature (vertical slices) inside each module; a thin `Shared.Kernel` for genuine invariants (`Money`, `Result<T>`).
- ✅ One `DbContext` per module. Separate schemas. Always.
- ✅ Map module boundaries to team ownership.
- ✅ Boring technology: Generic Host, built-in DI, EF Core, minimal APIs, `IOptions<T>`. Every exotic dependency multiplies the cost of every future change.

### 💾 Data
- ✅ Stateless app from day one (Redis, blob storage, no in-memory state).
- ✅ One database transaction per module per request.
- ✅ Cross-module consistency with events, **never** distributed transactions. (If you think you need a distributed transaction, your boundaries are wrong — use the outbox/sagas.)
- ✅ Use the outbox pattern for reliable messaging.
- ✅ Idempotent message handlers — duplicates are guaranteed eventually.
- ✅ Optimistic concurrency (`RowVersion`) over pessimistic locking for normal web workloads.
- ✅ Scale the database in this order: bigger server → replicas → cache → per-module DBs → partitioning. Don't jump to the end of the list.

### 💻 Code
- ✅ `CancellationToken` in every async method. No exceptions.
- ✅ `AsNoTracking()` for read queries; `ExecuteUpdateAsync`/`ExecuteDeleteAsync` for bulk operations (don't load rows just to change them).
- ✅ Write hot-path mapping by hand; avoid reflection mappers (AutoMapper) in speed-critical code.
- ✅ Validate configuration at startup (`ValidateOnStart` on options), not at 3 a.m. on the first request.
- ✅ Use keyed DI services (.NET 8+) instead of factories of factories.

### 🛡️ Protecting the Architecture (this is the secret)
- ✅ **Write architecture tests that fail the build** when someone breaks the rules (NetArchTest or ArchUnitNET):

```csharp
[Fact]
public void Modules_do_not_reach_into_each_others_internals()
{
    var result = Types.InAssembly(typeof(OrdersModule).Assembly)
        .ShouldNot().HaveDependencyOnAny(
            "MyApp.Billing.Internal", "MyApp.Customers.Internal")
        .GetResult();

    Assert.True(result.IsSuccessful,
        string.Join(", ", result.FailingTypeNames ?? []));
}

[Fact]
public void Domain_does_not_depend_on_infrastructure()
{
    var result = Types.InAssembly(typeof(OrdersModule).Assembly)
        .That().ResideInNamespace("MyApp.Orders.Domain")
        .ShouldNot()
        .HaveDependencyOnAny("Microsoft.EntityFrameworkCore", "Microsoft.AspNetCore.Http")
        .GetResult();

    Assert.True(result.IsSuccessful);
}
```

  > Rules people have to *remember* get forgotten. Rules that **fail the build** survive.

- ✅ Write short **decision records (ADRs)**: one page per big decision — context, decision, consequences, and *how to reverse it*. Keep them in the repo.
- ✅ Keep ~20% of capacity for cleaning up code. Debt is fine; *surprise* debt is not.
- ✅ Once a quarter, ask: "What can we delete?" Unused abstractions, dead flags, unused dependencies — delete them.
- ✅ Use **feature flags** to decouple deploying from releasing — this keeps single-deployable shipping safe as teams grow.

### 📊 Observability (before you need it)
- ✅ Structured logging (`ILogger<T>` + Serilog) and **OpenTelemetry** tracing with correlation IDs from the start. Retrofitting this *during* an outage is miserable.

### ❌ Don't Do These
- ❌ Microservices on day one
- ❌ A generic `IRepository<T>` wrapping `DbContext`
- ❌ One giant shared `DbContext`
- ❌ MediatR + CQRS everywhere "just because"
- ❌ AutoMapper in hot paths
- ❌ Distributed transactions (a signal your boundaries are wrong)
- ❌ Cross-module joins or shared tables
- ❌ Making everything async "for scale" when one SQL transaction would do
- ❌ The big rewrite (extract modules one at a time instead)
- ❌ Abstractions with exactly one implementation and no boundary reason to exist

---

## 8. Cheat Sheet: Which Architecture Should I Pick?

```
Prototype, internal tool, or probably-short-lived project?
  → Simple layered app or vertical slices. Done.

Complex business rules (finance, insurance, logistics)?
  → Modular monolith, with Clean Architecture INSIDE complex modules
    and plain slices for simple CRUD parts.

Team bigger than ~15 people? One module needs different scaling
or its own release schedule?
  → Extract THAT module into a service. One at a time.
    (Not "let's go microservices!")

Spiky background work (files, emails, webhooks)?
  → Queue workers or serverless functions at the edges.

Does the user need the answer right now?
  → Synchronous request/response. (This is the default answer, always.)
```

---

## 9. The Bottom Line

A simple architecture that scales isn't one that predicts the future — it's one that **responds to it cheaply**. The recipe:

1. **Start with one app**, split into modules with clear public interfaces and their own data.
2. **Keep the app stateless** so scaling up is just a settings change.
3. **Protect your boundaries with automated tests**, not good intentions.
4. **Grow step by step**: bigger server → replicas → cache → queues → extract one module at a time.
5. **Align modules with team ownership** so organizational and technical scaling reinforce each other.

> **Simplicity isn't the opposite of scalability — it's what makes scalability possible.**

Every piece of complexity in your system should have to *earn its place*. Add it when it's needed, not before.

---

## 10. Further Reading

- **Vladimir Khorikov** — *Unit Testing Principles, Practices, and Patterns* (testability without over-abstraction)
- **Sam Newman** — *Monolith to Microservices*; *Building Microservices* (2nd ed.)
- **Mark Richards & Neal Ford** — *Fundamentals of Software Architecture*; *Software Architecture: The Hard Parts*
- **Ford, Parsons, Kua** — *Building Evolutionary Architectures*
- **Steve Millett & Nick Tune** — *Patterns, Principles, and Practices of DDD in .NET*
- **Martin Fowler** — "MonolithFirst", "Microservices" (martinfowler.com)
- **Microsoft** — .NET Application Architecture Guides; eShop reference apps
- **NetArchTest.Rules / ArchUnitNET** — NuGet packages for architecture tests
