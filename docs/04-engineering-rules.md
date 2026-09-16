# 04 — Engineering Rules

Rules that govern every line of Tenqor code, every test, and every review. Each rule states its rationale and how it is verified.

---

## 1. Correctness before performance

**Rule.** No optimization is accepted at the cost of an invariant, a durability guarantee, or a failure-mode analysis. Performance work happens only after correctness is demonstrated and a real need is measured.

**Rationale.** Tenqor's product is reliability under failure. There is no "fast and occasionally wrong" mode.

**Verified by.** Concurrency and failure tests run and pass before any perf-sensitive change is merged; perf claims require a benchmark in the repo, not prose.

## 2. Explicit failure semantics

**Rule.** Every operation can fail, and failure is an explicit, classified outcome — never an exception swallowed, never a silent empty retry, never a catch-all `catch {}`.

**Rationale.** Unclassified failures make ambiguous outcomes unrecognizable.

**Verified by.** Code review: every `catch` and every non-success path maps to a documented classification ([03 §7.1]) or is marked `OPEN DECISION`.

## 3. No hidden retries

**Rule.** Retries are explicit, bounded, observable, and governed by the documented retry policy. No auto-retry lives inside the domain layer; no infinite retry; no retry that ignores version or state.

**Rationale.** Hidden retries convert deterministic failures into ambiguous outcomes and corrupt idempotency reasoning.

**Verified by.** Grep-review for retry loops; each retry decision cites the policy ([03 §7.4]).

## 4. No global mutable state

**Rule.** Tenqor code holds no global mutable state: no module-level counters, no ambient singletons, no hidden shared caches. State lives in the store; everything else is injected.

**Rationale.** Global mutable state makes concurrency and crash reasoning unsound and untestable.

**Verified by.** Review; tests that exercise the same engine instance from concurrent callers must behave deterministically.

## 5. No unnecessary abstractions

**Rule.** An abstraction layer exists only when there is a second, real consumer or a second, real implementation planned. No interfaces for interfaces' sake.

**Rationale.** Every abstraction is a place where semantics can drift or be misremembered.

**Verified by.** Review: name the second consumer/implementation of each abstraction or delete it. (Exceptions: the storage port is justified by [02 §4] requirements, and language-neutrality [00 §10].)

## 6. No premature distributed infrastructure

**Rule.** Nothing distributed is introduced until the failure it addresses is demonstrated. No Redis, Kafka, queues, or workflow engines in Phase 1 or Phase 2.

**Rationale.** Distributed systems multiply ambiguity; the transaction model must be simple and exact before it becomes distributed ([02 §6]).

**Verified by.** Dependency review: Phase 1 dependencies must not grow beyond the [00 §8] stack.

## 7. SQL behavior must be understood, not hidden

**Rule.** Every concurrency-critical SQL statement (the CAS update and the transaction boundaries) must be understandable by a reviewer, documented with its intent, and tested against real PostgreSQL. SQL is not a leak to be abstracted away; it is the contract.

**Rationale.** The database enforces the invariants, not the ORM. Hiding SQL hides the mechanism that makes Tenqor correct.

**Verified by.** [03 §10] requirements: integration tests run against Postgres via Testcontainers, not mocks.

## 8. Every concurrency guarantee must have a test

**Rule.** A concurrency guarantee with no test is a claim, not a guarantee. The failure probes ([03 §12]) are the test suite's spine; adding a guarantee without adding its probe is a review blocker.

**Rationale.** Concurrency bugs are the product's core risk; they do not show up in happy-path runs.

**Verified by.** CI: the probe suite runs on every PR against Postgres 18.

## 9. Every important invariant must be documented

**Rule.** Invariants named in the docs (Phase 1: version CAS, replay, durable rejection, terminal-finality, append-only history — per-tenant exclusivity is a Phase 3 invariant, not a Phase 1 one) are documented *where they are enforced* — next to the enforcement, in a comment tied to the doc section. Docs are the source of truth; comments point back to them.

**Rationale.** An invariant that survives only in a head is lost.

**Verified by.** Review checklist: enforcement site names its doc section.

## 10. No phase leakage

**Rule.** Phase N code implements only Phase N scope. Adapters, reconciliation, drift detection, and resource graphs do not exist in Phase 1 code, not even stubs. Where a future phase requires a different model, the seam is *documented* (as in [02 §6]) but not *built*.

**Rationale.** Premature seams become half-features and contradictory invariants.

**Verified by.** Review: every new file maps to a Section of this spec; files in later-phase scope are rejected.

## 11. The DB transaction model is exact

**Rule.** Phase 1 transitions are one atomic transaction ([03 §8]). Nobody widens, splits, or softens that boundary, and no code ever holds the transaction open across external I/O. The two transaction models ([02 §6]) are never blended.

**Rationale.** Blending the models either loses atomicity or leaks a transaction across I/O.

**Verified by.** Code review of the transition path; a test asserting rollback atomicity with an injected mid-transaction failure.

## 12. Doubt is recorded, not absorbed

**Rule.** If a behavior is not decided, it is marked `OPEN DECISION` with its constraint set. It is never "resolved" by choosing the most convenient implementation, and never hidden inside a config flag.

**Rationale.** The docs are the contract future implementers and reviewers rely on ([00 §11]).

**Verified by.** Review: any `OPEN DECISION` that matters to the code path being changed is called out in the review, not silently settled.

## 13. Honest claims in code and prose

**Rule.** No comment, README, or docs sentence asserts a guarantee the tests do not demonstrate. "Crash-safe" means a test simulates a crash and observes the outcome.

**Rationale.** Exaggerated claims in an OSS project become trust debt the moment a user hits them.

**Verified by.** Release checklist: user-facing claims (README, docs) must trace to a passing test or be marked as an aspiration/`OPEN DECISION`.