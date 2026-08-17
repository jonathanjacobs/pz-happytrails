# Happy Trails Design Space

## Status

Pre-alpha design-space document.

This document records **options, constraints, evidence, and unresolved tradeoffs**. It is intentionally not a committed architecture. Durable technical choices should be made only after comparative runtime evidence and recorded in ADRs.

See [`ENGINE-RESEARCH-B42.md`](ENGINE-RESEARCH-B42.md) for build-specific findings from the supplied Project Zomboid 42.20.2 decompiled source.

## Design objective

Represent repeated vehicle travel as persistent environmental change while keeping client, server, persistence, world-object, and network costs bounded.

The implementation should remain flexible enough to support higher-fidelity features later without requiring the first release to pay their runtime cost.

## Non-negotiable constraints

Any eventual design should:

- process only relevant movement rather than broad world areas;
- keep durable mutations server-owned;
- avoid unbounded historical state;
- avoid one persistent object for every historical wheel impression unless measurements prove that representation acceptable;
- allow performance/fidelity controls to reduce actual work;
- keep authoritative wear semantics independent from a particular rendering implementation;
- keep terrain classification replaceable;
- keep vegetation destruction separable from track wear;
- prefer engine-native collision/persistence/network behavior where it is safe and measurable;
- fail closed for unknown/destructive cases;
- make performance-sensitive work observable.

## 1. Traffic representation

### Option A — bounded per-square wear score

A modified square carries a bounded value and, if recovery is supported, perhaps a last-update time.

```text
Wear = clamp(0, MaxWear, Wear + PassageWeight)
```

Strengths:

- natural progressive wear;
- repeated passages can be coalesced;
- visual mutation only needs to occur when thresholds change;
- raw pass history is unnecessary.

Questions:

- persistence overhead per modified square;
- synchronization behavior if metadata is used;
- recovery bookkeeping.

### Option B — discrete state only

Store only the current tier plus minimal transition/recovery information.

Strengths:

- compact durable state;
- simple rendering contract.

Questions:

- whether enough information remains for gradual recovery/tuning;
- whether transient counters simply reintroduce complexity elsewhere.

### Option C — chunk/route aggregation

Aggregate traffic more coarsely and materialize square-level wear only where needed.

Strengths:

- potentially smaller storage on very long routes.

Questions:

- risks losing natural local path geometry;
- chunk lifecycle/reconstruction complexity;
- may be premature optimization if sparse square state is already cheap.

No representation is selected.

## 2. Vehicle observation

### Option A — server-side sampling of native replicated vehicles

The B42.20.2 engine review materially strengthens this candidate. PZ already replicates detailed vehicle transforms to the server, and vehicle script data exposes actual wheel offsets.

Candidate flow:

```text
native PZ vehicle replication
-> server considers only relevant moving vehicles
-> reject unchanged/stationary state cheaply
-> transform selected wheel offsets into world positions
-> interpolate/rasterize previous-to-current wheel segments
-> emit only newly crossed candidate squares
```

Potential strengths:

- simple authority model;
- likely no Happy Trails client movement protocol;
- actual vehicle-specific wheel geometry;
- work can be proportional to active moving vehicles and distance traveled rather than map size.

Questions:

- exact Lua visibility/cadence on a dedicated server;
- cost per moving vehicle;
- best rejection/distance thresholds;
- whether processing all wheels adds useful fidelity relative to representative left/right wheel paths.

### Option B — client passage reporting with server validation

This remains a fallback/comparison rather than the default assumption.

Potential strengths:

- can distribute geometry work to the driving client if server state is inadequate.

Potential costs:

- custom protocol and batching;
- duplicate/invalid report handling;
- validation may duplicate server work;
- additional network traffic despite PZ already sending vehicle state.

### Option C — native/event-assisted hybrid

Use engine-native collision and state changes where PZ already performs the relevant calculation, with Happy Trails sampling only continuous trail wear.

This is especially attractive for vegetation damage.

No movement strategy is selected until runtime measurements exist.

## 3. Vehicle footprint and path geometry

B42.20.2 vehicle scripts expose wheel count and local wheel offsets, and `BaseVehicle` can transform local positions into world coordinates.

Fidelity options therefore include:

1. centerline;
2. centerline plus generic width;
3. oriented vehicle footprint;
4. representative left/right wheel paths;
5. all actual scripted wheel paths.

Because server vehicle updates may be separated by roughly 150 ms while moving, any wheel-path implementation must interpolate between prior/current world positions. Sampling only the current tile is not sufficient at higher speeds.

The data model should remain independent from which fidelity level ultimately wins the benchmark.

## 4. Terrain classification

Candidate evidence sources:

- floor sprite properties;
- tile/sprite names as a bounded fallback;
- erosion/natural-floor metadata;
- zones;
- allow/deny tables;
- combinations of the above.

Classification should be centralized and cacheable where safe. Unknown or ambiguous terrain should produce no destructive mutation.

The engine's existing transient foliage logic also provides clues about how vanilla recognizes bushes/removable/attached-floor vegetation, but that should inform classification rather than become a persistence mechanism.

## 5. Visual representation

### Option A — existing floor-object overlay sprite

This is now the highest-priority visual experiment based on B42.20.2 engine research.

Engine findings indicate that an `IsoObject` overlay sprite:

