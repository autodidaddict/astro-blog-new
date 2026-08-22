---
title: "Baby's First CPU Cache Exposure"
description: "I fixed a performance problem related to CPU cache byte alignment"
published: 2026-08-22
draft: false
tags: ["Gaming", "MMO", "netcode", "Rust", "Umwelt", "simulation"]
---

In my [previous post](/posts/gather-and-notify-subscribers/) I talked a lot about memory layout and optimizing things for linear access, avoiding pointer de-referencing hops, and a bunch of other relatively low-level stuff to try and get the work done per tick down to a manageable level _at absurd scale_.

One of the first "low-hanging fruit" as we like to say in this industry was _parallelization_. In a gather pass, I could sweep the whole thing at once in a single thread, or I could split the work up into `len / cores` chunks and then have a thread-per-core do some work. 

I wrote the following benchmark:

```rust
fn parallel_gather(
    snap: &CellSnapshot,
    vs: &[Pos3],
    subs: &[Subscription],
    per_thread: &mut [DiscoveredEntities],
) {
    let threads = per_thread.len();
    assert!(threads > 0, "parallel_gather needs at least one buffer");
    let chunk = vs.len().div_ceil(threads);
    std::thread::scope(|s| {
        for ((vc, sc), buf) in
            vs.chunks(chunk).zip(subs.chunks(chunk)).zip(per_thread.iter_mut())
        {
            s.spawn(move || {
                for (v, sb) in vc.iter().zip(sc) {
                    buf.clear();
                    snap.gather_into(black_box(*v), black_box(*sb), buf);
                    black_box(buf.len());
                }
            });
        }
    });
}
```
Something I learned while doing this is that the `black_box` function is used specifically to _hint_ to the compiler that it shouldn't optimize away things. Tests have a way of using constant values that are easily optimized out, which would ruin my benchmarks. 

I kept confusing myself (and therefore making stupid decisions) with the `[DiscoveredEntities]` parameter. Remember that this type is actually a wrapper around a `Vec` of entities. `[DiscoveredEntities]` is a slice holding one buffer per _thread_, not per viewer. Each thread clears and refills its own buffer once for every viewer it handles, so 4 threads and 10,000 viewers means 4 buffers, each reused 2,500 times. That's why the thread count is `per_thread.len()`.

The goal of this benchmark is to split the work being done by the original `gather_into` function (unmodified from the last post) into independent chunks and run them in parallel. I want to find out how much faster I can get the gather sweep with the simplest first step of just doing things in parallel.

After running this through a bunch more tests and increasing the sample size, running it multiple times while I guzzled coffee, a conclusion here is that doing this in parallel _actively hurts_ performance. The 2,048 entity town square benchmark goes from 11.75ms serial to 39.65ms parallel at 4 threads. The 8k town square case at 2 threads is 470ms versus 180ms serial.

So, my first thought was, _"Self, what am I doing on each thread that could be interfering with the others?"_

The mutable slice is my _output buffer_. There should be no overlap in output buffers, so I don't _think_ this is what's causing it.

So, my parallel code is making things worse, but it appears as though my parallel code is well behaved enough that I shouldn't be doing anything bad. Time to google this crap.

