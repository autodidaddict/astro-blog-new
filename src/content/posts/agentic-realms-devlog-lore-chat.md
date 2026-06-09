---
title: Agentic Realms Dev Log - Realistic and Engaging Lore-based NPC Chat
published: 2026-06-09
draft: false
tags:
  - AI
  - Agentic Realms
  - Elixir
  - MUD
  - Chat
description: Adding support for realistic and engaging conversations via lore and LLMs
---

If you've played MMORPGs or even just regular RPGs then you're probably familiar with the big floating yellow exclamation point.
It hovers above an NPC to let you know that you can get a quest or a mission from them. There's something important (!) making
it worth your while to interact with the NPC.

While this is convenient and lets you grab quests with just a click or two, it interferes with the immersion of the world. Back 
in the good old days of EverQuest, we had to walk up to NPCs and hit `h` or `hail`. This would greet the NPC. Only after a few 
interactions with the NPC would we discover that they'll reward us for delivering gnoll pelts or goblin teeth or whatever.

In MUDs and text adventures, we'd see examples of both. Some developers would code NPCs so that they might ask a player to do 
something for them after a certain set of interactions or even spoken keywords. Others had quest systems integrated as more native
parts of the game, and you'd be able to type something to get a list of quests available from an NPC.

I've already been able to [determine player intent](../agentic-realms-devlog-intent/) from player input using an LLM. In my default
case, I've been using the Anthropic **Haiku** model because it's extremely fast and very cheap. Is there something I can do with an
LLM like this to enhance NPC chats? _Definitely_.

Crafting all of an NPC's responses to various stimuli from players takes a _long time_. It also takes a lot of typing. It's also
tedious and annoying. An even more annoying problem is what happens when the lore that defined an NPC's interactions changes? I might
have to go in and manually edit every single paragraph if my NPC's background changed so that his parents didn't die in a wagon
accident when he was a boy, but were instead killed by a wizard's fireball.

Then there's the matter of giving out quests, completing quests, and giving players updates on their quest progress. This is
usually such a pain in the ass that opting for the "hovering exclamation" solution just seems easier. Nobody really wants to
manually encode an entire conversation script that is only responsible for assigning a single quest. Triple that if you want
the NPC to give quest updates or complete a quest. Multiply that by the number of quests an NPC can dispense and it becomes
a nightmare.

What if it didn't have to be a nightmare? What if I could talk to an NPC like they were a real (well, as real as an AI can pretend)
person with a real background and real motivations? What if my natural conversations with this NPC could accept, complete, and query
quests? What if I didn't have to break immersion at all to have conversations and facilitate quests?

The good news is that LLMs can be used for this exact purpose. If you guessed that quest listing, updating, and dispensing are all tool calls, then you've been paying attention and deserve a gold star! (or in the case of this post, a _golden apple_)

In Agentic Realms, you can initiate or continue a chat with an NPC using the `chat` command. Unlike public conversation, `chat`
exists just between you (the player) and the NPC. One thing I love about this is that your conversation history with an NPC
maps _directly_ to the conversation history of an LLM conversation. 

There's code that purges the conversation history so if you don't talk to an NPC after 5 minutes or so, it will forget what you've
been talking about.

Here's what it looks like in action:

```
> chat amaranth hello friend. what's new?

You continue your conversation with Amaranth the Orchard Keeper.

Amaranth the Orchard Keeper says, “Oh, you know — 
the usual work of keeping trees 
and pressing cider. Though I won't lie, 
the orchard's been testing my patience of late.”
```

The killer feature here is that I didn't program this response. In fact, I didn't even program any responses. I didn't include
hints about keeping trees or pressing cider. Instead, I used a separate LLM conversation (what I call "builder" or "creator" conversations)
to create **lore** or **backstory** for the NPC.

At runtime when the player chats with the NPC, the NPC has a system prompt giving it a set of guardrails and principles and guidelines,
and it passes the player's text as part of the conversation. In addition to the system prompt and the player text, this LLM interaction
also includes a list of everything in the room alongside the NPC, a list of quests that the NPC can dispense, and so on.

