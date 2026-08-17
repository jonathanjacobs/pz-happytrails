# SPIKE-001 — Build 42 Environmental Wear Feasibility

## Status

In progress — comparative source review complete; runtime experiments not yet started.

## Question

Can Project Zomboid Build 42 support a performant, persistent, multiplayer-safe Happy Trails mechanic using Lua-accessible APIs and ordinary mod assets, without engine modification or brittle hacks?

A second question is equally important:

> Which implementation strategy provides the best client/server performance envelope without preventing higher-fidelity features later?

## Why this spike exists

The desired gameplay mechanic combines several concerns that must work together:

1. observing vehicle movement at sufficient spatial resolution;
2. classifying natural terrain and vegetation reliably;
3. representing repeated traffic without unbounded history;
4. mutating visual/world state persistently;
5. synchronizing durable mutations correctly in multiplayer;
6. doing all of the above with bounded client, server, object-count, persistence, and network cost.

Implementation should not begin by assuming any one API path works. This spike should produce evidence for later architectural decisions.

## Phase 0 — comparative reference-mod review

Three supplied Build 42 mod source packages were reviewed as research inputs:

- Workshop package `3457632586` — Footprints / zombie and bandit footprints;
- Workshop package `3690554902` — Vehicle Vegetation Destruction (VVD);
- Workshop package `3413150945` — More Damaged Objects.

These sources are comparison material only. No source from them has been incorporated into Happy Trails, and their licensing is not assumed to permit reuse.

Detailed observations are recorded in [`REFERENCE-IMPLEMENTATIONS.md`](REFERENCE-IMPLEMENTATIONS.md).

### Useful evidence from the review

The review established several techniques worth independently validating:

- Build 42 Lua can create world visual objects using `IsoObject` and add/remove them from squares.
- Existing mods use square/object transmission methods to synchronize durable object changes.
- Movement sampling, batching, caps, deferred work, and sparse tracking are all practical techniques, but their implementation details have major performance implications.
- Vehicle Vegetation Destruction demonstrates a bounded local scan plus server validation model, but also illustrates risks from repeated object enumeration and session-state growth.
- More Damaged Objects modifies sprite properties including `HitByCar` and `MinimumCarSpeedDmg`, suggesting that some collision work may be delegated to native engine behavior instead of reproduced with Lua spatial polling.
- More Damaged Objects also shows that vehicle-damaged bushes can produce branches/twigs, but its impact-detection path uses a broad local scan triggered by a world-sound signature and should not be assumed to be the best Happy Trails approach.
- The Footprints implementation contains substantial machinery for actor scanning, LOS gating, object creation/removal, cleanup, persistence bookkeeping, and MP synchronization. Its complexity is a warning against modeling every tire mark as an independently managed persistent object without measurement.

Separate review of the official Project Zomboid Java API documentation identified a native floor-blood/decal subsystem that warrants direct investigation. `IsoChunk` maintains bounded floor-blood collections, `IsoFloorBloodSplat` stores position/type/world-age state and implements save/load, and the network protocol includes `BloodSplatter` and `RemoveBlood` packet types. Server options also expose `BloodSplatLifespanDays`. This may provide either a reusable rendering path or, at minimum, a strong model for how Project Zomboid itself implements large numbers of lightweight environmental marks.

These are hypotheses and code/API observations, not validated Happy Trails design decisions.

## Scope

SPIKE-001 should answer the following.

### A. Vehicle observation

Determine what Build 42 exposes server-side and client-side for:

- active vehicles;
- vehicle position and movement;
- speed;
- orientation;
- occupied/unoccupied state;
- vehicle script/type/mass where useful;
- wheel positions or footprint information if available;
- event callbacks versus polling requirements;
- native collision callbacks/properties that may eliminate polling for vegetation damage.

Measure the practical event/sampling cadence needed to avoid gaps at normal and high driving speeds.

### B. Competing movement-detection strategies

At least these candidate strategies should be tested before one is selected:

#### Strategy A — server-side active-vehicle sampling

The server samples only relevant moving vehicles, derives affected squares, and owns all wear decisions.

Questions:

- Does the dedicated server expose sufficiently current vehicle transforms?
- What sampling cadence is required at highway speed?
- Can sampling be distance-driven rather than tick-driven?
- How much work is required per active vehicle?

#### Strategy B — client passage reporting + server validation

The driving client reports compact passage candidates; the server validates identity, vehicle ownership/driver state, position, and plausible distance before updating durable wear.

Questions:

