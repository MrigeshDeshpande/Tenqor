# 00 — Project Constitution

**Tenqor** — open-source, embeddable engine for reliable lifecycle management of resources belonging to multi-tenant applications.

Status: proposal. This document is the highest-level contract of the project. It changes only with explicit agreement.

---

## 1. Product definition

Tenqor is an open-source, embeddable engine that makes the **lifecycle of multi-tenant resources reliable**.

An application adopts Tenqor when it has resources that must be:
- created,
- provisioned,
- suspended,
- resumed,
- deleted,

in a way that survives retries, crashes, concurrency, and ambiguous outcomes — without the application re-implementing distributed-systems machinery for each tenant operation.

Tenqor is **domain-independent**. It understands only infrastructure-level concepts. It contains no business-domain concepts.

## 2. Product goal

> Declare the state a tenant's resources should be in, and make the system reliably converge toward that state.

The engine's primary responsibility is making tenant lifecycle operations reliable despite:

- retries
- process crashes
- duplicate commands
- concurrent workers
- partial failure
- timeouts
- ambiguous external outcomes
- stale workers
- temporary infrastructure failures

Correctness in the presence of these failures is the product. Performance, DX, and broad adapter coverage are secondary concerns that must never be bought with correctness.

## 3. Core philosophy

Tenqor is not primarily about CRUD. The central engineering question is:

> What happens when the system fails at every possible point?

Tenqor's model is a **durable, versioned state machine** persisted in a transactional store. State transitions are:

- **durable** — a committed transition is never lost, even if the process that made it dies;
- **idempotent** — re-delivering a command does not re-apply effects;
- **exclusive** — at most one in-flight operation applies to a tenant at a time;
- **versioned** — a transition is only applied against the state the caller actually observed.

The database is the source of truth. In-memory state is a cache, never an authority.

## 4. Scope

In scope (eventually, across phases):

- tenant lifecycle state model
- durable operations, idempotency, versioning, optimistic concurrency
- resource model and dependency graph
- durable resource execution and provider adapters
- desired-state reconciliation
- drift detection and repair

Phase-by-phase scope is defined in [03 — Phase 1 Specification](03-phase-1-spec.md) and the phase boundary rules in §8.

## 5. Non-goals

**Tenqor is not:**

- Terraform
- Kubernetes
- Crossplane
- Temporal
- Inngest
- a queue
- an authentication provider
- an authorization system
- a billing system
- a database
- a storage provider
- a generic tenant-context middleware library
- a complete SaaS control plane

Tenqor may integrate with these systems through adapters in later phases. It does not attempt to replace them.

Tenqor does not own: authentication, authorization, billing, databases, storage, search, or external services. Business entities — students, patients, invoices, courses, payments, doctors, employees, subscriptions — are **never** part of the core model. They may be represented only inside resource adapters, never inside Tenqor's core concepts.

Tenqor does not claim to solve:
- general workflow orchestration
- general job scheduling
- infrastructure provisioning at infrastruct scale
- platform-wide observability/alerting

## 6. Domain boundaries

Tenqor understands only infrastructure-level concepts:

| Concept | Meaning |
| --- | --- |
| Tenant | An isolated unit of tenancy whose resources are lifecycle-managed. |
| Resource | A managed artifact belonging to a tenant. |
| Desired State | The declared target configuration of a resource. |
| Observed State | The last verified configuration of a resource. |
| Operation | A durable, idempotent unit of lifecycle work against a tenant. |
| Transition | A state change of the tenant lifecycle, driven by an operation. |
| Dependency | A relationship between resources that constrains execution order. |
| Generation / Version | A monotonic marker of tenant/resource state changes. |
| Failure | An explicit, classified outcome of an operation. |
| Reconciliation | The loop that drives observed state toward desired state. |

These concepts are defined **independently of any programming language or business domain**.

## 7. Phase boundaries

Tenqor is built in strictly separated phases. A phase is implemented only after the previous phase's specification is reviewed and frozen.

| Phase | Deliverable | In scope summary |
| --- | --- | --- |
| **Phase 1** | Durable Tenant State Machine | tenant model, lifecycle states, transitions, transition validation, durable operations, idempotency, versioning/optimistic concurrency, PostgreSQL persistence, crash-safe transitions, operation history, concurrency & failure tests. Must be independently useful. |
| **Phase 2** | Resource model and dependency graph | resources, dependency graph, declared relationships. |
| **Phase 3** | Durable resource execution and provider adapters | adapter contract, out-of-transaction side-effect execution, durable effect/outcome recording. |
| **Phase 4** | Desired-state reconciliation | desired-state engine, convergence loop. |
| **Phase 5** | Drift detection and repair | observed-state polling, drift detection, repair plans. |

**Phase leakage is forbidden**: a later phase's concern must not be implemented or half-implied within an earlier phase's code or tests. Where a later phase will require a different model, the earlier phase documents the seam explicitly but does not build it.

## 8. Technology principles

Initial implementation stack:

- TypeScript
- Node.js 24 LTS
- PostgreSQL 18
- node-postgres (`pg`)
- Drizzle
- pnpm
- Turborepo
- Vitest
- Testcontainers
- Docker
- GitHub Actions

Rules:

1. Additional technologies must not be introduced without an explicit architectural reason and a review.
2. The core package must remain lightweight. It must **not** depend on Fastify, cloud SDKs, Redis, Kafka, Temporal, Inngest, or Kubernetes.
3. PostgreSQL is the reference store. Other stores may be added later only through the persistence abstraction, never by branching the domain model.
4. Runtime requirements may be pinned to a minimum Node LTS, never to the latest release. Versions are locked at the moment dependencies are first installed, not earlier.

**OPEN DECISION:** exact toolchain versions (TypeScript, pnpm, Vitest, Drizzle) are not pinned yet. They are pinned when Phase 1 implementation begins, after the architecture review.

## 9. OSS philosophy

Tenqor is open source:

- permissive license (MIT)
- public architecture docs before implementation
- adapter ecosystem intended (community providers), but adapters are a later-phase concern
- no proprietary control plane, no hosted-only features in the core

## 10. Language-neutrality principle

Tenqor is TypeScript-first, but its **domain model is language-neutral**.

Interoperability surfaces (planned, not yet built) are:

- JSON
- JSON Schema
- OpenAPI
- HTTP
- well-defined semantic contracts

The domain model must not be made unnecessarily TypeScript-specific (e.g., no reliance on `class` internals, `instanceof`, private fields, or TS-only types as part of the model's published contract). Python/Go SDKs are **not** in scope for Phase 1.

## 11. Decision record convention

Two kinds of unresolved item exist in the documents:

| Tag | Meaning |
| --- | --- |
| `OPEN DECISION` | Not decided. Must be reviewed and resolved before the concerned phase is implemented. A `OPEN DECISION` must say what constraints any resolution must satisfy. |
| `PROPOSED` | A concrete candidate with accompanying rationale. Still subject to review. |

When a decision is frozen, it is updated in place and a `DECIDED` record is added with the date and the reasoning. The docs are the source of truth, not the code.

## 12. Documentation style

Documents are written for experienced backend engineers:

- precise, technical, direct, honest
- no marketing language, buzzwords, or exaggerated claims
- no unsupported guarantees ("highly scalable" without evidence, fake benchmarks)
- unresolved items are marked, never silently invented