# Changelog

Human-readable history of notable Happy Trails changes. Git remains authoritative for exact diffs; this file summarizes behavior, architecture, diagnostics, documentation, and release milestones by development version.

## [Unreleased]

### Repository/template alignment
- Reorganized the distributable mod payload under `Contents/mods/pz-happytrails/`, matching the repository/package convention now used by Enshrouded Sleep.
- Standardized documentation naming and placement around `docs/ARCHITECTURE.md`, `docs/ROADMAP.md`, `docs/DEPLOYMENT.md`, `docs/VALIDATION_HISTORY.md`, `docs/spikes/`, and `docs/adr/`.
- Moved SPIKE-001 into `docs/spikes/SPIKE-001-environmental-wear-feasibility.md` and added `docs/spikes/README.md` as the spike convention/index.
- Established `docs/ROADMAP.md` as the single canonical roadmap; the top-level README now links to it instead of maintaining a second roadmap.
- Updated the root/documentation indexes so `Contents/` is treated as the distributable PZ/Workshop payload and engineering material remains outside it.

### Vanilla corpse-drag / floor-blood findings
- Revisited B42.20.3 `IsoFloorBloodSplat`/`IsoChunk` after observing corpse dragging create a fading blood trail in-game.
- Confirmed floor blood is represented as compact sub-tile records in a dedicated bounded `IsoChunk` queue rather than unlimited ordinary world objects.
- Confirmed the queue is capped at 1000 splats per chunk, with overflow moved to a short fade list.
- Confirmed each splat stores sub-tile x/y/z, type/index data, and creation `worldAge`, and is serialized with the chunk.
- Confirmed rendering lazily calculates visual age from `WorldAgeHours - creation worldAge` and applies a 72-world-hour blood aging/fade window.
- Promoted the architectural lesson—not direct blood-system reuse—to a first-class Happy Trails design precedent.

### Architecture changes
- Adopted a two-layer conceptual model:
  - **transient marks** for fresh high-resolution tire/mud/snow/flattening effects with bounded lifecycle and lazy age-based fading;
  - **persistent terrain wear** for sparse server-authoritative accumulated disturbance such as worn grass, exposed dirt, established ruts, and trails.
- Added explicit separation between persistent terrain wear and temporary carried-material contamination on hard surfaces.
- Added lazy elapsed-world-age evaluation as the preferred recovery/expiration principle instead of full-world timer scans.
- Added hysteresis/slow abandonment recovery as the intended behavior for established trails.
- Elevated sub-tile custom ground-decal feasibility to a primary rendering investigation while retaining existing-floor overlays as a strong candidate for persistent wear presentation.

### SPIKE-001 changes
- Added a dedicated experiment to trace the vanilla corpse-drag blood-emission cadence/spacing logic.
- Added a custom lightweight sub-tile decal smoke test before committing to ordinary `IsoObject` marks.
- Retained direct native blood-splat reuse only as an API/rejection-confirmation experiment; Happy Trails must not hijack vanilla blood tables/assets as a production strategy.
- Updated success criteria to require convincing transient marks, bounded mark lifecycle/state, persistent wear, MP/save persistence, and measured scaling.

### Previous research retained
- Comparative source review of three supplied Build 42 reference mods: Footprints (`3457632586`), Vehicle Vegetation Destruction (`3690554902`), and More Damaged Objects (`3413150945`).
- `docs/REFERENCE-IMPLEMENTATIONS.md` records useful API clues and performance risks without incorporating third-party source code.
- `docs/ENGINE-RESEARCH-B42.md` records behavioral findings from supplied/decompiled PZ source without storing or redistributing game source.
- Existing-floor overlays remain a candidate for persistent wear state.
- Native server vehicle replication plus wheel-position interpolation remains the leading vehicle-observation candidate.
- Native `HitByCar`/vehicle collision behavior remains the first vegetation-damage path to test before any broad Lua spatial scanner.
- Performance remains a first-class product requirement; at least one viable alternative must be benchmarked before accepting a production architecture.

### Planned next experiments
- Trace corpse-drag blood deposition cadence/spacing.
- Test whether Lua can create an arbitrary lightweight custom ground decal with sub-tile position, orientation/directional control, alpha, bounded lifetime, and acceptable MP/persistence behavior.
- Run the existing-floor overlay smoke test for established wear.
- Run the dedicated-server wheel-geometry/interpolation probe.
- Run controlled native vegetation-damage tests.
- Build no repeated-wear production prototype until those primitives are validated.

## [0.0.1] - 2026-08-16

Initial repository and project-development scaffold.

### Added
- Standardized Project Zomboid Build 42 repository scaffold.
- Stable Mod ID `pz-happytrails`.
- `VERSION` marker initialized to `0.0.1`.
- Canonical requirements, design/testing documentation, spike investigation, and ADR convention.
- Build 42/common media placeholders.
- Apache License 2.0 licensing with NOTICE attribution.
- Project-specific `.gitignore` covering local logs, archives, editor files, and helper-tool artifacts.

### Development status
- No functional gameplay implementation is claimed in v0.0.1.
- The project remains in pre-alpha feasibility analysis until SPIKE-001 produces sufficient evidence for the implementation architecture.
