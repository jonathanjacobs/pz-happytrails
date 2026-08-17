# Happy Trails Requirements

## 1. Purpose

Happy Trails is a Project Zomboid Build 42 mod intended to make repeated travel alter the landscape in persistent, visible, and mechanically coherent ways.

The core fantasy is that survivors create paths by using them. A route driven once should look different from a route driven repeatedly for weeks. Vehicles may also interact with small vegetation in a way that leaves evidence of passage.

This document is the canonical requirements specification. Implementation details may change as Build 42 feasibility work proceeds, but behavioral and performance requirements should only change deliberately and be reflected here and in the changelog.

## 2. Development status

Current version: `0.0.1`

Status: pre-alpha feasibility stage.

No gameplay behavior is considered implemented or validated until supported by explicit test evidence.

No production architecture has been selected. Reference mods are being reviewed only as comparative evidence for available Build 42 techniques, limitations, and performance failure modes.

## 3. Development principles

### 3.1 Performance is a first-class requirement

Happy Trails must be designed for long-running multiplayer servers and clients with modest hardware. Performance is not a post-MVP optimization task.

Before selecting an implementation architecture, feasibility work must compare plausible client/server approaches and measure their costs.

The project should prefer designs that:

- do work only when relevant movement occurs;
- avoid continuous broad-area scans;
- avoid one persistent world object for every historical tire impression unless measurements prove that approach acceptable;
- avoid unbounded actor, square, object, queue, or network state;
- batch or coalesce repeated events where doing so preserves gameplay semantics;
- perform durable visual mutations only when a wear threshold or other meaningful state change occurs;
- allow sampling cadence, visual fidelity, and optional effects to be reduced without corrupting the underlying world state;
- make server and client costs independently observable and tunable.

### 3.2 Requirements before architecture

The project should not commit to a specific polling loop, network protocol, persistence model, visual-object model, or module layout until SPIKE evidence supports that choice.

Candidate approaches should remain replaceable during pre-alpha development.

### 3.3 Extend the engine where possible

If Build 42 already exposes native collision, tile-property, world-object, persistence, or synchronization behavior that can safely perform part of the work, Happy Trails should evaluate using that behavior rather than reimplementing it in Lua.

Undocumented or incidental engine behavior must be validated before becoming a dependency.

### 3.4 Fail closed for destructive mutations

Unknown terrain or vegetation should remain unchanged rather than being modified by guesswork.

## 4. MVP scope

The MVP should establish a reliable, performant system for persistent vehicle-created travel wear on natural terrain in single-player and multiplayer.

### 4.1 Required for MVP

At minimum, the MVP should support:

1. detecting relevant vehicle travel over eligible terrain;
2. accumulating a bounded notion of repeated traffic;
3. visually distinguishing untouched, lightly traveled, and heavily traveled terrain with at least three wear states;
4. persisting those changes across save/load and dedicated-server restarts;
5. synchronizing persistent changes consistently in multiplayer;
6. avoiding mutation of paved roads, buildings, interiors, water, and other clearly ineligible surfaces;
7. avoiding runaway world-state growth, excessive object creation, or expensive broad scans;
8. exposing enough diagnostics to measure client, server, world-object, persistence, and network cost;
9. allowing performance-sensitive implementation parameters to be changed during testing without redefining wear semantics.

### 4.2 Candidate secondary features

These should be investigated but are not required to block the first viable track-wear MVP unless they prove inexpensive and clean:

- vegetation destruction;
- branch/twig debris;
- snow-specific tire tracks;
- mud/wet-ground appearance;
- recovery/regrowth;
- vehicle mass/class modifiers;
- speed-dependent wear;
- exact wheel-position rendering.

A feature may move into the MVP only after its cost and technical behavior are understood.

## 5. Progressive wear model

The intended player-facing behavior is progressive rather than binary.

A conceptual progression is:

```text
untouched terrain
-> compressed / flattened vegetation
-> visible wheel passage
-> established wheel ruts
-> heavily worn dirt track
```

The exact number of states, thresholds, representation, and visual assets are implementation decisions to be validated experimentally.

A single pass should not normally create a permanent road. Repeated traffic should be the primary driver of durable transformation.

The implementation must not require retention of an unbounded raw history of every vehicle pass.

## 6. Terrain sensitivity

The mechanic should distinguish between terrain classes where technically feasible.

Candidate classes include:

- grass;
- natural dirt;
- forest/natural ground;
- mud or wet soil;
- snow-covered natural terrain.

The system should not blindly replace every floor tile under a vehicle.

