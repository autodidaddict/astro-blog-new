---
title: "Sorting and Dropping Entities by Distance"
description: "Continuing my performance crusade, I decide to care about fewer entities"
published: 2026-08-22
draft: false
tags: ["Gaming", "MMO", "netcode", "Rust", "Umwelt", "simulation"]
---
I'm continuing my quest to build a simulation/game server framework that scales out to ridiculous, if not completely absurd, scales. Solving problems under rigid constraints is too much fun to pass up, and the **50ms** tick budget is my favorite.

The story so far:

1. [Introduction](/posts/before_the_entities/)
2. [Gathering Entity Candidates](/posts/gather-and-notify-subscribers/)
3. [CPU Alignment and Parallelism](/posts/babys-first-cpu-cache-fix/)

The little engine that should is doing really well on uniformly distributed groups as well as crowds smaller than 8,000. My current per-tick performance for the "town square" grouped entity clusters (at 4 threads):

| Entities in the Crowd | Gather Time |
|:--|:--|
| 2,048 | `3.31ms` |
| 4,096 | `13.32ms` |
| 8,192 | `50.19ms` |

## The Next Big Thing
What I want to tackle next is in cutting off the number of potential candidates that each thread processes for a viewer inside the `gather_into` function. I don't need to do that for the less dense cells, so my plan is to sub-divide dense cells and then walk sub-cells nearest-first. I'll stop the walk after some **N** candidates. 

It makes sense to me to build this before the _priority accumulator_ (despite that being the thing I set out to build first) because I don't need to prioritize information that isn't being sent to a client.

The crowd case is quadratic, which just means that its cost explodes and doesn't grow linearly. The big-O cost is viewers times entities, and in a town square those are the same number. 

A viewer in that crowd examines 8,192 entities to send 58 _(that number is explained in the next section)_. Every fix on my list is a way of shrinking that 141 to 1 ratio. The solutions differ in impact by orders of magnitude:

These are estimates, not measurements. None of them have been built yet:

- Capping at 512 candidates is maybe a 16x reduction, roughly 50 ms to 3 ms.
- SIMD on the distance test is maybe 2x to 3x at best, leaving 17 to 25 ms, which is a third of the tick for gather alone.
- Fusing the duplicated horizontal terms is maybe 1.25x.
- Aggregating a distant crowd into one record does nothing for the 8,192 viewers standing inside it, which remains over budget.

I also get diminishing returns from these after performing the cap operation because I'd be applying solutions to something like 512 iterations instead of 8,192.

### Spatial Sub-ordering

The cheap version of a cap is a rotating window: examine slice k of the cell this tick, slice k+1 next tick, no spatial structure needed. This fails because sixteen slices means every entity is examined once per sixteen ticks, including the person standing directly in front of you. Clients could get notified of unimportant things in the distance before what's happening in front of them.

What I think will turn out to be the linchpin is that visiting _sub-cells_ by distance is very different than visiting _entities_ by distance. Remember all that cool bit-twiddling stuff from the first post? That might pay off here, too.

### The Cost of Divide and Conquer (cap)

The secondary sort only happens in dense cells and it happens only once _per tick_, not _per viewer_. Sorting 8,192 entities into 64 sub-cells is two flat-cost passes. Benchmarking it afterward, `update` goes from **75.4µs** to **139.7µs** with an 8x8 sub-grid, so the second sort nearly doubles it in the worst case where the entire population sits in one cell. Still trivial against the tens of milliseconds it saves, but not free.

The structural cost is one extra offsets array per subdivided cell and a logic branch in the `gather` for whether a cell is subdivided.

The real cost is closer to the product-level, impacting user-visible game behavior. A viewer stops seeing everything within a radius and starts seeing the nearest N. Nearly every MMO ships with this kind of truncation. In many cases, the "fog of war" isn't just an in-theme boundary, it often correlates to this kind of "nearest N" cap.

## 58 Records per Packet
Why have I been saying that there is a 58 record budget per packet? Some of this is a guess, and some of it is based on actual numbers. First, I know that if I'm going to optimize the actual network transmission of this stuff, I can't be sending 1 packet for every single view notification per tick. In the town square herd, that means I'd be sending 8192x8192 packets, or 67,108,864, _per tick_. So what I need to do instead is pack some number **N** position update records into a packet and send _one per viewer per tick_.

Let's think about payloads, first. My calculations use a payload (the _full_ set of data transmitted in the packet) of **1200** bytes. Ethernet MTU is 1500, minus 40 for an IPv6 header and 8 for UDP leaves 1452. 1200 is the conservative figure that provides enough wiggle room for tunnels, VPNs, and PPPoE without fragmenting. QUIC mandates 1200 as its minimum datagram size for the same reason. In all my packet calculations, this is the one number I'll die on a hill for.

The state budget is the amount of stuff I can send given the data that needs to surround the state:

