# Monolith

A single deployable unit containing all the application's functionality. The default starting
point for most products, not a legacy pattern to be ashamed of.

## Why it wins early

- One codebase, one deploy pipeline, one place to `grep`. Refactoring across "service"
  boundaries is a compiler-checked function move, not a coordinated multi-repo release.
- In-process function calls instead of network calls: no serialization, no partial failure, no
  need for a service mesh or distributed tracing to understand a request's path.
- Transactions are just database transactions (ACID, one connection) — no distributed
  transaction/saga machinery needed (see `distributed-systems/distributed-transactions.md`).
- Cheapest to operate: one thing to deploy, monitor, and put on-call for.

## Where it breaks down

- **Deploy coupling**: any team's change requires a full-app deploy; a bug in one module can
  block every team's release.
- **Scaling is coarse**: if one code path is CPU-heavy and another is I/O-heavy, you scale the
  whole process to satisfy the hungriest path — you can't scale just the hot module.
- **Blast radius**: an unhandled exception or memory leak in one module can take the whole
  process down, including unrelated features.
- **Build/test time** grows with codebase size and eventually dominates the inner dev loop.

## The modular monolith

Enforce module boundaries inside the single deploy: separate packages/modules with explicit
public interfaces, a linter or architecture-fitness test (e.g. dependency-cruiser, ArchUnit)
that fails the build on a boundary violation, and no direct cross-module DB table access (go
through the module's own repository/service layer). This gets most of the maintainability
benefit of microservices — and, critically, makes a future extraction to a real service a
matter of moving one module out, not an untangling project.

## Staff-engineer notes

- Don't split a monolith to "fix" an org problem (two teams stepping on each other's code) if a
  clearer module-ownership model (CODEOWNERS, module boundaries, a deploy train) fixes it more
  cheaply.
- The strongest signal it's time to extract a service isn't complexity — it's a genuine need for
  independent deploy cadence, independent scaling, or a different language/runtime for one part
  of the system (e.g. a ML inference path that needs Python/GPU while the rest is a Go API).
- Watch for the "distributed monolith" anti-pattern on the other side: services that were split
  out but still deploy in lockstep and share a database — this has all of microservices' latency
  and operational cost with none of the independent-deploy benefit.
