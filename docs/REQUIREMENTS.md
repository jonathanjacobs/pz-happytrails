# Happy Trails Requirements

## 1. Purpose

Happy Trails is a Project Zomboid Build 42 mod intended to make repeated travel alter the landscape in persistent, visible, and mechanically coherent ways.

The core behavior is that survivors create paths by using them: a route driven once should look different from a route driven repeatedly for weeks.

Current version: `0.0.1`  
Status: **pre-alpha feasibility stage**

No gameplay behavior is considered implemented or validated until supported by explicit test evidence. No production rendering, persistence, networking, or sampling architecture has been selected.

## 2. Development principles

### R1 — Performance is a first-class requirement

Happy Trails must be suitable for long-running multiplayer servers. The implementation must:

- do work only when relevant movement/state is active;
- avoid broad/whole-world scans;
- avoid unbounded historical state;
- avoid one permanent ordinary world object for every historical tire impression unless measurements prove that representation acceptable;
- batch/coalesce redundant work where semantics permit;
- expose enough instrumentation to measure client, server, state, object, and network cost;
- allow fidelity controls to reduce actual processing/state rather than merely hide graphics.

### R2 — Requirements before architecture

SPIKE evidence must precede durable architecture selection. A promising decompiled/internal engine mechanism is not assumed to be a supported production API until runtime behavior is validated.

### R3 — Durable state is server-authoritative

Persistent terrain wear and destructive world mutations must be owned authoritatively by the server in multiplayer.

### R4 — Fail closed for destructive mutations

Unknown or ambiguous terrain/vegetation remains unchanged rather than being modified by guesswork.

## 3. Required two-layer world model

Happy Trails distinguishes **transient marks** from **persistent terrain wear**.

### R5 — Transient marks

Fresh passage should be capable of producing temporary high-resolution marks such as:

- tire impressions;
- compressed/flattened vegetation;
- mud tracks;
- snow tracks;
- carried mud/snow/dust deposited onto hard surfaces.

Transient marks should support, where technically feasible:

- sub-tile placement;
- travel-direction/orientation representation;
- opacity/tint/variant changes with age/material;
- bounded per-chunk/region retention;
- age-based fading/expiration;
- cleanup without whole-world scans.

A transient mark must not imply a permanent terrain mutation by itself.

### R6 — Persistent wear

Repeated traffic should accumulate a sparse durable terrain state representing the current condition of changed ground rather than retaining raw history of every pass.

Conceptual progression:

```text
UNDISTURBED
-> IMPRINT
-> WORN
-> ESTABLISHED
-> DEEP_RUT
```

The MVP may implement fewer visual tiers initially, but persistent architecture must allow richer progression later.

### R7 — Persistent state survives transient-mark expiration

Once a route crosses the established-wear threshold, expiration/fading of temporary tire/material marks must not restore the terrain to untouched state.

## 4. Vanilla architectural precedent

### R8 — Use floor-blood behavior as a design precedent, not a production dependency

Build 42.20.3 `IsoFloorBloodSplat` demonstrates a PZ-native pattern of compact sub-tile marks, bounded per-chunk storage, chunk serialization, creation-world-age timestamps, and lazy visual aging.

Happy Trails should evaluate analogous principles for transient marks.

The mod must not depend on hijacking/modifying vanilla blood sprite tables, blood assets, or blood-specific renderer behavior as its production design.

## 5. Vehicle observation and path geometry

### R9 — Vehicle travel is the primary MVP trigger

SPIKE-001 must determine whether server-visible replicated vehicle state is sufficient to observe moving vehicles with bounded work.

### R10 — Continuous paths must not contain speed-dependent gaps

If server/client transform updates are farther apart than desired mark spacing, Happy Trails must interpolate/rasterize between previous/current positions rather than processing only the current square.

### R11 — Wheel geometry should be supported without coupling semantics to it

The implementation should be capable of using representative left/right wheel paths or actual scripted wheel geometry where this proves useful and performant. Persistent wear semantics must remain independent from the chosen visual/path fidelity.

### R12 — Continuous mark deposition must be spatially bounded

SPIKE-001 must trace the corpse-drag/floor-blood emission path as a reference for how vanilla deposits continuous environmental marks. Tire-track emission should use a bounded spatial cadence rather than creating marks every rendering frame.

## 6. Terrain behavior

### R13 — Natural/deformable surfaces may accumulate persistent wear

Candidate eligible classes include:

- grass;
- natural dirt;
- forest/natural ground;
- mud/wet soil;
- snow-covered natural terrain.

### R14 — Hard surfaces do not accumulate persistent deformation

Paved roads, concrete, building interiors, and other non-deformable surfaces must not be permanently converted into rutted natural terrain by ordinary Happy Trails traffic.

Hard surfaces may receive temporary carried-material marks such as mud or snow when that feature is enabled.

### R15 — Unknown terrain fails closed

Unknown/modded terrain defaults to no destructive persistent mutation until classified safely.

## 7. Progressive wear

### R16 — A single pass should not normally create a permanent road

Repeated traffic is the primary driver of durable transformation.

### R17 — Wear is bounded

Wear values/tier counters must have defined bounds. Raw pass history must not be retained indefinitely.

### R18 — Repeated sampling of one physical passage must not inflate wear

Sampling/interpolation duplicates must be coalesced so one vehicle traversal cannot accidentally count many times simply because of callback/network cadence.

## 8. Persistence, recovery, and hysteresis

### R19 — Persistent wear survives save/reload and dedicated-server restart

Established state must persist across ordinary world lifecycle operations.

### R20 — Recovery must be bounded/lazy