```
state_budget = payload_bytes - (header_bytes + event_reserve_bytes)
             = 1200 - (16 + 256)
             = 928

records = state_budget / record_bytes
        = 928 / 16
        = 58
```

16 header bytes is a guess. No packet format has been designed (yet). It is likely to hold a sequence number, an ack or ack bitfield, a tick identifier, and some flags. 16 here seems reasonable, but it's unverified. I can't make it work without _some_ value, so I'll use 16 until benchmarks and tuning tell me otherwise.

The same goes for the "event reserve". Without a reserve, a dense crowd's position updates would fill every packet and a client could stand in the middle of an angry mob and never learn it died. If you decide to trip your PK flag in the town square and backstab a merchant... your client is going to need to receive the important events, especially your brutal demise. The actual number for the event reserve was chosen arbitrarily. This is again a situation where I needed _a_ number to proceed, and it's subject to change as benchmarks and data inform my reasoning.

That leaves `record_bytes`, the 16 I'm dividing by, and it's the weakest number of the four. No wire format exists yet. Working from the wire precision I already configured, a quantized position is 16 bits per horizontal axis and 14 vertical, so 46 bits or about 6 bytes, and a per-connection ghost id for a viewer seeing a few thousand entities needs about 12 more. A bare position update is nearer 8 bytes. The 16 assumes roughly double that for velocity, orientation, or a state mask, none of which I've specified. At 8 bytes a packet holds 116 records and at 24 it holds 38, so 58 could be off by a factor of two in either direction.

So after all that bloviating, now you know how I ended up with 58 bottles of beer on the wall.

## Subdivisions
The current design has cells occupying a contiguous range, defined by the values in `starts[c]..starts[c+1]`. I spent too long trying to figure out some fancy math way of doing this. I even thought (briefly) that Calculus might be the answer. In classic Kevin fashion, I overthought the problem.

Subdividing means sorting that range by sub-cell and recording where each sub-cell begins. I don't actually use a real container or bucket for the division. I just make sure the data appears in subdivision order. This is a common theme and it makes me appreciate the insane amount of work that goes into optimizing things like game engines or ECS engines.

I'll modify the `CellSnapshot` as follows:

```rust
#[derive(Debug, Clone)]
pub struct CellSnapshot {
    ids: Vec<EntityId>,
    xs: Vec<Fixed>,
    ys: Vec<Fixed>,
    zs: Vec<Fixed>,    
    starts: Vec<u32>,    
    cursor: Vec<u32>,
    cells: usize,
    cfg: WorldConfig,

    // New stuff for subdivisions
    sub_index: Vec<u32>,
    // For each subdivided cell, `axis * axis + 1` absolute offsets.
    sub_starts: Vec<u32>,
    sub_axis: u32,
    sub_threshold: u32,
    // Precomputed visit orders, built once and independent of entity data.
    sub_order: Vec<u8>,
    cell_order: Vec<(i8, i8)>,
}
```

And then expose some functionality for the subdivisions so that callers don't have to remember
how to do this in their loops:

```rust
impl CellSnapshot {
    /// Sub-cell ranges for a cell, or None if it was not dense enough.
    pub fn sub_cells(&self, cell: CellId) -> Option<SubCells<'_>>;
    pub fn with_subdivision(cfg: &WorldConfig, sub_axis: u32, sub_threshold: u32) -> CellSnapshot;
    pub fn subdivided_cells(&self) -> usize;
    /// Sub-cell visit order for a viewer, nearest first.
    pub fn sub_cell_order(&self, cell: CellCoord, viewer: Pos2) -> &[u8];
    /// Cell offsets from a viewer's own cell, nearest first.
    pub fn cell_order(&self) -> &[(i8, i8)];
}

pub struct SubCells<'a> { /* axis, offsets, and the four parallel slices */ }

impl SubCells<'_> {
    pub fn axis(&self) -> u32;
    pub fn occupants(&self, sx: u32, sy: u32) -> CellOccupants<'_>;
    pub fn count(&self, sx: u32, sy: u32) -> usize;
    /// By linear index, as yielded by `sub_cell_order`.
    pub fn occupants_at(&self, b: usize) -> CellOccupants<'_>;
    pub fn count_at(&self, b: usize) -> usize;
}
```
The one thing that sticks out at me like a giant middle finger is the use of a couple lifetime indicators. I consider myself a pretty good slinger of Rust, but even I get nervous when I see explicit lifetimes in the code. Separately, `sub_starts` is sparse: only cells that cross the density threshold occupy space in it, and `sub_index` is one `u32` per cell pointing into it. Storing sub-offsets for every cell would be 256KB of mostly zeros. 

I've always thought about keeping the number of allocations low, but historically I've done that in garbage collected languages like Java and C#. Being ruthless about allocations in Rust is... ゲーム, which is just "Game" phonetically in Japanese, but it has an aura of "the joy of a challenge" with it that I can't convey in a single English word.

