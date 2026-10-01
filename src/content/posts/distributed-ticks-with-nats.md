---
title: "Idempotent, distributed ticks in a cluster with NATS"
description: "Solving the distributed ticks problem with NATS, JetStream, and Go"
published: 2026-10-01
draft: false
tags: ["NATS", "Go", "Distributed Systems","Gaming", "simulation", "NO CARRIER"]
---

The "distributed tick" has always been one of my favorite (and most challenging) distributed systems problems. If you have many instances of the exact same code running in different places, how do you have code that runs on a timer? How do you "tick" once in the cluster and only have work appear once? If tick processing lags, how do you not end up with a cascading stampede of pending ticks? The problem is far more complicated than most of us give it credit for.

There's a subtle assumption here when I say _"exact same code running in different places"_. In this situation I mean the exact same binary with the exact same logic. This binary could be doing leader election to figure out where the real worker is, or it could have logic like `if !(primary) {...}`. I've built production systems using the latter and I regretted that decision almost from the moment of deployment.

I consider any code that has to branch based on some detected state, role, or hierarchy to be a liability. Each time the code branches that way, it produces an entirely new tree that I have to cover with tests and hope I catch all the edges. If one instance out of 12 in a cluster fails, now I have to capture which branch/role/state that instance was in when it failed and which of the thousands of possible paths it took to reach that point. This kind of thing looks simple when we're building on our workstation but it's a nightmare after the first release.

