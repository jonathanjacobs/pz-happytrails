# Happy Trails Documentation

Development and reference documentation lives in this directory.

- [`REQUIREMENTS.md`](REQUIREMENTS.md) — canonical functional requirements, constraints, MVP scope, and acceptance criteria.
- [`DESIGN.md`](DESIGN.md) — working architecture, state model, persistence concepts, multiplayer model, and performance considerations.
- [`SPIKE-001.md`](SPIKE-001.md) — initial Build 42 feasibility investigation for vehicle tracking, terrain mutation, persistence, and synchronization.
- [`TESTING.md`](TESTING.md) — repeatable single-player and dedicated-server test procedures and evidence standards.
- [`adr/`](adr/) — architecture decision records for significant technical choices once evidence supports a decision.

Repository-level documentation remains at the project root:

- [`../README.md`](../README.md) — project overview and current development status.
- [`../CHANGELOG.md`](../CHANGELOG.md) — human-readable development and release history.
- [`../LICENSE`](../LICENSE) — Apache License 2.0.
- [`../NOTICE`](../NOTICE) — project attribution notice.

New investigation notes should be added as numbered spike documents when the question is experimental. Durable architectural choices should be recorded as ADRs. This keeps exploratory evidence separate from long-lived design decisions.
