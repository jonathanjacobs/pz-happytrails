# SPIKE-001 — Build 42 Environmental Wear Feasibility

Status: **In progress**  
Primary target: Project Zomboid Build 42.20.x  
Architecture: [`../ARCHITECTURE.md`](../ARCHITECTURE.md)

## Question

Can Project Zomboid Build 42 support a performant, persistent, multiplayer-safe Happy Trails mechanic using Lua-accessible APIs and ordinary mod assets, without engine modification or brittle hacks?

A second question is equally important:

> Which implementation strategy provides the best client/server performance envelope without preventing higher-fidelity features later?

## Development rule

SPIKE-001 must narrow the design space with evidence. Promising Java/decompiled implementation details are research evidence, not production architecture, until the relevant behavior is verified from Lua in single-player and dedicated-server multiplayer.

## Evidence reviewed so far

Third-party Build 42 mods were reviewed as reference implementations. No source from those projects has been incorporated into Happy Trails. See [`../REFERENCE-IMPLEMENTATIONS.md`](../REFERENCE-IMPLEMENTATIONS.md).

Build 42 decompiled engine source has also been reviewed as research material. Findings are summarized in [`../ENGINE-RESEARCH-B42.md`](../ENGINE-RESEARCH-B42.md); decompiled game source is not distributed with Happy Trails.

## New vanilla precedent — corpse-drag/floor-blood trails

The B42.20.3 floor-blood implementation materially improves our understanding of how PZ can represent dense temporary ground marks.

Confirmed characteristics of `IsoFloorBloodSplat` / `IsoChunk`:

- compact dedicated records rather than ordinary persistent world objects;
- sub-tile floating-point x/y/z position;
- per-mark creation `worldAge`;
- serialization with the chunk;
- a bounded queue of 1000 floor splats per chunk;
- overflow moves old splats into a short fade list instead of allowing unbounded growth;
- visual aging is calculated lazily from `WorldAgeHours - splat.worldAge`;
- blood rendering fades/tints over a 72-world-hour age window.

This strongly supports a **two-layer Happy Trails model**:

1. high-resolution transient tire/material marks;
2. sparse persistent terrain-wear state.

Direct reuse/hijacking of vanilla blood splats is not assumed viable because the renderer and type tables are blood-specific.

## Required investigations

### A. Trace vanilla continuous trail deposition

Determine where corpse dragging or related movement logic decides to emit floor blood and how deposition is spatially throttled.

Questions:

- Is deposition distance-based, tick-based, animation-event-based, or stochastic?
- Does vanilla interpolate between movement samples?
- What spatial spacing produces a visually continuous trail without excessive marks?
- Is the relevant logic Lua-accessible or only engine-internal?

This is now useful precedent for tire-track sampling even if we cannot reuse the blood renderer itself.

### B. Custom lightweight decal feasibility

Determine whether Lua can create a custom lightweight ground mark with properties comparable to floor blood:

- custom tire/material texture;
- sub-tile position;
- orientation or directional variants;
- alpha/tint control;
- bounded lifetime;
- save/reload behavior where needed;
- dedicated-server/MP visibility;
- no full ordinary `IsoObject` per impression.

If no suitable generic decal path is exposed, document that result explicitly and benchmark alternatives.

### C. Server-side vehicle-state visibility

Determine whether dedicated-server Lua can reliably access:

- active vehicles;
- x/y/z and orientation;
- speed;
- vehicle script;
- wheel count/offsets;
- transformed wheel world positions;
- driver/occupancy state;
- enough information to reject stationary vehicles cheaply.

Measure actual time/distance between meaningful server-visible transform changes while a remote client drives.

### D. Gap-free wheel-path sampling

Compare representative left/right wheel tracks against all scripted wheel tracks. Interpolate previous/current wheel positions so high vehicle speed does not create gaps.

Measure:

- samples per moving vehicle per second;
- emitted marks per meter;
- duplicate/coalesced samples;
- path continuity at increasing speed;
- CPU/work scaling with multiple moving vehicles.

### E. Terrain classification

