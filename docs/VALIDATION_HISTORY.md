# Happy Trails — Validation History

This document records runtime and source-review evidence that materially changes the design. Current procedures live in [`TESTING.md`](TESTING.md); active investigations live under [`spikes/`](spikes/); durable decisions belong under [`adr/`](adr/).

## Current evidence boundary

No functional Happy Trails gameplay implementation has yet been validated. Current evidence is architectural/research evidence used to define SPIKE-001.

## Build 42 engine research

The project has reviewed Build 42 decompiled source and vanilla Lua as research inputs. Decompiled game code is not distributed with the mod.

Findings already incorporated into the architecture include:

- server-side vehicle state appears rich enough to justify a dedicated runtime wheel-geometry probe;
- vehicle scripts expose wheel geometry that may support representative left/right wheel tracks;
- existing floor-object overlays are a plausible persistent-wear presentation path;
- native vehicle/object collision should be investigated before building a custom vegetation neighborhood scanner;
- no general vehicle-motion Lua event has been assumed available, so bounded sampling remains an acceptable candidate pending measurement.

## Vanilla floor-blood / corpse-drag precedent — 2026-08-20

Observation in Build 42: dragging corpses can leave a blood trail that fades over time.

Follow-up source review of B42.20.3 established that floor blood uses `IsoFloorBloodSplat` records stored directly by `IsoChunk` rather than unlimited ordinary world objects.

Confirmed details:

```text
IsoChunk.floorBloodSplats        -> bounded queue, capacity 1000
IsoChunk.floorBloodSplatsFade    -> short fade-out list
IsoFloorBloodSplat position      -> sub-tile float x/y/z
IsoFloorBloodSplat age anchor    -> worldAge at creation
visual age                       -> current WorldAgeHours - creation worldAge
normal blood aging window        -> 72 world hours
```

When the queue is full, the oldest splat is removed from the bounded queue and moved to the fade list rather than allowing unbounded accumulation.

### Design consequence

This evidence materially strengthens a two-layer design:

1. **transient marks** — high-resolution, sub-tile, bounded, age-faded tire/material impressions;
2. **persistent wear** — sparse square-level terrain condition representing accumulated repeated traffic.

The engine precedent validates lazy age calculation and bounded per-chunk visual history as appropriate PZ-native design principles.

### Important limitation

The floor-blood system is blood-specific. Its sprite types, rendering/tint treatment, and queue semantics are not assumed to be a supported generic custom-decal API. Happy Trails should copy the architectural principles, not hijack vanilla blood tables or assets.

### New SPIKE-001 work

SPIKE-001 now explicitly requires:

- tracing the corpse-drag blood-emission cadence/spacing logic;
- testing whether Lua exposes a generic lightweight custom sub-tile decal path;
- comparing that path against persistent floor overlays and ordinary `IsoObject` alternatives;
- bounding mark count/lifetime and measuring continuous wheel-path quality.

## Next validation milestone

The next meaningful evidence should come from controlled runtime probes, beginning with corpse-drag emission tracing/custom-decal feasibility and then dedicated-server wheel geometry.
