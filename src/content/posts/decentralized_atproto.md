---
title: Decentralized Gaming with ATproto
published: 2025-09-19
draft: false
tags:
  - Gaming
  - ATproto
description: I walk through a thought experiment on what it might be like to build a decentralized game with user-generated content using ATproto
---

One day, a very long time ago, I wrote a blog post about [gaming in the fediverse](https://kevinhoffman.blog/post/fediverse_gaming/), which was really about how I might be able to twist and pervert `ActivityPub` into a way to game. I learn new technology by figuring out how to game with it. Some (3) years later, I am again taking a look at decentralized gaming, but this time with _ATproto_.

The AT Protocol ([ATproto](https://atproto.com/), pronounced "at proto") is an open, federated networking protocol designed to power decentralized _social_ applications. It's main claim to fame is that it separates identity, data storage, and application logic, giving users portable accounts and control over their own data while allowing developers to build _interoperable apps_ on top of a shared ecosystem. 

Unlike other social platforms, ATproto emphasizes user sovereignty and composability: identities are anchored in cryptographic keys, data lives in personal repositories (**PDS**s), and applications interact through a consistent, flexible API layer. This architecture encourages innovation—-[^1] whether in moderation, discovery, or entirely new interaction models. All of these features come without the lock-in of centralized social networks.

In this post, I'm going to talk about the _interoperable apps_ part, building on top of this shared ecosystem.

## A Game Concept

There's always been something that I find fascinating about the _"visit my world"_ multiplayer dynamic. We see this in **Animal Crossing** where you can visit a friend's neighborhood. You can visit other people's farms in **Stardew Valley**. You can visit other people's bases in **No Man's Sky**.

So for this hypothetical game, let's go with a colony theme. Let's say each player has their own colony (or _shard_ or _station_ or _moon_ or _planet_ or ...). When they interact with their own private little corner of the universe, the game rules can be whatever we like. To keep things simple, assume a harvest/refine/build/discover loop.

Every player gets a _shard_. Maybe this shard was originally part of a single unit that split apart in a catastrophic explosion sometime in the past. What will make this game a bit unique is how it will combine the idea of live interactive sessions (the kind of gameplay you would traditionally expect) and the consumption and production of **ATproto** records in a **PDS** (Personal Data Store) repository. The latter not only provides for async social interaction, but because of how ATproto works, it also means we have a _signed_ and _verifiable_ public record of everything important that happened in any player's shard.


## Playing in Live Sessions 

A _live session_ can be a few different things. If we decide to make the game "always online", then these sessions are ephemeral sessions managed by a server application somewhere. This game backend would manage all of the live sessions for all online players. The data maintained in a live session is essentially all of the ephemeral stuff that doesn't need to be made part of a shared public record. My current location, hitpoints, possibly even my equipment, in-game chats, and even the majority of activities around crop harvesting can be in-session.

Another way of thinking about it is this: anything in a live session is disposable. If you lose a live session due to crash or timeout and you log back in, loading data from your canonical record (your ATproto repository), then everything should be just fine.

As you'll see in the next section, it doesn't really make sense to manage _all_ of a game's global and session data within the "protosphere" (ATproto ecosystem). The public doesn't need to see that kind of spam, and you don't need to manage that kind of noise in your own profile. Only the important, persistent things matter outside a live session.

## Decentralized Public Data (ATproto)

If you take a look at the [ATproto specification](https://atproto.com/specs/atp), your eyes may likely water and you may feel as though you're losing consciousness. It's not the easiest of things to grasp, and the documentation is never really clear about responsibilities, ownership, and how the whole system fits together. It suffers a bit the way many one-company standards do in that the beginnings of the protocol and its documentation seem to be more about **Bluesky** integration than building other kinds of apps.

In ATproto, everyone has a distributed identity. This identity is the same no matter which PDS (repo) they are using at the time. Associated with an identity are any number of collections. For Bluesky, these collections are things like a user's posts and the list of other people's posts they've liked, their reposts, etc. All of that information is available to read for any user with a valid API authentication on an ATproto repository server.

Everyone playing our game has an ATproto identity--a **did** (distributed identity). ATproto uses [XRPC](https://atproto.com/specs/xrpc) under the hood, so it supports strongly typed remote function calls using the well-known ATproto APIs. These strong types come from **lexicons**, which are themselves records that you can find on a PDS. As you can probably guess, there are well-known lexicons for all things Bluesky, but ATproto doesn't limit people to just posts, likes, and re-posts.

Modeling the lexicons feels like one of the more fun aspects of gaming on ATproto. For a game where players have their own shards, and their **did** credentials give them the ability to write important records in those shards, we might have lexicons like this:

* `realm` - A realm is the top-level "thing" in the game. All activity in the game occurs within a specific realm. As I discuss in the next section, players can create their own realms, which is where this stuff gets really exciting.
* `npc` - Non-player characters. Their properties are defined as individual records, their shape described by the lexicon
* `item` - Objects that can exist in the game
* `quest` - A quest that can be undertaken by a player. The rewards and triggers are defined in a quest, and items related to it will link to the quest. We can also use _tags_ on records to provide a flat topology that helps us organize things.
* `location` - Represents a physical space within the realm
* `behavior` - Represents a behavior that can be associated with an NPC. 
* `ledger` - A collection of important, state-modifying events that occurred within a realm, such as an NPC dying or a player earning experience points. If you wanted, you could watch someone's raw ledger the same way you might refresh their Bluesky post list.

The `ledger` is essentially the event log used to produce player state, while the other records are used for the game _content_.

Because of how ATproto works, players could even choose to run their own PDS and have that be the home for their data graph. If a player has the ability to write records to their collections (remember they're signed, so they can't be faked or cheated) through the app, then there's nothing stopping them from creating their own content as well.

## User Generated Content

This is where things get pretty exciting (at least for nerds like me). In a typical game, the content is either all embedded in the single game client like a browser app or an offline console game. Other games have the content all stored in a single server (or cluster of them). What interests me about ATproto is that players can create their own content.

Each player gets their own repository. This could be hosted on a central PDS, but it could also be _self-hosted_ by a player on their _own PDS_. Regardless, they can create their own content simply by adding records to their `realm`. This may be kind of difficult to visualize since the creation flow is a bit unique. Let's walk through a sample.

```

[ Player opens browser ]
           │
           ▼
[ Login with AT Protocol / DID ]
           │
           ├─> AT Protocol PDS authenticates
           │
           ▼
[ Game Client (browser app) receives session token & DID ]
           │
           ▼
[ Fetch appropriate world shard (locations, NPCs, quests, behaviors) from PDS ]
           │
           ├─> For each location: fetch linked NPCs, quests
           ├─> For each NPC: fetch behaviors, dialogue, linked quests
           │
           ▼
[ Build in-memory world graph ]
           │
           ▼
[ Player explores world ]
           │
           ├─> Click / move to location → render map tile
           ├─> Interact with NPC → dialogue + triggers
           ├─> Pick up / complete quests → update local game state
           │
           ▼
[ Behavior & Quest Engine ]
           │
           ├─> Reads behavior records
           ├─> Applies triggers based on player actions
           ├─> Updates quest objectives
           │
           ▼
[ Session server updates state ]
           ├─> Backend server makes ATproto changes where appropriate
           │
           ▼
[ Player wants to create content ]
           │
           ├─> Open content editor in browser (Location, NPC, Quest, Behavior)
           ├─> Validate data against lexicon schemas
           ├─> Publish new record via AT Protocol API
           │       POST /xrpc/com.atproto.repo.createRecord
           |       doesn't require the session server
           │
           ▼
[ New record is federated / discoverable ]
           ├─> Other players fetch via feed / curated shard
           ├─> Can be reused by other content on other PDSs
           │
           ▼
[ Player sees their content in the game world ]
           │
           ▼
[ Iterative gameplay / world building continues ]
```


Again, the session server is managing all the ephemeral realtime stuff, and ATproto is managing both the player's public ledger of activity _and_ their content, if they've decided to create any. Now if we want to add a connection from the game's core/root content to someone else's shard, all I have to do is create an `exit` or `connection` or some other kind of ATproto record that refers to that content by its `did`! A truly decentralized, global content system that can create one amazing, sprawling world.

If another player creating their own content wants to use a sword or an NPC from someone else's realm, they can add that connection without ever needing changes made to the root or central shard.

An example `ledger` record (remember it has to conform to a `lexicon` record, which is similar to JSON schema):

```json


```

A sample quest:

```json
{
  "$type": "protoshards.realm.quest",
  "title": "The Knight’s Last Stand",
  "description": "Help Eryndor defend the Ruined Keep against the shadow beasts.",
  "giver": "at://did:plc:wxyz/protoshards.realm.npc/3k9gh",
  "location": "at://did:plc:abcd/protoshards.realm.location/3k7sd",
  "objectives": [
    { "type": "defeat", "target": "at://did:plc:core/protoshards.realm.npc/shadow_beast", "count": 5 },
    { "type": "escort", "target": "at://did:plc:wxyz/protoshards.realm.npc/3k9gh" }
  ],
  "rewards": [
    { "type": "item", "id": "at://did:plc:wxyz/protoshards.realm.item/loot123" },
    { "type": "reputation", "id": "at://did:plc:core/protoshards.realm.faction/knights_order", "amount": 50 }
  ],
  "tags": ["combat", "story"],
  "createdBy": "did:plc:kevin"
}
```

An NPC:

```json

{
  "$type": "protoshards.realm.npc",
  "name": "Eryndor the Tired",
  "species": "Human",
  "description": "A weary knight who has guarded the keep for decades, awaiting an heir that never came.",
  "location": "at://did:plc:core/protoshards.realm.location/3k7sd",
  "behaviors": [
    "at://did:plc:core/protoshards.realm.behavior/5fg9h"
  ],
  "quests": [
    "at://did:plc:core/protoshards.realm.quest/8lm2p"
  ],
  "tags": ["knight", "mentor"],
  "createdBy": "did:plc:kevin"
}
```

The location for the _Eryndor the Tired_ NPC:

```json

{
  "$type": "protoshards.realm.location",
  "name": "The Ruined Keep",
  "description": "An ancient fortress overgrown with vines and haunted by whispers.",
  "coords": { "x": 120, "y": -45 },
  "tags": ["ruins", "haunted"],
  "links": ["at://did:plc:core/protoshards.realm.npc/3k7sd"],
  "createdBy": "did:plc:kevin"
}
```

The vast majority of the game client's work will be in querying ATproto records and sending periodic ephemeral updates to the session server. Using ATproto, we can query for a list of locations within the realm (repository) with the following HTTP command:

```
GET /xrpc/com.atproto.repo.listRecords?repo=did:plc:core&collection=protoshards.realm.location
```

Creating new content for a player within their PDS:

```
POST /xrpc/com.atproto.repo.createRecord
{
  "repo": "did:plc:user123",
  "collection": "protoshards.realm.npc",
  "record": {
    "name": "Eryndor the Tired",
    "description": "A weary knight…",
    "locationId": "did:plc:core/protoshards.realm.location/3k7sd",
    "createdBy": "did:plc:user123"
  }
}
```

When all of this clicked for me, it was pretty inspiring. I've always loved the idea of players being able to create their own content (which is one reason I love **MUD**s so much). But for users to be able to securely create decentralized content and not have the waste and bloat from blockchains? That's fantastic!

In my infinite amount of spare time, _"all"_ I would have to do to create a game like this would be to stand up an off-the-shelf self-hosted PDS and then create the session server. Oh, then I'd have to create the UI for the game client. I'd wager the session server wouldn't have to talk to ATproto at all--all of that could be done inside the game client.

Should only take me a few hours to implement all this. _No problem_.

---

[^1]: I know this is an em dash, but trust me, I'm not an AI and an AI didn't write this post.