- Does this materially reduce server sampling cost?
- Can multiple square crossings be batched into one report?
- Can the server validate reports without recreating the same expensive scan?
- What is the network cost at different vehicle counts and speeds?

#### Strategy C — hybrid/event-assisted

Use native engine events/properties where available, with minimal sampling only for track-wear information the engine does not expose directly.

Questions:

- Can `HitByCar`, `MinimumCarSpeedDmg`, collision/world-sound events, or other native mechanisms eliminate Lua-side vegetation collision scanning?
- Can durable track-wear state be updated only after meaningful distance or threshold crossings?

No strategy is preferred until measured.

### C. Surface classification

Identify a robust way to distinguish:

- grass;
- dirt;
- forest/natural ground;
- paved road;
- building/interior floors;
- water or invalid terrain;
- snow-covered conditions where observable.

Compare classification based on floor sprite names, sprite properties, tile properties, zones, erosion data, or another API.

Measure the cost of classification and whether immutable results can be cached safely.

### D. Persistent visual mutation

Test plausible mutation approaches independently:

- replace/change floor;
- add/remove an `IsoObject`;
- attach or change an overlay/sprite;
- place a custom tile/object;
- use a native lightweight decal/splat mechanism if accessible;
- use another supported world-decoration mechanism.

For each approach record:

- visual result;
- save/load persistence;
- dedicated-server restart persistence;
- MP synchronization;
- collision/pathing side effects;
- removability/reversibility;
- number of persistent objects/state records created per kilometer of representative travel;
- mutation and cleanup cost;
- late-join behavior.

A central question is whether Happy Trails can represent a square's current **wear state** rather than creating a separate persistent visual object for every pass.

### D1. Native floor-blood/decal subsystem investigation

Project Zomboid already renders potentially large numbers of irregular floor marks without replacing the underlying terrain tile. This path should be investigated before Happy Trails commits to an `IsoObject`-based track renderer.

Official API surfaces to investigate include:

- `IsoChunk.FloorBloodSplats`;
- `IsoChunk.FloorBloodSplatsFade`;
- `IsoChunk:addBloodSplat(float x, float y, float z, int type)`;
- `IsoFloorBloodSplat` position, type, `worldAge`, sprite-map, save, and load behavior;
- `IsoGridSquare:splatBlood(...)`, `removeBlood(...)`, and `DoSplat(...)`;
- network packet types `BloodSplatter` and `RemoveBlood`;
- server option `BloodSplatLifespanDays`;
- client blood-decal rendering settings;
- `IsoGridSquare.bFlattenGrassEtc` as a separate native field potentially relevant to vegetation/ground presentation.

Questions to answer experimentally:

1. Is the floor-blood renderer callable from Lua in Build 42 server/client contexts?
2. Can it render custom tire-track sprite assets, or is it hard-wired to the built-in blood type table?
3. If custom sprite registration is possible, does it survive save/load and MP replication?
4. Is floor-blood state stored directly in chunks rather than as ordinary `IsoObject`s?
5. What are the actual queue/cap limits and eviction rules?
6. Would Happy Trails marks compete with or evict real blood splats if the same native collection were reused?
7. Can lifespan/fade be controlled independently for Happy Trails marks, or is decay globally tied to `BloodSplatLifespanDays` / native fade behavior?
8. Can marks be positioned continuously within a square and oriented/selected sufficiently to form convincing wheel tracks?
9. What is the render, save, load, and network cost of 100, 1,000, and several thousand native splats compared with equivalent `IsoObject` marks?
10. If direct reuse is too blood-specific, can the subsystem still inform a lighter Happy Trails representation that materializes visual decals only for loaded chunks?

This candidate is especially important because the engine already solves several problems Happy Trails otherwise has to solve separately: non-tile-replacing rendering, chunk-local storage, fading/age, save/load, and explicit multiplayer packet handling. None of those benefits should be assumed to generalize to custom marks until tested.

### E. Vegetation mutation

Identify representative vanilla grass, bush, sapling, and mature-tree objects.

Compare at least two approaches where feasible:

1. engine-assisted collision/damage using native tile properties or collision behavior;
2. explicit local spatial detection followed by server-authoritative mutation.

Determine whether vegetation can be classified and safely removed/replaced from the authoritative side.

Test debris separately. Measure persistent object/item proliferation before enabling debris by default.

### F. State storage

Compare bounded persistence strategies rather than assuming one:

- durable world tile/object state only;
- native lightweight decal/splat state if reusable;
- square/object `modData`;
- sparse server-side modified-square records;
- chunk-level or region-level aggregation if exposed and warranted.

