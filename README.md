# Tenqor

Open-source, embeddable engine for reliable lifecycle management of resources belonging to multi-tenant applications.

---

**Status:** Architecture — pre-implementation.

---

## Documentation

- [00 — Project Constitution](docs/00-project-constitution.md)
- [01 — Problem and Scope](docs/01-problem-and-scope.md)
- [02 — Architecture](docs/02-architecture.md)
- [03 — Phase 1 Specification](docs/03-phase-1-spec.md)
- [04 — Engineering Rules](docs/04-engineering-rules.md)

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

Contributors are encouraged to challenge assumptions, find failure cases, and improve the design. The goal is not simply to add code — it is to build reliable lifecycle infrastructure that remains understandable as Tenqor grows.

---

## License

MIT — see [LICENSE](LICENSE).