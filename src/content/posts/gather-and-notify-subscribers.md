---
title: "Gathering and Notifying During Simulation Ticks"
description: "The first post in the series that deals with entities inside cells"
published: 2026-08-21
draft: false
tags: ["Gaming", "MMO", "netcode", "Rust", "Umwelt", "simulation"]
---

Last time on _Storytime with Uncle Kevin_, I started talking about my plan to build out a simulation backend architecture that can scale to absurd sizes. I went through my thought process that ultimately led to a scalable design. By the end of the last post, I could take a viewer's position and answer, _Which cells are close enough to matter?_ The bounding box I built takes about **1.5 nanoseconds** to build.

In this post, we're going to figure out which entities inside those cells matter, where they are, and how we can get that information to 10s of thousands of clients without bringing the system to a halt. This is also the first thing I'll be building that can be impacted by the number of entities.

In the interest of keeping each of my design sessions short, I've got a single requirement for this one:

> _Given the cells to which a viewer is subscribed, produce the list of entities in them accompanied by the distance from the viewer_

I built the `CellCoord` and `Subscription` last time. Comparing a viewer's subscription against the previous tick's gives the cells they entered and left, and that difference is what tells a client _"you've entered this cell, start listening for its contents"_ or _"you've left this cell, ..."_. Cells are _addresses_ rather than _content_. This kind of subscription data is used to power spawn and de-spawn notifications on clients.

Cell coordinates tell a client where to look, not what's there. Let's look at this using some example numbers. With the default configuration, a cell is 128 meters across and the wire precision is 1/16th of a meter. This produces 2,048 distinguishable positions per axis inside a single cell.

So now I know that I need a search-narrowing data structure for the gather, some of it gets transmitted, some doesn't. Let's see what I end up with and how it performs.

```rust
#[derive(Clone, Copy, PartialEq, Eq, Debug)]
pub struct DiscoveredEntity {
    pub id: EntityId,
    pub dist_sq: DistSq,
}
```

The distance here is still squared and not the actual. Rather than doing the square root in order to sort, the full squared number still preserves ordering. If we need the actual distance later, then we can take the `sqrt` then.

And now a container to hold discovered entities:

```rust
/// Entities gathered for one viewer, in the order the cells were walked.
pub struct DiscoveredEntities {
    items: Vec<DiscoveredEntity>,
}
```
What you'll see if you're looking for the code is that there is one of these per worker thread, reused across every viewer that thread handles, rather than one per viewer. So the `Vec` allocates once at startup and never again, rather than 10,000 times per tick. When we're dealing with nano and microsecond budgets, getting decisions like this wrong can ruin the whole project.

What happens when a viewer is standing next to a crowd of 5,000 other entities? This is the whole reason why I'm making this engine. We can let the `Vec` grow with a generous reserved amount at startup. After warmup no growth takes place. Once I can benchmark some of this stuff, I'll have a better idea of how much to reserve, etc.

## Not-so-Magic: The Gathering
With `DiscoveredEntities`, I've got a place to put entities I find when gathering, but I need to understand a few more higher level things before I can write a `gather` function.

In the simulation's memory, entities live in 3 arrays and a bitset. There is an array per axis storing the entity's `Fixed` position, and the bitset records which slots currently hold a live entity. There is no array of IDs, because an entity's ID _is_ its index into the position arrays. These entities are owned by my library, but the consumer (the game or simulation server) is responsible for moving them each tick.

This kind of design is pretty common to optimize reading a bunch of stuff in a row, so we don't have to chase 10,000 scattered addresses and pay a cache miss on each one. This kind of "struct of arrays" design is also what underpins nearly every [ECS (Entity Component System)](https://en.wikipedia.org/wiki/Entity_component_system).

The gathering needs to lay things out in cell order when we're maintaining them in ID-order. An entity's ID gives you an idea of (relatively) when it was created, while the cell tells you where the entity is. Reading positions by ID while walking a cell means jumping all over memory, because those two orderings have nothing to do with each other.

After the game's hook into the tick runs, the tick in the library rearranges everything into cell order. This is my `CellSnapshot` struct. Entities in this snapshot are laid end-to-end, grouped by cell. The positions are stored right next to the entity IDs so iterating over this snapshot gets the entity ID and its position not just without a lookup, but in a way that keeps the data resident in L1 and L2 and lets the CPU's prefetcher see where the loop is going.

The gather function walks the list of cells a viewer is subscribed to and then walks the entities within that cell. While it might look like we're making some naive looping mistakes, it's optimized under the hood.

The gather function (on the `CellSnapshot` struct):

```rust
pub fn gather_into(
    &self,
    viewer: Pos3,
    sub: Subscription,
    out: &mut DiscoveredEntities,
) {
    let cfg = self.config();
    let radius_sq = DistSq::from_radius(cfg.horizontal_view_radius());
    let viewer_h = viewer.horizontal();

    for coord in sub.cells() {
        let entities = self.entities_for_cell(cfg.cell_id(coord));
        for i in 0..entities.len() {
            if viewer_h.dist_sq(entities.horizontal(i)) > radius_sq {
                continue;
            }
            out.push(DiscoveredEntity::new(
                entities.ids[i],
                viewer.dist_sq(entities.pos(i)),
            ));
        }
    }
}
```

