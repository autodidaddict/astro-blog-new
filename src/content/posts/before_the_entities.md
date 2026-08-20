---
title: "Positions Aren't Messages, and Other Early Simulation Server Discoveries"
description: "Doing some evidence-based design on hyper-scale simulation servers"
published: 2026-08-20
draft: false
tags: ["Gaming", "MMO", "netcode", "Rust", "Umwelt", "simulation"]
---

Last November I wrote a couple of posts about building MMO netcode in Elixir. I got a [packet codec and dispatcher](../dispatching_mmo_packets/) working, I did a big [theory dump on movement simulation](../simulating_movement_mmo_server/), and then, like I always do, I got distracted by some other shiny thing.

The distraction was never total, though. No matter how many times I've decided to dabble in the realm of high volume
servers, MMOs, and simulations, there's always been one constant. There's been a design question that nags at
me every time I'm in line waiting for an ice cream or I'm taking a long elevator ride.

> _When something happens in the game world, how do you figure out who needs the update, and how do you do that with tens of thousands of players in the same world?_

When I ask people at the post office or the grocery store this question, I get very strange looks. So this time I decided to sit down and work through the problem on my own, and not just in theory. This time, I was going to _do the math_.

I've since discovered that there's a name for the thing gnawing at my brain. It's called **interest management**. In this blog post, I'm going to talk about diving into this head first so I can learn, explore, build, and maybe talk about what I've learned.

## My Napkin Math Was Perfect and Wrong

In my post about moving massive numbers of objects in a game server, I did some back-of-the-napkin math about a brute force approach: **100,000** tracked objects, **10 billion** iterations, against maybe **2.3 billion** instructions in a **100ms** tick. I concluded that I needed a spatial index and maybe some R-tree sauce on top. I was right, but it solves the wrong problem.

The problem isn't about updating the position of a hojillion things in memory every tick. That's what I've been focused on, but the math proves it's the least of my worries. The real challenge is in _visibility_ and _notification_. What do clients see and how can those clients get the updates they need within the time budget?

Let's say we've got **10,000** players, each of whom can see around **100** other players (or entities) and the server's loop tick is running at **20Hz**. This means **10,000 x 100 x 20 = 20,000,000**. 20 million entity updates per second. _This_ is where I need to spend my optimization time. Another way to think about 20Hz is that every tick only has **50ms** to get _all of its work_ done and _push all of its notifications_.

What's missing from this set of math is the time it takes to discover nearby entities. Given the fastest R-tree sauce and the fastest spatial index in the universe (quantum spatial indexing??), the fact remains that we still have to _produce and ship_ 20 million updates. Spatial indexing makes an already cheap part of the loop cheaper, but does nothing to tackle the hardest part of it.

The way to optimize this problem is by _not sending_ the updates. Figure out every way you can possibly avoid sending a packet, and now we've got the makings of something that can really scale to ridiculous numbers above and beyond what most MMOs today average.

As I go through the post, I'll "show my work" that supports my conclusion. Give each client a fixed byte budget, score every candidate entity by how stale the client's picture of it has gotten, and each tick you ship only the top **N** that matter. Distant entities naturally settle into a low update rate. There's no starving consumers, and no separate code branch for crowded areas vs empty. Per-client cost stays bounded no matter how many people cram into the town square or how many people follow Leroy Jenkins to their doom.

Genius, right? _Of course it is_. Because it's not my idea. It was originally implemented in [Starsiege TRIBES](https://www.gamedevs.org/uploads/tribes-networking-model.pdf), which shipped in December 1998, and written up by Frohnmayer and Gift at GDC the following year. I knew there was a reason I loved that game. As far as I can tell, nobody has come up with anything more than incremental improvements on the design. The Tribes design involves the use of a _priority accumulator_, and that'll probably be the subject of my next post.

## Going BEAMless

I've built a lot of things in Elixir and I'll surely build a ton more. `Horde.Registry` and `via_tuple` combine their super powers to form a clustering easy button. The binary pattern matching used for building codecs is some of the most elegant language syntax I've ever used, without exception. However, for the simulation layer that I'm discussing now, I've called up my old friend **Rust**.