Test:

- save/reload;
- server restart;
- late client join;
- state removal when wear returns to zero;
- approximate storage overhead for representative routes;
- whether cost grows with historical travel or only with currently modified terrain.

### G. Multiplayer authority

The server must own durable wear state and destructive mutations, but SPIKE-001 must determine the cheapest reliable way for movement evidence to reach the server.

Candidate flows include:

```text
server observes movement
-> server computes passage
-> server mutates world
-> native synchronization distributes result
```

and:

```text
client observes local movement
-> client batches compact passage candidates
-> server validates/coalesces
-> server updates durable wear only on meaningful change
-> native or explicit synchronization distributes result
```

The spike must compare, not assume.

## Performance test methodology

### Baseline

Record a mod-disabled or Happy-Trails-disabled baseline using the same route, vehicle, player count, and server configuration.

### Required counters

Prototype instrumentation should count at least:

- movement samples/events;
- squares generated from those samples;
- square-object enumerations;
- terrain classifications;
- cache hits/misses where applicable;
- wear events created versus coalesced/skipped;
- world mutations;
- native splat/decal entries if tested;
- active/persistent state entries;
- queue lengths/high-water marks;
- custom network messages and items per batch;
- cleanup/recovery operations.

### Stress dimensions

Tests should vary:

- driving speed;
- route duration/distance;
- repeated travel over the same route versus exploration into new terrain;
- simultaneous moving vehicles;
- dense vegetation versus open fields;
- one client versus multiple clients;
- high-fidelity versus reduced-fidelity settings once such controls exist.

The goal is to identify scaling behavior, not merely prove that a short solo test works.

## Prototype constraints

SPIKE-001 code should be diagnostic and deliberately narrow.

It should:

- operate on a small controlled test area initially;
- use conspicuous logging prefixes;
- avoid permanent broad map mutation;
- include a kill switch or `Enabled` guard;
- avoid spawning large numbers of items;
- avoid hidden background scans;
- bound all queues/tables used by the prototype;
- expose counters for work performed;
- record enough information to reproduce conclusions.

Suggested log prefix:

```text
[HappyTrailsSpike001]
```

## Test environment

Minimum tests:

1. local single-player Build 42.20+;
2. dedicated Build 42.20+ server;
3. two-player multiplayer test;
4. multiple simultaneous moving vehicles if practical;
5. late-join validation after a terrain mutation;
6. server restart and reconnect;
7. repeated-route versus long-distance exploration comparison.

## Evidence to capture

For every candidate API/path, record:

- Build version;
- Lua execution context (`client`, `server`, `shared`);
- exact method/event/property names used;
- representative logs/counters;
- observed persistence behavior;
- observed network behavior;
- failures/exceptions;
- CPU/frame/tick observations available from the test environment;
- object/state growth behavior;
- conclusion: viable, viable-with-caveats, or rejected.

## Success criteria

SPIKE-001 is successful if it identifies at least one credible implementation path that can demonstrate all of the following in a controlled test:

1. detect a moving vehicle crossing natural terrain;
2. identify at least one eligible terrain type and one ineligible paved type;
3. create a visible track/wear state on the eligible terrain;
4. persist that state through save/reload or server restart;
5. show the same state to a second or late-joining client;
6. demonstrate bounded processing/state growth during repeated travel;
7. compare the selected candidate against at least one plausible alternative, including the native splat/decal path if Lua access permits it;
8. identify a viable vegetation-damage path or explicitly split it into a later spike;
9. identify which performance/fidelity parameters meaningfully reduce cost.

## Go / no-go outcomes

### GO

Proceed to a functional v0.0.x prototype if a measured candidate implementation provides persistent/synchronized track wear with bounded scaling characteristics and acceptable gameplay responsiveness.

### CONDITIONAL GO

Proceed with a reduced MVP if core track wear is viable but vegetation damage, snow, wheel-level positioning, recovery, or debris requires separate investigation.

### NO-GO / REDESIGN

Redesign the concept if durable visual mutation cannot be synchronized/persisted cleanly, if every viable visual representation requires unacceptable object/state proliferation, or if movement detection cannot be bounded to an acceptable performance envelope.

## Deliverables

At completion:

- update this document with measured results;
- update `docs/DESIGN.md` only with validated constraints and selected options;
- update `docs/REQUIREMENTS.md` where feasibility changes scope or adds numeric budgets;
- update `CHANGELOG.md`;
- create ADR(s) only for decisions supported by the spike;
- create follow-on spikes only for unresolved questions that materially block implementation.
