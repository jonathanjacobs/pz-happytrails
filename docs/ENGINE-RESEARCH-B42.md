# Build 42 Engine Research Notes

## Purpose

This document records behavior discovered by inspecting a locally decompiled Project Zomboid installation supplied for Happy Trails interoperability and feasibility research.

The decompiled game source is **not** part of this repository and must not be redistributed through Happy Trails. This document records only behavioral findings, relevant public class/method/property names, and implications for our own independent implementation.

## Examined build

The supplied source identifies itself as:

```text
Project Zomboid 42.20.2
Git revision: ffe7a8a4b1
```

These findings are therefore build-specific until verified against later Project Zomboid releases.

## Status terminology

- **Engine-confirmed** — behavior is directly present in the examined Java implementation.
- **Lua-unverified** — Java behavior exists, but Happy Trails has not yet proven that the required API is callable/reliable from Lua in the relevant SP/client/server context.
- **Runtime-unverified** — expected behavior still requires an actual game/server test.

No item in this document by itself constitutes a Happy Trails architecture decision.

---

## 1. Floor blood splats: useful model, poor direct extension point

### Engine-confirmed behavior

`IsoChunk` owns a bounded queue of floor blood splats with a capacity of **1000 per chunk**. `IsoChunk.addBloodSplat(...)` creates a compact `IsoFloorBloodSplat`, evicts the oldest entry when the queue is full, invalidates the chunk render cache, and on a dedicated server sends the game's native blood-add packet to relevant clients.

Floor splats are serialized with the chunk and restored during chunk loading. The compact splat record contains chunk-relative/sub-square position, z, type, age information, and a small index used by blood-decal density handling. The persisted representation is substantially smaller than a full ordinary world object.

The native blood lifespan option is consulted while loading: blood older than the configured lifespan may be discarded.

### Critical limitation discovered

The renderer is not a generic decal registry in this build.

`IsoFloorBloodSplat` defines a fixed set of **21 built-in floor-blood texture types**. The FBO rendering path validates the splat type against that fixed table and resolves the texture from that table. It also applies blood-specific tint/darkening/fade behavior and respects the client's blood-decal rendering setting.

Consequences for Happy Trails:

- arbitrary new tire-track types cannot simply be assigned a new integer type through ordinary data-driven configuration;
- Happy Trails marks would share the same 1000-entry-per-chunk queue with real gore if this subsystem were reused directly;
- marks would inherit blood-specific rendering controls and lifespan semantics;
- direct modification of the native type table would be brittle and would risk replacing/interfering with vanilla blood rather than cleanly extending it.

### Current assessment

**Direct reuse: not preferred / likely reject for normal Lua implementation.**

The subsystem remains extremely valuable as a design reference: PZ itself demonstrates that environmental marks can be represented as compact chunk-local state and rendered through cached chunk rendering rather than as one heavyweight `IsoObject` per individual mark.

A very small Lua-accessibility smoke test is still reasonable if it costs little, but SPIKE-001 should no longer assume that native blood splats are the leading production rendering substrate.

---

## 2. Existing-object overlay sprites are a stronger visual candidate

### Engine-confirmed behavior

`IsoObject` has an `overlaySprite` and optional overlay color. The public overlay setter accepts a sprite name and updates the object's overlay without replacing the object's primary sprite.

Changing an overlay:

- invalidates the relevant FBO/chunk rendering state;
- can be propagated with the game's native overlay-update networking path;
- is saved as part of the object's normal serialized state;
- is reconstructed during load;
- does not inherently require creating a second world object when applied to an existing floor object.

The persistence path can store a literal sprite name when a dictionary ID is unavailable, which is promising for custom Happy Trails sprite assets.

### Why this may fit Happy Trails

A natural floor square already has a floor object. If a tire/wear graphic can live in that floor object's overlay slot, a wear-state transition may be representable as:

```text
existing floor object
+ Happy Trails overlay sprite
```

rather than:

```text
existing floor object
+ additional persistent IsoObject for every visual mark
```

This could provide:

- preservation of the underlying terrain identity;
- custom sprite support;
- normal chunk/FBO caching;
- native save/load behavior;
- native multiplayer overlay synchronization;
- no extra ordinary world object for the visual layer.

### Open risks

Each `IsoObject` appears to have only one overlay slot. Happy Trails must determine:

- how often natural floor objects already use overlays;
- whether replacing an existing overlay would damage vanilla or another mod's presentation;
- whether overlays can be cleared/restored safely;
- how orientation/state variants look on the isometric floor;
- whether the overlay API is fully usable from Lua in dedicated-server multiplayer.

### Current assessment

**High-priority SPIKE-001 candidate. Lua/runtime verification required.**

---

## 3. PZ already computes transient vehicle/foliage interaction for rendering

### Engine-confirmed behavior

The render path maintains a transient `flattenGrassEtc` flag on `IsoGridSquare` near visible vehicles. During rendering, the engine marks a bounded region around each vehicle, and foliage-like objects are routed through vegetation-specific rendering behavior.

The engine's relevant foliage classification includes concepts such as bushes, removable vegetation, and attached-floor vegetation.

### Implication

This is **not** a persistent trail system. It appears to be a temporary rendering mechanism associated with vehicles and vegetation presentation.

It is nevertheless useful evidence because:

- PZ already has a notion of foliage that should be treated differently near vehicles;
- the engine's own vegetation classification may help us identify candidate objects;
- Happy Trails should not mistake this transient flag for durable deformation state.

### Current assessment

**Useful classification/rendering clue, not a persistence mechanism.**

---

## 4. Native vehicle collision already performs vegetation-local collision work

### Engine-confirmed behavior

`BaseVehicle` already enumerates nearby relevant squares/objects as part of its collision processing and performs precise object collision tests. The collision path recognizes properties including `CarSlowFactor` and `HitByCar` and also calls native plant-collision handling for vegetation candidates.

The plant collision path considers trees, bushes, and selected plant tiles and ignores very low vehicle speeds before applying collision effects.

### Implication

Happy Trails should avoid creating a second continuous Lua-side vegetation scanner unless native behavior proves insufficient. The engine is already paying the cost to identify actual vehicle/object interaction.

The preferred investigation is therefore:

1. determine which vegetation sprites can safely opt into native vehicle damage;
2. test the relevant sprite properties on a controlled object;
3. allow the engine to perform collision detection;
4. add only the minimum Happy Trails aftermath/state work that native behavior does not provide.

### Current assessment

**Strong candidate for vegetation damage; runtime/Lua tile-definition validation required.**

---

## 5. `HitByCar` and `MinimumCarSpeedDmg` have substantial native behavior

### Engine-confirmed behavior

The generic object collision path checks `HitByCar`. When present, it reads `MinimumCarSpeedDmg`, evaluates vehicle speed, and can route the collision through native vehicle damage handling.

For objects with the appropriate properties, native behavior can:

- decrement object damage;
- emit impact/world sound effects;
- switch to a configured damaged sprite at an intermediate damage threshold;
- transmit the sprite update to multiplayer clients;
- remove the destroyed object from the square at a later threshold;
- transmit that removal through normal multiplayer world-object synchronization.

### Implication for Happy Trails

A bush/sapling progression may be able to use native engine behavior such as:

```text
intact vegetation
-> damaged vegetation sprite
-> removed vegetation
```

without Happy Trails continuously searching nearby squares and independently implementing object-removal synchronization.

This does **not** mean Happy Trails should globally attach `HitByCar` to every vegetation sprite. Classification, compatibility, minimum speeds, damaged-sprite availability, and interaction with vanilla collision must be tested narrowly first.

### Current assessment

**Very promising performance-first vegetation approach.**

---

## 6. Vehicle scripts expose actual wheel geometry

### Engine-confirmed behavior

The vehicle scripting API contains wheel definitions with data including:

- wheel ID/model;
- front/rear status;
- local vehicle-space offset;
- radius;
- width.

`VehicleScript` exposes wheel count/wheel access, while `BaseVehicle` exposes methods that transform a local vehicle-space position into world coordinates.

The vehicle also exposes current speed and other physical state useful for later wear weighting.

### Implication

Happy Trails does not necessarily need to approximate the track as the vehicle's center square or a generic vehicle-width rectangle.

A possible low-cost geometry path is:

```text
for each selected wheel path
    transform script wheel offset to current world position
    compare with previous sampled wheel position
    rasterize/interpolate the short segment between them
    process only newly crossed candidate squares
```

This preserves the option for convincing left/right wheel tracks without requiring a physics simulation of our own.

