# Happy Trails — Architecture and Design Space

## Status

Pre-alpha architecture/design-space document.

This document records current constraints, candidate architectures, engine evidence, and unresolved tradeoffs. It is intentionally not a final architecture. Durable choices should be selected only after runtime evidence and then recorded in ADRs.

See [`ENGINE-RESEARCH-B42.md`](ENGINE-RESEARCH-B42.md) for build-specific engine findings and [`spikes/SPIKE-001-environmental-wear-feasibility.md`](spikes/SPIKE-001-environmental-wear-feasibility.md) for the active feasibility investigation.

## Design objective

Represent repeated vehicle travel as environmental history: fresh tracks appear at sub-tile fidelity, repeated traffic produces durable terrain wear, abandoned routes recover, and multiplayer/server costs remain bounded.

## Non-negotiable constraints

The implementation should:

- process active/relevant movement rather than scanning broad world areas;
- keep durable terrain state server-authoritative;
- avoid unbounded historical state;
- avoid one permanent ordinary world object for every historical tire impression;
- keep authoritative wear semantics independent from any particular rendering mechanism;
- support bounded cleanup/recovery without whole-world iteration;
- keep terrain classification replaceable and fail closed for unknown/destructive cases;
- keep vegetation destruction separable from track wear;
- prefer native persistence/network/rendering behavior where safe and measurable;
- expose performance-sensitive work through diagnostics during development.

## 1. Two-layer world model

The current strongest model separates **transient marks** from **persistent terrain disturbance**.

### Layer A — transient marks

Examples:

- fresh tire impressions;
- mud or snow tracks;
- recently flattened vegetation;
- carried mud/snow/dust deposited onto hard surfaces.

Desired properties:

- sub-tile position;
- orientation and/or geometry appropriate to wheel paths;
- compact records;
- bounded per chunk or loaded region;
- age-based visual fading;
- no requirement to preserve every mark indefinitely.

### Layer B — persistent terrain disturbance

Examples:

- worn grass;
- exposed soil;
- established wheel ruts;
- durable two-track trails;
- long-lived suppression of regrowth along frequently used routes.

Desired properties:

- sparse storage only where player activity has materially changed terrain;
- bounded wear score or discrete wear tier;
- last-traffic timestamp for lazy recovery;
- server authority and save/restart persistence;
- visual representation selected separately from the semantic state.

A representative progression is:

```text
UNDISTURBED
  -> IMPRINT
  -> WORN
  -> ESTABLISHED
  -> DEEP_RUT
```

Recovery should use hysteresis so established routes do not oscillate rapidly between states.

## 2. Vanilla floor-blood system as architectural precedent

Build 42.20.3's `IsoFloorBloodSplat` system is now an important design reference.

Confirmed engine properties:

- floor blood is stored in a dedicated bounded queue on each `IsoChunk` rather than as unlimited ordinary world objects;
- the queue is capped at 1000 splats per chunk;
- each splat stores compact sub-tile position (`x`, `y`, `z`), type/index data, and creation `worldAge`;
- splats serialize with the chunk;
- when the bounded queue fills, the oldest splat is moved into a short fade-out list rather than allowing unbounded growth;
- rendering computes age from current `WorldAgeHours - splat.worldAge`;
- blood tint/alpha is reduced over a 72-world-hour visual aging period.

This validates several Happy Trails principles:

1. high-resolution environmental marks can be represented independently from square-level terrain mutation;
2. sub-tile coordinates are practical and desirable for paired wheel paths;
3. creation-time + lazy age evaluation is preferable to decrementing every mark every tick;
4. transient mark storage should be explicitly bounded;
5. transient appearance and persistent terrain condition should be separate concepts.

Directly hijacking `IsoFloorBloodSplat` is **not** the intended production design. Its sprite table, tinting, queue semantics, and rendering path are blood-specific. The useful lesson is the architecture, not reuse of vanilla blood assets or tables.

## 3. Vehicle observation

The leading candidate remains server-side observation of PZ's replicated vehicle state.

Candidate flow:

```text
native vehicle replication
-> reject stationary/irrelevant vehicles cheaply
-> read vehicle transform and wheel geometry
-> transform representative wheel offsets to world coordinates
-> interpolate previous/current wheel segments
-> emit newly crossed mark/path samples
-> update transient marks and persistent wear separately
```

A client-reporting protocol remains a fallback if dedicated-server Lua cannot observe sufficiently frequent/accurate vehicle transforms.

## 4. Wheel geometry and sampling

B42 vehicle scripts expose wheel count/local wheel positions and `BaseVehicle` exposes transforms that can place them in world space.

Fidelity options remain:

1. vehicle centerline;
2. centerline + generic width;
3. representative left/right wheel paths;
4. all scripted wheels.

