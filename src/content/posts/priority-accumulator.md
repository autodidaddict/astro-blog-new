---
title: "Building a Priority Accumulator for the Simulation Server"
description: "I (finally) get to add the priority accumulator to the simulation code"
published: 2026-08-26
draft: false
tags: ["Gaming", "MMO", "netcode", "Rust", "Umwelt", "simulation"]
---

[Throughout this series](https://kevinhoffman.blog/tags/umwelt/) I've been ruthlessly optimizing the work done during a single tick of the
simulation server so that it can handle extreme scale as a worst case scenario. In my testing, that
scenario has been to cram 8,192 entities into a 128 meter square area and then do the work at 20 Hz (20 times per second). That gives me a latency budget of **50ms**.

To give you some perspective, a fountain with benches around it can be represented in a single 128 meter cell. That fountain right in front of the boards in FFXIV, if running in my engine, would theoretically be able to handle 8,000 people without slowing down. I know this value is going to go down with more heavy game logic, but it's a good start.

I've accumulated quite a bit of evidence and conclusions since the start. If I keep the packet size down to 1200 bytes, then it fits under the standard 1500 MTU and so (most of the time), that packet should be sendable and routable without having to split it. I've determined that a single position update costs **12 bytes**, so a single message carries **98** updates. That makes my budget 98 records per player per tick pass. 

This brings me to the problem that I need to solve. We need to build packets targeted at specific viewer entities, 20 times per second. There are two extreme ends of the spectrum that need to work. First, we have a player standing in a world where the entities are all uniformly distributed. 

![Uniform distribution](/images/umwelt-layout-uniform.svg)

_**Figure 1** - 8,192 objects over 16.8 square kilometers. About 100 fall inside the
sight radius and the search examines all of them, 96.7 on average, against a
limit of 256._

In the uniform distribution case, the number of entities a player can see averages out to almost the same number of records carried by a single packet. This seems easy, but it can mess with algorithms in weird ways.

The worst case scenario is...well, worse. A player can be standing in an area where they see far more than 98 other entities. In my own "crowded town square" scenario, a single player could be standing within sight range of more than 8,000 other entities. Naively implemented, it would take ~800 packets to convey all the positions, which is (800 / 20 = 40 ticks) ~2 seconds. 

![Single-cell crowd](/images/umwelt-layout-town-square.svg)

_**Figure 2** -  The same 8,192 objects inside one cell. The search examines 322.8 and
stops, leaving 7,900 untouched. Above 512 objects a cell is divided into 16 m
sub-cells (dashes) so a crowd can be walked nearest-first without sorting it.
Gray dots thinned to one in three._

The crowd in Figure 2 is 80 times denser and costs 3.3 times more to examine.

Two seconds might seem like no big deal, but in an online game, it's an eternity. Every object in the list would appear jerky, standing still for 2 seconds and then teleporting to a new location. We could order that list by distance from the player. But then all we've done is choose which objects we get updates about and which ones we miss. 

Regardless of the algorithm, if we send 8,000 updates, then we'll have to wait 2 seconds to get an update from each entity. So the first thing I did (which I started doing a few posts ago) was to cut off what gets sent. But even that isn't good enough. I still need to eventually get position updates from things further away.

## Prioritizing Location Updates
The only way to solve this problem by keeping the simulation ticks as small as possible while sending the fewest number of updates is to intelligently prioritize what information goes out on each packet.

Some terms that I use in this section that might be new if you're not following along with the code:

* `odometer` - Each entity in the simulation has an odometer. It tracks an always-increasing value of distance traveled. My engine doesn't actually _do_ the movement (I don't have a physics engine or even a first-class concept of velocity). All I do is track when entities move. Performing the move is up to the game itself.
* `cell` - A logical subdivision of geometric space in which entities reside. Defaults to 128 meters.
* `region` - Each simulation is responsible for a single region. Much of my benchmarking is done against a 16km² region, divided into cells of 128 meters.
* `viewer` - An entity that cares about what's around it. Some entities may exist in the simulation without caring about position updates.

One last quirk worth mentioning: The position updates occur on all 3 axes (`x`, `y`, and `z`). However, the `z` axis is treated a bit differently. I don't use true Euclidean distance there, instead using [Manhattan](https://en.wikipedia.org/wiki/Taxicab_geometry) distance. So instead of pure 3D, think of the simulation's view of the world as 2D where each position has a vertical cylinder extending upward from it. 

