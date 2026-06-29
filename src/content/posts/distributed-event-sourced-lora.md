---
title: Creating a distributed, event-sourced data store on a LoRa radio mesh
published: 2026-06-29
draft: false
tags:
  - Event sourcing
  - LoRa
  - Reticulum
  - mesh
  - meshlogd
description: I've started to build a no-leader, distributed, event-sourced data store that works over radio with no internet.
---
You're in the middle of a disaster relief effort and you're trying to control logistics in an actively hostile area. You've got hundreds of volunteers, you've got a laptop on solar battery at the base camp and you've got multiple stations that are handing out supplies. You've also got multiple teams dedicated to search and rescue.

There's no Wifi and all the nearby cell towers are either broken or have no power. There's no connectivity or access to the cloud. Search and rescue teams are trying to keep track of the progress of their grid search and information on the people they've found. Each of the supply relief stations needs to keep track of their inventory and you need to see that in realtime so you can plan additional bulk supply deliveries. 

Throughout the day, people need to report when and where they've spotted hazards like down (and still live) power lines, broken water mains, possible gas leaks, etc. Looting needs to be prevented and also tracked.

You're standing thirty feet from a colleague, close enough to see each other. You need to get a data update from your phone onto theirs. You can't. There's no Wi-Fi, so your normal tools are dead. There's no cellular service. And most of your teams are scattered well past Bluetooth range. You're surrounded by perfectly capable devices with no way to talk to one another.

You've heard that one of those generator-powered mobile emergency Internet units might be on the way, but there's no ETA. What if you had Internet, but using it was a risk? There are times and places where using Internet from the field can put people in danger. Protests and other civil actions might be in danger if a hostile government controls the Internet.

Now we get to the part where you say to yourself, "There has to be a better way."

