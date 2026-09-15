# 01 — Problem and Scope

This document explains the real problem Tenqor solves, why existing infrastructure does not fully solve it, and precisely what Tenqor does and does not claim.

---

## 1. The multi-tenant lifecycle problem

Multi-tenant applications manage resources that belong to tenants: a database schema, a storage bucket, an index, a service account, a deployed workload. These resources have lifecycles:

- on signup, they must be **provisioned**;
- on churn or abuse, they must be **suspended**;
- on recovery, they must be **resumed**;
- on deletion, they must be **removed**.

This is easy to describe and hard to do reliably, because each step exists in two worlds at once:

1. the **application's record** of what should be true, and
2. the **external system's** actual state.

These two diverge whenever anything fails, and the failure modes are not exotic. They are normal.

## 2. The failure modes that matter

### 2.1 Partial failure

Provisioning a tenant is rarely one atomic action. It is a sequence: create database, apply schema, create role, provision search index, call a webhook. Any step can succeed while a later step fails. The tenant is now half-provisioned, and someone must decide whether the half is acceptable, rolled back, or completed.

### 2.2 Retries

Systems retry. HTTP clients retry on timeout. Queue consumers retry on failure. Operators retry on flaky failures. Every retry is a duplicate command. If the retry re-executes a step that already succeeded, the system must detect that and not double-apply it.

### 2.3 Concurrency

The moment more than one worker exists, two workers can operate on the same tenant at the same time — one suspending while another provisions, one deleting while another resumes. Without a per-tenant control mechanism, these operations interleave unpredictably.

### 2.4 Ambiguous outcomes

A request times out. The caller assumes failure and retries. But the request may actually have completed on the external system. Now the system has performed the side effect twice, or recorded failure when the effect succeeded. The ambiguity is unresolvable by the caller; it must be resolved by the system that owns the operation.

### 2.5 Crash mid-operation

A worker is suspended, killed, or the process restarts between "I started the external call" and "I recorded the result." From the store's perspective, the operation was in progress. From the external system's perspective, it may or may not have run. Someone must decide what the truth is and complete or roll back the work.

### 2.6 Stale workers

A worker dies while holding an operation. Its lock/heartbeat is caducous. A new worker must detect that the previous worker is gone and take over — otherwise the tenant is stuck in an in-progress state forever (or worse, two workers both believe they own the operation).

### 2.7 Desired vs observed state

The application records the state it *wants* (desired state). The actual system drifts: a manual change, a failed job, an orphaned resource. Eventually the observed state diverges from the desired state. Reconciliation — the loop that detects and corrects divergence — is the only defense against drift being permanent.

## 3. Why existing infrastructure does not solve this problem

Each category of existing tooling solves a nearby problem, not this one.

| Existing tool | What it is good at | Why it does not close the loop |
| --- | --- | --- |
| **Queues** (SQS, RabbitMQ, etc.) | At-least-once delivery of a message. | A queue delivers a task twice and has no knowledge of the tenant's state, the operation's validity, or whether the effect was already applied. Partial failures and ambiguous outcomes are the consumer's problem. |
| **Workflow engines** (Temporal, etc.) | Durable execution of long-running workflows with retries and replay. | They handle the *mechanics* of durable execution, but they do not own tenant lifecycle semantics. The application still models the state machine, the transitions, the idempotency, and the reconciliation itself. Tenqor may integrate with such engines later; it does not depend on them. |
| **Infrastructure as code** (Terraform, etc.) | Declarative provisioning of infrastructure, diff-based apply. | It is external to the application's database, not embedded, and it is not the origin of truth for an application's tenant lifecycle decisions. A tenant is more than a set of Terraform resources. |
| **Kubernetes / Crossplane** | Reconciliation loops and controllers over cloud resources. | Operates at cluster/infra scope, not application-tenant scope. Tenants within an app are not K8s custom resources, and the app would not run K8s to get tenant lifecycle. |
| **SaaS control planes** (Temporal Cloud, etc.) | Managed versions of the above. | Same gap, plus the app keeps ownership of semantics. |
| **Tenant-context middleware** | Passing `tenantId` through requests. | That is request plumbing, not lifecycle management. Different problem entirely. |

The gap: **an application-grade, durable state machine for tenant lifecycle, with idempotency, concurrency control, crash safety, and reconciliation semantics, that lives inside the application's own stack.**

## 4. The central idea

> Declare what state a tenant's resources should be in; make the system reliably converge toward that state.

"Reliably" means: after any number of the failures in §2, an operator inspecting the store can always answer two questions without guessing:

1. What is the current lifecycle state of this tenant? — **Always exact.**
2. What operation is or was in progress, and what was its outcome? — **Always recorded.**

## 5. Concrete examples from different domains

The same structural problem appears in every domain. The nouns change; the failure modes do not.

- **B2B SaaS:** on workspace creation, provision a database schema, an Elasticsearch index, and a storage bucket. On cancellation, deprovision all three, but only after billing confirms the final invoice. Suspend a workspace whose payment failed — then resume it if payment succeeds.
- **EdTech:** a school deploys a dedicated backend namespace and an ingestion queue per school year. At year end, rotate and archive. A crash mid-archive leaves the school in "archiving" forever without a stale-worker mechanism.
- **FinTech:** a merchant's settlement account must be actively suspended before writes are blocked, then recreated precisely, in order, before read-only access is restored. Inconsistent state here is not a cosmetic bug.
- **Healthcare:** a clinic creates a compliance boundary, a data store, and a set of audit hooks per site. Hoops on failure: partial provisioning must be visible and repairable, never silently half-done.
- **Marketplaces:** onboarding a new shop provisions subdomains, API keys, and payment routing. Two onboarding requests (double-click, retried HTTP) must yield one provisioned shop.
- **CRM:** syncing an org's integration credentials across regional clusters; suspending an org must not leave one region active.
- **Internal multi-tenant platforms:** a platform team provisions per-team environments; the internal contract demands the same durability as external tenants.

Every case reduces to: *durable lifecycle state per tenant, idempotent transitions, per-tenant exclusivity, crash recovery, and eventually, reconciliation against the real world.*

## 6. Scope of the claim

**Tenqor solves the lifecycle of tenant-scoped resources.** This is a specific, bounded claim. Tenqor explicitly does not claim to solve:

- general workflow/job orchestration (why: not tenant-lifecycle-shaped)
- infrastructure provisioning at infrastructure scale (why: that is Terraform's domain)
- generic scheduling, generic message delivery, generic retry layers
- platform-wide observability, alerting, or incidents
- any business-domain logic (billing, auth, permissions — these are external)

Tenqor will not work correctly if these are pushed into it. Out-of-scope concerns belong outside the engine, reachable only through adapters in later phases.

## 7. Honest limits

- Tenqor does not make external systems reliable. If a cloud API returns garbage, Tenqor records what happened and exposes it; it cannot fix the API.
- Reconciliation is eventual. Tenqor reduces the *window* and *likelihood* of divergence; it does not eliminate physical impossibility (e.g., a provider that permanently loses a resource).
- Durability is bounded by the store. Tenqor is as durable as the PostgreSQL transaction that records its state.
- Tenqor's correctness guarantees are about the **state machine and its operations**, not about the correctness of code that lives in adapters.

These limits are constraints on the design, not marketing caveats.