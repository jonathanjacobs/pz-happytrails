# Happy Trails Requirements

## 1. Purpose

Happy Trails is a Project Zomboid Build 42 mod intended to make repeated travel alter the landscape in persistent, visible, and mechanically coherent ways.

The core fantasy is that survivors create paths by using them. A route driven once should look different from a route driven repeatedly for weeks. Vehicles should also interact with small vegetation in a way that leaves evidence of passage.

This document is the canonical requirements specification. Implementation details may change as Build 42 feasibility work proceeds, but behavioral requirements should only change deliberately and be reflected here and in the changelog.

## 2. Development status

Current version: `0.0.1`

Status: pre-alpha feasibility stage.

No gameplay behavior is considered implemented or validated until supported by explicit test evidence.

## 3. MVP goals

The MVP should establish a reliable, performant system for persistent travel wear on natural terrain in single-player and multiplayer.

At minimum, the MVP should support:

1. detecting vehicle travel over eligible terrain;
2. accumulating a lightweight notion of repeated traffic;
3. visually distinguishing untouched, lightly traveled, and heavily traveled terrain;
4. persisting those changes across save/load and dedicated-server restarts;
5. synchronizing persistent changes consistently in multiplayer;
6. avoiding mutation of paved roads, buildings, interiors, water, and other clearly ineligible surfaces;
7. avoiding runaway world-state growth or expensive full-map scans;
8. providing deterministic restoration or degradation behavior where route recovery is enabled.

## 4. Progressive wear model

The intended player-facing behavior is progressive rather than binary.

A conceptual progression is:

```text
untouched terrain
-> compressed / flattened vegetation
-> visible wheel passage
-> established wheel ruts
-> heavily worn dirt track
```

The exact number of states, thresholds, and visual assets are implementation decisions to be validated experimentally.

A single pass should not normally create a permanent road. Repeated traffic should be the primary driver of durable transformation.

## 5. Terrain sensitivity

The mechanic should distinguish between terrain classes where technically feasible.

Candidate classes include:

- grass;
- natural dirt;
- mud or wet soil;
- snow-covered natural terrain;
- forest-floor or similar non-paved surfaces.

The system should not blindly replace every floor tile under a vehicle.

Eligible terrain must be identified using stable and maintainable criteria, ideally existing tile/object properties or a bounded allowlist/denylist abstraction.

## 6. Vehicle interaction

Vehicle travel is the primary MVP trigger.

The system should, where APIs permit, account for factors such as:

- repeated passage count or accumulated wear;
- vehicle mass or class;
- speed;
- direction of travel;
- wheel position or vehicle footprint;
- weather or surface state.

Not all factors are required for the first functional implementation. The minimum viable model may begin with simple repeated passage and later incorporate weight or speed.

## 7. Vegetation damage

Where Build 42 APIs support safe object mutation, vehicles should be capable of damaging small vegetation.

Candidate behavior:

```text
grass / very small vegetation -> flattened or removed
small bush -> crushed / removed
large bush -> damaged or removed
sapling -> broken / removed
mature tree -> not automatically destroyed by normal travel
```

The exact object classifications must be validated against vanilla Build 42 objects.

The system must not make mature trees trivial to destroy merely because a vehicle overlaps their square.

## 8. Debris

When vegetation is destroyed, the mod may create environmentally appropriate debris if doing so is technically reliable and does not cause excessive item/object proliferation.

Candidate outputs include:

- twigs;
- branches;
- small logs;
- visual-only debris objects.

Debris generation is secondary to reliable vegetation destruction and may be deferred beyond the first MVP if persistence or performance costs are unacceptable.

## 9. Recovery and persistence

Tracks and terrain wear should persist across normal save/load cycles and dedicated-server restarts.

Long-term recovery is desirable but must not require expensive whole-map processing.

Potential recovery behavior:

- lightly disturbed grass recovers relatively quickly;
- moderate wear recovers slowly;
- heavily established dirt tracks recover very slowly or effectively persist until abandoned for a long period;
- renewed traffic resets or increases the wear state.

