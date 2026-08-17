# Happy Trails Design

## Status

Working design for pre-alpha feasibility work. Nothing in this document should be treated as validated engine behavior until confirmed by a spike or test result.

## Design objective

Represent repeated travel as a bounded, persistent environmental state transition rather than as continuous terrain deformation.

The preferred architecture should be:

- local to recently traveled squares;
- server-authoritative for durable world changes;
- lazy rather than scan-based;
- tolerant of unknown terrain and third-party content;
- visually expressive without requiring custom engine modifications.

## Candidate state model

A candidate per-square wear model is:

```text
0 = untouched
1 = disturbed / flattened
2 = visible track
3 = established rut
4 = heavily worn path
```

The actual state count may change after visual and API feasibility is known.

A square should only carry explicit Happy Trails state if vanilla world state cannot encode the same result directly.

## Candidate wear accumulator

Instead of storing raw pass counts indefinitely, use a bounded score:

```text
WearScore = clamp(0, MaxWear, WearScore + PassageWeight)
```

Possible passage inputs:

- base vehicle passage weight;
- vehicle mass/class modifier;
- speed modifier;
- wet/snow surface modifier;
- repeat-pass threshold hysteresis.

State transitions occur only when the wear score crosses configured thresholds.

## Recovery model

Recovery should be evaluated lazily when a modified square is revisited, loaded, or otherwise touched by the system.

Candidate formulation:

```text
RecoveredWear = max(0,
    StoredWear - RecoveryRate * ElapsedWorldTime)
```

This avoids continuous global processing.

If terrain visuals themselves can encode durable state without custom metadata, recovery may instead use object age/timestamp data stored only for squares that can recover.

## Vehicle sampling

The first spike should determine the least expensive reliable trigger for vehicle movement.

Candidate approaches:

1. vehicle update/event callback;
2. player update while seated in a moving vehicle;
3. periodic server sampling of active vehicles only;
4. client-side movement reporting with server validation, only if necessary.

The design should not process every vehicle every game tick unless measurements show that this is safe and unavoidable.

A distance threshold should prevent repeated processing while a vehicle remains on the same square.

## Track footprint

The MVP may begin with a coarse footprint:

```text
vehicle center square + direction-derived adjacent wheel path squares
```

A later implementation may derive actual wheel positions if accessible through stable APIs.

Visual plausibility is more important than physically exact tire geometry for the first release.

## Terrain classification

Terrain should be classified through a dedicated adapter rather than scattered sprite-name checks throughout the code.

Conceptual interface:

```lua
classifySurface(square) -> surfaceClass | nil
```

Possible classes:

```text
grass
dirt
forest
mud
snow
ineligible
unknown
```

Unknown defaults to no mutation.

## Visual mutation strategy

SPIKE-001 should compare these approaches:

- changing floor tiles;
- adding/removing world objects;
- overlay sprites;
- custom tiles/sprites;
- supported decal-like objects if available.

Selection criteria:

- persists correctly;
- synchronizes in MP;
- does not break map collision/pathing;
- looks acceptable from all relevant orientations;
- can be reversed or advanced safely;
- does not create excessive world objects.

## Vegetation damage adapter

Vegetation handling should be isolated from track wear.

Conceptual interface:

```lua
classifyVegetation(object) -> vegetationClass | nil
applyVehicleDamage(object, vehicleContext) -> result
```

Candidate result values:

```text
none
flattened
removed
destroyed-with-debris
blocked-by-large-object
```

This allows the MVP to ship track wear even if vegetation mutation proves unreliable.

## Persistence

Preferred order of persistence strategies:

1. encode durable state directly in vanilla world tile/object state;
2. use square/object modData where safely persisted and synchronized;
3. use bounded server-side tables keyed only to modified squares if required.

Avoid a global history database unless testing proves simpler mechanisms inadequate.

## Multiplayer authority

Durable mutation should occur on the server.

A likely flow is:

```text
vehicle movement observed
-> server identifies affected squares
-> server classifies terrain
-> server updates wear state
-> threshold crossing mutates world object/tile
-> normal PZ object/world synchronization propagates state
```

If the server lacks sufficient high-frequency vehicle information, a client may report a candidate passage event, but the server must validate and own the resulting mutation.

## Performance controls

Candidate safeguards:

- process only moving occupied vehicles or active network vehicles;
- skip if the vehicle remains within the same sampled tile footprint;
- cache immutable terrain classification where safe;
- mutate visual state only on threshold changes;
- use sparse modified-square metadata;
- use lazy recovery;
- aggregate or suppress repetitive debug logs;
- avoid debris item creation by default until cost is understood.

## Module boundaries

A likely implementation layout is:

```text
42/media/lua/shared/HappyTrails/
    Constants.lua
    SurfaceClassifier.lua
    WearModel.lua

42/media/lua/server/HappyTrails/
    VehicleTracker_Server.lua
    TerrainWear_Server.lua
    VegetationDamage_Server.lua
    Persistence_Server.lua

42/media/lua/client/HappyTrails/
    DebugDisplay_Client.lua   # development only, if needed
```

This is provisional. Spike evidence may simplify or change it.

## Failure behavior

The mod should fail closed for destructive world mutations.

If a surface or vegetation object cannot be confidently classified, Happy Trails should leave it unchanged and optionally emit a rate-limited debug diagnostic.

If a mutation API throws an error, the current event should be abandoned without cascading changes to neighboring squares.

## Architectural decisions requiring ADRs

Create an ADR when evidence supports a durable choice about:

- world-state representation;
- visual mutation method;
- MP authority/synchronization path;
- recovery model;
- terrain classification strategy;
- third-party asset dependency or custom sprite pipeline.
