# Reference Implementations Review

## Purpose

This document records observations from third-party Build 42 mod source packages supplied for comparative research during Happy Trails development.

These mods are **not** dependencies or architectural templates. They are used to identify potentially useful Project Zomboid APIs, implementation patterns, performance risks, and questions for independent validation.

No third-party source code has been copied into Happy Trails.

The supplied archives did not establish a reusable license for these codebases, so Happy Trails must treat their implementation as read-only reference material unless licensing is separately confirmed.

## Reviewed packages

| Workshop/package ID | Mod | Primary relevance |
|---|---|---|
| `3457632586` | Footprints / zombie and bandit footprints | Persistent ground marks, actor sampling, cleanup, MP synchronization |
| `3690554902` | Vehicle Vegetation Destruction (VVD) | Vehicle-local scanning, vegetation classification/destruction, server validation |
| `3413150945` | More Damaged Objects | Native vehicle-hit sprite properties, damaged-object replacement, branch/twig aftermath |

## 1. Footprints (`3457632586`)

### What it demonstrates

The implementation demonstrates that a B42 mod can:

- instantiate visual ground objects with `IsoObject`;
- add and remove those objects from `IsoGridSquare` instances;
- tag them with object `modData`;
- transmit object additions/removals in multiplayer;
- batch custom client/server footprint synchronization;
- maintain capped tracking collections;
- defer visual creation until visibility conditions are met;
- use weather/snow state to influence cleanup;
- synchronize existing tracked marks to a joining client.

### Performance-relevant structure

The shared runtime is approximately two thousand lines and maintains multiple state collections for:

- placed footprints;
- FIFO cleanup order;
- pending removals;
- player movement anchors;
- zombie movement anchors;
- priority zombies;
- LOS caches;
- deferred footprint reveals;
- cleanup and erosion bookkeeping.

Default configuration includes a footprint cap of 150 and several explicit scan/batch limits.

Player movement is sampled on a real-time interval. Zombie processing also runs on a real-time interval and includes candidate discovery, distance checks, priority tracking, optional LOS checks, and a bounded scan budget.

Each visual footprint is represented as an `IsoObject`. Creation, removal, tracking, cleanup, and synchronization therefore operate at the footprint-object level.

### Useful ideas

- Explicit caps are preferable to unlimited visual-object growth.
- Batch create/remove network messages are preferable to one message per mark.
- Deferred work and per-tick limits can smooth cleanup cost.
- Tagging created world objects makes later cleanup possible.
- Late-join synchronization must be planned explicitly when transient bookkeeping is involved.
- Work counters and debug instrumentation are valuable during performance investigation.

### Risks Happy Trails should avoid or improve

- A persistent-object-per-print model can produce high object lifecycle and bookkeeping cost.
- Actor-level footprint generation creates much more work than Happy Trails needs for an MVP focused only on vehicles.
- Zombie discovery and LOS processing are irrelevant to our core problem and demonstrate how quickly environmental marking can become CPU-heavy when actor scope expands.
- Multiple parallel collections increase cleanup and consistency complexity.
- Queue removal from the front of ordinary Lua arrays can become costly at scale.
- Synchronizing many individual marks to late joiners can become expensive even when batched.
- Visual marks should ideally represent durable **wear states** rather than every individual vehicle passage.

### Happy Trails takeaway

The most important lesson is not “use this footprint system.” It is that durable ground marking is possible, while an object-per-mark lifecycle is precisely the kind of design Happy Trails should benchmark against more compact state-transition approaches.

## 2. Vehicle Vegetation Destruction (`3690554902`)

### What it demonstrates

VVD demonstrates a client-observation/server-mutation pattern:

1. the driving client identifies its current vehicle;
2. every sixth tick it samples the vehicle;
3. it checks the current occupied square for soft vegetation;
4. it builds a short directional cone in front of the vehicle for trees/structures;
5. matching squares are sent to the server as destruction/damage requests;
6. the server validates that the player is the vehicle driver and that the requested position is near the vehicle;
7. the server damages vehicle parts and mutates world objects;
8. server confirmations are sent back to clients.

The implementation also queues destruction work and processes at most ten queued entries per tick.

### Useful ideas

- Limit processing to an actual driver in a vehicle.
- Skip meaningful forward scanning below a minimum speed.
- Bound spatial sampling to a small area around the moving vehicle.
- Deduplicate candidate positions before processing.
- Validate client-generated destructive requests on the server.
- Bound server mutation throughput with a queue budget.
- Separate object classification from movement geometry.

### Risks Happy Trails should avoid or improve

#### Repeated object enumeration

Each candidate square enumerates its object list to determine whether something matches. This work is repeated while driving and could become significant in object-dense areas.

Happy Trails should test whether native properties, cached classification, or threshold-gated processing can reduce this cost.

#### Long-lived processed-square metadata

VVD stores processed keys in `propsMeta` and the reviewed code does not show a general cleanup path for those keys during the session. Long-distance exploration could therefore grow session memory/state continuously even after the relevant object is gone.

Happy Trails should require explicit bounds or lifecycle rules for every persistent/transient lookup table.

#### Coarse vehicle geometry

