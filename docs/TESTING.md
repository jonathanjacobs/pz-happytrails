# Happy Trails Testing

## Purpose

This document defines how Happy Trails experiments and releases should be validated. Pre-alpha testing should privilege reproducible evidence over subjective impressions.

The active investigation is [`spikes/SPIKE-001-environmental-wear-feasibility.md`](spikes/SPIKE-001-environmental-wear-feasibility.md).

## Test environments

Use, as appropriate:

- local single-player Build 42.20+;
- local hosted multiplayer;
- dedicated server Build 42.20+;
- two or more clients for synchronization tests.

Record the exact Project Zomboid build, Happy Trails commit/version, and relevant server settings for every material result.

## Evidence standard

For any claim about engine behavior, capture:

- exact test version/commit;
- server settings relevant to the test;
- map location/test surface;
- exact player/vehicle actions;
- relevant client/server logs;
- screenshots/video where visual behavior matters;
- object/state counts when performance matters;
- save/restart and late-join behavior where persistence matters.

A visual impression alone is not sufficient evidence for synchronization, persistence, or performance.

## SPIKE-001 experiment order

### 1. Corpse-drag blood-trail trace

Use vanilla corpse dragging as a reference behavior and determine, through source tracing plus controlled observation where possible:

- how frequently/at what distance new floor-blood splats are emitted;
- whether deposition is distance-, tick-, animation-, or event-driven;
- whether movement gaps are interpolated;
- approximate spatial mark density required for a continuous-looking trail.

Do not modify the vanilla blood system.

### 2. Custom transient-decal smoke test

Using one conspicuous temporary Happy Trails test asset, determine whether Lua can create a lightweight ground mark with:

- sub-tile x/y placement;
- direction/rotation or directional variants;
- alpha/tint control;
- bounded lifetime;
- safe cleanup;
- multiplayer visibility;
- save/reload behavior if the candidate is intended to persist through unload/reload;
- no full ordinary `IsoObject` per impression, if the candidate claims to avoid that cost.

Record object counts before/after.

### 3. Persistent floor-overlay smoke test

Use one controlled natural square. Record:

- original floor sprite/overlay;
- resulting overlay;
- object count before/after;
- visibility to a second client;
- late-join result;
- save/reload result;
- dedicated-server restart result;
- clear/restore behavior;
- behavior when the overlay slot is already occupied.

### 4. Server wheel-geometry probe

No terrain mutation.

Drive a vehicle from a remote client while the server records, at a bounded diagnostic cadence:

- vehicle ID;
- position/orientation changes;
- speed;
- wheel count/offsets;
- transformed wheel positions;
- elapsed real time and distance between meaningful server-visible changes;
- interpolated/rasterized path segments.

Verify that representative left/right wheel paths can remain gap-free at increasing speeds.

### 5. Terrain classification

Create a controlled route containing natural, paved, indoor, water/invalid, and unknown/modded surfaces.

Verify:

- eligible natural surfaces are recognized;
- hard surfaces do not accumulate persistent terrain wear;
- hard surfaces remain eligible for temporary carried-material marks where that feature is enabled;
- water/interiors/unknown destructive cases fail closed.

### 6. Repeated-pass progression

Once a minimal wear prototype exists, drive the same route repeatedly.

Verify:

- a single pass creates only transient/light disturbance;
- repeated traffic increases persistent wear monotonically;
- persistent state changes only at intended thresholds;
- repeated sampling of one physical passage is coalesced;
- established wear remains after transient marks fade;
- direction changes do not corrupt state.

### 7. Persistence and multiplayer convergence

After creating established wear:

1. save/exit and reload;
2. restart the dedicated server;
3. verify the authoritative state remains;
4. have Client A add additional wear;
5. reconnect a previously absent Client B;
6. verify Client B receives the current state without manual repair/reload.

### 8. Transient aging / bounded lifecycle

For each transient-mark candidate:

- record creation world age;
- verify appearance/expiry is derived lazily from elapsed time rather than requiring a whole-world timer scan;
- verify the per-chunk/region record count has a hard or effectively hard upper bound;
- stress repeated driving in one chunk and verify old marks are expired/coalesced rather than growing forever.

### 9. Vegetation damage

Test native vehicle/object collision behavior first. For every supported vegetation class:

- identify the exact object/sprite/property set;
- test below/above damage thresholds;
- verify flatten/damage/removal behavior;
- verify vehicle response;
- verify mature/protected objects remain safe;
- verify synchronized removal/damaged sprite state in MP;
- verify no duplicate debris.

### 10. Recovery

Once recovery exists:

- establish known persistent wear;
- advance world time without traffic;
- revisit/reload the area;
- verify lazy recovery/hysteresis;
- verify deep/established wear recovers substantially more slowly than light disturbance;
- drive the abandoned route again and confirm previous disturbance can be re-established naturally.

### 11. Performance

Record a Happy-Trails-disabled baseline using the same route/vehicles/clients.

Measure at least:

- vehicles examined and rejected;
- meaningful vehicle samples;
- wheel transforms/interpolated segments;
- transient marks emitted/expired/high-water count;
- candidate squares generated/coalesced;
- terrain classifications;
- persistent state reads/writes/high-water count;
- visual mutations;
- custom network traffic if any;
- cleanup/recovery operations;
- server/client stutter or tick impact.

Performance controls must reduce actual work/state, not merely hide graphics.

## Regression checklist

Before promoting a functional build:

- no Lua errors in ordinary driving;
- no mark creation while stationary;
- paved roads are not permanently deformed;
- transient marks expire/bound correctly;
- established wear persists after transient marks disappear;
- MP clients converge and late join works;
- save/restart persistence works;
- no uncontrolled ordinary-world object growth;
- no mature-tree deletion from normal passage;
- disabling the mod stops further processing;
- `VERSION`, `Contents/mods/pz-happytrails/mod.info`, and `Contents/mods/pz-happytrails/42/mod.info` agree;
- README, requirements, roadmap, validation history, testing notes, and changelog reflect the tested behavior.

## Debug logging conventions

```text
[HappyTrails][SERVER]
[HappyTrails][CLIENT]
[HappyTrailsSpike001]
```

High-frequency diagnostics must be opt-in, rate-limited, and bounded.
