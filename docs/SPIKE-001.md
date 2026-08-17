# SPIKE-001 — Build 42 Environmental Wear Feasibility

## Status

Planned.

## Question

Can Project Zomboid Build 42 support a performant, persistent, multiplayer-safe Happy Trails mechanic using Lua-accessible APIs and ordinary mod assets, without engine modification or brittle hacks?

## Why this spike exists

The desired gameplay mechanic combines four concerns that must all work together:

1. observing vehicle movement at sufficient spatial resolution;
2. classifying natural terrain and vegetation reliably;
3. mutating visual/world state persistently;
4. synchronizing those mutations correctly in multiplayer.

Implementation should not begin by assuming any one API path works. This spike should produce evidence for the architecture.

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
- event callbacks versus polling requirements.

Measure the practical event/sampling cadence needed to avoid gaps at normal and high driving speeds.

### B. Surface classification

Identify a robust way to distinguish:

- grass;
- dirt;
- forest/natural ground;
- paved road;
- building/interior floors;
- water or invalid terrain;
- snow-covered conditions where observable.

Document whether classification is based on floor sprite names, sprite properties, tile properties, zones, erosion data, or another API.

### C. Persistent visual mutation

Test at least the plausible mutation approaches available from Lua:

- replace/change floor;
- add/remove an `IsoObject`;
- attach or change an overlay/sprite;
- place a custom tile/object;
- use another supported world-decoration mechanism.

For each approach record:

- visual result;
- save/load persistence;
- dedicated-server restart persistence;
- MP synchronization;
- collision/pathing side effects;
- removability/reversibility;
- object-count implications.

### D. Vegetation mutation

Identify representative vanilla grass, bush, sapling, and mature-tree objects.

Determine whether they can be classified and safely removed/replaced from the authoritative side.

Test whether appropriate debris can be created without duplication or excessive persistent object/item cost.

### E. State storage

Validate whether square/object `modData` or another persistent mechanism can store a bounded wear value and last-update timestamp.

Test:

- save/reload;
- server restart;
- late client join;
- state removal when wear returns to zero;
- approximate storage overhead for a representative modified route.

### F. Multiplayer authority

Determine whether the server receives sufficient vehicle movement information to own the mechanic directly.

Preferred outcome:

```text
server observes vehicle
-> server computes passage
-> server mutates world
-> normal PZ synchronization distributes result
```

If this is not viable, identify the smallest client-reporting protocol needed and define how the server can validate it.

## Prototype constraints

SPIKE-001 code should be diagnostic and deliberately narrow.

It should:

- operate on a small controlled test area;
- use conspicuous logging prefixes;
- avoid permanent broad map mutation;
- include a kill switch or `Enabled` guard;
- avoid spawning large numbers of items;
- avoid hidden background scans;
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
4. late-join validation after a terrain mutation;
5. server restart and reconnect.

## Evidence to capture

For every candidate API/path, record:

- Build version;
- Lua execution context (`client`, `server`, `shared`);
- exact method/event names used;
- representative logs;
- observed persistence behavior;
- observed network behavior;
- failures/exceptions;
- performance observations;
- conclusion: viable, viable-with-caveats, or rejected.

## Success criteria

SPIKE-001 is successful if it identifies a credible implementation path that can demonstrate all of the following in a controlled test:

1. detect a moving vehicle crossing natural terrain;
2. identify at least one eligible terrain type and one ineligible paved type;
3. create a visible track/wear marker on the eligible terrain;
4. persist that marker through save/reload or server restart;
5. show the same marker to a second or late-joining client;
6. safely remove or alter at least one class of small vegetation, or provide clear evidence that vegetation damage should be split into a later spike;
7. do so without full-map scanning or an obviously unbounded per-tick workload.

## Go / no-go outcomes

### GO

Proceed to a functional v0.0.x prototype if the server can authoritatively observe movement and persist/synchronize a practical visual mutation.

### CONDITIONAL GO

Proceed with a reduced MVP if track wear is viable but vegetation damage, snow, wheel-level positioning, or recovery needs separate investigation.

### NO-GO / REDESIGN

Redesign the concept if durable visual mutation cannot be synchronized/persisted cleanly or if vehicle movement cannot be observed with acceptable performance using available mod APIs.

## Deliverables

At completion:

- update this document with results;
- update `docs/DESIGN.md` with validated architecture;
- update `docs/REQUIREMENTS.md` where feasibility changes scope;
- update `CHANGELOG.md`;
- create ADR(s) for durable design choices;
- create follow-on spikes only for unresolved questions that materially block implementation.