## Today I Learn About the Perf Command
I spent some time searching and asking some AI assistants about my problem. One thing kept creeping up over and over: [false sharing](https://medium.com/@ali.gelenler/cache-trashing-and-false-sharing-ce044d131fc0). As I understand it, false sharing happens when two CPU caches, which batch load data from their cache in _lines_, modify variables in use by the other's cache. This makes one CPU dump the line while the other re-reads it. If the cores spend more time invalidating and reloading each other's lines than they do processing the data, nothing gets done.

⚠️ **_Disclaimer_**: I only just learned about this while researching my problem. I claim no credit for being innovative or clever by figuring out what _false sharing_ is. False sharing is apparently well known to everyone who does real coding at this level.

So now rather than guessing, I can make changes, use the `perf c2c` (cache-to-cache) command, and actually examine the data that shows the cache line usage per core while my program is running!

Here's some fun facts. CPU caches typically operate on 64-byte chunks. The bigger the chunk, the higher the chance for two caches invalidating each other when work is being done in parallel. Sometimes a batch optimization will grab 2 64-byte chunks at once. If you want to guarantee that two separate structures never share a line, or share an adjacent pair of lines, Rust lets you fix the _alignment_ of that structure. 

My hypothesis: my `DiscoveredEntities` inner `Vec` uses 24 bytes for the header (this is some stuff I googled. I did not know that off the top of my head like the real hoodie-wearing hackers might). If I force the alignment of the `DiscoveredEntities` struct to 128-bytes (the docs say _at least_ 128, since a field needing more alignment would win, but nothing in this struct needs more than 8, so it is exactly 128), then my parallel work should no longer "false share" and bump data out of another core's cache.

Something else I learned that felt important: the _alignment_ does pad the structure. `size_of::<DiscoveredEntities>()` was 24 bytes and is now 128, because Rust rounds a type's size up to a multiple of its alignment. The alignment forces every instance to start at an address that is a multiple of 128, and the padded size is what forces the _next_ instance in an array to start a full 128 bytes after it. So when two get loaded into two different caches, neither should have an overlapping memory address that can cause cache invalidation. The cost is 128 bytes per buffer instead of 24, which at 8 threads is 1 KB.

Clear as mud! 

### Unaligned Benchmark
First, I want to take a look at the cache lines as they operate when I'm not forcing the alignment of my data structures. This is my current "slow" state. To be honest, the idea that I can actually prove something like this (hopefully) is giving me a nerd rush. I rarely get a chance to do the scientific method at this level.

I ran my benchmark with the unaligned build. The shared cache line:

| cache line | HITM share | local HITM | records touching it |
|---|---|---|---|
| `0x5d8e9f2f8480` | 99.47% | 6,024 | 79,823 of 271,016 |

**HITM** here is "hit modified". It's a load that found the line it wanted sitting in another core's cache in modified state. In other words, the **HITM** is bad and we want
as few of them as possible.

Here's a per-offset breakdown within that line. Don't worry if it's confusing, I just barely understood this:

| offset | HITM | stores (L1 hit / L1 miss) | records | what sits here |
|---|---|---|---|---|
| 0x00 | 0.41% | **96.79% / 85.23%** | 808 | buffer A, written field |
| 0x08 | 0.27% | — | 15,584 | buffer A, read field |
| 0x10 | 1.59% | — | 15,987 | buffer A, read field |
| 0x18 | 0.21% | **1.38% / 1.14%** | 213 | buffer B, written field |
| 0x20 | 0.22% | — | 15,162 | buffer B, read field |
| 0x28 | 1.59% | — | 15,722 | buffer B, read field |
| 0x30 | 0.10% | **1.83% / 3.41%** | 218 | buffer C, written field |
| 0x38 | 95.60% | — | 16,129 | buffer C, read field |

All contending code resolves to `umwelt::gather::<impl umwelt::...>`, which is
`gather_into` with `DiscoveredEntities::push` inlined into it. So not only do we know where the bad things are, we know that my `push` call is doing it.

The first table says all of it happens in one place. 6,024 of 6,056 HITM events in the entire program are on a single 64-byte cache line, and a third of all sampled memory traffic touches it.

The stores are only at offsets 0x00, 0x18, and 0x30. This is a stride of exactly 24 bytes, which is the size of a Vec header and therefore of an unaligned `DiscoveredEntities`. The offsets in between carry the loads, which is `push` reading the pointer and capacity before it writes the length.

### Aligned Benchmark
Here's the shared cache line summary with the `#[repr(align(128))]` on `DiscoveredEntities`.

| cache line | HITM share | local HITM | records touching it |
|---|---|---|---|
| `0xffff88f90f6af540` | 7.50% | 9 | 30 |
| `0xffff88f9d37c2bc0` | 3.33% | 4 | 11 |
| `0xffff88f9c163b540` | 2.50% | 3 | 16 |
| `0x7855e020a180` | 2.50% | 3 | 39 |
| `0xffffffffb7409a80` | 1.67% | 2 | 15 |

And here's the per-offset breakdown for the worst line, `0xffff88f90f6af540`:

| offset | HITM | stores (L1 hit / L1 miss) | records | what sits here |
|---|---|---|---|---|
| 0x00 | 100.00% | — | 21 | kernel, unresolved |

And for `0x7855e020a180`, the only one of the five in userspace:

| offset | HITM | stores (L1 hit / L1 miss) | records | what sits here |
|---|---|---|---|---|
| 0x08 | 100.00% | — | 4 | glibc `malloc`, malloc.c:3301 |
| 0x10 | — | 6.90% / 0% | 2 | glibc `get_free_list`, arena.c:712 |

What we're seeing here is that the aligned version is **9.3x** faster, with **50x** fewer cache-line transfers between cores. That's pretty crazy for not changing any code and just forcing a 128-byte alignment!

## Moar Benches
Here is some more benchmark data run against the _aligned_ version of the parallel code so that it can be reliably compared to the timings from the previous post before the parallel enhancement.

| threads | uniform 10k | speedup | town square 2,048 | speedup | town square 8,192 | speedup |
|---|---|---|---|---|---|---|
| 1 | 10.25 ms | 1.00x | 11.82 ms | 1.00x | 180.5 ms | 1.00x |
| 2 | 5.31 ms | 1.93x | 6.13 ms | 1.93x | 95.2 ms | 1.89x |
| 4 | 2.87 ms | 3.58x | 3.31 ms | 3.58x | 50.2 ms | 3.60x |
| 8 | 2.25 ms | 4.56x | 3.44 ms | 3.44x | 53.8 ms | 3.36x |

Against a 50 ms tick budget:

| threads | uniform 10k | town square 2,048 | town square 8,192 |
|---|---|---|---|
| 1 | 21% | 24% | 361% |
| 2 | 11% | 12% | 190% |
| 4 | 5.7% | 6.6% | 100% |
| 8 | 4.5% | 6.9% | 108% |

Times are the mean of two independent 50-sample runs. The speedups reproduce within a few percent between runs while the absolute times drift up to 9%, so I'm paying more attention to the relative timings and ratios than the absolute values.

ⓘ Two things that really stuck out to me this time: 8 threads beats four in the uniform column and loses to four in both town square columns, which held across both runs and all three scenarios. And the 8,192 town square at four threads is exactly at 100% of the tick budget for the gather alone, before any simulation, scoring, or packet work. ⓘ

## Wrapping Up
This experience had a lot more numbers in it than I'm usually comfortable with. I thought I would take the "low-hanging fruit" and ended up deep down a `perf` rabbit hole. As with most of my rabbit hole digs, I came away from this one knowing a lot more and having a far broader perspective on ... everything.

The before and after here is more than enough evidence to prove to me that the `#[repr(align(128))]` really was a fix and not just some random fluke of a non-deterministic test. 

| | unaligned | aligned |
|---|---|---|
| time, 2,048 town square, 4 threads | 32.3 ms | 3.49 ms |
| total HITM events | 6,056 | 120 |
| HITM on the worst single line | 6,024 | 9 |
| loads slow enough to sample (30+ cycles) | 115,262 | 21,806 |

The difference is _so huge_ that 50x fewer loads even qualified for being sampled because there was no contention.

I hope you found some small piece of this enlightening, because I certainly learned a _ton_. I've still got a lot of work to do as the aligned version is still burning more time than I have for the 8,192 town square herd, while the 2,048 town square and 10,000 uniform distribution are all under budget.

My next post will likely be where I try and reduce that 361% budget consumption for the 8k town square on a single core. There's got to be a bottleneck somewhere. My first suspicion is that I'm performing the gather sweep for every viewer. I wonder if there's an optimization where I can reduce the sweep sizes when viewers share the same cells or have overlapping view radii. 

Tune in next time to find out just how wrong I am!