And the `entities_for_cell` function, which is the one that (from the outside) looks like we're doing a naive list traversal:

```rust
#[inline(always)]
pub fn entities_for_cell(&self, id: CellId) -> CellOccupants<'_> {
    let c = id.index();
    debug_assert!(c < self.cells, "cell {c} out of range for {} cells", self.cells);
    let lo = self.starts[c] as usize;
    let hi = self.starts[c + 1] as usize;
    CellOccupants {
        ids: &self.ids[lo..hi],
        xs: &self.xs[lo..hi],
        ys: &self.ys[lo..hi],
        zs: &self.zs[lo..hi],
    }
}
```
This is where some of the almost-magic happens. The `starts` array holds starting indexes that correspond to a `CellId`'s `index`. With that index, we can just use the `ids`, `xs`, `ys`, and `zs` arrays and use the slice from `lo` to `hi`. 

What comes out of the `gather_into` is a list of _candidates_, not _decisions_. Using my default world sizing parameters and assuming (best case) uniform distribution of entities, a single viewer will gather ~95 entity candidates. The size optimizations done to the wire packet (I haven't shown socket serialization yet... but it's in my head somewhere) based on my napkin protocol limit a packet to around ~58 position updates. So a typical viewer gathers more candidates than will fit in a packet, and something has to decide which ones get dropped.

## Just How Fast Can We Gather?
I wrote a handful of benchmarks here, but I'll spare you some of the eye-watering details. First, I wanted to measure how long it takes to gather from empty cells. This is when the viewer is in the cell but no one else is within the radius.

```
22.96 cells walked, 0 entities   →  94.5 ns per viewer  =  4.12 ns per cell
```

That's 95 ns of the 906 ns uniform case, so the loop overhead itself is only about 10% of the work.

Ok so we know it takes **4.12ns** to sweep an empty cell. So we can subtract that from other cell results for different cell sizes to see how the cost changes.
```
┌───────┬───────────────────┬──────────┬───────────────┐
│ cell  │ entities per cell │ examined │ ns per entity │
├───────┼───────────────────┼──────────┼───────────────┤
│ 64 m  │ 2                 │ 151      │ 7.06          │
├───────┼───────────────────┼──────────┼───────────────┤
│ 128 m │ 8                 │ 185      │ 4.38          │
├───────┼───────────────────┼──────────┼───────────────┤
│ 256 m │ 32                │ 264      │ 3.38          │
├───────┼───────────────────┼──────────┼───────────────┤
│ 512 m │ 128               │ 972      │ 1.81          │
└───────┴───────────────────┴──────────┴───────────────┘
```
What's interesting here is that the cost _per entity_ is 4 times _cheaper_ for the 128-entity cell than it is for the 2-entity cell. This is the **_opposite_** of what I expected to happen.

I expected to see the per-entity cost either stay flat while the per-cell cost went up, or maybe the per-entity cost goes up with number of entities. It going down is surprising but actually makes sense.

Everything has been designed around optimizing for linear, gapless reads. This gives us the opportunity for cache affinity on the CPU as well as avoiding pointer whack-a-mole. So it makes sense that doing the gather in one long, contiguous run without gaps is the most efficient per entity.

The size of the cell isn't the important thing, since we've seen a mostly fixed cost to construct those. The important bit is how many entities sit in one contiguous run. The math per entity is identical in every row of that table, so the math can't be what's changing between them. The variable is the memory layout.

The "crowded town square" scenario, where 8,192 entities are gathered in a single cell, costs **19.46µs** for a single viewer. In a town square the entities _are_ the viewers, so all 8,192 of them do that same walk, which is **159ms** _per tick_ on one core. The budget is **50ms**, so this is a problem. Eight cores brings it to roughly 20ms, which fits and leaves nothing for anything else. And I don't think the priority accumulator, where we limit the packets being sent, is going to help here.

## Wrapping Up
I am continuing on my journey to optimize a distributed system architecture to support online simulations (which includes MMORPGs) at huge, possibly even absurd scale. I want to solve the real, tough, challenging problems in this space rather than just adopting the "build now, pray later" approach.

This post was a bit short, but I wanted to have a nice progression between my first post and some more cool stuff: the _priority accumulator_. This is what will let me make decisions about what packets actually need to get sent to a client, rather then just dumping everything in range. 

What I've realized I have to do before the priority accumulator post is address the elephant in the room: _in my current design, 8,192 entities in a cell puts the gather at more than 3x the per tick budget on one core_. Remember that the game itself has to do work (physics, combat, timers, etc) during that same tick.

I bet an actual smart person would've been able to avoid this detour, but I'm happy I took it. I've learned quite a bit more about how I want to design and build this system since I started adding tests and benchmarks.