---
title: "Building a Seamless Multiplayer Universe"
description: "Supporting massive worlds without load pauses or seam transitions"
published: 2026-09-07
draft: false
tags: ["Gaming", "MMO", "netcode", "Rust", "Umwelt", "simulation"]
---
Back in the gaming days of yore, when we dialed up to the Internet using modems that made strange squeals and beeps and whistles, things were different. We suffered through poor gaming experiences
so you don't have to. You're welcome.

Time was you had to sit and wait while your character moved to a different `zone` in the game world.
We called this activity _zoning_ and it could take so long that we often planned our game strategy around it: make sure you pick up everything you need in the city zone before you leave, because it was
a pain in the ass to have to zone back in.

This wasn't fast travel like teleporting, this was running up against a zone boundary and your game would give you the ancient equivalent of the spinning beachball. EverQuest used to just display a little bit of text in the window like `Zoning...please wait.` This wasn't the worst of it. If you disconnected abrubptly while your character was transitioning from one zone to the other, you could lose progress or equipment. If you hit a zone boundary while an angry NPC was nearby, you could "wake up" on the other side of the zone boundary as a dead body. We had it rough, kids.

There are two competing concerns at work and game developers used to have to choose one or the other, since the available technology rarely made both possible at the same time. In order to provide a massive world for players to explore, games had to split worlds into distinct sections. This allowed the backend to scale and, on early graphics cards and PCs with slow hard drives, also allowed a game to only load small portions of the world at a time. The idea of streaming maps as players drove high speed cars around the world was unheard of.

So far as I've been building [Umwelt](https://kevinhoffman.blog/tags/umwelt/), I've spent a ton of time focusing on how to make a single region server (the "sim") run as fast as possible at absurd scale: handle 10,000 entities in a 128 meter square without slowing down past 20Hz, and still be able to transmit updates to edges and clients without running out of bandwidth.

Now I want to tackle the next step: how do we provide a seamless experience where the player can cross region boundaries (which are hosted by different servers) without them knowing and without anyone around them knowing. This design leverages the fact that I have separate edge servers and reinforces the decision that forbids any one region from knowing about any other region.

To start with, I will divide the universe into a grid, with each section of the grid being called a `square` (to avoid overloading the `cell` terminology from internal geometry in the sim). Each square in the grid can be empty, or it can be _fully_ occupied by a region.

Take a look at the following diagram and I will explain the architecture after:

![Seamless world transition diagram](/images/seam.svg)

Let's start with the two adjacent regions, **Region A** and **Region B**. In the diagram we see that there's an entity with a heading moving toward region A's outer boundary. The first requirement of the system being truly seamless is that players can't know there's a boundary.

This means that a player's visibility radius can't be compressed by a region boundary. Instead, when the entity's view radius collides with a region boundary, the _edge_ creates a shadow entity in the adjacent region. It aggregates position updates from the player's current region as well as those within range from the adjacent region. This makes it so that players can "see across" a region boundary without knowing it's there.

In the diagram, you can see that the region sends `ViewCollision` and `BoundaryCollision` events to the edge. On the view collision, we put a shadow (an observer with no avatar) in the adjacent region. On the actual boundary collision, we perform the `teleport` process that already exists to move players from one region to another. This "migration" clocks in at about **15-20ms**, with a 20Hz single-tick budget of **50ms**. This means that under normal circumstances, it takes _less than a tick_ to move a player's entity from one region to another. While the client is rendering positions and smoothing using extrapolation, this teleportation is invisible. In fact, the edge is a complexity barrier that makes it physically impossible for the game client to see this transition. It just keeps getting position updates in _world coordinates_.

In the diagram, you can see the game view looks like one huge universe to the player. They're unaware of the complexity happening on the back end. This design also works perfectly with _ad hoc_ regions, such as ones that might get spun up for dungeon instances, raids, special event zones, etc. The region simulator doesn't know the difference. 

In the current version of the system, there's no control plane that tells region servers to start or stop. I'll get to that after I've solved a number of other problems.

## Look Ma, No Zones!
This design lets the regions keep working at scale and speed and keeps game clients ignorant of the division of the world into grid squares. Edges can drop and come back up and it doesn't destroy what's being managed by the region simulations. In theory, we can scale not only the number of concurrent players, but the size of the game world itself, just by spinning up more edges and regions.

I've been building a sample game, [Mildew Valley](https://github.com/umwelt-sim/mildew-valley), that exercises **Umwelt** the way a real game would. I'm looking forward to seeing how this new design withstands first contact with a massive mildew valley map, while seamlessly managing internal region coordinates and external/global world coordinates.