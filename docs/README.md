# Happy Trails Documentation

Development and reference documentation lives in this directory.

- [`REQUIREMENTS.md`](REQUIREMENTS.md) — canonical functional requirements, scope boundaries, performance/scalability requirements, and MVP acceptance criteria.
- [`DESIGN.md`](DESIGN.md) — open design space: alternatives, constraints, performance tradeoffs, and questions. This is intentionally not a committed architecture during pre-alpha feasibility work.
- [`SPIKE-001.md`](SPIKE-001.md) — Build 42 feasibility and comparative performance investigation for vehicle observation, terrain mutation, persistence, synchronization, and vegetation damage.
- [`REFERENCE-IMPLEMENTATIONS.md`](REFERENCE-IMPLEMENTATIONS.md) — comparative review of supplied third-party B42 mods used only as research inputs and performance cautionary examples.
- [`TESTING.md`](TESTING.md) — repeatable single-player and dedicated-server test procedures and evidence standards.
- [`adr/`](adr/) — architecture decision records for significant technical choices only after spike evidence supports a decision.

Repository-level documentation remains at the project root:

- [`../README.md`](../README.md) — project overview and current development status.
- [`../CHANGELOG.md`](../CHANGELOG.md) — human-readable development and release history.
- [`../LICENSE`](../LICENSE) — Apache License 2.0.
- [`../NOTICE`](../NOTICE) — project attribution notice.

The documentation order for pre-alpha work is:

```text
requirements and scope
-> comparative evidence / spikes
-> measured alternatives
-> ADR-backed architecture decisions
-> implementation
```

Reference code is not treated as project code. New experimental questions should be captured as spike work; durable choices should be recorded as ADRs only after evidence exists.
