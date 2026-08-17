# Happy Trails Design Space

## Status

Pre-alpha design-space document.

This document intentionally records **options, constraints, and questions**, not a committed architecture. Nothing here should be treated as validated engine behavior until confirmed by a spike or test result. Durable choices should be made only after comparative evidence and then captured in ADRs.

## Design objective

Represent repeated travel as persistent environmental change while keeping client, server, persistence, world-object, and network costs bounded.

The implementation should remain flexible enough to support higher-fidelity features later without requiring the first release to pay their runtime cost.

## Non-negotiable constraints

Any eventual design should:

- process only relevant movement rather than broad world areas;
- keep durable mutations server-owned;
- avoid unbounded historical state;
- allow work to be reduced by performance/fidelity settings;
- avoid coupling core wear semantics to a single rendering technique;
- keep terrain classification replaceable as Build 42 knowledge improves;
- allow vegetation destruction to be enabled, disabled, or deferred independently of track wear;
- fail closed for unknown/destructive cases;
- make performance-sensitive work measurable.

## Open dimension 1 — traffic representation

Several representations remain plausible.

### Option A — bounded per-square wear score

A modified square stores a bounded value and possibly a last-update timestamp.

Conceptually:

```text
Wear = clamp(0, MaxWear, Wear + PassageWeight)
```

Advantages:

- natural progressive wear;
- threshold-based visual mutation;
- repeated traffic can be coalesced;
- raw pass history need not be retained.

Risks/questions:

- persistence overhead per modified square;
- synchronization behavior of metadata;
- recovery bookkeeping.

### Option B — discrete state only

Squares store only their current wear tier and enough information to know when a transition is allowed.

Advantages:

- smaller durable state;
- potentially simpler persistence.

Risks/questions:

- may require additional transient counters;
- less granular tuning/recovery.

### Option C — aggregated route/chunk state

Traffic is aggregated at a coarser granularity and materialized into square-level visuals only when needed.

Advantages:

- potentially lower persistence overhead on very long routes.

Risks/questions:

- may lose convincing local track geometry;
- depends on available chunk/region persistence hooks;
- more complex reconstruction.

No option is selected yet.

## Open dimension 2 — vehicle observation

Candidate approaches:

### A. Server-side active-vehicle sampling

The server periodically samples only moving/relevant vehicles and converts traveled distance into candidate affected squares.

Potential strengths:

- simple authority model;
- little or no custom client reporting.

Potential costs:

- server-side sampling load scales with moving vehicles;
- cadence must avoid gaps at high speed.

### B. Client passage reporting with server validation

The local driving client derives compact passage candidates and sends them in batches. The server validates and owns durable state transitions.

Potential strengths:

- distributes movement geometry work to clients;
- can naturally follow local vehicle updates.

Potential costs:

- additional network protocol;
- validation must be strong enough to prevent duplicate or implausible mutations;
- server validation must not duplicate all client work.

### C. Native/event-assisted hybrid

Use Build 42 collision/events/properties for what the engine already knows, and sample only what remains necessary for trail wear.

Potential strengths:

- potentially lowest Lua-side work;
- may provide more accurate collision behavior.

Potential costs:

- undocumented/native behavior may be difficult to generalize;
- event coverage may not provide continuous wheel-path information.

SPIKE-001 should benchmark these approaches before an ADR selects one.

## Open dimension 3 — affected vehicle footprint

Possible fidelity levels include:

1. center-square sampling;
2. centerline plus width approximation;
3. oriented vehicle bounding footprint;
4. approximate left/right wheel paths;
5. actual wheel positions if stable APIs expose them.

A lower-cost representation may be adequate for wear accumulation while higher-fidelity visuals are generated only at thresholds.

The state model should not depend on exact wheel coordinates unless testing shows they are both inexpensive and reliable.

## Open dimension 4 — terrain classification

Possible evidence sources include:

