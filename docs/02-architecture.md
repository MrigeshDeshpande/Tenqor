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
- define invariants (e.g., concurrent transition attempts on a tenant serialize such that exactly one commits; a transition only applies when the observed version matches; terminal states are terminal)
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
- decide replay vs. new execution (idempotency semantics — [03 §5])
- execute the transition atomically through the version-guarded store operation (concurrency semantics — [03 §6])
- resolve a command whose outcome is commit-result-ambiguous by replay through the idempotency key
- classify and record outcomes and failures
- distinguish **domain rejections** (durable recorded results) from **transaction/infrastructure failures** (rolled back, nothing recorded)

This layer is where **durability is held**: it never returns success to a caller until the store has durably recorded it. It never reports a deterministic rejection as a system failure, and it never hides an infrastructure failure as a domain result.

## 4. Persistence abstraction (ports)

The core depends on interfaces that state required semantics, not on a database. Phase 1 requires at least these capabilities:

| Capability | Semantic required | Why |
| --- | --- | --- |
| Atomic transition transaction | Either the whole transition commits or none of it does; a deterministic rejection — including a CAS version conflict — is durably recorded as a *committed* result without changing tenant state (CAS=0 is never a blanket rollback) | crash safety, no partial transitions, observable rejections |
| Compare-and-swap on tenant version | A state transition must fail (not silently succeed) if the version it observed is stale | optimistic concurrency, no lost updates, transition serialization |
| Durable operation records | Operations and their outcomes (including rejections) persist; lookup by operation id per tenant | idempotency replay, audit, future takeover |
| Append-only operation history | History entries are never mutated | audit, "what happened, exactly?" |

The abstraction must be honest about the boundary between Phase 1 and later phases (§6). **Phase 1 does not expose worker claims, leases, heartbeats, or takeover through the persistence port** — Phase 1 has no out-of-transaction work and therefore no worker ownership. Phase 3 introduces the worker-execution persistence contract (durable claim, lease, takeover) when there is a real reason for it.

**OPEN DECISION:** the exact shape of the storage interface (single read-modify-write operation vs. composed primitives; how tx boundaries and the rejection path are expressed). Constraint: the interface must not *force* a future Phase 3 implementation to hold a transaction open across external I/O (§6), it must not expose Phase 1 to the worker-claim semantics that Phase 3 will add, and it must express the CAS-rejection path (CAS=0 → committed durable rejection, or idempotent replay) **distinctly** from infrastructure rollback ([03 §8.1], [04 §2]).

## 5. PostgreSQL implementation

`packages/postgres` implements the persistence abstraction with Drizzle + `pg` against PostgreSQL 18.

Scope in Phase 1:

- schema and Drizzle migrations
- repository + transaction code implementing atomic transitions and the rejection path
- version CAS enforcement as a real database constraint
- operation history tables
- test support (Testcontainers-managed Postgres for integration tests)

The schema is exactly what the abstraction requires; there is no schema for features not yet in scope.

## 6. The two transaction models — and the seam between them

Phase 1 and later phases require **different transaction models**. The architecture must not blur them.

### 6.1 Phase 1 (now): everything transactional

```text
BEGIN
  validate command against tenant snapshot + observed version

  if invalid (domain rejection):
      insert operation (FAILED, classified rejection)
      insert history (REJECTED)
  else:
      compute next state
      update tenant (state, version + 1)          -- CAS: WHERE id = $id AND version = $observed
      if 0 rows (version conflict):               -- deterministic rejection, not an error
          insert operation (FAILED, VERSION_CONFLICT)
          insert history (REJECTED)
      else:
          insert operation (SUCCEEDED, outcome)
          insert history
COMMIT   ← single atomic unit
```

The detailed algorithm, including the after-CAS idempotency re-check for the same-`OperationId` race, is in [03 §8.1]; this sketch and [03 §8.1] are the same transaction model.

Phase 1 has **no external side effects**, so the entire state transition + operation persistence is **one atomic DB transaction**. Either the tenant moved to the next state *and* the outcome is recorded, or (for a rejection) the rejection is durably recorded while the tenant is unchanged. A transaction/infrastructure failure (connection loss, serialization failure, infra-caused constraint error) rolls back entirely and records nothing. Classification is by **semantic cause**, not by the raw database error: a duplicate idempotency key is absorbed by the idempotency path before insertion, a schema/programmer bug is surfaced as such, and only genuine infra-caused errors roll back ([03 §7.1]).

Phase 1 operation statuses are `SUCCEEDED` and `FAILED`; a `PENDING`/`RUNNING`/`PROPOSED` claim state does not exist in Phase 1. There is no durable moment between "command accepted" and "committed result" that any other process can observe, so there is no claim to own, lease, or take over.

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
- outcome recording may be a *different* transaction from claim; a crash between side effect and outcome recording is an **external ambiguous outcome** that reconciliation or operator action resolves;
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
| OPEN DECISION | Storage interface shape | Must not force tx held open across external I/O in later phases; must let Phase 1 implement atomic transitions; must express the CAS-rejection path (CAS=0 → committed durable rejection or idempotent replay) distinctly from infrastructure rollback; must be satisfiable by PostgreSQL in Phase 1. |
| OPEN DECISION | Whether a transition may run in-transaction in Phase 1 | Only because Phase 1 performs no external I/O; must keep a migration path to out-of-tx effects. |
| OPEN DECISION | Teal exposure (public API) of the domain model | Must remain language-neutral per [00 §10]; must not leak `pg`/Drizzle types. |