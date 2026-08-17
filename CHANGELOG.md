# Changelog

Human-readable history of notable Happy Trails changes. Git remains authoritative for exact diffs; this file summarizes behavior, architecture, diagnostics, documentation, and release milestones by development version.

## [Unreleased]

### Added
- Comparative source review of three supplied Build 42 reference mods: Footprints (`3457632586`), Vehicle Vegetation Destruction (`3690554902`), and More Damaged Objects (`3413150945`).
- `docs/REFERENCE-IMPLEMENTATIONS.md` documenting useful API clues, performance risks, and questions promoted into SPIKE-001 without incorporating third-party source code.
- `docs/ENGINE-RESEARCH-B42.md` documenting behavioral findings from the supplied decompiled Project Zomboid 42.20.2 (`ffe7a8a4b1`) source tree without storing or redistributing game source.
- Explicit SPIKE-001 comparison of server-side vehicle sampling, client passage reporting, and native/event-assisted hybrid approaches.
- Required performance instrumentation for samples/events, affected squares, object enumeration, state growth, queue high-water marks, mutations, and custom network traffic.
- Engine research into floor blood splats, existing-object overlays, native vehicle replication, scripted wheel geometry, transient foliage flattening, and vehicle/vegetation collision handling.

### Engine research findings
- Confirmed the native floor-blood subsystem is compact, chunk-local, bounded to 1000 entries per chunk, persisted, and explicitly networked, but the current renderer is hard-wired to 21 built-in blood types and blood-specific lifecycle/render controls. Direct Happy Trails reuse is therefore no longer the preferred visual candidate.
- Identified `IsoObject` overlay sprites as a stronger candidate: overlays are persisted with the existing object, participate in native overlay synchronization, invalidate cached chunk rendering when changed, and may allow custom track/wear sprites without adding another ordinary world object.
- Confirmed vehicle scripts expose actual wheel offsets and `BaseVehicle` can transform local wheel positions to world coordinates, allowing actual wheel-path sampling to be benchmarked against coarser approximations.
- Confirmed native vehicle physics networking already feeds detailed vehicle transforms/state to the server, with moving updates targeting roughly 150 ms periods in the examined build. This strengthens the server-side sampling candidate and may eliminate the need for a custom client movement-reporting protocol.
- Confirmed native vehicle collision already performs localized object/plant collision work.
- Confirmed `HitByCar` / `MinimumCarSpeedDmg` can drive native damage, damaged-sprite transitions, synchronized sprite updates, and eventual synchronized object removal for compatible objects.
- Confirmed the engine has transient vehicle-adjacent foliage rendering through `flattenGrassEtc`; this is useful classification/rendering evidence but not persistent trail state.
- Found no general vehicle-motion Lua event in the examined event registrations; a bounded periodic/tick sampler remains a candidate to benchmark.

### Changed
- Reframed performance as a first-class product requirement rather than a later optimization step.
- Clarified that Happy Trails remains requirements- and evidence-driven and has not selected a production architecture.
- Reworked `docs/DESIGN.md` into an open design-space document rather than a provisional module architecture.
- Split required MVP behavior from secondary features such as vegetation destruction, debris, snow/weather effects, exact wheel positioning, and recovery.
- Added the requirement that at least one plausible implementation alternative be benchmarked before the functional MVP architecture is accepted.
- Added graceful fidelity reduction as a requirement: performance settings must reduce actual processing/object/network work rather than only hide graphics.
- Added explicit lifecycle/bounding requirements for all runtime tables, queues, persistent state, and cleanup mechanisms.
- Downgraded direct native blood-splat reuse from a priority rendering candidate to a secondary smoke-test/reference-model path based on engine implementation constraints.
- Promoted existing-floor overlays to the first visual runtime experiment.
- Promoted native server vehicle replication plus wheel-position interpolation to the first movement-observation experiment.
- Promoted native `HitByCar` collision behavior to the first vegetation-damage experiment before any custom Lua area scanner.

### Planned
- Establish a repeatable Happy-Trails-disabled performance baseline before runtime experiments begin.
- Run a one-square existing-floor overlay smoke test covering custom sprite rendering, object count, MP visibility, late join, save/reload, restart, clearing, and occupied-overlay behavior.
- Run a dedicated-server wheel-geometry probe to measure Lua-visible vehicle transform cadence and verify gap-free interpolation between actual wheel positions at representative speeds.
- Run a controlled native vegetation-damage test using `HitByCar`, `MinimumCarSpeedDmg`, and optional damaged-sprite behavior.
- Build no repeated-wear production prototype until those primitives are validated.
- Compare at least one viable alternative representation before accepting an architecture.
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