Here's my new implementation of gather, `gather_into_capped`, that takes advantage of cell subdivision and distance ordering:

```rust
pub fn gather_into_capped(
    &self,
    viewer: Pos3,
    sub: Subscription,
    cap: usize,
    out: &mut DiscoveredEntities,
) {
    let cfg = self.config();
    let radius_sq = DistSq::from_radius(cfg.horizontal_view_radius());
    let viewer_h = viewer.horizontal();
    let center = cfg.cell_of(viewer_h);
    debug_assert!(
        sub.contains(center),
        "subscription does not contain the viewer's own cell"
    );

    for &(dx, dy) in self.cell_order() {
        let cx = center.x as i32 + dx as i32;
        let cy = center.y as i32 + dy as i32;
        if cx < sub.x0 || cx > sub.x1 || cy < sub.y0 || cy > sub.y1 {
            continue;
        }
        let coord = CellCoord::new(cx as u16, cy as u16);
        let cid = cfg.cell_id(coord);

        match self.sub_cells(cid) {
            Some(grid) => {
                for &b in self.sub_cell_order(coord, viewer_h) {
                    take(viewer, viewer_h, radius_sq, grid.occupants_at(b as usize), out);
                    if out.len() >= cap {
                        return;
                    }
                }
            }
            None => {
                take(viewer, viewer_h, radius_sq, self.entities_for_cell(cid), out);
                if out.len() >= cap {
                    return;
                }
            }
        }
    }
}
```

Another thing that changed is that now cells are visited outward from the viewer's own cell rather than the original _row-major_ strategy I started with. The ring order is precomputed once from the `cell_radius`. If we capped on a row-major visit order, then we would fill the cap on whatever cell happened to be first.

## The Results
I think I'm in a good place, performance-wise. There's probably some small tweaks that can be made, but I'm no longer in the "I don't have enough per-tick time budget to do all this" panic mode. 

Check out the numbers at 4 cores:

| cap | 20 samples | 50 samples | % of 50 ms tick |
|---|---|---|---|
| uncapped | 44.10 ms | 42.09 ms | 84% |
| 2,048 | 12.04 ms | 10.33 ms | 21% |
| 1,024 | 6.00 ms | 5.32 ms | 11% |
| 512 | 3.11 ms | 3.05 ms | 6.1% |
| 256 | 1.81 ms | 1.77 ms | 3.5% |

8,192 town square, 4 threads, every entity also a viewer. The percentage column is from the benchmark test with a sample size of 50.

There are some very important points here. I think the biggest one for me is this:

<font color="red">_**At a cap of 512 the crowd is 6.1% of tick against the uniform case's 5.7%. A dense crowd is no longer a special case.**_</font>

Remember that when I first started the Optimization World Tour (get your tickets now!), a single core was handling an 8,192 entity crowd in about 180ms. Next I went to multiple cores, and now I've added a culling strategy. Nothing better than a good culling.

And now the absolute worst-case scenarios, the crowd of unwashed masses all not washing on a single core, with the culling/cap enabled:

**Single core, 8,192 town square:**

| | time | % of 50 ms tick |
|---|---|---|
| original, uncapped | 152.14 ms | 304% |
| cap 512 | 10.89 ms | 22% |
| cap 256 | 6.18 ms | 12% |

**Single core, 10,000 town square:**

| | time | % of 50 ms tick |
|---|---|---|
| uncapped | 229.19 ms | 458% |
| cap 512 | 13.78 ms | 28% |
| cap 256 | 7.29 ms | 15% |

Another headline: <font color="red">_**8,192 in one cell went from 304% of a single core's tick budget to 22%, a 14.0x reduction, with no threads involved**_</font> (even better if you look at the 256 entity cap value). One caveat on that 152.14ms: I measured the same uncapped scenario at 178.79ms and 182.13ms in earlier runs. This machine drifts 15% to 28% between runs on the same benchmark with no code change at all, so I can't tell you whether the gap is the new outward ring traversal or just noise. The 14.0x ratio comes from a single run where uncapped and capped were measured together, so that part is internally consistent.

Now all 8,000 of my closest friends can hang out right next to the town square's job board and the engine will handle it as smoothly as if we were all evenly spread across the whole cell.

## Wrapping Up
It all started when I was just a boy, 4 days ago, when I had a dream of making a super-scalable simulation server framework for people to make games with. I've come so far. Uncapped, the same gather is 14 times slower than the capped one. It was _nowhere near_ the tick budget on a single core.

Now, the worst-case scenario crowd on a single core consumes 22% of the tick budget at 8,192 and 28% at 10,000, with the option to get that lower with tuning and a stricter culling threshold. I can't remember the last time I ever made such a huge performance impact on any code I've written. I'm going to be riding the nerd high from this for a while... or at least until I find another hard problem to solve in the simulator engine.