## Low-bandwidth Radio Mesh
There is a type of radio communication called [LoRa](https://en.wikipedia.org/wiki/LoRa). The word is a portmanteau of _Long Range_. There are a couple of things that make LoRa an incredibly appealing data transport. First, it uses license-free spectrums (they vary depending on where you are). This means that you don't need to pay a fee to use the spectrum, nor do you need a special HAM-radio license to communicate on those frequencies.

LoRa is, as the name implies, _long range_. It is also _low bandwidth_. The speeds are measured in Kilobits per second. The data rate is typically between 0.3 kbps to 11 kbps, with one region supporting 50 kbps and regulations in some regions set a floor on the data rate.

When I first discovered LoRa, I thought it might be a cure-all. It does solve a ton of problems, but not without cost. In addition to the low bandwidth, LoRa transmissions can fail, causing LoRa-based networks to partition frequently.

## Ditch the Sync
When trying to apply LoRa to the problem described at the beginning of this post, it's only natural to ask _"How do I synchronize nodes on a network like this?"_ We might start designing a protocol that has a complex system of retries, back-off periods, buffers, caches, and _conflict resolution_.

If you're familiar with any of my rantings and even some books, you'll know that I'm a huge proponent of event sourcing. Event sourcing in a situation like this seems like a way to remove the complexity of building our own sync protocol.

If every event in this network is _immutable_ and _content-addressable_, then the event log (history) is just a [grow-only set](https://en.wikipedia.org/wiki/Conflict-free_replicated_data_type#G-Set_(Grow-only_Set)), and merging data from two nodes is a deterministic _set union_ operation. A union like this is _automatically_ conflict-free, idempotent, and order-independent. Convergence stops being something that the code has to perform and instead becomes a property of the whole system.
 
## Creating Radio Meshes
If you think about a LoRa device as a network card (or, if you're old enough, a modem) it might help to visualize this. A LoRa board is an antenna, usually some kind of power source, and a microcontroller running firmware that drives the radio: how it behaves on its spectrum, when it transmits, how it frames packets.

That firmware layer is where a lot of off-grid projects live. [Meshtastic](https://meshtastic.org/), for example, is firmware (plus an app and a protocol) that turns these boards into a simple messaging mesh. But firmware that drives a radio was only half of what I needed. I also needed an actual network stack on top of it that provided identity, encryption, addressing, routing. These are all provided by a different layer.

That layer is [Reticulum](https://reticulum.network/). Reticulum is a networking stack that runs on the host (a phone, a Raspberry Pi, a laptop) and talks to the radio over USB or Bluetooth. The radio itself runs firmware called **RNode**, which turns a $15 commodity LoRa board into a Reticulum-speaking modem.

Reticulum is leaderless by design. There's no root authority for anything in Reticulum: no DNS, no DHCP, no central _anything_. Networks self-configure and self-heal. Its Interface Access Codes (IFAC) let me control which devices are allowed onto a given virtual network, again with no central authority.

Reticulum had already solved the parts of this problem I didn't want to build on my own like multi-hop routing, store-and-forward through intermediaries, the RNode firmware that makes cheap hardware usable, and a working app ecosystem (Sideband/LXMF) that proves the whole stack runs on real hardware, including Android.

Out in the field, this is what lets volunteers securely join a mesh and exchange data while the network constantly reshapes via devices dropping off or transmissions getting lost, without anyone having to think about it. We can finally move data between people hundreds of meters, even kilometers, apart without routing it through the cloud.

Meshtastic and Reticulum are both worth your time if you're diving into rabbit holes in this space, but note that they sit at different layers. I chose Reticulum mostly because of the layer in which it sits.

## Enter meshlogd
I've started working on a new project called **meshlogd**. It is an event-sourced, decentralized, distributed data store that converges over a LoRa radio mesh with no servers or Internet.

You have a mesh full of devices, each publishing events. Sometimes there will be no other radio in range to "hear" the event, other times partitions might happen multiple hops along the trajectory as the event publishes. 

If you're building an application on top of **meshlogd**, none of that matters to you. It's all taken care of. You can write code that runs against bound LoRa devices that publishes sensor reading events. No matter how the network reshapes and re-organizes, eventually, all devices in the mesh will have the same view of shared state.

To make **meshlogd** work, I needed a _dial-tone_ (another old people reference), not an application. Reticulum is the dial-tone that makes meshlogd possible.

## Publish and Reduce
I've hidden the manual creation of CRDTs and all of the low-level details and simplified the developer experience down to just two activities: _publish_ and _reduce_.

* **Publish** - Emit your own domain-specific events. Meshlogd will wrap it in an envelope, but you can use any shape you like on the wire[^1]
* **Reduce** - You can also call this _fold_, depending on which lexicon you like best. Your application provides a _reducer_, a pure function that accepts the current state and a received event and produces new state. This is the core of how event sourcing _applies_ events to state.

This is the power of event sourcing, magnified as it is layered on top of a secure, radio-based mesh network. Event sourcing is what makes possible the reliable developer experience I want.

### Some Use Cases
In the sample I described at the beginning of the blog post, your application might be emitting events like `supplies.dispensed`, `medical.event.occurred`, `survivor.spotted`, `grid.searched`, etc. Your reducer function would use these events to update remaining stock on hand, current survivor locations, and search status within areas. That's all your app needs to worry about.

In a search and rescue or field expedition app, you might have events like `waypoint.created`, `waypoint.moved`, or `team.checked.in`. Deploying scientific instruments into the field might produce `sensor.reading` events or `sensor.failure`. 

You can even have relatively boring off-grid chat with events like `message.posted`, or `channel.created`. All the while keeping in mind that meshlogd is designed for secure and verified author identities and authorization enforcement.

## Summary and What's Next
This blog post was just to touch on the fact that I'm building this eventually-consistent, always-convergent, secure, distributed event log. I wanted to touch on the motivation for building it and what it might look like to build an app using it.

I've got tests that prove the log works and I've run simulations that show convergence across hundreds of random partitions, but I've yet to run this against real radios. I'm planning on assembling a batch of test radios tonight.

I've created some open source repositories for this and I will be sharing them soon (once I am less embarrassed by the code). In the next few blog posts, I'll describe the _how_ and _why_ of the implementation.

---
[^1]: _Within reason_. Just because the developer experience is great doesn't mean that you can ignore the constraints of a slow-data-rate and highly partition-prone network.