Identify a cheap central classifier for eligible natural surfaces versus pavement, water, interiors, and unknown/modded surfaces. Unknown terrain must fail closed for persistent mutation.

### F. Persistent visual wear

Benchmark an existing-floor overlay for established wear. Verify:

- underlying terrain remains intact;
- object count does not grow unnecessarily;
- persistence across save/reload/restart;
- existing-client and late-join convergence;
- safe behavior when an overlay slot is already occupied;
- clearing/restoration.

Retain additional `IsoObject` and floor replacement as measured fallbacks.

### G. Persistent wear-state storage

Compare bounded per-square/discrete sparse state representations. State growth should follow **currently meaningful changed terrain**, not every historic pass.

Candidate record:

```lua
HappyTrailsTileState = {
    wearLevel = 3,
    lastTrafficWorldAge = 1234.5,
    trafficScore = 27,
    dominantTrackAngle = 42.5,
    trackWidth = 0.72,
    surfaceType = "grass",
}
```

Exact schema remains provisional.

### H. Native vegetation damage

Test PZ's existing vehicle/object collision path, including `HitByCar`, minimum-speed damage, damaged-sprite transition, removal, vehicle response, and multiplayer synchronization. Only benchmark a custom Lua spatial scanner if the native path cannot cover the required vegetation classes.

## Runtime experiment order

1. **Corpse-drag trail trace** — identify blood emission cadence/spacing and relevant calls.
2. **Custom transient-decal smoke test** — one conspicuous custom mark at a sub-tile position; test orientation/alpha/persistence/MP.
3. **Persistent floor-overlay smoke test** — one controlled natural square; test save/reload, restart, late join, object count, conflicts.
4. **Server wheel-geometry probe** — no terrain mutation; log transforms and rasterized wheel paths.
5. **Native vegetation-damage probe** — controlled objects only.
6. **Minimal repeated-wear prototype** — transient marks plus a three-state persistent wear model.
7. **Alternative benchmark** — compare at least one viable alternative before architecture selection.

## Prototype constraints

All spike code must be bounded and removable. It should:

- operate on controlled areas initially;
- use conspicuous `[HappyTrailsSpike001]` logging;
- include a kill switch;
- avoid broad world scans;
- bound queues/tables;
- avoid persistent debris;
- rate-limit logging;
- expose work counters;
- fail closed on unknown terrain/objects.

## Success criteria

SPIKE-001 succeeds when evidence demonstrates a credible path that can:

1. observe moving vehicles from the authoritative server with sufficient fidelity;
2. derive continuous wheel paths without high-speed gaps;
3. create convincing transient ground marks at useful spatial resolution;
4. keep transient mark count/lifetime bounded;
5. classify at least one eligible natural and one ineligible hard surface;
6. create at least one persistent non-destructive wear state;
7. synchronize persistent state to existing and late-joining clients;
8. survive save/reload and dedicated-server restart;
9. show bounded CPU/state/network scaling;
10. validate a low-cost vegetation-damage path or explicitly defer it;
11. compare the selected candidates against at least one alternative.

## Go / no-go

**GO:** proceed to a functional v0.0.x prototype when measured candidates provide transient tracks plus persistent/synchronized wear with bounded scaling.

**CONDITIONAL GO:** proceed with core track/wear only if vegetation, snow, mud, debris, or recovery require separate investigation.

**NO-GO / REDESIGN:** redesign if custom ground marks cannot be represented acceptably, persistent wear cannot be synchronized cleanly, viable approaches require unacceptable object/state growth, or vehicle path observation cannot be bounded.

## Deliverables

- measured results added to this document;
- [`../ARCHITECTURE.md`](../ARCHITECTURE.md) updated with validated constraints;
- [`../REQUIREMENTS.md`](../REQUIREMENTS.md) updated where evidence changes requirements;
- [`../VALIDATION_HISTORY.md`](../VALIDATION_HISTORY.md) updated with completed test evidence;
- [`../../CHANGELOG.md`](../../CHANGELOG.md) updated;
- ADRs created only for durable choices supported by measurements;
- follow-on spikes created only for unresolved blocking questions.