- is a visual layer on the existing object;
- is serialized with the object;
- has native MP update handling;
- invalidates cached/FBO chunk rendering when changed;
- can persist a literal custom sprite name;
- does not inherently require adding another ordinary world object when applied to the existing floor.

Potential strengths:

- preserves underlying floor identity;
- potentially very low steady-state render overhead due to chunk caching;
- native persistence/networking;
- custom track/wear sprite assets appear feasible;
- object count need not grow merely because a square becomes visually worn.

Critical questions:

- Lua access and dedicated-server behavior;
- how frequently natural floors already occupy the overlay slot;
- compatibility with vanilla/other-mod overlays;
- orientation and blending quality;
- safe restoration/clearing.

An occupied overlay must not be blindly overwritten.

### Option B — additional ordinary `IsoObject`

Reference mods prove this is practical, but object count, cleanup, late-join sync, and lifecycle cost are concerns.

Keep this as a benchmark/fallback, not the default.

### Option C — floor/tile replacement

Potentially compact, but risks altering terrain identity, erosion interaction, mapping, reversibility, and compatibility. Test only if less invasive layers fail.

### Option D — native floor-blood splats

B42.20.2 confirms that PZ's blood system is an excellent **model** for lightweight environmental marks but a poor direct generic extension point.

Confirmed constraints include:

- 1000-entry bounded queue per chunk;
- fixed 21-type built-in blood texture table in the renderer;
- blood-specific color/age treatment;
- shared gore queue/lifecycle;
- blood-decal rendering controls;
- blood lifespan semantics.

Therefore direct reuse is currently **not preferred** and may be rejected after a minimal Lua smoke test. Happy Trails should not modify/hijack vanilla blood tables as a production strategy.

### Option E — custom visual materialization

If overlays are insufficient, a later approach could separate compact authoritative wear from visual materialization for loaded chunks. This remains open but should not be implemented until simpler native-backed options are measured.

## 6. Persistence

Candidate state stores remain:

- state encoded directly by durable world-object visual state;
- square/object `modData`;
- sparse server-side modified-square records;
- coarser chunk aggregation if justified.

Selection criteria:

- save/restart reliability;
- late-join convergence;
- storage growth per kilometer of route;
- recovery/cleanup cost;
- compatibility with erosion and modded maps;
- independence from unrelated vanilla systems.

A useful target is to store the **current meaning** of a modified square, not a history of every pass that produced it.

## 7. Vegetation damage

### Option A — native `HitByCar` / collision-property path

This candidate is substantially stronger after engine review.

B42.20.2 already performs localized vehicle/object collision work. For suitable objects, native `HitByCar` handling can evaluate minimum damage speed, decrement damage, switch to a damaged sprite, synchronize that sprite, and eventually remove/synchronize the destroyed object.

Potential strengths:

- no duplicate continuous Lua neighborhood scanner;
- engine already computes actual collision;
- native multiplayer object mutation;
- damaged-state progression may be available essentially for free once sprites/properties are configured.

Questions:

- which vanilla vegetation objects can safely opt in;
- appropriate speed/damage semantics;
- compatibility with existing tile properties;
- availability of suitable damaged sprites;
- whether custom vegetation mods inherit behavior gracefully.

### Option B — explicit bounded Lua spatial scan

Retain only as a fallback/comparison if native collision cannot cover required vegetation classes.

### Option C — hybrid aftermath

Let native collision own collision/damage/removal; Happy Trails reacts only to add optional aftermath such as wear increment or controlled debris.

Vegetation remains optional relative to core track wear.

## 8. Debris

Possible forms:

- inventory/world items;
- decorative objects;
- transient/local presentation;
- disabled entirely in performance-focused configurations.

Persistent item proliferation must be measured before debris is enabled by default.

## 9. Recovery

Potential approaches:

- lazy elapsed-time evaluation when a modified square is revisited;
- chunk-load evaluation;
- integration with vanilla erosion where practical;
- no recovery in early MVP while persistence/performance stabilize.

Continuous full-world scans are excluded.

## 10. Performance controls

A meaningful control must reduce actual work, not merely hide graphics.

Candidate control dimensions include:

- vehicle sampling rejection/distance threshold;
- number of wheel paths sampled;
- interpolation resolution;
- candidate-square coalescing;
- threshold-only visual mutation;
- terrain classification caching;
- optional vegetation damage;
- optional debris;
- optional snow/mud embellishment;
- recovery cadence;
- diagnostic verbosity.

## 11. Current experimental priority

The engine review changes test order without selecting architecture:

1. **Existing-floor overlay smoke test** — persistence, MP, object count, conflicts.
2. **Server wheel-geometry probe** — actual Lua-visible transforms/cadence and gap-free interpolation.
3. **Native vegetation-damage probe** — controlled `HitByCar`/minimum-speed/damaged-sprite behavior.
4. **Minimal repeated-wear prototype** — only after those primitives are validated.
5. **Alternative benchmark** — required before ADR selection.

## 12. When architecture may be selected

No option becomes preferred/final until SPIKE-001 has runtime evidence for:

- server movement observation and cost;
- path continuity at speed;
- visual persistence and MP convergence;
- render/world-object/state growth;
- terrain classification cost;
- scaling with simultaneous vehicles and exploration distance;
- compatibility/failure behavior.

At that point, durable decisions should be captured in ADRs with measurements and rejected alternatives.
