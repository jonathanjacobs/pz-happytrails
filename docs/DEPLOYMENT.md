# Happy Trails — Deployment Guide

Current status: **pre-alpha / not ready for a valued persistent world**.

No functional Happy Trails gameplay implementation has yet passed the full SPIKE-001 persistence and multiplayer acceptance criteria.

## Packaging layout

Happy Trails follows the same repository/package convention as Enshrouded Sleep:

```text
Contents/
└── mods/
    └── pz-happytrails/
        ├── mod.info
        ├── 42/
        │   ├── mod.info
        │   └── media/
        └── common/
            └── media/
```

The `Contents/` tree is the distributable Project Zomboid/Workshop payload. Repository documentation, licenses, validation evidence, and development files remain outside that tree.

## Pre-alpha testing

Until a release candidate exists:

- use a disposable/test-server world;
- back up saves before persistence experiments;
- use identical source snapshots on server and clients;
- keep prototype diagnostics bounded and rate-limited;
- do not enable destructive vegetation experiments outside controlled test areas;
- preserve logs and the exact commit/version used for each test.

## Rollback principle

Happy Trails must be designed so disabling/removing the mod stops future wear processing without requiring a separate external database service.

Persistent world mutations already written by a test build may not be automatically reversible unless the specific prototype implements restoration. During pre-alpha, the reliable rollback remains restoring a pre-test world backup.

## Public deployment gate

Do not treat the mod as public-server ready until the roadmap/SPIKE criteria confirm:

- bounded transient mark lifecycle;
- stable persistent wear state;
- save/restart behavior;
- multiplayer and late-join convergence;
- no unacceptable object/state growth;
- tested disable/rollback behavior;
- no known destructive interaction with ordinary terrain or erosion.