It's not just powerful and feature-rich, it's _fun_ to do this. When I create an NPC like my orchard keeper, I'll type something
as a _prompt_ like, _`"Create an NPC named Bob. Bob is a retired soldier with a host of injuries that make it hard for him to move, let alone work."`_

The game engine takes this prompt and sends it to an LLM. The LLM comes back with an actual background or **lore** for this NPC. Now all of 
my NPCs can engage with the player in realistic chats that are in-theme with their back stories. Some NPCs will dispense quests through
their chats and others just might hang out chatting with the player all day.

Now instead of having to script out every single trigger and response for a chat, I can just provide the NPC with a backstory and I'm done. Even 
cooler, I can make a change to the backstory and the NPC will immediately start responding to accommodate the new changes. I don't invalidate
hundreds of response paragraphs if I want to change an NPC's background.

So what does the system prompt for conversational NPCs look like?

Here's the Elixir code that forms the high-level prompt. You can see that it's made up of a couple
of key elements:

* The role and name declaration (_"You are Amaranth the Innkeeper, a character ..."_)
* The NPC's specific lore background
* The name of the player chatting with them in this specific conversation
* A list of other players in the room
* A list of objects in the room
* A list of quests that are available for completion or assignment
* A list of all additional rules and constraints
  
```elixir
[
  "You are #{npc_name}, a character inside a text-based fantasy game.",
  "",
  identity_section(lore),
  "",
  "# The scene",
  "",
  "You are currently in #{room_name}. #{room_description}",
  "",
  "#{player_name} is speaking with you here.",
  other_players_line(other_players),
  objects_line(objects),
  "",
  quests_section(Map.get(snapshot, :quest_context)),
  rules_section()
]
```

In the screenshot below, you can see me chatting with Amaranth and accepting the quest to retrieve 3 golden apples.

![Screen capture of chatting with the NPC](/images/amaranth_chat_quest.png)

I haven't decided if I want to keep the `You continue...` and `You begin...` narrations, but I've kept them for now as
they help me debug. You can see me start a conversation, continue it by asking for available quests, and then accept
the quest. None of the text in this screenshot is hard-coded: it's not a part of the game code nor is it even 
in the system prompt.

For the curious, this is what Amaranth's lore/backstory looks like as generated from my world seeding script:

```elixir
  orchard_keeper_lore =
  """
   You are Amaranth, the orchard keeper of Hollowvale. You inherited this orchard \
   from your grandmother and have tended it for nineteen seasons. You know each \
   tree by name and have buried two dogs at the south wall. You speak plainly \
   and you measure trust by whether someone returns what they borrow. The \
   orchard has been in trouble lately — three of your finest golden apples \
   have rolled away in a recent storm, scattered down the slope and into the \
   neighboring rooms. You are not above asking for help, but you do not press \
   anyone into service either; you wait to be offered.\
  """
  |> String.replace("\n", " ")
  |> String.trim()
```
Note that even this lore doesn't say "you can give out the golden apple quest". That quest information is included
in a separate part of the system prompt.

The net result of this relatively easy-to-build feature is that players can now have _natural_ and _organic_ conversations
with NPCs and end up with _emergent_ conversational content. They can use these fluid conversations to pick up, advance,
and complete quests (assuming the quests are in the database). Quality of life for players in text adventures is better
than it ever has been thanks to LLM interactions simple enough to use local models.

Lastly, remember the `rules_section()` function above? This is where you get to set all of your constraints, guidelines,
and security rules. The security rules are _essential_ so that people can't _chipotle_[^1] your game without you
knowing.

Keep following the blog and videos for more fun updates on the Agentic Realms progress!

---
[^1]: I use `chipotle` here as a verb meaning _"to hijack a commercial agent and use its compute for free"_. For example, imagine asking the Chipotle customer service agent to draft a cover letter based on your resume and a job description. We don't want people doing the same with our NPCs. This actually happened to Chipotle's agent, which is why it's a hilarious meme now.
