---
title: "Eliminating Complexity with NATS"
description: "I was generating massive amounts of complexity until I dumped it all for NATS"
published: 2026-08-31
draft: false
tags: ["Gaming", "MMO", "netcode", "Rust", "Umwelt", "simulation", "NATS"]
---
In my journey building the [Umwelt](https://github.com/umwelt-sim/umwelt-rs) library and associated samples, I spent a ton of time and effort trying to get the internal simulation execution to work with 10,000 entities in a small region of space and still be able to keep up at **20Hz**. 

Once I'd gotten the simulator to a point where I felt I could move on (I still think it needs to be faster, but optimizations cost more than they're worth right now), my next step was to build the communications transport and protocol between the simulation/region servers and the edge servers.

I started with the assumption that the edge servers would communicate with the simulations via TCP, streaming packets back and forth. I'd already optimized the simulator to churn out very dense packets that fit under the MTU of most network cards.

An early implementation was the edge server starts up and it points at a single region. The edge server then receives position updates from, and sends commands to, the region sim. This looked okay in testing, but that was only because I was running a very limited set of tests. The big problem with this is that it meant that every game client on an edge had to point to the same region, forcing game clients to open multiple connections to get to different regions.

I then made it so edges could talk to multiple regions. Now I had a connection-per-region and the edge was relaying the appropriate data. At that point, I had to solve a bunch of problems:

* How does the edge authenticate to the region?
* How does an edge find region servers?
* How do I manage entities moving from one region to another without impacting the game client?
* How do I manage the routing tables for entities? Regions need to send to an edge for a given entity
* Crash recovery. If an edge crashes, how does it come back up? Is it completely empty? How does it rediscover the addresses for the regions? If a region crashes, how do the edge clients react?

This is one of the most common forks in the road for developers. The consequences of this choice could haunt them for the rest of their lives (not me, of course...some other guy). At this point we can see this mounting complexity as a challenge and declare, _"Yes! I will emerge victorious after conquering this complexity!"_ Or, we could see this same complexity and declare, _"No! I refuse to accept that this much complexity is required to solve this problem!"_

So I took a step back and looked at my system from a requirements perspective, without assuming any technology choice was set in stone. I just needed an incredibly fast way to get messages between regions and edges. I needed my solution to support many different kinds of topologies, specifically because the _edge_ should be deployed, well, _at the edge_. I needed my system to be secure and resilient. This backbone needed to be rock solid.

I've done a metric crap-ton of work over the years with [NATS](https://nats.io)[^1], including building a distributed workload execution framework on top of it. In my early decisions about architecture, I hadn't even thought about NATS. I was so focused on the idea that I had to control the speed of the TCP transmission that I felt I _had_ to write it by hand.

So I ran a few benchmarks. It turned out that I could send to NATS at _exactly_ the same rate that I was sending my own TCP packets manually. Over and over again, every benchmark I did showed that the send/receive speed over NATS was within the benchmark's margin of error. Once I saw that, there was much rejoicing[^2].

I set about ripping out all of the comms between the region simulator and the edge. There's nothing quite like the feeling of deleting huge swathes of code and not just maintaining feature parity, but making things better in the process.

I no longer needed to figure out how edges discovered regions. Authenticated to the same NATS operator environment, edges and clusters didn't need to discover anything. Using simple subjects got rid of that problem. I also no longer needed to figure out how to authenticate the endpoints since NATS took care of that for me.

I kept a running list of things that NATS made possible as I did the rewrites. I've kept it as a terse bullet-style so your eyes don't water trying to read it all:

* **Connection management** — No accept loop, no per-edge socket, no connect/reconnect logic. Both sides take a pre-connected `async_nats::Client` and never take the configuration required to make one. Transmission got faster because I removed a map lookup for the edge that owned an entity.
* **Edge discovery** — Wildcard subscriptions (`umwelt.*.edge.{me}.*`) match every region, including ones that are brought online later. No service registry, no advertise address, no DNS. Note that the subject tokens define a _source_ and a _destination_ and NOT a parent and child. So a `umwelt.12.edge.abcdf.state` is a state update message _from_ region **12** _to_ edge **abcdf**.
* **Migration connectivity** — An entity moving between regions now costs zero subscription overhead. The edge is already receiving from the destination before anything else needs to tell it that destination exists.
* **Message framing** — NATS delivers whole messages. The old length-prefixed TCP framing (`wire.rs`) was deleted entirely. ✂️
* **Message routing** — Subject-based pub/sub routes each payload to exactly the right edge. No routing table, no fan-out code, no subscriber tracking.
* **Write batching** — The NATS client buffers internally. One `flush()` per tick replaces three syscalls per payload that capped the old TCP transport at 170k payloads/s.
* **Authentication** — NATS accounts, JWTs, and `.creds` files handle it. The old `auth.rs` bearer-secret checking was deleted. The library no longer enforces any opinions on credentials. The game developer can choose any NATS scheme they like.
* **Reconnection** — `async-nats` retries forever by default. No backoff or retry logic in the codebase. The design contract is that an edge takes a fresh name after a partition so it doesn't resume commanding despawned entities, so this was annoyingly fussy in the original code.
* **Clustering and HA** — Transparent. A three-node cluster matched single-server throughput in the ADR benchmark. The library is topology-agnostic.
* **Edge lifecycle** — No registration handshake, no session teardown. A region learns an edge exists when it first sends a command and forgets it when it goes silent. Starting an edge is a NATS connection and two subscriptions (~3 ms). Stopping one leaves nothing to clean up.
* **Warm-on-approach connections** — The ADR lists four features that existed solely to work around TCP's connection cost, all of which were deleted. These were warm-on-approach, idle-close timer, region-to-address mapping, and advertise address (where I had to rely on configuration to know at what address a given edge could be reached). Getting rid of warn-on-approach is a load off my shoulders. This was going to be where the library detects when an entity is approaching the edge of a region. It then creates and warms up a TCP connection to the adjacent region so it's primed before the entity migrates. I still get a headache thinking about it. Now all of that complexity doesn't have to be implemented.

I could go on and on about some of the more subtle things that NATS gives me, but the list above has the highlights. 

When I talk about this kind of transition with people, some of them object saying that I've now made NATS a SPOF (Single Point of Failure, because we love our acronyms, or WLOA). It's not a single point of failure because of the topologies it supports, and NATS is one of those parts of your infrastructure that just works and never breaks. It's the electricity that you run all of your truly fragile stuff on.

The real takeaway from this isn't that NATS is great technology, it's that it's boring and lets me focus on the really important parts of my application or library. In my case, I've been able to start thinking about the domain logic for the region simulator that I want for my sample game. Had I not switched to NATS, I would still be pulling out my one remaining follicle trying to juggle all the moving parts in the point-to-point TCP architecture.

---
[^1]: Full disclosure, I used to work for Synadia, the creators of NATS. I loved NATS and used it long before that, however.
[^2]: Yay.