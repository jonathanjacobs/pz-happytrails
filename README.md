# Happy Trails for Project Zomboid B42

Happy Trails is a Project Zomboid Build 42 mod project focused on making repeated survivor and vehicle travel leave a persistent, visible mark on the world.

The intended system allows commonly traveled routes to evolve from untouched terrain into flattened vegetation, tire marks, ruts, worn paths, and eventually recognizable informal roads. Vehicle movement may also crush small vegetation, break saplings, and leave appropriate debris behind.

> **Development status:** pre-alpha / feasibility stage. The repository currently contains project structure, requirements, design notes, and test planning. No functional Happy Trails gameplay implementation has been validated yet.

## Core concept

Project Zomboid already models the world reclaiming civilization through erosion. Happy Trails explores the inverse: survivors gradually reshape the landscape through repeated use.

Planned behaviors include:

- vehicle tracks on suitable natural terrain;
- progressive wear from repeated traffic rather than a single binary terrain change;
- flattened grass and vegetation along frequently traveled routes;
- dirt, mud, snow, and grass-specific visual states where technically feasible;
- destruction of small bushes, saplings, and similar vegetation by vehicles;
- broken branches or other debris where appropriate;
- persistence across multiplayer sessions;
- gradual recovery of lightly used routes where practical;
- server-authoritative state changes for multiplayer consistency;
- performance safeguards so traffic history does not become an unbounded world-state cost.

The exact mechanic is intentionally not fixed until the relevant Build 42 Lua hooks, tile/object mutation APIs, persistence behavior, and multiplayer synchronization paths have been validated.

## Current development phase

The first development milestone is a technical feasibility spike. It must establish whether Build 42 exposes enough information and mutation capability to implement the mechanic cleanly without brittle engine hacks.

See:

- [`docs/REQUIREMENTS.md`](docs/REQUIREMENTS.md) — canonical requirements and MVP acceptance criteria.
- [`docs/DESIGN.md`](docs/DESIGN.md) — working architecture and data-model concepts.
- [`docs/SPIKE-001.md`](docs/SPIKE-001.md) — first feasibility investigation and go/no-go criteria.
- [`docs/TESTING.md`](docs/TESTING.md) — repeatable test strategy and evidence requirements.
- [`CHANGELOG.md`](CHANGELOG.md) — human-readable development history.

## Stable project identity

Repository:

```text
pz-happytrails
```

Stable Project Zomboid Mod ID:

```text
pz-happytrails
```

Preferred local mod folder:

```text
pz-happytrails/
```

The repo and Mod ID intentionally match so a GitHub source archive can be renamed predictably for local or dedicated-server testing.

## Repository layout

```text
pz-happytrails/
├── .github/
├── 42/
│   ├── mod.info
│   └── media/
├── common/
│   └── media/
├── docs/
│   ├── adr/
│   ├── DESIGN.md
│   ├── README.md
│   ├── REQUIREMENTS.md
│   ├── SPIKE-001.md
│   └── TESTING.md
├── .gitignore
├── CHANGELOG.md
├── LICENSE
├── NOTICE
├── README.md
├── VERSION
└── mod.info
```

This structure follows the same general conventions used across the related Project Zomboid mod projects maintained in this GitHub account: root-level project metadata, explicit Build 42 content, a human-readable changelog, semantic development versions, canonical documentation under `docs/`, and Apache 2.0 licensing with a NOTICE file.

## Versioning

Development uses semantic-style versions:

```text
0.0.x  experimental spikes, diagnostics, and pre-alpha implementation
0.1.x  first functional MVP and stabilization work
1.x    stable public releases after the behavior and compatibility contract is mature
```

`VERSION`, root `mod.info`, and `42/mod.info` should be updated together whenever the project version changes.

## Development principles

Happy Trails should prefer documented or experimentally validated Project Zomboid APIs over invasive workarounds. Multiplayer behavior should be server-authoritative where persistent world state is involved. Every non-trivial Lua function should be documented sufficiently for another developer to understand its inputs, outputs, side effects, failure modes, and important control-flow decisions.

Experiments should be captured as spike documents rather than allowed to become undocumented prototype code. Significant architectural decisions should be recorded under [`docs/adr/`](docs/adr/).

## Compatibility target

Initial development targets Project Zomboid Build 42.20 or later. Exact minimum compatibility may change as the feasibility work identifies required APIs.

The project should avoid unnecessary assumptions about specific vehicle, map, vegetation, or graphical mods. Compatibility with major content mods will be evaluated after the vanilla mechanic is validated.

## License

Copyright 2026 Jonathan Jacobs.

Licensed under the **Apache License, Version 2.0**. See [`LICENSE`](LICENSE) and [`NOTICE`](NOTICE).

Project Zomboid is developed by The Indie Stone. This project is an independent community mod and is not affiliated with or endorsed by The Indie Stone.