The blood-trail observation reinforces that distance-based deposition should be investigated. A continuous moving source should not create one permanent object per frame; instead marks should be emitted at bounded spatial intervals and interpolated across gaps between meaningful transform updates.

SPIKE-001 should trace the vanilla corpse-drag blood emission path to see how PZ decides when to deposit another splat while a corpse is moving.

## 5. Terrain classification

Candidate inputs include:

- floor sprite/tile properties;
- erosion or natural-floor metadata;
- bounded sprite-name allow/deny tables;
- zones;
- combinations of the above.

Classification should be centralized and cacheable. Unknown surfaces should receive no destructive persistent mutation.

Hard/non-deformable surfaces should not accumulate persistent terrain wear, but may receive temporary carried-material marks such as mud tracked onto pavement.

## 6. Visual representation candidates

### A. Custom lightweight ground-decal path

Now the highest-value rendering question because the vanilla blood implementation demonstrates the desired behavior class.

Required capability:

- custom texture/sprite;
- sub-tile x/y placement;
- orientation/rotation or sufficient directional variants;
- alpha/tint control;
- bounded lifecycle;
- persistence/synchronization appropriate to the selected mark class.

Key question:

> Can Lua create a lightweight arbitrary custom ground decal with persistence/fading characteristics comparable to `IsoFloorBloodSplat`, without creating a full ordinary `IsoObject` for every tire impression?

### B. Existing floor-object overlay

Still a strong candidate for **persistent wear-state presentation**. Engine research indicates overlay sprites can persist and synchronize without necessarily adding another ordinary world object.

This may be better suited to square-level established wear than to dense sub-tile fresh tire impressions.

### C. Additional `IsoObject`

Known to be practical from reference mods but remains a fallback because object-count, cleanup, late-join, and persistence costs may be significant.

### D. Floor/tile replacement

Potentially compact but more invasive. Risks terrain identity, erosion interaction, reversibility, and compatibility. Test only if less invasive approaches fail.

### E. Native blood splats

Use only as a minimal API/rejection experiment and architectural reference. Do not modify/hijack vanilla blood tables as a production strategy.

## 7. Persistent wear representation

Candidates:

- bounded per-square wear score plus last-traffic time;
- discrete wear tier plus minimal recovery metadata;
- sparse server-side records for modified squares;
- state encoded partly in durable world visual state if measurements support it.

The durable record should represent the **current meaning** of changed terrain, not every vehicle pass that produced it.

## 8. Aging and recovery

Transient marks should use lazy age evaluation:

```text
age = currentWorldAgeHours - mark.createdWorldAge
```

No full-world timer scan should be required.

Persistent wear should recover on a much slower timescale using lazy evaluation when a square/chunk is revisited or loaded. A likely long lifecycle is:

```text
DEEP_RUT -> WEATHERED_RUT -> OVERGROWN_RUT -> FAINT_TRAIL -> RECOVERED
```

Weather may alter appearance or recovery rate without erasing the underlying persistent state accidentally.

## 9. Weather/material model

Persistent **wear** and temporary **material transfer** are distinct.

Examples:

```text
grass -> flattened/worn/exposed dirt       persistent wear
muddy field -> muddy road-edge tracks       temporary transfer
snow -> compressed tire impressions         temporary surface mark
```

Rain, snow, thaw, and regrowth should be able to modify appearance/lifetime independently of the semantic wear tier.

## 10. Vegetation damage

Prefer native vehicle/object collision semantics where possible, including `HitByCar`/minimum-speed behavior and native synchronized object mutation. A broad Lua vegetation neighborhood scanner should be a fallback, not the first design.

Vegetation damage remains separable from track-state persistence so it can be disabled or deferred independently.

## 11. Performance principles

A performance/fidelity control must reduce actual work. Candidate dimensions:

- active-vehicle rejection thresholds;
- wheel paths sampled;
- mark spacing;
- interpolation resolution;
- transient marks retained per chunk;
- candidate-square coalescing;
- threshold-only persistent-state mutation;
- terrain-classification caching;
- recovery cadence;
- optional vegetation/debris/weather embellishment;
- diagnostic verbosity.

## 12. Current experimental priority

1. trace corpse-drag blood deposition to identify vanilla continuous-mark spacing logic;
2. test whether a custom lightweight sub-tile decal path is Lua-accessible;
3. benchmark existing-floor overlays for established/persistent wear;
4. probe dedicated-server wheel geometry and gap-free interpolation;
5. probe native vegetation damage;
6. build the smallest repeated-wear prototype only after these primitives are understood;
7. compare at least two viable representations before recording an ADR.

No rendering or persistence strategy is considered final until SPIKE-001 produces runtime evidence.