Eligible terrain must be identified using stable and maintainable criteria. Candidate evidence sources include tile properties, floor sprite metadata, zones, erosion data, or bounded classification tables.

Unknown terrain should default to no mutation.

## 7. Vehicle interaction

Vehicle travel is the primary MVP trigger.

The system may eventually account for:

- accumulated wear;
- vehicle mass or class;
- speed;
- direction of travel;
- wheel position or vehicle footprint;
- weather or surface state.

The first implementation should use the minimum set of inputs needed to create convincing, performant behavior.

The system must not assume that per-frame or per-tick vehicle processing is necessary. SPIKE-001 must compare event-driven, sampled, and hybrid approaches.

## 8. Vegetation damage

Vegetation damage is desirable but subordinate to the track-wear performance envelope.

Where Build 42 APIs support safe object mutation, vehicles may be capable of damaging small vegetation.

Candidate behavior:

```text
grass / very small vegetation -> flattened or removed
small bush -> crushed / removed
large bush -> damaged or removed
sapling -> broken / removed
mature tree -> not automatically destroyed by normal travel
```

The project should explicitly investigate whether native tile properties or engine vehicle-collision behavior can perform vegetation collision/damage more efficiently than Lua-side spatial scanning.

The system must not make mature trees trivial to destroy merely because a vehicle overlaps their square.

## 9. Debris

When vegetation is destroyed, the mod may create environmentally appropriate debris if doing so is technically reliable and does not cause excessive persistent object/item proliferation.

Candidate outputs include:

- twigs;
- branches;
- small logs;
- visual-only debris objects;
- no debris, where performance settings or server policy disable it.

Debris generation is optional until object-count and persistence costs are measured.

## 10. Recovery and persistence

Tracks and terrain wear should persist across normal save/load cycles and dedicated-server restarts.

Long-term recovery is desirable but must not require continuous processing of every modified square.

Candidate recovery behavior:

- lightly disturbed grass recovers relatively quickly;
- moderate wear recovers slowly;
- heavily established dirt tracks recover very slowly or effectively persist until abandoned for a long period;
- renewed traffic resets or increases the wear state.

Recovery should be evaluated lazily, from timestamps, from vanilla erosion behavior, or by another bounded mechanism. The specific persistence representation remains an open design choice until measured.

## 11. Weather interaction

Weather-dependent appearance is desirable but not mandatory for the first functional MVP.

Candidate behavior includes:

- snow tracks appearing readily and fading under snowfall or thaw;
- wet ground producing darker or muddier tracks;
- dry conditions producing less dramatic rutting;
- rain affecting recovery or appearance.

Weather mechanics must be driven by available Build 42 APIs and should not duplicate vanilla climate simulation unnecessarily.

Snow implementation must not inherit a high-frequency per-actor/per-footprint processing model merely because an existing footprint mod uses one.

## 12. Multiplayer requirements

Persistent world mutations must have a clearly defined authority model, but the mechanism by which movement reaches that authority remains open until measured.

Candidate approaches include:

- server directly observes active vehicles and computes passage;
- client reports compact movement/passage candidates and the server validates/coalesces them;
- hybrid approaches in which clients provide low-cost movement hints while the server owns durable state transitions.

Regardless of approach:

- the server must own durable wear state and destructive world mutations;
- duplicate reports must not multiply damage or wear incorrectly;
- late-joining clients must converge on existing modified terrain;
- network traffic should represent state changes or compact batches rather than every frame of motion;
- a lower-fidelity client must not change authoritative world outcomes.

Any client-authoritative durable mutation requires an explicit ADR and strong justification.

## 13. Performance and scalability requirements

### 13.1 Baseline-first measurement

SPIKE-001 must establish a vanilla/mod-disabled baseline and then measure candidate implementations against the same driving scenarios.

No hard CPU, memory, object-count, or network budget should be invented before baseline measurements are available. After measurements, explicit budgets should be added here.

### 13.2 Measurements required

At minimum, diagnostics should make it possible to observe or estimate:

- vehicle samples or movement events processed per second;
- grid squares considered per vehicle sample;
- square objects inspected;
- terrain classifications performed and cache hit rate if caching is used;
- wear events accepted, coalesced, or ignored;
- visual/world mutations performed;
- persistent Happy Trails state count;
- active queue sizes and high-water marks;
- client-to-server and server-to-client message counts/batch sizes where custom networking is used;
- cleanup/recovery work performed;
- client frame-time/FPS impact during representative driving;
- dedicated-server tick/CPU impact where observable.

### 13.3 Required performance properties

The final implementation must avoid:

- full-map scans;
- continuous scans of large areas around every vehicle;
- unbounded tables containing every visited natural square;
- permanent tracking of destroyed/nonexistent objects without cleanup;
- repeated enumeration of unchanged square contents where it can be avoided safely;
- FIFO implementations that become increasingly expensive with large queues;
- one network message for every insignificant positional update;
- cleanup algorithms whose cost grows with total world history rather than active/relevant state;
- uncontrolled persistent debris or visual-object growth.

### 13.4 Graceful fidelity reduction

The design should permit performance-oriented server or client settings without requiring a separate code path for every profile.

Candidate tunable dimensions include:

- vehicle sampling distance/cadence;
- minimum movement distance before reprocessing;
- track visual density;
- mutation batching;
- recovery/cleanup cadence;
- vegetation damage;
- debris generation;
- snow/weather embellishments.

A performance setting should reduce work, not merely hide graphics while expensive underlying processing continues.

## 14. Compatibility requirements

Initial development targets vanilla Project Zomboid Build 42.20 or later.

The mod should avoid hard dependencies on specific map, vehicle, weather, or vegetation mods.

Compatibility with custom vehicles and maps should be graceful where possible. Unknown terrain should default to no mutation rather than destructive guessing.

The mod should fail safely if expected APIs or object properties are unavailable.

## 15. Configuration

The first functional release should expose only settings that materially improve administration, compatibility, or performance.

Candidate sandbox settings include:

- enable/disable the system;
- wear-rate multiplier;
- recovery-rate multiplier;
- performance/fidelity controls validated during testing;
- vegetation destruction enable/disable;
- debris generation enable/disable;
- optional debug/performance telemetry.

The project should not expose speculative knobs before the underlying mechanic is stable.

## 16. Logging and diagnostics

Pre-alpha builds should include targeted diagnostic logging sufficient to prove:

- vehicle movement is observed at the expected cadence;
- eligible surface classification is correct;
- wear state changes occur only at thresholds;
- vegetation mutations occur once and on the authoritative side;
- persistence survives reload/restart;
- multiplayer clients converge correctly;
- processing and network work remain bounded during stress tests.

Diagnostics should be rate-limited and disabled by default in public releases.

## 17. Documentation requirements

Every non-trivial Lua function should document:

- purpose;
- expected inputs;
- return values;
- side effects;
- authority context where relevant (`client`, `server`, or shared);
- failure/fallback behavior.

Loops, branching logic, state transitions, synchronization decisions, and non-obvious API workarounds should include concise inline comments explaining why they exist.

Performance-sensitive loops or queues should document their expected complexity and bounding mechanism.

## 18. Third-party reference code and licensing

Reference mods may be inspected to understand Build 42 APIs, patterns, and known performance problems.

They are research inputs, not implementation dependencies.

Third-party source code must not be copied into Happy Trails unless its license is explicitly established as compatible with Apache License 2.0 and all required attribution/notice obligations are satisfied.

Where licensing is unclear, Happy Trails may independently implement a technique after validating the underlying Project Zomboid API behavior.

## 19. Explicitly out of scope for the initial MVP

Unless feasibility evidence makes them essentially free, the first MVP does not require:

- deformable 3D terrain geometry;
- physics-based mud depth;
- wheel-specific suspension simulation;
- vehicle getting stuck in ruts;
- NPC pathfinding preference for player-created roads;
- erosion-system replacement;
- pedestrian footpath formation;
- zombie-created trails;
- complex seasonal vegetation regrowth;
- custom sound effects;
- extensive compatibility patches for third-party maps and vehicles.

## 20. MVP acceptance criteria

A functional MVP is acceptable when a controlled test demonstrates all of the following:

1. a vehicle can repeatedly traverse an eligible natural route without errors;
2. the route progresses through at least three visually distinguishable wear states;
3. an adjacent unused route remains unchanged;
4. paved or explicitly ineligible terrain remains unchanged;
5. wear persists after saving and reloading;
6. wear persists after a dedicated-server restart;
7. two multiplayer clients observe the same resulting terrain state;
8. a late-joining client observes the existing terrain state;
9. repeated passage does not generate duplicate world objects or runaway network traffic;
10. processing cost remains bounded as the test duration and travel distance increase;
11. the chosen implementation has been benchmarked against at least one plausible alternative from SPIKE-001;
12. lower-fidelity/performance settings demonstrably reduce processing, object, or network cost where applicable;
13. normal vehicle movement remains responsive during representative multiplayer testing;
14. at least one supported vegetation class can be safely crushed/removed, or vegetation destruction is explicitly deferred based on performance/API evidence;
15. all experimentally discovered limitations are documented before `0.1.0`.
