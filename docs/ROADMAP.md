# Happy Trails Roadmap

This is the single canonical project roadmap. The top-level README should summarize current status and link here rather than maintain a second roadmap.

## Current phase — pre-alpha feasibility

Primary investigation: [`SPIKE-001`](spikes/SPIKE-001-environmental-wear-feasibility.md).

Current goals:

- validate server-visible vehicle/wheel geometry;
- trace vanilla corpse-drag blood deposition as a reference for continuous transient trail generation;
- determine whether Lua can create lightweight custom sub-tile ground decals;
- validate a persistent square-level wear representation;
- validate save/reload, dedicated-server persistence, multiplayer synchronization, and late joins;
- test native vehicle/vegetation collision paths;
- establish measurable performance bounds before implementing broad gameplay behavior.

### Pre-alpha exit criteria

Move to a functional alpha prototype when SPIKE-001 demonstrates:

- continuous vehicle paths without high-speed gaps;
- convincing transient tire marks with bounded lifecycle/state;
- at least one persistent non-destructive wear state;
- safe terrain classification;
- multiplayer/save persistence;
- bounded CPU/state/network growth;
- a viable vegetation strategy or an explicit decision to defer it.

## Functional alpha — v0.0.x

Implement the smallest complete gameplay loop:

```text
vehicle traffic
-> transient wheel marks
-> accumulated wear
-> at least three persistent wear levels
-> save/restart/MP persistence
-> slow abandonment recovery
```

Initial scope should focus on grass/dirt and a small controlled asset set. Diagnostics should remain available but disabled by default.

## Public Alpha — v0.1.x candidate

Broaden field testing and content coverage:

- multiple vehicle types and wheelbases;
- longer travel routes and exploration-heavy play;
- 3–12+ player dedicated-server use;
- rain/mud/snow presentation where feasible;
- carried material onto hard surfaces;
- vegetation destruction;
- erosion/regrowth interaction;
- compatibility with important map/vehicle/environment mods;
- administrator performance/fidelity controls.

## Public Beta

Focus on stabilization rather than expanding the core model:

- persistence/schema stability;
- recovery tuning and hysteresis;
- long-session performance;
- migration/disable/rollback behavior;
- asset polish and directional coverage;
- compatibility matrix;
- Steam Workshop packaging/documentation;
- regression testing across B42 updates.

## v1.0 readiness

A stable release should require:

- no known high-severity save/world corruption risk;
- representative multiplayer scale validated;
- bounded transient and persistent state growth;
- established trails survive restart and recover predictably when abandoned;
- installation, upgrade, disable, and rollback documented;
- compatibility claims limited to combinations actually tested;
- all distributed code/assets have clear provenance and licensing.

## Post-MVP candidates

Possible later features, not commitments:

- richer mud-depth/water-filled rut visuals;
- more sophisticated vehicle mass/speed/traction effects;
- player/zombie foot traffic contributing to paths;
- repair/restoration actions;
- route-history/admin visualization;
- compatibility helpers for specific terrain or vehicle mods.