Recovery must not require continuous processing of every modified square. Candidate strategies include elapsed-time evaluation when a square/chunk is revisited or loaded.

### R21 — Established wear uses hysteresis

Recovery thresholds/times should differ from wear-creation thresholds so frequently used routes do not oscillate rapidly between states.

Expected pattern:

- light disturbance recovers relatively quickly;
- worn terrain recovers slowly;
- established/deep ruts recover very slowly after prolonged abandonment.

### R22 — Weather presentation is distinct from underlying state

Snow, rain, mud, thaw, or regrowth may obscure/alter a trail's appearance without accidentally erasing the semantic persistent wear state.

## 9. Vegetation damage

### R23 — Vegetation damage is desirable but separable

Vehicle interaction may flatten/remove small vegetation and saplings, while mature trees should not become trivial to destroy through ordinary overlap.

### R24 — Native collision is tested before custom broad scans

SPIKE-001 must evaluate native vehicle/object collision behavior (`HitByCar`, minimum damage thresholds, damaged sprites/removal/sync where applicable) before implementing a custom Lua neighborhood scanner.

### R25 — Debris is optional and bounded

Branches/twigs/logs or decorative debris may be added only if persistent object/item proliferation remains acceptable. Debris can be disabled entirely without affecting core wear semantics.

## 10. Multiplayer requirements

### R26 — Server owns persistent wear/destruction

Clients may assist with presentation or compact movement hints only if explicitly justified; they must not independently own durable world-state outcomes.

### R27 — Duplicate network/movement reports cannot multiply wear

Any custom network path must be deduplicated/coalesced.

### R28 — Late join converges

A late-joining client must see current persistent terrain state without manual repair/reload.

### R29 — Network traffic is state-oriented/batched

Happy Trails must not emit one custom network message for every insignificant positional update.

## 11. Performance/scalability requirements

### R30 — Baseline-first measurement

SPIKE-001 must establish Happy-Trails-disabled baselines and compare candidates using the same route, vehicles, clients, and server configuration.

### R31 — Required counters

Development diagnostics should make it possible to measure or estimate:

- vehicles examined/rejected;
- meaningful movement samples;
- wheel transforms/interpolated segments;
- transient marks emitted/expired/high-water mark;
- candidate squares generated/coalesced;
- terrain classifications;
- persistent wear reads/writes/high-water mark;
- visual/world mutations;
- custom network messages/bytes if any;
- cleanup/recovery work;
- client frame-time/FPS and server tick/CPU impact where observable.

### R32 — All transient storage has an explicit bound

Every queue/table/cache retaining transient marks must have an expiration/eviction rule and measurable high-water behavior.

The vanilla floor-blood queue demonstrates the principle; Happy Trails does not have to use the same capacity.

### R33 — Persistent state tracks meaningful changed terrain, not world history

State growth should scale with currently relevant altered terrain, not every natural square ever crossed by a vehicle.

## 12. Configuration

The first functional release should expose only settings that materially improve administration/performance, such as:

- enabled/disabled;
- wear-rate multiplier;
- recovery-rate multiplier;
- validated fidelity/performance controls;
- vegetation destruction enable/disable;
- debris enable/disable;
- diagnostics enable/disable.

Speculative knobs should not be added before the underlying mechanic is stable.

## 13. Documentation/code-quality requirements

Every non-trivial Lua function should document:

- purpose;
- inputs;
- return values;
- side effects;
- authority context (`client`, `server`, shared);
- failure/fallback behavior.

Performance-sensitive loops/queues should document their complexity and bounding mechanism. Non-obvious engine/API workarounds require concise explanatory comments.

## 14. Third-party/code provenance requirements

Reference mods and decompiled PZ source may be used as research evidence only. They must not be copied into Happy Trails unless licensing/permission explicitly allows it and all attribution/notice obligations are satisfied.

The repository must continue to comply with the Project Zomboid modding policy documented in [`PZ_MODDING_POLICY.md`](PZ_MODDING_POLICY.md) and the root compliance/third-party files.

## 15. MVP acceptance criteria

A functional MVP is acceptable when controlled testing demonstrates all of the following:

1. vehicle travel over eligible terrain is observed without errors;
2. wheel/path sampling remains continuous at representative driving speeds;
3. a single pass can create a convincing transient mark without permanently converting the route;
4. transient mark count/lifetime is explicitly bounded;
5. repeated traffic progresses through at least three visually distinguishable wear states;
6. established persistent wear remains after transient marks fade;
7. an adjacent unused route remains unchanged;
8. paved/hard terrain remains free of persistent deformation;
9. persistent wear survives save/reload and dedicated-server restart;
10. two multiplayer clients observe the same persistent state;
11. a late-joining client observes existing persistent state;
12. repeated passage does not generate duplicate/unbounded ordinary world objects;
13. processing/state/network cost remains bounded as duration/travel distance increase;
14. the selected rendering/persistence approach has been compared against at least one plausible alternative;
15. performance/fidelity settings demonstrably reduce actual work/state where applicable;
16. at least one vegetation class can be safely handled through a validated path, or vegetation is explicitly deferred;
17. all experimentally discovered limitations are documented before `0.1.0`.

## 16. Explicitly out of scope for the initial MVP

Unless evidence makes them essentially free, the first MVP does not require:

- deformable 3D terrain geometry;
- physics-based mud depth;
- vehicles physically becoming stuck in ruts;
- NPC pathfinding preference for player-created roads;
- erosion-system replacement;
- pedestrian/zombie-created trails;
- extensive compatibility patches;
- complex seasonal vegetation simulation.
