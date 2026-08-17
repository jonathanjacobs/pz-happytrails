# Changelog

Human-readable history of notable Happy Trails changes. Git remains authoritative for exact diffs; this file summarizes behavior, architecture, diagnostics, documentation, and release milestones by development version.

## [Unreleased]

### Planned
- Complete SPIKE-001 to validate Build 42 vehicle-position/event access, terrain/object mutation, persistence, and multiplayer synchronization options.
- Identify a minimal representation for track wear that does not require unbounded per-square world state.
- Validate whether vegetation destruction can be implemented with vanilla object/tile APIs and appropriate vehicle collision information.
- Determine whether tire-track visuals can use existing tiles/overlays, custom sprites, or another supported rendering approach.
- Define recovery/decay behavior for lightly traveled routes after feasibility is established.
- Establish a repeatable dedicated-server test matrix before implementing the functional MVP.

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
