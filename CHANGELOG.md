# Changelog

Human-readable history of notable Happy Trails changes. Git remains authoritative for exact diffs; this file summarizes behavior, architecture, diagnostics, documentation, and release milestones by development version.

## [Unreleased]

### Added
- Comparative source review of three supplied Build 42 reference mods: Footprints (`3457632586`), Vehicle Vegetation Destruction (`3690554902`), and More Damaged Objects (`3413150945`).
- `docs/REFERENCE-IMPLEMENTATIONS.md` documenting useful API clues, performance risks, and questions promoted into SPIKE-001 without incorporating third-party source code.
- Explicit SPIKE-001 comparison of server-side vehicle sampling, client passage reporting, and native/event-assisted hybrid approaches.
- Required performance instrumentation for samples/events, affected squares, object enumeration, state growth, queue high-water marks, mutations, and custom network traffic.
- Investigation of Project Zomboid's native floor-blood/decal subsystem as a potential lightweight rendering/persistence/networking path for tire tracks.
- SPIKE questions covering `IsoChunk.FloorBloodSplats`, `IsoFloorBloodSplat`, `BloodSplatter`/`RemoveBlood` packets, `BloodSplatLifespanDays`, custom sprite feasibility, queue/eviction behavior, and performance relative to ordinary `IsoObject` marks.
- Investigation of `IsoGridSquare.bFlattenGrassEtc` as a potentially relevant native grass/ground presentation mechanism.

### Changed
- Reframed performance as a first-class product requirement rather than a later optimization step.
- Clarified that Happy Trails remains requirements- and evidence-driven and has not selected a production architecture.
- Reworked `docs/DESIGN.md` into an open design-space document rather than a provisional module architecture.
- Split required MVP behavior from secondary features such as vegetation destruction, debris, snow/weather effects, exact wheel positioning, and recovery.
- Added the requirement that at least one plausible implementation alternative be benchmarked before the functional MVP architecture is accepted.
- Added graceful fidelity reduction as a requirement: performance settings must reduce actual processing/object/network work rather than only hide graphics.
- Added explicit lifecycle/bounding requirements for all runtime tables, queues, persistent state, and cleanup mechanisms.
- Added investigation of native `HitByCar` / `MinimumCarSpeedDmg` sprite properties as a potential way to offload vegetation collision work to the engine.
- Expanded the visual design space to distinguish ordinary `IsoObject` marks, floor/tile replacement, the native floor-splat path, generic overlays, and hybrid state-to-visual materialization.

### Planned
- Establish a repeatable baseline with Happy Trails disabled before runtime experiments begin.
- Run SPIKE-001 vehicle-observation experiments and measure server-side sampling versus client/hybrid alternatives.
- Run a narrow native-decal experiment before building a custom track renderer: determine Lua access, custom sprite feasibility, persistence, MP replication, cap/eviction semantics, and interaction with vanilla gore settings.
- Validate the native vehicle-hit property path for representative bushes/saplings before designing a custom vegetation scanner.
- Compare track visual representations by persistence, MP convergence, object count, late-join cost, reversibility, and rendering cost.
- Identify a minimal representation for progressive wear that does not require unbounded per-pass history or one persistent object per tire impression.
- Add explicit numeric performance budgets to `docs/REQUIREMENTS.md` only after baseline and candidate measurements exist.

## [0.0.1] - 2026-08-16

Initial repository and project-development scaffold.

### Added
- Standardized Project Zomboid Build 42 repository layout based on the related mod-development projects.
- Root and Build 42 `mod.info` metadata using stable Mod ID `pz-happytrails`.
- `VERSION` marker initialized to `0.0.1`.
- Canonical requirements, design, testing, and documentation index files under `docs/`.
- `docs/SPIKE-001.md` for the initial feasibility investigation.
- `docs/adr/README.md` for future architecture decision records.
- Build 42 and common media placeholders required to preserve the intended source layout in Git.
- Apache License 2.0 licensing with project NOTICE attribution.
- Project-specific `.gitignore` covering local logs, archives, editor files, and helper-tool artifacts.

### Development status
- No functional gameplay implementation is claimed in v0.0.1.
- The project remains in pre-alpha feasibility analysis until SPIKE-001 produces sufficient evidence for the implementation architecture.
