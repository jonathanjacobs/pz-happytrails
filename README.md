# Happy Trails

**Dynamic tracks and terrain wear for Project Zomboid Build 42.**

Status: **Pre-alpha / feasibility investigation**  
Current development version: **v0.0.1**  
Current research baseline: **Project Zomboid 42.20.x**

Happy Trails is a Project Zomboid Build 42 mod project focused on making repeated survivor and vehicle travel leave a persistent, visible mark on the world.

Project Zomboid already models nature reclaiming civilization through erosion. Happy Trails explores the inverse: repeated human activity gradually reshapes the landscape.

## What it is intended to do

Planned behavior includes:

- fresh vehicle tracks on suitable natural terrain;
- progressive wear from repeated traffic rather than a one-pass binary mutation;
- flattened grass, exposed soil, wheel ruts, and established informal trails;
- temporary mud/snow/material tracks that can fade independently of persistent wear;
- vegetation damage from vehicle contact where a safe native path exists;
- save/restart and dedicated-server persistence;
- multiplayer synchronization and late-join convergence;
- slow recovery/regrowth on abandoned routes;
- bounded transient/persistent state so travel history cannot grow without limit.

No functional gameplay implementation has yet passed SPIKE-001. The repository currently contains the package skeleton, requirements, architecture research, testing plan, and formal feasibility investigation.

## Current design insight: transient marks vs persistent wear

Build 42.20.3's vanilla floor-blood system provides a strong architectural precedent. PZ stores floor blood as compact sub-tile records in a bounded per-chunk queue, timestamps them with world age, serializes them with the chunk, and calculates visual aging lazily from elapsed world time.

Happy Trails is therefore investigating a two-layer model:

```text
TRANSIENT MARKS
fresh tire impressions / mud / snow / flattened vegetation
-> high spatial fidelity
-> bounded lifecycle
-> age-based fading

PERSISTENT TERRAIN WEAR
worn grass / exposed dirt / established ruts and trails
-> sparse server-authoritative state
-> survives reload/restart
-> slow abandonment recovery
```

The blood system is a design reference, not a plan to hijack vanilla blood sprites or tables.

See [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) and [`docs/spikes/SPIKE-001-environmental-wear-feasibility.md`](docs/spikes/SPIKE-001-environmental-wear-feasibility.md).

## Current development phase

The active milestone is SPIKE-001. It must determine whether Build 42 exposes a clean, bounded path for:

- server-observed vehicle/wheel geometry;
- gap-free path interpolation at driving speed;
- lightweight custom sub-tile ground marks;
- persistent non-destructive wear presentation/state;
- terrain classification;
- native vegetation damage;
- save/restart, multiplayer, and late-join behavior;
- acceptable CPU/state/network scaling.

The full project roadmap is maintained only in [`docs/ROADMAP.md`](docs/ROADMAP.md).

## Stable project identity

Repository and stable Project Zomboid Mod ID:

```text
pz-happytrails
```

## Repository/package layout

Happy Trails now follows the same repository structure used by Enshrouded Sleep and intended as the standard template for these Project Zomboid mod projects:

```text
pz-happytrails/
├── .github/
├── Contents/
│   └── mods/
│       └── pz-happytrails/
│           ├── mod.info
│           ├── 42/
│           │   ├── mod.info
│           │   └── media/
│           └── common/
│               └── media/
├── docs/
│   ├── adr/
│   ├── spikes/
│   ├── ARCHITECTURE.md
│   ├── DEPLOYMENT.md
│   ├── PZ_MODDING_POLICY.md
│   ├── README.md
│   ├── RELEASE_CHECKLIST.md
│   ├── REQUIREMENTS.md
│   ├── ROADMAP.md
│   ├── TESTING.md
│   └── VALIDATION_HISTORY.md
├── ASSET_LICENSE.md
├── CHANGELOG.md
├── COMPLIANCE.md
├── LICENSE
├── NOTICE
├── README.md
├── THIRD_PARTY_NOTICES.md
└── VERSION
```

`Contents/` is the distributable Project Zomboid/Workshop payload. Project documentation, research notes, licensing, and development material remain outside it.

## Versioning

```text
0.0.x  experimental spikes, diagnostics, and pre-alpha implementation
0.1.x  functional alpha/beta stabilization
1.x    stable public releases after behavior and compatibility are mature
```

`VERSION` and package `mod.info` files should be updated together whenever the project version changes.

## Development principles

Happy Trails should prefer documented or experimentally validated Project Zomboid APIs over invasive workarounds. Persistent world changes should be server-authoritative. High-frequency work must be bounded and measurable. Whole-world scans and unbounded historical records are out of scope.

Experiments belong under [`docs/spikes/`](docs/spikes/). Durable architecture choices belong under [`docs/adr/`](docs/adr/). Detailed evidence belongs in [`docs/VALIDATION_HISTORY.md`](docs/VALIDATION_HISTORY.md).

## Documentation

- [`docs/README.md`](docs/README.md) — documentation index
- [`docs/ROADMAP.md`](docs/ROADMAP.md) — single canonical roadmap
- [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) — architecture/design space
- [`docs/REQUIREMENTS.md`](docs/REQUIREMENTS.md) — canonical requirements and acceptance criteria
- [`docs/TESTING.md`](docs/TESTING.md) — test strategy
- [`docs/VALIDATION_HISTORY.md`](docs/VALIDATION_HISTORY.md) — accumulated evidence
- [`docs/spikes/`](docs/spikes/) — formal investigations
- [`docs/adr/`](docs/adr/) — architecture decision records
- [`docs/DEPLOYMENT.md`](docs/DEPLOYMENT.md) — packaging/test deployment/rollback guidance
- [`CHANGELOG.md`](CHANGELOG.md) — human-readable change history

## License

Copyright 2026 Jonathan Jacobs.

Licensed under the **Apache License, Version 2.0**. See [`LICENSE`](LICENSE) and [`NOTICE`](NOTICE).

Project Zomboid is developed by The Indie Stone. Happy Trails is an independent community mod and is not affiliated with or endorsed by The Indie Stone.
