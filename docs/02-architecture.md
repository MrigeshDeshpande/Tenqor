# 02 — Architecture

The initial architecture, its layers, dependency direction, and the boundary the later phases will require.

---

## 1. Layer model

Tenqor is organised as additive layers with strict dependency direction:

```text
┌──────────────────────────────────────────────┐
│              Application / Service           │  use cases, command intake,
│              (packages/core)                 │  idempotency, orchestration
├──────────────────────────────────────────────┤
│                  Domain                      │  states, transitions,
│              (packages/core)                 │  invariants, version rules
├──────────────────────────────────────────────┤
│          Persistence Abstraction             │  ports/interfaces the domain
│              (packages/core)                 │  depends on (Store, Tx)
├──────────────────────────────────────────────┤
│          PostgreSQL Implementation           │  Drizzle + pg, transactions,
│             (packages/postgres)              │  migrations, constraints
└──────────────────────────────────────────────┘
```

Rules:

1. **Dependency direction is downward only.** The domain layer must never import the persistence layer, the application layer must never import a concrete store, and nothing outside `packages/postgres` may import `pg` or Drizzle.
2. **`packages/core` has no runtime dependencies.** It depends only on TypeScript's standard library plus its own declared abstractions.
3. Every store-specific behavior of consequence — transaction isolation, locking, unique constraints, partial indexes — must be reached through an interface whose contract states the required semantics in terms the domain can reason about.
4. The connect of the core to a store is **dependency injection at the boundary**: the application layer is handed a `Store` implementation. It never constructs one.

## 2. Domain layer

Holds concepts from [00 — Constitution §6]: `Tenant`, `LifecycleState`, `Operation`, `Transition`, `Version`, plus the rules around them.

Responsibilities:

- define the lifecycle state set and the transition table (allowed/prevented, with reasons)
- define invariants (e.g., a tenant never has two in-flight operations; a transition only applies when the observed version matches; terminal states are terminal)
- define the operation/transition semantics, independent of any store
- expose pure functions that validate a command against a tenant snapshot

The domain layer is **store-agnostic and host-agnostic**. It does not know about transactions, SQL, or processes. It knows about states, versions, commands, and validation results.

## 3. Application / Service layer

Holds the use cases the engine exposes to a host application, e.g.:

- provision a tenant
- suspend a tenant
- resume a tenant
- delete a tenant
- query tenant / operation state
- resolve a retried or ambiguous operation

Responsibilities:

- accept commands with idempotency keys
- decide replay vs. new execution (idempotency semantics — [03 §6])
- claim per-tenant exclusivity (concurrency semantics — [03 §7])
- orchestrate the transition through the store interface
- classify and record outcomes and failures

This layer is where **durability is held**: it never returns success to a caller until the store has durably recorded it. It never claims a failure until the outcome is unconditioned, or the ambiguity is explicitly handed to the caller.

## 4. Persistence abstraction (ports)

The core depends on interfaces that state required semantics, not on a database. Phase 1 requires at least these capabilities:

| Capability | Semantic required | Why |
| --- | --- | --- |
| Atomic read-modify-write of a tenant + operation | Either the whole transition commits or none of it does | crash safety, no partial transitions |
| Compare-and-swap on tenant version | A state transition must fail (not silently succeed) if the version it observed is stale | optimistic concurrency, no lost updates |
| Per-tenant in-flight exclusivity | At most one claimable operation per tenant at the store level | two workers cannot both own a tenant |
| Durable operation records | Operations and their outcomes persist; replay is possible | idempotency, audit, takeover |
| Lease / heartbeat on claims | A claim can be observed as stale; takeover is possible | stale-worker recovery |

The abstraction must be honest about the boundary between Phase 1 and later phases (§6).

**OPEN DECISION:** the exact shape of the storage interface (single read-modify-write operation vs. composed primitives; how tx boundaries are expressed in the interface; whether the interface exposes "run work while holding a claim"). Constraint: the interface must not *force* a future Phase 3 implementation to hold a transaction open across external I/O (§6).

## 5. PostgreSQL implementation

`packages/postgres` implements the persistence abstraction with Drizzle + `pg` against PostgreSQL 18.

Scope in Phase 1:

- schema and Drizzle migrations
- repository + transaction code implementing atomic transitions
- version CAS and per-tenant claim enforcement as real database constraints
- operation history tables
- test support (Testcontainers-managed Postgres for integration tests)

The schema is exactly what the abstraction requires; there is no schema for features not yet in scope.

## 6. The two transaction models — and the seam between them

Phase 1 and later phases require **different transaction models**. The architecture must not blur them.

### 6.1 Phase 1 (now): everything transactional

```text
BEGIN
  claim operation (RUNNING)
  validate tenant snapshot + version
  compute next state
  update tenant (state, version + 1)
  record operation outcome (SUCCEEDED / FAILED)
  insert operation history
COMMIT   ← single atomic unit
```

Phase 1 has **no external side effects**, so the entire state transition + operation persistence is **one atomic DB transaction**. Either the tenant moved to the next state *and* the outcome is recorded, or neither happened.

**OPEN DECISION:** whether the domain/application code is structured so a "transition work" callback may run inside the transaction in Phase 1 (valid only because it performs no external I/O). Constraint: the seam must be designed so that Phase 3 can move work *outside* the transaction without redesigning the domain layer.

### 6.2 Phase 3 (future): durable operation, non-transactional effects

Once real side effects exist (cloud APIs, storage, auth), we cannot hold a DB transaction open across external I/O. The later model is:

```text
DB transaction           → durable operation record (PENDING/RUNNING) + claim
worker claims operation  → exclusive per tenant
external side effect     → performed by worker, outside any DB transaction
record outcome           → separate DB transaction: state transition + outcome
```

Key consequences to design for later, not now:

- the operation must be durable *before* the side effect starts, so a crash leaves a recoverable record;
- the "run the side effect" step is re-entrant: a retry must re-run it, an idempotent operation must absorb the duplicate (idempotency keys and provider idempotency);
- outcome recording may be a *different* transaction from claim; a crash between side effect and outcome recording is an **[03] ambiguous outcome** that reconciliation or operator action resolves;
- lease/takeover must decide what a stale-run state means (abort? resume? classify outcome?).

**This model is out of scope for Phase 1. It is documented here only so Phase 1 builds the right seams and never pretends the Phase 1 model will survive unchanged.**

## 7. What is not in the architecture

Nothing of these is in the initial architecture:

- no HTTP server, no CLI, no daemon
- no queue, no scheduler, no workflow engine
- no cloud SDKs, no provider adapters
- no reconciliation loop (Phase 4)
- no drift detection (Phase 5)
- no resource graph (Phase 2)

Each is a deliberate, documented boundary.

## 8. Architecture decisions still open, with constraints

| OPEN | Decision | Constraints any resolution must satisfy |
| --- | --- | --- |
| OPEN DECISION | Storage interface shape | Must not force tx held open across external I/O in later phases; must let Phase 1 implement atomic transitions; must be satisfiable by PostgreSQL in Phase 1. |
| OPEN DECISION | Whether a transition may run in-transaction in Phase 1 | Only because Phase 1 performs no external I/O; must keep a migration path to out-of-tx effects. |
| OPEN DECISION | Teal exposure (public API) of the domain model | Must remain language-neutral per [00 §10]; must not leak `pg`/Drizzle types. |