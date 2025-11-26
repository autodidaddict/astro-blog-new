---
title: Dispatching Network Packets in an Elixir MMO
published: 2025-11-26
draft: false
tags:
  - Gaming
  - MMO
  - netcode
  - Elixir
description: A recent milestone in my tinkering with an MMO backend is dispatching network packets to interested parties
---

Some of you may be able to relate to the idea that I am perpetually starting new side projects. I rarely ever finish them, but most
of the time I learn a bunch of useful things along the way and, if I'm lucky, I might even have some fun.

I tend to favor distributed systems when I spin up a brand new side project, and this time is no different. A few weeks ago I was sitting
in the car waiting for someone and an idea hit me: _I should build the backend netcode for an MMO in Elixir_. I've made MUDs in Elixir and
I've made gaming backends that I would consider toys, but I've never actually made a "real" gaming server with raw TCP and UDP packets and
full clustering, sharding, zones, etc.

After doing a little poking around and setting up a scaffold project that used Erlang networking and the `Horde` library for self-forming
clusters, I googled around for some examples of encoding and decoding packets. I knew Elixir's binary pattern matching syntax was amazing, so I found a 
couple examples of raw interactions.

In this post I'll talk about building a packet codec (**co**der/**dec**oder) and some of the fun aspects of Elixir, OTP, and Horde that made packet dispatching a breeze.

## Building a Packet Codec
In my travels looking for some examples of Elixir codecs, I actually stumbled across someone else who was building an MMO backend. They'd made a behavior to describe each network packet so that everything you need to know about a packet is in the packet's module. If you've seen some other networking code in other languages, you know things aren't always this organized. The last time I messed with a C++ code base with packet encoding, I had to read 4 different files to reverse engineer how each packet worked.