Recovery should be based on lazy evaluation, timestamps, world age, erosion callbacks, or another bounded mechanism rather than scanning every modified square continuously.

## 10. Weather interaction

Weather-dependent appearance is desirable but not mandatory for the first functional MVP.

Candidate behavior includes:

- snow tracks appearing readily and fading under snowfall or thaw;
- wet ground producing darker or muddier tracks;
- dry conditions producing less dramatic rutting;
- rain affecting recovery or appearance.

Weather mechanics must be driven by available Build 42 APIs and should not duplicate vanilla climate simulation unnecessarily.

## 11. Multiplayer requirements

Persistent world mutations must have a clearly defined authority model.

Preferred model:

- the dedicated server decides durable wear state and vegetation destruction;
- clients may detect local movement for presentation or event reporting only if necessary;
- clients must converge on the authoritative world state;
- duplicate events from multiple clients must not multiply damage or wear incorrectly;
- late-joining clients must see existing modified terrain correctly.

Any client-authoritative shortcut must be justified by evidence and documented as an architectural decision.

## 12. Performance requirements

Happy Trails must remain suitable for long-running multiplayer servers.

The implementation must avoid:

- per-tick scanning of large map regions;
- tracking every historical vehicle position forever;
- unbounded tables containing every natural square ever visited;
- excessive network messages for unchanged state;
- spawning large quantities of persistent debris items;
- repeated expensive sprite/object enumeration where results can be cached safely.

Processing should be event-driven or movement-local whenever possible.

Modified-square state should be stored only when necessary, and state that can be represented directly by world objects/tiles should not be duplicated without reason.

## 13. Compatibility requirements

Initial development targets vanilla Project Zomboid Build 42.20 or later.

The mod should avoid hard dependencies on specific map, vehicle, weather, or vegetation mods.

Compatibility with custom vehicles and maps should be graceful where possible. Unknown terrain should default to no mutation rather than destructive guessing.

The mod should fail safely if expected APIs or object properties are unavailable.

## 14. Configuration

The first functional release should expose only settings that materially improve administration or compatibility.

Candidate sandbox settings include:

- enable/disable the system;
- wear-rate multiplier;
- recovery-rate multiplier;
- vegetation destruction enable/disable;
- debris generation enable/disable;
- optional debug logging.

The project should not expose speculative knobs before the underlying mechanic is stable.

## 15. Logging and diagnostics

Pre-alpha builds should include targeted diagnostic logging sufficient to prove:

- vehicle movement events are observed at the expected cadence;
- eligible surface classification is correct;
- wear state changes occur only at thresholds;
- vegetation mutations occur once and on the authoritative side;
- persistence survives reload/restart;
- multiplayer clients converge correctly.

Diagnostic logging should be removable or disabled by default in public releases.

## 16. Documentation requirements

Every non-trivial Lua function should document:

- purpose;
- expected inputs;
- return values;
- side effects;
- authority context where relevant (`client`, `server`, or shared);
- failure/fallback behavior.

Loops, branching logic, state transitions, synchronization decisions, and non-obvious API workarounds should include concise inline comments explaining why they exist.

Documentation should be sufficient for another developer to audit the logic without reverse-engineering intent from the code.

## 17. Licensing and attribution

The project is licensed under Apache License 2.0.

The repository and distributed source should retain:

- `LICENSE`;
- `NOTICE`;
- relevant copyright and attribution notices;
- prominent notices on modified third-party files if any are incorporated under compatible terms.

Third-party code must not be copied into the project unless its license is compatible and its attribution requirements are satisfied.

## 18. Explicitly out of scope for the initial MVP

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

These may become later features once the core system is stable.

## 19. MVP acceptance criteria

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
10. the implementation does not noticeably degrade normal vehicle movement during a representative multiplayer test;
11. at least one supported vegetation class can be safely crushed or removed by vehicle passage, or the project explicitly documents vegetation destruction as deferred based on API evidence;
12. all experimentally discovered limitations are documented before `0.1.0`.