- floor sprite names;
- tile/sprite properties;
- zones;
- erosion/natural-floor metadata;
- bounded allow/deny tables;
- combinations of the above.

Classification should be centralized behind a replaceable interface. Unknown terrain should produce no destructive mutation.

Performance questions:

- which properties are cheap to read repeatedly;
- which results are immutable and safe to cache;
- whether classification can be skipped entirely until a wear threshold is likely to change.

## Open dimension 5 — visual representation

Candidate approaches:

### A. One additional world object per modified square/state

Potential strengths:

- straightforward custom sprites;
- independent of base floor replacement.

Potential costs:

- object-count growth;
- add/remove/transmit overhead;
- cleanup complexity;
- late-join synchronization volume.

This is specifically a risk identified from reviewing the Footprints reference implementation and must be benchmarked rather than assumed.

### B. Floor/tile replacement

Potential strengths:

- may encode state directly in the world;
- potentially fewer auxiliary objects.

Potential costs:

- could interfere with base terrain identity, erosion, maps, or third-party tiles;
- reversibility must be proven.

### C. Native floor-splat/decal path

Project Zomboid has a specialized floor-blood subsystem rather than representing every blood mark as an ordinary world object. Official API documentation exposes chunk-local bounded floor-blood collections, a compact `IsoFloorBloodSplat` representation containing coordinates/type/world age, explicit save/load behavior, and dedicated `BloodSplatter` / `RemoveBlood` network packet types.

Potential strengths if the mechanism is reusable or extensible:

- preserves the underlying terrain rather than replacing it;
- continuous/sub-square positioning may produce more organic tire marks;
- uses a rendering path already designed for many environmental splats;
- chunk-local bounded storage may be cheaper than ordinary `IsoObject` lifecycle management;
- native age/fade, persistence, and MP behavior already exist for blood.

Major unknowns:

- whether Lua can create/control these splats safely in Build 42;
- whether custom tire-track sprites can be registered or whether the system is hard-coded to blood types;
- whether custom marks would share caps/eviction with real blood and therefore interfere with gore;
- whether lifespan/fade can be separated from the server-wide blood-splat lifespan;
- whether orientation and sprite selection are adequate for coherent wheel tracks;
- whether the network packet format supports custom visual types;
- whether direct use would create compatibility risk with engine updates.

This option is now a priority SPIKE-001 experiment, but it is not a selected architecture.

### D. Generic overlay/decal-like representation

If the blood-specific path cannot support custom assets cleanly, other overlay or attached-sprite mechanisms may still preserve the base floor.

Potential strengths:

- may preserve underlying terrain;
- may support custom tire-track assets without sharing blood limits.

Potential costs:

- persistence, synchronization, and object-count behavior remain unknown until tested.

### E. Hybrid state-to-visual materialization

Durable wear state is stored separately; nearby/loaded squares materialize only the visuals needed for the current state.

Potential strengths:

- could decouple world history from rendered-object count;
- could potentially use a lightweight native decal path for visible chunks while retaining compact authoritative wear state separately.

Potential costs:

- chunk load/unload lifecycle complexity;
- late-join and cleanup behavior must be carefully validated.

No rendering approach is selected yet.

## Open dimension 6 — persistence

Candidate strategies include:

- encode state directly in durable world tile/object mutation;
- native splat/decal state if reusable without interfering with vanilla blood;
- square/object `modData`;
- sparse server-side state keyed only to modified squares;
- chunk/region aggregation if available and justified.

Selection criteria:

- save/restart reliability;
- late-join convergence;
- storage growth per kilometer of traveled route;
- cleanup/recovery cost;
- compatibility with map and erosion systems;
- independence from unrelated vanilla systems such as gore limits/lifespan where required.

There is no preferred ordering until SPIKE measurements exist.

## Open dimension 7 — vegetation damage

The reference implementations reveal at least two distinct approaches worth testing.

### A. Native collision/property assisted