So when I saw [this post](https://medium.com/@ygorcastor/building-a-ragnarok-online-server-in-elixir-4c6d75a61d74) I was inspired. I decided to take that example to heart and build a packet _behavior_.

Let's take a look at the first packet I built: the _login_ packet. In most games, this is the first packet sent or is part of the early handshake process. My data structure is pretty simplistic, but there's enough of a skeleton there to build on it later.

```elixir
use Yggdrasil.Network.Packet

@packet_id 0x0010
@packet_size 55

defstruct [:version, :username, :password, :client_type]

@impl true
def build(%__MODULE__{} = packet) do
  username_padded = pack_string(packet.username, 24)
  password_padded = pack_string(packet.password, 24)

  data = <<
    packet.version::32-little,
    username_padded::binary,
    password_padded::binary,
    packet.client_type::8
  >>

  build_packet(@packet_id, data)
end

@impl true
def parse(<<@packet_id::16-little, data::binary>>) do
  parse(data)
end

def parse(
      <<version::32-little, username::binary-size(24), 
        password::binary-size(24), client_type::8>>
    ) do
  {:ok,
   %__MODULE__{
     version: version,
     username: extract_string(username),
     password: extract_string(password),
     client_type: client_type
   }}
end

def parse(_), do: {:error, :invalid_packet}
```

Importantly, the packet behavior requires callbacks for both `build` and `parse`. Build produces a binary from the structure fields while parse does the reverse. Declaring the packet this way makes it self-documenting for developers and also easily tested to ensure the data can perform a lossless round trip.

I love the syntax for Elixir's (and, to nearly the same extent, Erlang) binary pattern matching. I've seen a ton of different languages and I can't think of one that makes it as easy to work with raw data payloads as Elixir.

## Dispatching Network Packets to GenServers
Once the basics are in place for encoding and decoding, I needed a way to _dispatch_ these packets to the interested parties. My game server has a number of different OTP supervisors, including a **world** supervisor, a **zone** supervisor, and a **session** supervisor. I also want to be able to dispatch directly to specific things (like a unique session) without having to go through those supervisors.

This is where `Horde.Registry` comes to the rescue! This provides a cluster aware (and optimized) registry. It's eventually consistent, but at any given time I can dispatch a message to any GenServer anywhere in the cluster without ever having to know explicitly on which node it's running.

If you ask an AI assistant to come up with a dispatch scheme (or if I ask myself when I'm not thinking clearly), you could probably get away with a structure like this:

```elixir
case packet_type do
  0x10 -> ...
  0x11 -> ...
  0x12 -> ...
end
```
AI assistants don't particularly mind this kind of code because verbosity doesn't pose an obstacle for them. But it poses an obstacle for me, my sanity, and the ability of my colleagues to read and maintain my code. The "super giant switch" pattern is used everywhere and in some cases is even the most efficient way to do dispatching when every nanosecond matters.

But for a server-authoritative MMO, I need performance, but not so much that I can't replace a big `case` statement with a couple of **O(1)** lookups and function calls.

I also want the dispatch target of a packet to be _declared inside the packet_. I don't want to maintain some external dispatch lookup table because I've been there before and hated it. So I updated my packet ability slightly and I can now declare the dispatch target right alongside the packet type and size:

```elixir
@packet_id 0x0010
@packet_size 55
@dispatch_target :session

defstruct [:version, :username, :password, :client_type]

@impl true
def packet_id, do: @packet_id

@impl true
def packet_size, do: @packet_size

@impl true
def dispatch_to, do: @dispatch_target
```

On line 14 you can see that I've got a new callback, `dispatch_to`. This can return the atoms `:session`, `:player`, `:zone`, or `:world`. It should be somewhat trivial to add new packets that dispatch to new targets. Right now I am following **YAGNI** in that I don't need multi-target dispatch yet. If I ever do get to the point where it looks like that, I will try and refactor my packet design to see if I can avoid multiple targets. My gut just tells me multi-dispatch for these kinds of packets is more complexity than its worth and could negate any performance benefit I'm getting from the **O(1)** lookups.

Now I can create a simple module that performs dispatch (note that I deliberately didn't make this a `GenServer` as it doesn't need to be another queue bottleneck):

```elixir
 def dispatch(packet_module, parsed_packet, session_id) do
   case get_route_target(packet_module, parsed_packet, session_id) do
     {:session, _session_id} ->
       Session.handle_packet(session_id, parsed_packet)

     {:zone, _zone_id} ->
       # TODO: dispatch to zone
       Logger.debug("Zone dispatch")

     {:world} ->
       # TODO: dispatch to world
       Logger.debug("World dispatch")

     {:gateway} ->
       # TODO: dispatch to gateway
       Logger.debug("Gateway dispatch")

     {:error, reason} ->
       Logger.warning("Failed to route packet #{inspect(packet_module)}: #{reason}")
       {:error, reason}
   end
 end
```

Here, `packet_module` is an actual Elixir module. I get this information from the `Packet` behavior. The second parameter, `parsed_packet`, is the packet that has been decoded from binary received over the wire. This is an efficient binary read because I always know the exact size of the incoming packet. Lastly, I have the `session_id`, which is either used directly to dispatch to a session or to look up a zone (sessions contain the player's current zone).

At first glance this might look the same as a switch on packet type, but `get_route_target` isn't like that. It's responsible for adding the runtime data like the current session ID or the zone ID to the dispatch target already defined by `packet_module`.

And now we get to one of my absolute favorite things about Elixir registries, especially `Horde`'s, which is replicated and consistent across an entire Erlang cluster. If you look at the code listing above, you'll see `Session.handle_packet/2` being invoked. This forces us to figure out how we go from a `session_id`, which is a string, to the `pid` of the running session process. 

At first I was using a manual lookup and then sending the resulting `pid` to `GenServer.call`, but then I remembered we have even more magic sauce: `via_tuple`!

Let's see what `Session.handle_packet` looks like, keeping in mind that this function is not a `GenServer` call:

```elixir
def handle_packet(session_id, packet) do
   GenServer.cast(via_tuple(session_id), {:handle_packet, packet})
end

def via_tuple(session_id),
  do: {:via, Horde.Registry, {Yggdrasil.Registry, {:session, session_id}}}
```

It may seem like a strange request, but stop for a minute and just look at this code. Try and soak up all of the things that are being made trivial because code like this is possible. It's tiny little snippets like this that make me find even more reasons to love Elixir.

First, I get to use `GenServer.cast` to asynchronously send a message to a target process. I don't have to look up the process _at all_, I can use the `:via` tag and say that I want to send that message to a process in the globally clustered `Horde` registry with a key of `{:session, session_id}`.  This is an underrated super power.

## Wrapping Up
I can now take a network packet that I received over the wire via TCP (or UDP, which I'll be coding soon), decode it into a native Elixir structure, and dynamically dispatch that packet to the appropriate OTP server process _anywhere in my network cluster_ without my code having any coupling to my network topology or my supporting infrastructure. 

Next I'm going to deal with movement. When done, we'll be able to take _movement requests_ from the client, validate them, and turn them into immutable movement _events_ that are then not only sent to the calling client, but sent to _all other clients that should know about that movement_. If we can get this fundamental pillar of MMO backends working cleanly, then we know we can build everything else. I like tackling the harder parts first.
