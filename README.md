# Tenqor

Open-source, embeddable engine for reliable lifecycle management of resources belonging to multi-tenant applications.

---

## The Problem

Multi-tenant applications manage resources (database schemas, storage buckets, indexes, workloads) that must be provisioned, suspended, resumed, and eventually removed. Each step exists in two worlds at once: the application's record of what should be true, and the external system's actual state. These diverge whenever anything fails, and the failure modes are not exotic; they are normal:

- **Partial failure**: a multi-step provisioning run where half the steps succeed and the rest do not
- **Retries**: every retry is a duplicate command that must not double-apply
- **Concurrency**: two workers operating on the same tenant at the same time
- **Crashes**: a process dies mid-operation, leaving the outcome unknown
- **Ambiguous outcomes**: a timeout whose external effect may actually have succeeded
- **State drift**: observed state drifting from what the system expects

---

## The Solution

Tenqor provides a durable lifecycle engine that:

- **Models lifecycle explicitly**: a tenant is a versioned state machine, not scattered flags
- **Persists state transitions**: a committed transition is never lost, even if the process that made it dies
- **Makes operations idempotent**: re-delivering a command never re-applies its effects
- **Handles concurrency deterministically**: concurrent transitions against the same tenant are resolved through optimistic version checks; stale attempts are durably rejected
- **Keeps durable history**: every committed outcome is recorded, including rejected commands
- **Reconciles desired vs observed state**: being built toward this in later phases; Phase 1 is the durable lifecycle foundation

---

## Why Tenqor?

Existing tools solve workflow execution, infrastructure provisioning, or individual platform concerns. Tenqor focuses on the gap between them: an application-grade, durable state machine for tenant lifecycle with idempotency, concurrency control, and crash safety, embedded inside the application's own stack.

---

## Current Status

- **Documentation contract frozen:** `00`–`04`, architecture reviewed
- **Phase 1 (Durable Tenant State Machine):** specification complete; implementation not yet started
- **Later phases** (resource execution, desired-state reconciliation, drift detection and repair) are designed by phase boundary only; nothing is advertised that the repository does not yet have

---

## Documentation

- [00: Project Constitution](docs/00-project-constitution.md)
- [01: Problem and Scope](docs/01-problem-and-scope.md)
- [02: Architecture](docs/02-architecture.md)
- [03: Phase 1 Specification](docs/03-phase-1-spec.md)
- [04: Engineering Rules](docs/04-engineering-rules.md)

---

## Repository structure

```text
tenqor/
├── docs/
├── packages/
│   ├── core/
│   └── postgres/
├── apps/
│   └── playground/
├── tests/
├── package.json
├── pnpm-workspace.yaml
├── turbo.json
└── tsconfig.json
```

---

## Contributing

Tenqor is being built as a long-term open-source project, and contributions are welcome.

You can contribute in many ways:

* Report bugs and edge cases
* Improve documentation
* Add tests and failure cases
* Improve developer experience
* Fix implementation issues
* Propose new capabilities
* Review existing code and pull requests
* Build examples and integrations
* Work on future phases as they are opened for contribution

### Before You Contribute

Start by reading the project documentation, especially:

* `docs/00-project-constitution.md`
* `docs/01-problem-and-scope.md`
* `docs/02-architecture.md`
* `docs/03-phase-1-spec.md`
* `docs/04-engineering-rules.md`

These documents describe what Tenqor is, what it is not, and the engineering rules the project follows.

### Issues, Discussions & Pull Requests

If you find a bug, have an idea, or want to work on something:

1. Check whether an existing issue or discussion already covers it.
2. For bugs and implementation fixes, open an issue or submit a PR with a clear description and tests where appropriate.
3. For new capabilities or architectural changes, open a discussion/issue first so the problem and proposed direction can be understood before significant implementation work begins.
4. Keep pull requests focused on one meaningful change.
5. Make sure the change is consistent with the current phase and documented contracts.

You do not need permission to fix a typo, improve documentation, add a missing test, or fix a clearly understood bug.

For larger changes, start a conversation first.

### Development Philosophy

Tenqor is developed incrementally over multiple phases.

Each phase is designed to be independently useful and has explicit boundaries. A future capability should not be pulled into an earlier phase simply because it may eventually be useful.

The project follows:

> **DESIGN → CONTRACT → IMPLEMENT → TEST → BREAK IT → FIX IT → DOCUMENT → FREEZE**

Frozen contracts are not changed casually. If a contribution reveals that a frozen contract needs to change, the change should be discussed and documented before implementation.

### Pull Requests

A good PR should make it easy to answer:

* What problem does this solve?
* Why does it belong in Tenqor?
* Which phase does it belong to?
* What behavior changed?
* How is the behavior tested?
* Does it introduce any new dependency or architectural responsibility?

Contributors are encouraged to challenge assumptions, find failure cases, and improve the design. The goal is not simply to add code. It is to build reliable lifecycle infrastructure that remains understandable as Tenqor grows.

---

## License

MIT: see [LICENSE](LICENSE).