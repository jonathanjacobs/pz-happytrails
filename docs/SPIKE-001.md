# SPIKE-001 — Build 42 Environmental Wear Feasibility

## Status

In progress — comparative mod review and Build 42.20.2 engine-source review complete; runtime experiments not yet started.

## Question

Can Project Zomboid Build 42 support a performant, persistent, multiplayer-safe Happy Trails mechanic using Lua-accessible APIs and ordinary mod assets, without engine modification or brittle hacks?

A second question is equally important:

> Which implementation strategy provides the best client/server performance envelope without preventing higher-fidelity features later?

## Development rule

SPIKE-001 must narrow the design space with evidence. It must not turn a promising Java implementation detail into a production architecture before the relevant behavior is verified from Lua in single-player and dedicated-server multiplayer.

## Evidence reviewed so far

### Third-party comparison mods

Three supplied Build 42 mod source packages were reviewed as research inputs:

- Workshop package `3457632586` — Footprints / zombie and bandit footprints;
- Workshop package `3690554902` — Vehicle Vegetation Destruction (VVD);
- Workshop package `3413150945` — More Damaged Objects.

Detailed observations are recorded in [`REFERENCE-IMPLEMENTATIONS.md`](REFERENCE-IMPLEMENTATIONS.md). No source from these mods has been incorporated into Happy Trails.

### Decompiled installed game build

The user supplied a locally decompiled `zombie/` source tree from the exact installed game build used for development testing. The examined code identifies itself as:

```text
Project Zomboid 42.20.2
Git revision ffe7a8a4b1
```

The decompiled source is research material only and is not stored in this repository. Behavioral findings are summarized in [`ENGINE-RESEARCH-B42.md`](ENGINE-RESEARCH-B42.md).

## Engine findings that change experimental priority

### 1. Native blood splats are optimized but blood-specific

The engine stores floor blood in a compact bounded per-chunk queue and persists/networks it efficiently. However, the examined renderer uses a fixed 21-type blood texture table, validates splat types against that table, applies blood-specific visual aging/tinting, shares the native blood queue, and is governed by blood-decal/lifespan settings.

Therefore direct reuse is no longer the leading Happy Trails rendering candidate. A small Lua smoke test is still acceptable, but we should not build around modifying/hijacking the blood system unless runtime evidence reveals a clean extension point that was not apparent in the Java implementation.

### 2. Existing-object overlay sprites are a stronger candidate

`IsoObject` supports an overlay sprite that is serialized with the object, invalidates cached chunk rendering when changed, and has a native multiplayer overlay-update path. A floor object may therefore be able to display a custom Happy Trails track/wear sprite without replacing the floor and without adding a second ordinary `IsoObject`.

This becomes the first visual runtime experiment.

### 3. Actual wheel geometry exists

Vehicle script wheel definitions expose wheel count and local wheel offsets, and `BaseVehicle` can transform local positions to world positions. Happy Trails can therefore test actual left/right wheel paths instead of assuming vehicle-center or generic-width geometry.

### 4. Detailed vehicle transforms already reach the server

Native vehicle networking writes replicated position/orientation/physical state into the server-side `BaseVehicle`. Moving vehicles target roughly 150 ms physics-network intervals in the examined build. A server-side Happy Trails sampler may therefore need no custom client movement-reporting protocol.

Because 150 ms can span multiple tiles at speed, the candidate implementation must interpolate/rasterize between previous and current wheel positions rather than processing only the current square.

### 5. Native collision already performs vegetation-local work

`BaseVehicle` already performs localized vehicle/object collision checks and includes native plant-collision handling. Generic object collision also interprets `HitByCar` and `MinimumCarSpeedDmg`; compatible objects can use native damage, damaged-sprite transitions, removal, and normal MP synchronization.

This substantially weakens the case for a continuous custom Lua vegetation-area scanner.

### 6. No general vehicle-motion Lua event was found

No generic `OnVehicleUpdate`-style Lua event was found in the examined event registrations. A bounded periodic/tick sampler remains a likely candidate, but it should be judged by measured work rather than rejected merely because it uses a tick callback.

## Scope

SPIKE-001 should answer the following questions before architecture is selected.

### A. Lua visibility of server-side vehicle state

Determine in dedicated-server Lua whether we can reliably access:

- active vehicles;
- current x/y/z and orientation;
- current speed;
- vehicle script;
- wheel count;
- wheel offsets;
- transformed wheel world positions;
- occupancy/driver state;
- enough state to reject stationary/irrelevant vehicles cheaply.

Measure the actual interval between meaningful server-visible vehicle transform changes while a remote client drives.

### B. Vehicle-path sampling strategies

Compare at least:

#### Strategy A — bounded server-side sampling

```text
native PZ vehicle replication
-> server samples only relevant moving vehicles
-> transform wheel positions
-> interpolate previous/current wheel segments
-> process only newly crossed candidate squares
```

Questions:

- How much work occurs per moving vehicle per second?
- Can unchanged transforms be rejected before terrain work?
- Can distance thresholds reduce processing further?
- Does segment interpolation eliminate high-speed gaps?

#### Strategy B — client passage reporting + server validation

Retain this as an alternative only if server-visible transforms are too stale or expensive. If tested, compare network and validation cost against Strategy A.

#### Strategy C — native/event-assisted hybrid

Use engine-native collision and other state changes wherever possible, limiting Happy Trails sampling to track-wear information that PZ does not already compute.

No strategy is selected until measured.

### C. Terrain classification

Identify a cheap, centralized way to distinguish eligible natural surfaces from roads, interiors, water, and unknown/modded surfaces.

Compare floor sprite properties, tile properties, erosion/natural-floor metadata, bounded allow/deny tables, or combinations. Unknown terrain must fail closed.

### D. Visual representation