Let's take a look at _why_ Rust is ideal for this problem. Writing a handful of integers into an array takes a couple of nanoseconds. Alternatively, the cheapest possible _message_ you can send starts with an atomic operation, and Jeff Dean's [latency numbers](https://gist.github.com/jboner/2841832) put a mutex lock/unlock at **25** nanoseconds. That's just the starting point, before you add queue bookkeeping or a cache line bouncing between cores. Sending to an out-of-process broker is a different universe entirely: Dean puts a bare round trip within a single datacenter at **500 microseconds**, and that's before any broker, TLS, or client library buffering gets involved. In practice you're looking at milliseconds.

Routing these entity updates through any message-passing system incurs a big pile of penalties in exchange for nothing that I actually need. The simulation writes every entity's position back to back into one flat array in memory. When it's time to build outbound packets, every client's send loop reads from that same array. No new allocations, and no lock on shared mutable state.

You might ask yourself, _"Self, what stops a reader from seeing half-updated data?" The answer is that the writer never mutates an array anyone is reading. The sim finishes a tick, publishes the completed array, and starts writing into a fresh one. Readers always get a frozen snapshot of a completed tick, which is exactly what you want for position data anyway.

Let's take the example of a new player moving in the overcrowded town square with 500 people in "interest range". In a message-passing design, one move produces 500 messages all carrying the same sixteen bytes. Here, the mover writes 16 bytes once and 500 readers go look at them. Nothing is copied, and the sim doesn't send anything to anybody. To be clear, packets still go out to clients with one per client per tick. What disappears is the _internal fan-out_. The sim doesn't send N messages to N session actors which are then forwarded to sockets. The sim writes _once_, and each client's send loop reads from the array to build its own packet.

Contiguous(ness?) also matters. Walking a flat array means every cache line the CPU pulls in is full of data you're about to use anyway, and the prefetcher can see where you're headed. Following pointers between scattered objects instead makes you pay a main memory reference for each miss. Those misses are unnoticeable at small scale and can be a bottleneck at huge scale.

To put a fine point on it, I want to know where my data is, how it's laid out, and how quickly I can read and write it. Rust gives me this. There's no chance of a garbage collection pause making a tick exceed its deadline and cause cascading failures and I even get access to batched syscalls which I can use to optimize network sends. BEAM is still the right tool for my control plane, but that's a topic for another post.

## I Read a 1934 Essay About Ticks

I've decided to call the library I'm building `umwelt`. This comes from an essay by [Jakob von Uexküll](https://germanhistory-intersections.org/en/knowledge-and-education/ghis:document-143)[^1] that starts with a discussion of a tick, of all things. A tick's entire perceptual universe consists of only 3 signals: _butyric acid_, _warmth_, and _hair_. That is the tick's entire world. According to the tick, there is nothing else. You and I inhabit the same universe as the tick, but our perception of it couldn't be more different.

Uexküll's word for the perceptual world a creature actually inhabits, as distinct from the true world out there, is `umwelt`. Every creature has one. They're all different, and they're all incomplete.

This is exactly what the library I'm working on does. Every client gets a budgeted approximation of the same world and no two of them match. As a bonus vocabulary, I get `merkwelt` if I want to distinguish between what a creature can sense versus what it can act on. I'm also going to create a dogfood (or is that bloodfood for ticks?) game called `herd` whose entire job is to generate obscene amounts of load that should crush lesser simulations.

## It's Integers All the Way Down

If I've learned one thing in my time developing software, it's that tabs are better than spaces. If I've learned two things, the second is that floating point will betray you the moment you need two machines to agree on an answer.

To be fair to floats, IEEE 754 is precisely specified, so a given machine running given code produces the same answer every time. The problem isn't accuracy. It's **reproducibility across implementations**, which is just a big computer science topic that's out of scope for this post.

So positions in this design are _fixed point_ (as opposed to _floating point_. See what I did there?). A `Fixed` is an `i32` with 10 fractional bits, so a single scalar value on one axis carries both a whole number part and a fraction in one integer. The 10 fractional bits mean that the optimized divide and multiply (via _bit shifting_) influence the size of the smallest unit of travel. I don't want to do any floating point math, so rather than have an entity move .81 meters, it needs to move a whole number of some other units.

Some nerds (like me) would call this smallest unit a _quantum_ or an _atom_. The engine doesn't give it a name because naming things is hard. Instead, we just know that if we want to divide the world up into some amount of meters that supports even shifting by 10 digits, one meter is 1024 (2^10) of these units. 1/1024th of a meter is about **0.98mm**. So I'm making a huge performance optimization in exchange for one unit of movement being just shy of a millimeter.

I could have gone with 8 fractional bits instead, which gives 1/256th of a meter, or about **3.9mm**. That's still finer grained than anything a player would notice. I went with 10 because the `i32` gives me 22 whole-number bits either way and I'd rather have the headroom in precision than in range I'll never use.

So much math, make it stop!

If you remember my movement post, and I'm sure you've got it memorized, you'll remember that I was using floats in tensors and treated one unit as one meter. Aside from the previously mentioned math optimization, my reason for switching is crash recovery. The plan is checkpoint-and-replay disaster recovery. I dump state periodically, log all the entity inputs, and then on crash reload the checkpoint and replay forward. This is a classic strategy, but it only works cleanly if replays produce _bit-identical_ results.

Floating point numbers don't give me that. The stored bits are portable as an `f32` written on one machine reads back identically on another. What isn't portable is the _result of computing_ on those bits during replay. A compiler can fuse `a * b + c` into a single multiply-add that rounds differently than doing it in two steps. Further, trig and exponential functions come out of a math library that IEEE 754 only _recommends_ round correctly rather than requiring it, so `glibc`, `musl`, and Apple can each hand you a different last bit for the same `sin` call. 😮

Interestingly, `sqrt` is _not_ in that boat. IEEE 754 mandates correct rounding for square root along with the four basic operations, so that one's actually portable. Who knew?

Using integer-only math also gave me an unexpected bonus. Every derived value in the configuration (you'll see that in a bit) remains const-evaluable. Since I don't use `sqrt` or `ceil`, all of my derivation functions stay nice `const fn`s.[^2]

The tradeoff is that multiplication can be a trap. This is why I created an abstraction (yay free abstractions in Rust!) so I only get this right or screw it up in one spot:

```rust
impl Mul for Fixed {
    type Output = Fixed;
    #[inline(always)]
    fn mul(self, rhs: Fixed) -> Fixed {
        Fixed(((self.0 as i64 * rhs.0 as i64) >> FIXED_SHIFT) as i32)
    }
}
```

Here we're not only doing the bit shifting, but we're widening the type out to `i64` prior to that because the 32-bit values can overflow before the bit shift can fix that.

Since _scaling_ a fixed point (e.g. position) by a value is an entirely different operation than multiplying two `Fixed` values, we can use another implementation (thanks, Rust!):

```rust
impl Mul<i32> for Fixed {
    type Output = Fixed;
    #[inline(always)]
    fn mul(self, rhs: i32) -> Fixed {
        Fixed(self.0 * rhs)
    }
}
```

This time there's no shift. The type system will save me from myself here.

## Just the Top Bits

There are a number of settings that are configurable rather than fixed constants. We can choose the size of a region, which is an area of cells, and we can choose the cell size and some other parameters. The default in my library picks a 4096m square region cut into 128m cells. This gives us 32 cells per axis, and since there are two axes, 32 × 32 = 1024 total cells within the region. Don't worry if you didn't come up with those numbers in your head because I sure as hell didn't. 🖩

Here's the bit I had entirely too much fun with (get it? BIT??). A position value along one axis is 22 bits. A cell is 2¹⁷. So there are 2⁵ = 32 cells per axis, and the cell index is literally the top five bits of that position. If that trick went by too fast, I'll walk through the steps.

Keeping in mind that the **4096m** region size is configuration and not a constant, when I say "a position is 22 bits", I'm describing the _value range_: A 4096m region is 4,194,304 units, so a coordinate inside that region occupies 22 bits total. Those 22 are 12 integer bits plus 10 fractional ones.

Here's the actual layout:

![bit positions](/images/position_bits_cell_meters_fraction.png)

We shift away 17 bits, which is 7 bits of whole meters and 10 bits of fractional ones. A 128m cell is 2⁷ meters, so the cell size in meters and the width of that middle field are the same fact from different points of view. Change the cell size to 256m and the middle field grows to 8 bits while the cell index shrinks to 4, giving 16 cells per axis. The 10 fractional bits never move.

Let's do a concrete example. **300544** stored in a single `i32`.

```
   00010  0100101  1000000000
   └─5──┘ └──7──┘  └───10───┘
   cell 2  37m      0.5m
```

And now in human-readable table form:

| field | bits | value | means |
|:--|:--|:--|:--|
| cell index | 21–17 | 2 | cell 2 |
| meters in cell | 16–10 | 37 | 37m into that cell |
| fraction | 9–0 | 512 | 512/1024 = 0.5m |

Note that this is a value _along a single axis_. Storing **300544** in the `x` field of a `Pos2` means we are 293.5m along the x axis, which is 37.5m into cell 2. There's a separate 22-bit value for `y`, and a third for `z` with its own extent. The 32-cells-per-axis figure becomes a 1024-cell grid only when you pair the x and y cell indices together.

One small little caveat. _**If**_ the game had compile-time constant values for the grid size and cell sizes, then we could use regular division and the compiler would give us the shift optimization for free since LLVM turns division by any constant into a multiply-and-shift. Since these are runtime configuration, the compiler can't do that for me, which is why my `build()` function rejects any cell size that isn't a power of two. The cell-index-in-the-top-bits trick works either way.

Your assignment is to do this homework problem in reverse. Enjoy, kids![^3]

## The Battle of the Bounding Box and the List

A viewer subscribes to every cell within `cell_radius` of whatever cell they're standing in. By "subscribes to", I mean that the viewer has a vested interest in any _changes_ happening in any of those cells in range.

My first instinct was to store this subscription as a list of cell coordinates. A `Vec<CellCoord>` allocates, which I don't want happening ten thousand times per tick, and a fixed-size array means picking a maximum `cell_radius` up front and paying for the worst case on every viewer. At the default radius of 2 that's 25 cells, but bounding it for a radius of 4 means every subscription carries space for 81.

Then I noticed the set of cells in range is always a rectangle, so I don't need to store the cells at all. I can store the four edges:

```rust
pub struct Subscription {
    pub x0: i32,
    pub x1: i32,
    pub y0: i32,
    pub y1: i32,
}
```

This is always sixteen bytes, no matter how many cells are inside that bounded region, and no matter how many entities are inside those cells. Better still, asking "is this cell in range?" becomes four integer comparisons rather than scanning a list.

## Benchmarking and Evidence

With enough of the basic math available to me in Rust code, I could write some benchmarks. What I want to measure is how long it takes to do specific tasks within a full tick's worth of subscription work. We've got **N** viewers, one pass, with each viewer doing progressively more work as identified in the table columns. So, **N=10,000** means 10 thousand viewers each had their bounding box rebuilt from scratch during that tick. The "per viewer" number is the row's total divided by **N**.

If I know how long it takes to do the work, then I can figure out how much of my **50ms** budget (**20 Hz**) is being taken up by that task. This will tell me where I have room to optimize and where complexity trade-offs might pay time dividends.

The columns are cumulative — each one includes everything to its left.

| viewers | build the box | find the cell, then build it | ...then walk its cells |
|---|---|---|---|
| | *4 adds, 4 clamps* | *+ load x/y, shift to cell index* | *+ visit ~23 cells* |
| 100 | 1.51 ns | 2.65 ns | 28.7 ns |
| 1,000 | 1.62 ns | 2.72 ns | 28.6 ns |
| 10,000 | 1.50 ns | 2.78 ns | 29.1 ns |
| 100,000 | 1.60 ns | 2.65 ns | — |

The membership test (_"is this cell inside this box?"_) takes a flat **0.67ns**. In fact, all three subscription operations are flat across three orders of magnitude. This is the kind of math I can get behind. Each viewer's work is bounded and independent of every other viewer's, so per-viewer cost shouldn't drift.

There's no cache "cliff" in the per-item cost at 100k either. A cliff is what you see when your working set outgrows a cache level and per-item cost jumps rather than creeps. This makes sense when we do the math (again): **100,000** viewers is **800KB** of position data against **8MB** of L3 cache. If I'm bored I might add a 1 million case to the bench, which would be right at L3 capacity, and see where the curve actually bends.

The cell iteration math checks out, too. The mean subscription size over a 32×32 grid at radius 2 is 23.16 cells once you account for edges and corners getting clipped. 28.7 ÷ 23.16 = **1.24ns** per cell, so the loop is doing real work in the benchmark and wasn't optimized out.

The net result is the fantastic part, though.

_**10,000 viewers cost 27.8 microseconds per tick against a 50ms budget**_. That's just **0.056%** of my tick budget. I've got money to burn, baby!

## Killing the Darlings

I had a whole plan for the next feature where I update subscribers. When a viewer moves, you need to know which cells they entered and which they exited, because that's what drives spawn and despawn on the client. The obvious approach is to build both bounding boxes and diff them. But since a single-cell move only ever changes one row and one column, you can compute just those two strips directly and never build either box at all.

That trick depends on an invariant, and it's one my config already enforces. At 40 m/s on a 20Hz tick, an entity moves 2m per tick against 128m cells, so it can only ever cross one cell boundary per axis per tick. My `build()` refuses to construct a config where that isn't true:

```rust
if max_move_per_tick.raw() as u32 >= cell_raw {
    return Err(SpeedExceedsCellPerTick {
        move_per_tick: max_move_per_tick,
        cell_size,
    });
}
```

I'd been calling the fast version the "strip delta" and I was looking forward to writing it. It was going to be so clever[^4]. I had the diagonal-corner edge case worked out in my head. I had a testing strategy where the naive version acts as the source of truth for the fast one. It was going to be great.

Rebuilding a box measured **1.5 nanoseconds**.

The strip version might get that to 0.5ns. At ten thousand viewers that saves 10 microseconds per tick (two one-hundredths of one percent of the budget) in exchange for two nasty edge cases and an entire truth test suite to prove the clever one agrees with the boring one.

So I'm skipping the clever and keeping the simple. The delta function still has to exist, but it's going to be the dumb implementation: build both boxes and test membership in both directions. That's roughly 23 cells checked against the old box plus 23 against the new one, so about 46 comparisons at 0.67ns, plus building two boxes. At ten thousand viewers all moving at once, that's 340 microseconds, or 0.7% of the tick. Still got money burning a hole in my pocket!

The key point is that this is measuring _cells_, not entities. The cost is identical whether there are 10,000 entities in a cell or 10. We'll talk about gathering entities when I post about the _priority accumulator_.

A while back I built a beautiful hand-rolled state machine parser for my OCaml Redis and then deleted it when I realized `Eio.Buf_read` _already handled_ the hard part. I don't lack for evidence when backing up my "clever is killable" stance. That one I killed because I found out the work was already done for me. This one I killed because I measured it.

The difference this time is that the evidence took ten minutes to collect. Seems like a pretty good trade for not maintaining an optimization forever.

## Wrapping Up

This post has been all about how to split the world up into manageable chunks and have viewers declare
their interest in what's happening inside those chunks. Everything I've measured so far is viewer-side bookkeeping that produces cell bounds. I haven't actually done any peeking at the entities _within_ a cell.

Subscription is O(N) in viewers with a fixed per-viewer constant so crowding doesn't change the number of cells in a bounding box. A zombie horde of 5,000 costs exactly what 5,000 scattered players cost. Gathering entities is O(viewers × entities per viewer).

In my geek shopping for next post, I'm going to need a cell index mapping cells to entity IDs, entity storage in a struct of arrays, and a gather pass that walks each viewer's subscribed cells and computes distance to everything it finds. That's the first thing in this whole project whose cost scales with crowd size instead of viewer count, so if I do it wrong, it could completely kill my ability to scale.

When I write the bot harness I'm going to make most of the population converge on a single point because the p99 from having 8 entities in each cell won't tell me anything.

Tune in next time to find out whether I was right about any of this.

---

[^1]: An interesting side note is that this man's signature is on the German professors' document supporting Hitler. That doesn't change the meaning of the word `umwelt`, which is ordinary German for "environment" or "surroundings" and long predates his use of it as a technical term.
[^2]: `const fn` isn't really an optimization as the compiler already constant-folds regular function calls when the arguments are constants. What it buys me is the _capability_ to call these functions in `const` declarations and array lengths, which regular functions can't do.
[^3]: Remember when we were kids and we told ourselves that computers were fast enough that we'd never need to bit twiddle ever again? Good times.
[^4]: The moment something becomes clever, that's when we need to start seriously questioning it.