⚠️ **NOTE** - This is an opinion that can have material impact on gameplay. I can do this optimization because the world is "mostly flat" (hills and mountains still matter). But a world in which the Z axis isn't sparse like a sci-fi space game would need different algorithms that would have real impact on performance.

### Once per Tick
These steps are performed at the beginning of each tick. This is work done once regardless of the number of players within the region.

1. **The consumer's game callback runs** - This callback gets mutable access (no allocations) to the position arrays and can spawn, move, and despawn. Everything else below is downstream of the per-tick work performed by the game.
2. **The odometer walks every slot** - For each live entity it takes the distance from where that entity was on the previous pass (`|dx| + |dy| + |dz|`) and adds it to a per-entity running total. Then it records the current position as the new previous. The total wraps rather than saturating since a saturating total would pin at its maximum and freeze that entity's score forever.
3. **The snapshot is rebuilt** - Every live entity is sorted into its cell, producing an array of positions ordered by cell. Dead entities are absent. This data structure becomes the source for per-player search next.

### Per Player per Tick
Players are split across worker threads. Each of the following happens per player, within one
of the threads.

1. **Evaluate skip conditions** - Not registered, not due this pass (a client can be served every Nth pass), avatar dead, or avatar outside the simulation's region.
2. **Diff the subscription box** - The cells within sight of the avatar's cell, which is 5 × 5 at the default settings. If the box differs from last pass, that counts as a subscription change.
3. **Perform the search** - Walk the box's cells in order of distance from the player, collecting candidates, and stop once the candidate count reaches the walk limit. The limit is checked at cell boundaries rather than per object, so the result overshoots slightly (322.8 against a limit of 256 in the crowded case).
4. **Cut at the budget limit** - Take the client's declared message size, subtract the header and anything held for pending events (I haven't implemented events yet), then take the despawn records queued from last pass (despawns notify on the following tick, not current), capped at half the message. What remains, divided by the 12-byte record size, is the number of slots available for position updates.
5. **Scoring** - We're going to sort the candidate list, so we need to provide a meaningful score, not just a naive distance or staleness value.
   1. If the client already holds a copy, drift is the entity's odometer total now minus the total recorded when it was last sent. That subtraction wraps, which is why the total is allowed to wrap.
   1. If the client holds no copy, drift is a fixed constant standing in for "could be wrong by a whole view radius," and the entry is flagged as new.
   1. Score is drift multiplied by the weight for that distance band. One multiply, saturating. The band is ilog2 of the squared distance, so it costs a shift and a table lookup, no square root. Distance bands are logarithmic distance from the viewer so we don't have linear priority based on distance.
6. **Sort and cut** - Sort by score descending, ties broken by position in the candidate list, which is nearest-first. Take the first slot's entries, then cut that prefix again at the first score of zero. An entity that has not moved since the client last heard about it scores zero and consumes nothing.
7. **Bookkeeping on the ghost table** - For each entity sent, record the odometer total at this moment. For each entity in the set but not sent, stamp it as still present without touching its recorded total, so its drift keeps accruing. Then, evict any entry not stamped within the grace window, and collect those IDs as departures.
8. **Assembly** - Write the header, then the despawn records, then the position updates, into a buffer the worker reuses.
9. **Handoff** - Pass the bytes to a game-supplied entity sink. Queue this pass's departures as next pass's despawn records.

This is a lot to soak in so I'm not going to do a code dump here. Feel free to take a look, but honestly it's a bunch of really annoying math that takes a lot of optimization shortcuts that aren't self-describing in the least. Without Google and AI, I wouldn't have been able to get the algorithms back down to the right time cost without spending _a lot_ of time running experiments.

## Wrapping Up
With the new priority accumulator in place, in the _worst case scenario_ my library's overhead is only **16ms** out of the **50ms** tick budget, or **32%**. Best-case scenarios are coming in at around **5%**.

Specifics: Measured at 50,000 objects and 8,192 players in a single crowded cell on a laptop[^1], a pass takes 16.4 ms
of the 50 available, 20.5 ms in the worst one percent, none missed a deadline,
and messages come back 97.7 of 98 slots full. Only 0.2% of objects ever qualified as "never notified".

Next I'm either going to start working on the first socket serving code or I might add event support so I can do benchmark checking on `Game` implementations that handle combat and emit events.

---
[^1]: A 2020 M1 Macbook Pro, running a bunch of other consumer stuff during the benchmarks like Slack, Chrome, Terminal, Signal, etc.