Compare the following without coupling authoritative wear semantics to any one renderer.

#### D1 — existing floor-object overlay

Priority experiment. Determine whether a custom track/wear overlay can be applied to a natural floor object and:

- render correctly;
- preserve the underlying terrain;
- avoid creating another ordinary world object;
- synchronize to a second client;
- persist through save/reload and dedicated-server restart;
- appear for a late joiner;
- be cleared/restored safely;
- coexist with or safely decline to modify floors that already use an overlay.

#### D2 — additional `IsoObject`

Retain as a comparison because reference mods prove it works. Measure object-count, mutation, persistence, cleanup, and late-join costs rather than assuming it is acceptable.

#### D3 — floor/tile replacement

Test only if overlays prove inadequate. Verify reversibility and compatibility with erosion, maps, and terrain identity.

#### D4 — native blood-splat path

Now a secondary/rejection-confirmation experiment. The Java implementation is blood-specific. Test only enough to confirm Lua accessibility and whether an unexpectedly clean extension point exists.

#### D5 — other/custom materialization paths

Remain open if A–D fail or benchmarks reveal a better approach.

### E. Vegetation mutation

Test native collision/property handling before any custom area scanner.

For representative controlled vegetation objects:

1. determine whether `HitByCar` can be applied safely;
2. test `MinimumCarSpeedDmg`;
3. test optional damaged-sprite transition;
4. confirm native removal when appropriate;
5. verify vehicle response;
6. verify dedicated-server authority and MP synchronization;
7. confirm no custom broad square/object scan is needed.

Only if this path is insufficient should SPIKE-001 benchmark an explicit local scanner.

### F. Wear-state persistence

The visual layer and authoritative wear state may be separate concerns.

Compare bounded state representations such as:

- state encoded directly by a durable visual/world mutation;
- square/object `modData`;
- sparse server-side records for modified squares;
- chunk-level aggregation if justified by measurements.

State growth should follow currently meaningful modified terrain, not every historical pass.

## Runtime experiment order

### Experiment 1 — floor-overlay smoke test

Use one controlled natural square and one conspicuous temporary custom sprite. No vehicle logic.

Record:

- original floor sprite and overlay;
- resulting overlay;
- object count before/after;
- client visibility;
- late-join result;
- save/reload result;
- server restart result;
- clear/restore behavior;
- behavior when an overlay is already occupied.

### Experiment 2 — server wheel-geometry probe

No terrain mutation.

Drive a vehicle from another client while the server records, at a rate-limited diagnostic cadence:

- vehicle ID;
- transform changes;
- speed;
- wheel count;
- wheel offsets;
- transformed wheel positions;
- elapsed real time between server-visible changes;
- rasterized/interpolated squares between consecutive wheel samples.

The critical result is whether the server alone can produce gap-free wheel paths with bounded work.

### Experiment 3 — native vegetation damage probe

Modify only a deliberately controlled test vegetation sprite/object. Verify below/above-threshold collision, damage/sprite/removal behavior, and MP synchronization.

### Experiment 4 — minimal wear-state prototype

Only after Experiments 1–3 establish the underlying primitives should we add repeated-pass wear and compare at least two viable representations.

## Performance methodology

Record a Happy-Trails-disabled baseline using the same route, vehicles, clients, and server configuration.

Prototype counters should include at least:

- vehicles examined;
- vehicles rejected as stationary/irrelevant;
- meaningful vehicle samples;
- wheel transforms;
- interpolated segments;
- candidate squares generated;
- duplicate/coalesced candidate squares;
- terrain classifications;
- square-object enumerations;
- wear-state reads/writes;
- visual mutations;
- persistent state count/high-water mark;
- custom network messages/bytes, if any;
- cleanup/recovery operations, if any.

Stress dimensions should include speed, route length, repeated-route versus exploration, dense vegetation, and multiple simultaneous moving vehicles.

## Prototype constraints

All SPIKE code must be diagnostic, bounded, and removable. It should:

- operate on controlled areas initially;
- use conspicuous `[HappyTrailsSpike001]` logging;
- include a kill switch;
- avoid broad world scans;
- bound all queues/tables;
- avoid persistent debris;
- rate-limit logging;
- expose work counters;
- fail closed on unknown terrain/objects.

## Success criteria

SPIKE-001 succeeds when evidence demonstrates a credible path that can:

1. observe a moving vehicle from the authoritative server with sufficient fidelity;
2. derive a continuous affected path without high-speed gaps;
3. classify at least one eligible natural and one ineligible paved surface;
4. create at least one persistent non-destructive visual wear state;
5. synchronize that state to existing and late-joining clients;
6. survive save/reload and dedicated-server restart;
7. show bounded CPU/work/state/network scaling in representative tests;
8. compare the chosen visual/movement candidate against at least one alternative;
9. validate a low-cost vegetation-damage path or explicitly defer it;
10. identify meaningful performance/fidelity controls.

## Go / no-go

### GO

Proceed to a functional v0.0.x prototype when a measured candidate provides persistent/synchronized wear with bounded scaling and acceptable responsiveness.

### CONDITIONAL GO

Proceed with track wear only if vegetation, snow, mud, debris, recovery, or other secondary features require separate investigation.

### NO-GO / REDESIGN

Redesign if durable visual wear cannot be persisted/synchronized cleanly, if viable representations require unacceptable object/state growth, or if vehicle path observation cannot be bounded to an acceptable performance envelope.

## Deliverables at completion

- measured results added here;
- `docs/DESIGN.md` updated with validated constraints;
- `docs/REQUIREMENTS.md` updated with numeric budgets where evidence supports them;
- `CHANGELOG.md` updated;
- ADRs created only for durable choices supported by measurements;
- follow-on spikes created only for unresolved blocking questions.
