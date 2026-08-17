# Happy Trails Testing

## Purpose

This document defines how Happy Trails experiments and releases should be validated. Pre-alpha testing should privilege reproducible evidence over subjective impressions.

## Test environments

Development should use, as appropriate:

- local single-player Build 42.20+;
- local hosted multiplayer where useful;
- dedicated server Build 42.20+;
- two or more clients for synchronization tests.

Record the exact Project Zomboid build for every material result.

## Evidence standard

For any claim about engine behavior, capture enough evidence to reproduce it:

- test version/commit;
- server settings relevant to the test;
- mod version;
- map location or controlled test surface;
- exact player/vehicle actions;
- relevant client and server logs;
- screenshots where visual state matters;
- save/restart behavior where persistence matters.

A visual impression alone is not sufficient evidence for synchronization or persistence.

## Baseline test matrix

### 1. Load and initialization

Verify:

- mod loads without Lua errors;
- stable Mod ID is `pz-happytrails`;
- expected modules load in the correct client/server context;
- disabling the mod leaves the world unchanged.

### 2. Surface classification

Create a controlled route containing representative terrain types.

For each square, record the classifier result and verify that:

- eligible natural surfaces are recognized;
- paved roads are rejected;
- interior/building floors are rejected;
- water/invalid squares are rejected;
- unknown third-party surfaces fail closed.

### 3. Single-pass behavior

Drive once across an eligible route.

Verify:

- only the intended footprint is sampled;
- no duplicate wear is applied while stationary;
- adjacent untouched squares remain unchanged;
- a single pass produces only the configured low-level disturbance.

### 4. Repeated-pass progression

Drive the same route repeatedly using a controlled number of passes.

Record the wear state after each threshold.

Verify:

- progression is monotonic while traffic continues;
- visual state changes only at intended thresholds;
- repeated sampling of the same physical passage does not inflate wear unexpectedly;
- direction changes do not corrupt state.

### 5. Persistence

After creating visible wear:

1. save and exit;
2. reload the world;
3. verify wear and stored metadata;
4. on a dedicated server, restart the server;
5. reconnect and verify again.

### 6. Multiplayer convergence

With two clients:

1. Client A drives the test route;
2. Client B observes from nearby;
3. compare resulting terrain state;
4. disconnect Client B;
5. add additional wear with Client A;
6. reconnect Client B;
7. verify the current authoritative state is visible.

No manual client reload should be required beyond normal joining/loading semantics.

### 7. Vegetation damage

For every supported vegetation class:

- identify the exact object/sprite;
- test low-speed contact;
- test higher-speed contact if speed matters;
- verify whether the object is flattened, removed, or ignored;
- verify mature trees and protected objects remain safe;
- verify no duplicate debris is created in MP.

### 8. Recovery

Once recovery exists:

- establish a known wear score/state;
- advance world time without traffic;
- revisit/reload the square;
- verify lazy recovery produces the expected state;
- drive the route again and confirm wear resumes from the recovered state.

### 9. Performance

Test a representative active multiplayer scenario rather than only a single stationary vehicle.

Measure or observe:

- server log/event frequency;
- number of processed vehicle samples;
- number of state-changing mutations;
- amount of stored modified-square state;
- obvious server tick or driving stutter;
- network message volume if custom networking is used.

Diagnostic logging should be disabled for performance comparisons when verbose logs themselves materially affect results.

## Regression checklist

Before promoting a functional build:

- no Lua errors in normal driving;
- paved roads remain unchanged;
- terrain wear persists;
- MP clients converge;
- late join works;
- no duplicate debris objects/items;
- no uncontrolled wear while parked;
- no mature-tree deletion from ordinary passage;
- disable/fail-safe behavior stops further mutation;
- version markers agree across `VERSION`, root `mod.info`, and `42/mod.info`;
- README, requirements, testing notes, and changelog reflect the tested behavior.

## Debug logging conventions

Development builds should use clear prefixes such as:

```text
[HappyTrails][SERVER]
[HappyTrails][CLIENT]
[HappyTrailsSpike001]
```

Logs should emphasize state transitions and evidence rather than flooding every tick. High-frequency sampling logs should be explicitly opt-in.