More Damaged Objects sets sprite properties such as `HitByCar` and `MinimumCarSpeedDmg`. If Build 42's native vehicle collision system can safely handle relevant vegetation classes, Happy Trails may be able to avoid continuous Lua spatial scans for those objects.

This must be independently validated.

The official API also exposes `IsoGridSquare.bFlattenGrassEtc`. Its exact semantics and Lua accessibility are undocumented in the public JavaDocs, but the field is sufficiently relevant to warrant a narrow experiment before implementing custom grass-flattening logic.

### B. Explicit local spatial detection

Vehicle Vegetation Destruction samples squares near/in front of a moving vehicle, classifies objects, and requests server-side destruction.

This proves a local-scan approach is possible, but Happy Trails should test tighter geometry, better state cleanup, and lower object-enumeration cost rather than inheriting that implementation.

### C. Hybrid

Native collision handles destructible objects; Happy Trails observes only enough state to add optional aftermath such as debris or a wear increment.

Vegetation damage must remain separable from the core trail system.

## Open dimension 8 — debris

Options include:

- real world inventory items;
- decorative objects;
- temporary/local-only visual debris where appropriate;
- no debris in performance-focused configurations.

Debris should not be enabled by default until persistent-object/item cost is measured.

## Open dimension 9 — recovery

Potential approaches:

- lazy timestamp evaluation when a modified square is touched;
- recovery during chunk load;
- integration with vanilla erosion behavior if possible;
- native splat aging/fading if it can be controlled independently and matches intended semantics;
- no recovery for early MVP while persistence and performance are stabilized.

Continuous whole-world recovery scans are excluded.

## Performance control dimensions

The eventual implementation should allow meaningful reduction of actual work through some combination of:

- movement sampling cadence;
- distance threshold before new geometry is processed;
- affected-square density;
- wear-event coalescing;
- threshold-only visual mutation;
- mutation batching;
- sparse-state cleanup;
- optional vegetation damage;
- optional debris;
- optional weather/snow embellishments;
- reduced client visual density if visual state can be safely decoupled from authoritative wear.

A control that only hides visuals without reducing expensive detection/network/state work is not considered a performance optimization.

## Reference-code and native-engine lessons, not architectural decisions

The supplied comparison mods and official API documentation reveal useful techniques but also potential traps:

- Footprints: batching, caps, deferred visibility and tagged world objects are useful; actor scanning, LOS work, per-print object lifecycle, synchronization, and cleanup create significant complexity.
- Vehicle Vegetation Destruction: local bounded scanning and server validation are useful; repeated square-object enumeration, coarse footprint geometry, and long-lived processed-square metadata should be improved or avoided.
- More Damaged Objects: native `HitByCar`/`MinimumCarSpeedDmg` properties are promising; using a magic world-sound signature followed by a 5x5 object scan is too indirect to adopt without strong evidence.
- Native blood rendering: PZ already has a bounded chunk-level environmental-splat system with persistence and dedicated networking. It may be reusable, extensible, or simply instructive; the spike must establish which.

See [`REFERENCE-IMPLEMENTATIONS.md`](REFERENCE-IMPLEMENTATIONS.md) and [`SPIKE-001.md`](SPIKE-001.md).

## Failure behavior

The mod should fail closed for destructive world mutations.

If a surface or vegetation object cannot be confidently classified, Happy Trails should leave it unchanged and optionally emit a rate-limited diagnostic.

If a mutation API fails, the current event should be abandoned without cascading changes to neighboring squares.

## When architecture may be selected

An architecture should not be described as preferred or final until SPIKE-001 produces comparative evidence on:

- movement-observation cost;
- visual representation cost, including the native splat/decal candidate where accessible;
- persistence behavior;
- MP synchronization;
- scaling with distance and simultaneous vehicles;
- object/state growth.

At that point, each durable choice should be captured in an ADR with the alternatives considered and the measurements supporting the decision.
