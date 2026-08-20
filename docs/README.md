# Happy Trails Documentation

The repository root is intentionally concise. Detailed design, testing, validation evidence, spikes, architecture decisions, release guidance, and Project Zomboid policy notes live under `docs/`.

## Current project state

- Development version: `v0.0.1`
- Status: pre-alpha / feasibility investigation
- Active investigation: [`SPIKE-001`](spikes/SPIKE-001-environmental-wear-feasibility.md)
- Functional gameplay implementation: not yet validated

## Start here

- [`ROADMAP.md`](ROADMAP.md) — the single canonical project roadmap.
- [`ARCHITECTURE.md`](ARCHITECTURE.md) — current architecture/design space, including the transient-mark vs persistent-wear model.
- [`REQUIREMENTS.md`](REQUIREMENTS.md) — canonical behavior and acceptance criteria.
- [`TESTING.md`](TESTING.md) — repeatable test strategy.
- [`VALIDATION_HISTORY.md`](VALIDATION_HISTORY.md) — accumulated runtime/source-review evidence.
- [`DEPLOYMENT.md`](DEPLOYMENT.md) — package layout, test deployment, and rollback guidance.

## Engine/reference research

- [`ENGINE-RESEARCH-B42.md`](ENGINE-RESEARCH-B42.md) — Build 42 engine findings used to prioritize experiments.
- [`REFERENCE-IMPLEMENTATIONS.md`](REFERENCE-IMPLEMENTATIONS.md) — observations from comparison mods; no third-party source is incorporated into Happy Trails.

The B42.20.3 floor-blood/corpse-drag system is now explicitly treated as an architectural precedent for lightweight, bounded, sub-tile, age-faded environmental marks. The current design does **not** assume the blood renderer itself is a supported generic decal API.

## Formal spike investigations

- [`spikes/SPIKE-001-environmental-wear-feasibility.md`](spikes/SPIKE-001-environmental-wear-feasibility.md) — current feasibility investigation covering corpse-drag trail emission, custom transient decals, vehicle/wheel observation, persistent wear, vegetation, MP persistence, and performance.

See [`spikes/README.md`](spikes/README.md) for the spike convention.

## Architecture Decision Records

See [`adr/README.md`](adr/README.md). ADRs should record durable choices only after runtime evidence is sufficient to select among alternatives.

## Release / compliance

- [`RELEASE_CHECKLIST.md`](RELEASE_CHECKLIST.md) — release-readiness checks.
- [`PZ_MODDING_POLICY.md`](PZ_MODDING_POLICY.md) — project-specific Project Zomboid mod-policy guidance.
- [`../COMPLIANCE.md`](../COMPLIANCE.md) — repository compliance index.
- [`../ASSET_LICENSE.md`](../ASSET_LICENSE.md) — asset licensing/provenance policy.
- [`../THIRD_PARTY_NOTICES.md`](../THIRD_PARTY_NOTICES.md) — third-party notices/provenance.
- [`../LICENSE`](../LICENSE) and [`../NOTICE`](../NOTICE) — Apache-2.0 licensing/notices.

## Repository/package convention

Happy Trails follows the same structural convention as Enshrouded Sleep:

```text
Contents/mods/pz-happytrails/
```

contains the distributable Project Zomboid payload, while engineering documentation and project metadata remain at repository level.

## Documentation policy

The top-level README should remain a project landing page, not a second engineering notebook. There should be one canonical roadmap (`ROADMAP.md`), one canonical architecture document (`ARCHITECTURE.md`), and spike investigations should live under `docs/spikes/` rather than directly under `docs/`.
