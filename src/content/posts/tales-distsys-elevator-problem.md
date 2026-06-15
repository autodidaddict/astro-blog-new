---
title: "Tales from the Distributed Systems Crypt - The Elevator Problem"
published: 2023-08-03
#author: "Kevin Hoffman"
type: post
description: "Tales from the Distributed Systems Crypt - The Elevator Problem"
tags: 
    - Distributed Systems
    - MUD
---
_In this post I talk about learning a valuable distributed systems lesson the hard way._

It was a cold winter morning on a weekend back in <font color="red">**_(redacted)_**</font>. I was sitting at my desk in my dorm room hacking away on my favorite [MUD](https://en.wikipedia.org/wiki/Multi-user_dungeon) (_Multi-user Domain/Dungeon_), **Genesis**. I would eventually become the _Archwizard of Code_ for that MUD, but not this day... Nay, this day I would stumble headlong into my first hard-learned distributed systems pattern.

For those of you not familiar with the classics, at that time a **MUD** server was a single Unix process running on a machine, typically hosted at a University that graciously donated the compute resources. This meant that everyone playing the game was doing so against the same monolithic process. MUDs would routinely restart because the server process eventually consumed too much RAM. You had to use a `telnet` client to connect to the MUD (TCP sockets) or you could use a MUD client that did the telnetting for you.

The appeal of these games can't be overstated. They were multiplayer games when the best console device you could get at the time was an 8-bit Nintendo Entertainment System (**NES**), which didn't have a modem, let alone the ability to handle networking.

I had literally spent months writing code for an area of the MUD. It had its own weather system, multiple guilds, tons of really cool NPCs, it was fantastic and I was proud of this work and I couldn't wait to show it off to players. So I got to work building the grand entrance to this area. Players would fight their way through to the base of a cliff as tall as "the wall" from Game of Thrones (of course this predates GoT). They could pull a lever that would raise a platform from the base of the cliff to the top. In short, it was an elevator meant to be the first thing players would see when they entered the area. It was the first impression I couldn't mess up.

So naturally I messed it up.

Most MUDs are made up of _rooms_, in which players reside. To get to different rooms, players need to go through exits, which can invoke custom code (in my case written in [LPC](https://www.cs.hmc.edu/~jhsu/wilderness/basics.html), a variant of C designed specifically for this class of MUD). In my setup I had `lift.c`, the room that held players on the lift, `cliff_bottom.c` and `cliff_top.c` - three rooms.

Operation should be straightforward, right? Player enters `cliff_bottom.c`, pulls the lever, and the lift moves up to `cliff_top.c`. Easy peasy.

Except it wasn't.

The core of this problem is shared mutability. The lift is a shared resource. There's code in the lift that checks internal state (`position` could be `TOP` or `BOTTOM`). Players can pull the lever at the top of the cliff to beckon the lift. Players at the base of the cliff can do the same thing. My problem was that they could do this while the elevator was in "motion". To simulate the motion I had a timer that would change the lift's position when it expired.

To keep players from falling into the void, they could only step onto the lift (via the `lift` exit) when the lift was in the right position. The lift position had to be `TOP` for folks in `cliff_top.c` to get into the lift and `BOTTOM` for folks in `cliff_bottom.c` to get into the lift. The problem arose when people at the top and bottom of the cliff were pulling the lever (the equivalent of pushing call buttons on an elevator) at the same time. 

The code was clearly written by someone who had never dealt with shared mutability before (me). 

```c
void on_lever_pull() {
    if (position == TOP) {
        position = BOTTOM;
        call_out("arrive", 5);
    } else {
        position = TOP;
        call_out("arrive", 5);
    }
}
```

The code would set the position of the lift (the shared resource) blindly and then "arrive" at its destination 5 seconds later. The problem was that if someone pulled the lever at the top of the cliff, the lift would start moving up. If someone pulled the lever at the bottom of the cliff, the lift would start moving down. If the lift was already moving, the position would be set to the opposite of what it was before. This meant that the lift would never arrive at its destination. It would just keep moving up and down forever.

It was even worse than this. Due to some internal details I didn't know about, it was entirely possible to create too many timers. After enough people on the top and bottom pulled the lever, my code would no longer be allowed to use timers. Remember when I mentioned earlier that the MUD was a single process? You can't have a single process that lets untrusted code starve resources, so naturally everybody's code had quotas.

Once I exceeded my quota, the lift could be in a state where `arrived` was `false` and there were no timers pending to change that state. The code in the exits made sure to check that both `arrived` was true and the position was either `TOP` or `BOTTOM`. This made sure that players could never get onto the lift in this state, _nor could the players already in the lift get off_.

In many solutions to the elevator problem, the elevator has a default state (floor) that it will attempt to return to when it has no other active requests. When the elevator is moving up, it has a destination in mind and will stop at all floors greater than the current floor and less than the max floor. The inverse is true when the elevator is descending. If there are no floors actively requesting the lift after some timeout period, then the elevator returns to the default position. In my case, that would've been the cliff base.

In this solution, new resources (like timers) are not created every time a lever is pulled or a button is pushed. Each floor's status is either on or off. The elevator, which has its own internal state, decides whether to stop based on the state of each floor. Because each floor is a single room with a single button/lever, each floor is now an isolated mutable resource that can be managed using standard player command queue mechanics.

The problem gets even easier to solve if you embrace an event sourced approach and the lift and floor lobbies are separate aggregates determining their state from event flows, and summoning an elevator involves making a command request. Duplicate requests to summon the elevator to a given floor are just discarded.

As a new developer, I never asked myself questions like:
* What happens if someone wants to mutate state while a timer is active?
* What happens if multiple players try to mutate the same state? 
* How can I avoid the state ever getting into an inconsistent mode?
* Do I need locks to prevent concurrent mutation, or can I design my way around a distributed lock?

On opening day for my new area, I trapped 10 people on the lift. They couldn't get out. People on the top couldn't leave the area because the lift was stuck in an indeterminate state. People on the bottom couldn't get in because the lift was never available there, either. 

In the end, after teleporting my victims out, I opted for a magical portal that instantly whisked people from the bottom to the top and back again. This was entirely multiplayer safe and didn't attempt to maintain a shared mutable state. These days regardless of what my actual distributed systems problem is, I ask myself, _"Can this trap people in an elevator?"_ as a metaphor for the entire class of shared mutable distributed state problems that everyone (including me) always overlooks.