During another weekend where, for some inexplicable reason, I had some spare inspiration and started yet another side project, I bumped into the "distributed tick" problem. Again. In [another side project](https://kevinhoffman.blog/tags/umwelt/), I'm building dedicated processes that tick 20 times per second in memory. That problem isn't in the distributed systems category, it's a speed and efficiency problem where every extra blocking call can ruin the game's ability to keep up with load.

The distributed tick is slower and requires less accuracy. This tick needs to be guaranteed to happen, needs to not duplicate work, and needs to recover from crashes. Just running a fast loop in a single node's memory won't do the trick.

## How Not to Tick
A first stab at this problem might include just having a process emit the ticks on a message bus. This does work, and I've seen production apps written this way. However, it usually requires having either a separate "ticker" binary, or having a branch and state switch gated with a `if (imTheClock)` check. I don't like either of those options and neither pass the SPOF (Single Point of Failure) test.

What if each node then starts its own timer and performs the work every minute? In this scenario, we'd have to ensure that the clocks of each node are ticking at the same rate, which likely involves NTP or some other clock sync between VMs. There are so many ways this can go horribly wrong I don't think I can enumerate them all. The first issue is that you'd have to synchronize the ticks in addition to the clocks. If every node is ticking every 60 seconds, but they're all doing at _at different offsets_, then we get duplicate ticks.

Here's a fun fact. Did you know that even if you're not using something like an AWS _spot instance_, that your cloud provider likely can and will _pause_ your entire VM? Your code can pause for 5 seconds and then wake up with the clock only showing a 1 second gap. Those kinds of gaps usually involve a race of aliens and require either Guinan or Commander Data's internal chronometer to solve.

## Hail to the King
If you've been around the distributed systems block a few times, then you're probably thinking, _"just do a leader election, problem solved!"_. Let's play this out and see how it fits. We grab a **Raft** leader election library off the shelf, put it in our code, and now all nodes participate in leader election. Remember my earlier objection to having branches like `if !(leader) { ... }` in the code? Raft is well-known and the implementations are ubiquitous, so I'm not worried about the logic being wrong. But I'm still worried about distsys failure.

Let's say everything is going well. We've got 5 nodes, the leader election has quorum, and 1 of the 5 nodes is now emitting the single tick. (hint: _if the tick is work dispatch, we might have a better way to solve this_). The tick emitter crashes and now we have to wait some amount of time for a new leader to be picked. The new leader takes their rightful position on the Throne of Clocks, figures out how far behind they are, and starts pumping out new ticks.

This is fine, and we routinely see raft picked as a way of keeping identical nodes in their place. However, I've also seen leader election fail to pick a leader. I've seen it get stuck in a cycle where a leader is elected and then another leader is elected (called "flapping") and so the group has to re-decide. All of this can take long enough to miss a tick period. That's also no big deal, you just write some code where when a node assumes the leader role, it figures out the time until next tick, and the number of missed ticks. For missed ticks, do you just burst those out or do you ignore them and a missed tick is never dispatched? Also, now we have `when (became_leader) {...}` code, which is another branch chipping away at my code coverage and confidence.

## Clock Ticks are Work Dispatch Loops
When I got down to the bottom of the leader election rabbit hole, I'd added a pile of complexity I wasn't comfortable with. I had to keep track of the current leader, I had to manage leadership and quorum metadata. I was instinctively reaching for `etcd` to help me deal with this. Then I remembered that [NATS](https://nats.io) JetStream streams already deal with writer-leaders and already give me things like sequence numbers at different scopes. JetStream can help me manage the dispatch part of this. But how do I do the clock-based looping?

When I started thinking of this problem as work dispatch, the solution was to double down on my use of NATS and realize that a single point doesn't _emit_ clock ticks, each worker attempts to _claim_ or _win_ a clock tick. The tick winner performs the work, publishes the work they did, and another worker can try and claim the next tick. There's a certain elegance to this kind of simplicity that you can feel when you see it, especially as a distributed systems problem. 

Now we have a design where every single node is running its own internal clock timer. We don't need clock synchronization between machines nor do we need lock-step synchronization so everyone ticks at the exact same moment. When the node's timer fires, it can try and win/claim the next tick. If it can't claim it, then it happily ignores that tick and will try again next time. Only the "tick winner" gets to do work. Because the nodes are the timer tickers, then every node but 1 can crash and the ticks will continue.

## Implementing Tick Loops on NATS
Now that I've walked through the problem and some of the solutions I rejected, let's talk about how NATS made this solution pretty easy, but still required me to do a little bit of work (in this case, in Go).

I'm going to start off with a stream called `CLOCK` with a subject of `clock.universe`. I can get away with this being a durable stream because there are a small number of ticks (one per minute is a tiny amount). I'm using a universe here but it doesn't have to be game-themed. This stream starts with a `UniverseCreated` event that carries some initial parameters, like the period between ticks. Thereafter, this stream will contain `TickAdvanced(T)` events where `T` is the tick number. Each advanced event contains the instance ID of the node that "won" that tick.

Each time a node starts mid-game, it replays the `CLOCK` stream to the head before it starts any countdown timer.

Each node is running the exact same loop:

1. A `TickAdvanced(T)` event arrives
2. Start a countdown timer. If my ID is the instance ID in the event, then I count down **60 seconds**. If not, then I count down **62+jitter** seconds.
3. If a newer `TickAdvanced(T+1)` event arrives before my countdown ends, toss the countdown and continue with the loop from step 1 with the new event.
4. If the countdown ends without receiving a newer event than `T` (e.g. T+1 or later), then try to append a new `TickAdvanced(T+1)`. The stream will take exactly one append because I'm using the `Nats-Expected-Last-Subject-Sequence` header set to the stream sequence of the `TickAdvanced(T)` message. The stream leader's compare-and-set is what creates the duplicate tick prevention.
5. If I'm the winner (I published the `TickAdvanced` event with my instance ID), then I will dispatch the commands associated with `T+1`. On an ambiguous publish error, re-read the head, and if it's `TickAdvanced(T+1)` carrying my ID, I won and must dispatch.
6. Crash recovery comes from the standby timers. If the tick driver (last winner) dies, no event arrives, the standbys' longer (62+jitter) timers fire, and the first guarded append to get through wins, designating the new driver. The cost is the grace period (e.g. 2 seconds) paid once.

## Ticking the Universe in the NO CARRIER Game
I've started building a game that uses this kind of tick supported by NATS and JetStream, called [NO CARRIER](https://github.com/nocarrier-ai/nocarrier). In this game, players interact with the game by submitting standing orders (AI prompts) that dictate the behavior of their fleet. Every game tick, each of these admirals (an agent) decides on its action plan and the game executes some of that plan.

I need an idempotent distributed tick so the agents don't make decisions twice (because the decisions aren't likely to be identical) and so they don't consume metered resources like an LLM or Jev multiple times per tick.

The following is an architecture diagram that shows how players use the web (or **`telnet`**!!) to get into the game. 

![Deployment architecture for the NO CARRIER game](/images/nocarrierd_deployment_architecture.svg)

The players publish changes to their fleet admiral's standing orders and that dictates what happens during the game loop, which is progressed forward in an idempotent fashion by the swarm of `nocarrierd` processes. These processes don't need to talk to each other or discover each other. They don't need to form a cluster. Everything they need from the communications and storage substrate they get from NATS and JetStream.

## Wrapping Up
I've lost count of how many times I've been down this road before. I start designing a system and I encounter a pretty nasty problem. I try and solve the problem iteratively and, before I know it, I'm drowning in accidental complexity. 

These days I know enough to see if using NATS will take care of some of that complexity. Pretty much every time, it does the trick. It would be really easy to just accept the complexity and hand-roll a solution to it. After all, that's what us developers are supposed to do, right?

There's some measure of pride that comes from encountering a tough problem and coding a solution to it. However, that kind of pride often comes with a blindspot that obscures the practical state of the situation. Can someone else's tool solve it? Can someone else's highly specialized and battle-tested technology either solve my problem or make it like I never had the problem to begin with?

Even if I don't know if there's a magic bullet when I start, I usually spend time researching and digging around. I'm never going to be the first person to have encountered and solved a problem, so I might as well learn from how other people dealt with it and pass that benefit on to the users of my software.