Soft vegetation is checked only on the vehicle center square. Forward collision detection uses a hand-built cone with fixed offsets/depth/width rather than the vehicle's actual collision or wheel geometry.

This may be adequate for some destruction effects but should not define Happy Trails' track geometry without comparison.

#### Destruction breadth

The server queue handler removes all non-floor objects from a selected square once a prop destruction event is accepted. Happy Trails should classify and mutate the intended object specifically rather than treating an entire square as disposable.

#### Network/authority tradeoff

The client does much of the candidate detection. That distributes work but introduces custom request/validation traffic. Happy Trails should compare this against direct server vehicle observation rather than assuming either is superior.

### Happy Trails takeaway

VVD is strong evidence that bounded local vehicle scanning and server-authoritative mutation can work. It also provides concrete examples of why we need explicit memory growth, object-enumeration, validation, and geometry requirements before selecting that pattern.

## 3. More Damaged Objects (`3413150945`)

### What it demonstrates

This source contains the most interesting engine-native clue found in the comparative review.

During `OnLoadedTileDefinitions`, the mod assigns selected sprites properties named:

```text
HitByCar
MinimumCarSpeedDmg
```

with a minimum vehicle speed value.

This strongly suggests that Build 42 contains native tile/object behavior for vehicle impacts that may be usable for at least some vegetation/destructible-object interactions.

The mod also demonstrates:

- replacing damaged object sprites;
- transmitting object additions/removals;
- adding world inventory items such as branches and twigs after bush destruction;
- reacting to vehicle-associated world-sound events and then modifying nearby objects.

### Useful ideas

- Investigate engine-native `HitByCar` behavior before implementing custom collision scanning.
- Use native collision handling where it can safely eliminate Lua polling.
- Damaged-state sprites may be preferable to simple deletion for some vegetation classes.
- Debris can be generated as ordinary world inventory items if desired.

### Risks Happy Trails should avoid or improve

#### Indirect impact detection

The reviewed common client code registers for `OnWorldSound` while the player is in a vehicle, looks for a specific radius/volume/no-source signature, waits one tick, and scans a 5x5 area around that position.

That is an indirect heuristic. It may work for the mod's purposes, but Happy Trails should not depend on magic sound characteristics if a direct engine event/property or vehicle-state path exists.

#### Broad local scan after each inferred impact

A 5x5 square scan with object enumeration can be expensive if triggered frequently. It also makes the event's exact collision target less precise.

#### Debris proliferation

Real branches/twigs are persistent world inventory items. Frequent automated creation could create long-term server/world-state cost. Happy Trails should measure this before enabling debris broadly.

### Happy Trails takeaway

The primary value of this source is the clue that **native vehicle-hit sprite properties may let the engine do work we otherwise might poll for in Lua**. Validating this should be an early SPIKE-001 experiment because it could materially improve vegetation-damage performance.

## Cross-implementation comparison

| Concern | Footprints | VVD | More Damaged Objects | Happy Trails investigation |
|---|---|---|---|---|
| Movement detection | player/zombie interval sampling | driver vehicle polling every 6 ticks | inferred vehicle impact sound + native properties | compare server sampling, client batching, native/event-assisted hybrid |
| Spatial scope | actor path + nearby zombie discovery | center square + short forward cone | 5x5 around inferred impact | derive minimum necessary squares from vehicle movement |
| World visual state | one `IsoObject` per footprint | removes matched objects | replaces/adds object sprites/items | compare state-per-square, floor mutation, overlays, objects |
| MP model | custom batched sync + world object changes | client request, server validation/mutation | mixed transmission helpers | server owns durable state; movement evidence path remains open |
| Persistence | world objects + runtime bookkeeping/ModData cleanup | world destruction persists | world object mutation | compare direct world state, ModData, sparse server state |
| Cleanup/state bounds | explicit caps/queues, complex cleanup | bounded mutation queue; processed-key table appears unbounded | mostly world-state driven | all runtime tables/queues must have explicit lifecycle/bounds |
| Main performance risk | many actors + many managed objects + sync/cleanup | repeated square/object scanning + growing metadata | heuristic 5x5 impact scans + debris | threshold/state-based mutation with measured work budgets |

## Questions promoted into SPIKE-001

The review adds these priority questions:

1. Can the dedicated server observe active vehicle transforms cheaply enough to avoid client passage reporting?
2. If client reporting is cheaper, can traveled squares be batched/coalesced by distance rather than sent per tick?
3. Can `HitByCar` / `MinimumCarSpeedDmg` or related native properties handle bush/sapling collision without explicit Lua scans?
4. Can track wear be represented as one current state per affected square rather than one object per tire impression?
5. Which visual representation has the lowest persistent object and late-join synchronization cost?
6. Can immutable terrain classification be safely cached, and at what granularity?
7. How does each candidate strategy scale with long-distance exploration versus repeated use of the same route?
8. What state can be discarded when a chunk unloads versus what must remain durable?

## Licensing rule

Until a third-party mod's license is explicitly confirmed, Happy Trails will not copy its source code.

We may independently validate the underlying Project Zomboid API/engine behavior and implement our own solution based on our own requirements, measurements, and architecture decisions.