The MVP may still choose a coarser representation if benchmarks show it is materially cheaper, but exact wheel geometry is available to compare.

### Current assessment

**Engine capability confirmed; dedicated-server Lua visibility and cost require runtime verification.**

---

## 7. Multiplayer already replicates detailed vehicle transforms to the server

### Engine-confirmed behavior

The game's native vehicle-physics network packet carries vehicle position/orientation and detailed physical state from the authorized vehicle simulation owner to the server. The server writes those values into its authoritative `BaseVehicle` representation and relays updates onward.

The moving-vehicle network cadence is substantially more frequent than the stationary cadence; in the examined implementation the moving path targets roughly **150 ms** update periods, versus a slower approximately **300 ms** period when appropriate.

### Implication

A custom Happy Trails client protocol for basic vehicle position may be unnecessary.

A potentially efficient server-side design candidate is:

```text
native PZ vehicle networking
-> server BaseVehicle transform changes
-> Happy Trails samples only relevant moving vehicles
-> actual wheel positions are transformed
-> previous/current wheel points are interpolated
-> only newly crossed natural squares produce wear work
```

At high speeds a vehicle can move across multiple squares during a 150 ms interval, so interpolation/rasterization between samples is essential; simply inspecting the current square would create gaps.

### Current assessment

**This materially strengthens the server-side sampling candidate.** Runtime testing must verify how frequently those replicated changes are visible to Lua on the dedicated server.

---

## 8. No general vehicle-update Lua event was found

### Engine-confirmed observation

The examined Lua event registration includes vehicle-related events such as enter/spawn/damage-texture events, but no general `OnVehicleUpdate`-style motion event was found.

### Implication

Happy Trails may need a periodic/tick callback for vehicle observation. That is not automatically a performance problem if the hot path is aggressively bounded:

```text
iterate current relevant vehicles
-> reject stationary/irrelevant vehicles immediately
-> reject unchanged transform/sample distance
-> perform geometry/classification only after a meaningful movement threshold
```

The decision should be based on measured work per active vehicle, not on avoiding `OnTick` categorically.

---

## 9. Revised SPIKE-001 priorities

Based on the engine review, the first runtime experiments should be narrower than a complete track prototype.

### Experiment A — existing-floor overlay smoke test

On a controlled natural floor square:

1. inspect whether the floor already has an overlay;
2. apply one conspicuous Happy Trails test overlay;
3. confirm no additional ordinary world object is created;
4. verify second-client visibility;
5. verify save/reload;
6. verify dedicated-server restart;
7. verify late join;
8. clear/restore the overlay;
9. repeat on a floor that already has an overlay if one can be found safely.

### Experiment B — server wheel-geometry probe

On a dedicated server:

1. observe active vehicle transforms from Lua;
2. read wheel count and local offsets;
3. transform wheel positions to world coordinates;
4. record actual observed server update intervals while another client drives;
5. interpolate between consecutive samples;
6. verify that the resulting crossed-square sequence has no gaps at representative speeds.

No terrain mutation is required for this experiment.

### Experiment C — native vegetation damage probe

On a deliberately selected test vegetation object/sprite:

1. add/test `HitByCar` and a controlled `MinimumCarSpeedDmg`;
2. optionally configure a known damaged sprite;
3. drive into it at below/above threshold speeds;
4. verify native damage/sprite transition/removal behavior;
5. verify server authority and client synchronization;
6. confirm no custom Lua neighborhood scan is necessary.

### Experiment D — only then prototype wear state

If A and B succeed, build the smallest possible repeated-pass wear experiment and compare at least one alternative representation before selecting an architecture.

---

## 10. Design implications without design commitment

The decompiled source substantially narrows the search space:

- **Direct native blood-splat reuse:** less attractive than initially expected because the renderer is blood-specific and shares gore lifecycle controls.
- **Existing floor overlay:** more attractive than initially expected because persistence, native synchronization, and FBO invalidation are already implemented.
- **Vehicle center-square approximation:** no longer technically necessary; actual wheel geometry exists.
- **Custom client passage protocol:** may be unnecessary because native PZ vehicle replication already feeds the server detailed transforms.
- **Continuous Lua vegetation scan:** may be unnecessary because native vehicle/object collision plus `HitByCar` can already perform much of the work.

These are changes in **experimental priority**, not ADR-level architecture decisions.
