---
title: Fun Designing a Combat API
published: 2026-07-19
draft: false
tags:
  - Elixir
  - API Design
  - MUD
  - Agentic Realms
description: I had some fun crafting the API for my game's combat system
---

Recently I had one of those inspiring conversations that reminds you why you're doing this kind of thing in the first place. We talked about API design and threw out ideas with no judgment, just 
possibilities and real concern for quality and ergonomics and efficiency. I loved it.

I left that conversation inspired and so I decided to tackle an API design project I knew was on my roadmap: designing the combat system for my own game, [Agentic Realms](/tags/Agentic%20Realms/).

On my mental blank canvas I knew I was going to need some kind of random number generation. Above that, I was going to need to handle weapons and armor and attacks and defenses and skills and spells and death and healing and ... and ... it was a lot.

As I paced around my office thinking about the kinds of rules I'd need to define, I recognized one of my own limitations. While I could model the API, there was no way I was going to be able to devise a set of combat rules and internal mechanics that would be balanced and enjoyable for players. API design is _not_ the same as well-balanced game design.

If I couldn't create my own mechanics, I knew where I could find some: _Dungeons & Dragons_. While we can't legally use any of the trademarked and copyrighted material from D&D, **Wizards of the Coast** have graciously created the [System Reference Document](https://www.dndbeyond.com/srd). This is a set of rules, data, and metadata that you can use free of charge in your own game (assuming you provide source attribution). Even games like Baldur's Gate 3 use rules from the SRD.

My API design exercise just got a _lot_ more fun. My game's core mechanics would already be familiar to anyone who's played tabletop or digital D&D games. Deciding on the domain shape is usually the hardest part of API design, and the SRD gave me a head start.

## The SRD 5e Elixir Library
I've encapsulated the results of my API design into a reusable Elixir library called **[srd_5e](https://srd-5e.hexdocs.pm/readme.html)**. It's not complete, but there's enough in there to handle basic combat rounds. The rest of this post talks about the decisions that went into making this library.

## The Core Constraints
Tabletop RPGs rely on dice rolls. It's unusual for any kind of meaningful decision or action to happen without these dice. Dice produce random numbers, and randomness makes testing and verification impossible. 

This makes the crucial part of the SDK the `Dice` module. It would be very easy to let randomness spread throughout the API, but it's much better for the API developer and for consumers to have randomness constrained to just one place. The `Srd.Dice.roll` function is the _only_ place in the entire library that makes RNG calls. 

The output of a call to `roll` is a `Roll` struct. This is the most frequently used data structure in the library. It gets passed to all of the rules. A roll result has the following fields:

* `count`
* `sides`
* `modifier`
* `dice`
* `reduce`
* `total`

I'm not going to list out everything in the entire library, but I think this data structure helps illustrate the design philosophy. The result of rolling dice is an immutable structure that contains the list of raw roll results (the `dice` list), number of times the die was rolled, the sides, and so on. Some rules just need the `total` field, but other rules have extra logic that uses the original raw dice. 

As a convenience, you can `parse` a dice expression like `"3d4"` or `"2d10+4"` and roll directly from that friendly string. When I get to the _content_ defined in the SRD, you'll see how this little convenience pays big dividends.

Everything in the rest of the library revolves around the `Roll` struct, especially tests. Tests just fabricate roll results and pass them into the rules to verify their behavior.

In addition to confining randomness, the other core constraint on the library is that all of the types
must be serializable. Enforcing serializable not only makes wire transmission possible, but it keeps people from sticking functions or lambda closures into a data structure. Data stays as pure data. Serializable data is also data that can be easily kept in an event stream (which is what the game consuming this library uses).

## Boundaries
Another challengingly fun aspect of API design is figuring out the layers. We rarely ever abstract a flat surface area. Most of the time, we do things like build foundations (e.g. `Dice`) and layer additional features on top.

If we get these boundaries wrong, we can do things like create circular references or internal details can bleed across these layers and ruin the cohesion of the entire library. I've got two rules when it comes to boundaries/layering:

* One-way layering from top down. Content (I'll get to that in a bit) has metadata that feeds into dice. Completed dice rolls are used by rules. For example, the `ArmorClass` module resolves the AC by accepting a raw map (`%{category: :heavy}`) rather than a `%Content.Armor{}`. This keeps rules from ever having to depend on content.
* _Rules resolve, they don't orchestrate_. The rules are all invoked as pure functions. They have no side effects and they are deterministic. Die rolls take place elsewhere and are used as parameters. If the rules could roll their own dice, this library would be a huge mess. HP management, turn bookkeeping, dice rolling, etc are all done by the caller.

## Types as Interfaces
Another activity we have to manage in an API is curating public interfaces and hiding what should stay hidden. I'm always a fan of "less is more" here, and I start with everything hidden and only broaden visibility when I need to.

* Every outcome of every pure function is a struct with a `@type t`, not a loose map or a tuple, for e.g. `%Attack{hit?, critical?, margin}`. While Elixir is a dynamically typed language, using type specs can trigger build failures on mismatches and the new "gradually typed" features are eerily good at catching problems.
* Closed SRD sets are created as typed unions like damage types, weapon mastery, cover, skills, etc. Illegal values for these union types cause early failures at the boundaries instead of leaking their way into the core as unrecognized atoms.
* Required vs optional is visible in the signature (which means it's visible in your IDE's contextual popups). For example, `Damage.resolve(roll, :fire, resist: [...])`. The damage type is positional and required, and the defenses/resistances are options. This looks a bit weird to non-Elixir developers because most other languages don't use the concept of _keyword lists_ like Elixir and Erlang do.

## Policy
This was a simple decision but it helps illustrate supporting the _principle of least surprise_. We don't want functionality to surprise a developer. Even if the result is supposedly a positive one, giving developers unexpected functionality is never good.

Because the SRD rules vary from one edition to the next, some rules like _exhaustion_ have completely different formulas. You'll get a different result in SRD 5.1 then you do in SRD 5.2. To keep the functionality unsurprising, everything in the library corresponds to SRD 5.2. 

If I want to expose the "old" exhaustion formula, I can do it as an opt-in choice made by the caller with an option like `edition: 5.1`.

## Resolving a Combat Round with My Library
So let's see what this API looks like in action. Here's an example day in the life of a game resolving combat turns using SRD:

```elixir
alias Srd.Content.{Weapons, Armors}
alias Srd.Dice
alias Srd.Rules.{Initiative, Attack, Damage, Hitpoints, DeathSaves}

# Order the combatants. The caller rolls each initiative and captures the roll.
Initiative.order([
{"knight", Dice.Rolls.initiative(2)},
{"orc", Dice.Rolls.initiative(1)}
])
#=> [{"orc", %Srd.Dice.Roll{...}}, {"knight", %Srd.Dice.Roll{...}}]   # highest first

# The knight attacks the orc with a longsword, against its chain-mail AC.
longsword = Weapons.get("longsword")          # 1d8 slashing, versatile 1d10
orc_ac = Armors.get("chain-mail").base_ac     # 16

attack = Dice.Rolls.attack(5)                 # 1d20 + attack bonus
result = Attack.resolve(attack, target_ac: orc_ac)
#=> %Srd.Rules.Attack{hit?: true, critical?: false, natural: 14, total: 19, target_ac: 16}

# On a hit, roll the weapon's damage (parsed from content) and apply defenses.
if result.hit? do
damage = Dice.roll(longsword.damage)
dealt = Damage.resolve(damage, longsword.damage_type)
#=> %Srd.Rules.Damage{type: :slashing, raw: 4, final: 4}

# Apply to the orc's hit points. The outcome drives what happens next.
Hitpoints.new(15, 15) |> Hitpoints.damage(dealt.final)
#=> %Srd.Rules.Hitpoints{hp: 11, max_hp: 15, temp_hp: 0, outcome: :ok}
end
```

When `Hitpoints` reports `:downed`, the caller starts death saves; a hit taken
while at 0 hit points is a failure it routes in.

```elixir
down = Hitpoints.new(4, 15) |> Hitpoints.damage(4)
down.outcome
#=> :downed

DeathSaves.new() |> DeathSaves.record_save(Dice.Rolls.death_save())
#=> %Srd.Rules.DeathSaves{successes: 1, failures: 0, status: :dying}
```
There are other rules and conveniences in the library, but this gives you an idea of how the API has been tailored to the right audience - people making a game that sits atop SRD rules.

## Embedding Content
Some of my favorite functions in the preceding code list are `longsword = Weapons.get("longsword")` and then `damage = Dice.roll(longsword.damage)`. There's a module in the library called `Srd.Content.Weapons`. This module exposes a function `get/1` to query for a weapon by its normalized slug. 

If I'm building the game, I don't want to have to remember the damage dice for each weapon. Since the SRD makes that data available, I've imported all of that content. The library has `Content.Weapons`, `Content.Armors`, and `Content.Conditions`. These were all relatively small data sets, so embedding them right in the compiled output is fine. I'm currently thinking through what I want to do for monsters and spells, both of which could add several MB to a deployed artifact using the library.

One option is to create another package, `srd_5e_content`, which would then let developers opt-in
to the large content tables. If they don't need it, they can leave that extension out. I'm leaning toward this option but if you've got other solutions, please let me know.

## Wrap-Up
I love getting nerd sniped/inspired. Inspiring conversations are my favorite aspect of attending or speaking at conferences. I used a bunch of inspiration points to make this library and I had a ton of fun doing it. I think what I'll do soon is create an `Encounter` aggregate in the actual **Agentic Realms** game, and then have the managed, turn-by-turn encounter utilize the SRD 5.2 rules.