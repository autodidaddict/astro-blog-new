---
title: Playing D&D-based Games in Unison
published: 2026-08-09
draft: false
tags:
  - Unison
  - API Design
  - MUD
description: I created a Unison port of my SRD 5e API
---

In [my last blog post](/posts/combat-api-design/) I talked about designing a combat API in Elixir. In this post, I want to talk about building a far more complete SRD 5e library in Unison. I had a lot of fun building the functionality in Elixir that covered some of the rules and content available in the Standard Reference Document (**SRD**).

I'm always curious about what it feels like to model solutions in different languages. I love figuring out how
the unique features of each language lend themselves to solving problems in different ways. Unison is even more interesting because it is pure functional (with effects) and doesn't have source code files. In Unison, the code is in your codebase or it isn't, there's no middle ground.

To recap, the **SRD** is the subset of rules and content from the world of Dungeons & Dragons that is available for free public use so long as you provide attribution (the **CC-BY** license). When I'd modeled this in Elixir, I had created some primitives for invoking specific rules, but that's where the abstraction ended. 

This posed a problem. At that abstraction level, the game developer would have to know when and where the rules applied in order to figure out which functions to call. An abstraction isn't much good if it doesn't actually hide any complexity from the user. This time around, I wanted to do better.

This is when I decided on the **Encounter** [abstraction](https://share.unison-lang.org/@autodidaddict/srd-5e/code/releases/0.1.0/latest/types/encounters/Encounter). An encounter is a state machine that represents an ongoing, turn-based combat session. In D&D (and therefore the SRD), once combat begins everything switches to turn-by-turn resolution. Players try and perform actions like swinging swords, casting spells, leaping, etc. When the player has exhausted their resources for a turn, they choose to _end_ it. When all participants in a combat session have completed their turns, a new _round_ starts.

What I want for the high-level interface to my library needs to be simple for game developers. The game engine asks my library which commands are available for any given combatant in the current round. The engine presents those choices to the player somehow (could be a GUI menu, a multiple-choice list, text sent over a socket, etc). The player then decides which command they want to execute. The library handles _everything_ after the engine submits a command to the encounter.

This is classic _event-sourced_ design, but it might feel weird to some bedcause it's not _real-time_, it's turn-based. We might have interactions with the encounter _aggregate_ (I use this word in its logical definition, not as a class/object/bit of code) like this:

* `Attack(me, them, weapon)` submitted, emits `ParticipantAttacked`, which is then _applied_ by the aggregate to produce new state.
* `CastSpell(me, magic missile, them, ...)` -> `SpellCast`
* `CastSpell(me, spell-i-dont-know, them, ...)` -> _Rejection_

As with all good event sourced systems, a rejection of a command is not an event. Events are immutable representations of things that have happened and from the aggregate's point of view, the result of rejecting a command is that nothing happened.

With this pattern in mind, an entire combat encounter involving multiple parties and all kinds of different characters is really just a `fold`: the encounter's accumulator starts with the initial state of combat (participants, initiative order, relative locations, etc), commands are validated, events are emitted and applied to the accumulator until the encounter is over.

Step 1 was stating my requirements. I need to be able to process all aspects of combat encounters without requiring consumers to know when and where rules apply. I need to be able to test all rule application deterministically. This meant I needed to deal with dice similar to the way I designed in the Elixir library, but I had a hunch that I'd be able to use Unison's _abilities_ to let me emit events from a pure function.

I'll start with the lowest level primitive - evaluating dice expressions:

```haskell
Roll.parse "2d6+3"
> Some (Expr 2 6 +3)
```

This converts the human-friendly dice expression "2d6+3" (roll 2 six-sided dice, add 3 to the result) into an `Expr`, which is like a mini-AST for dice expressions. Once I've got an expression, I can roll it (which requires `Random`):

```haskell
splitmix 9 do Expr.roll (2 * d8)
> DieResult 8 10 +0
```

This uses an ability handler (`splitmix`) to roll a manually created roll expression. In the example above, the dice result shows that it used 8-sided dice with a final result of `10` and no modifier was applied. Everything in the game builds on the foundation of representing a roll of the dice.

Probably the single most frequently executed rule in a D&D game is the _attack roll_. Like every rule in my library, it takes a completed die roll, the required context, and produces a result. The first form of `rules.attack` takes the roll on which a critical occurs, the die result, and the target's armor class (**AC**). Since many attack rolls can be simplified as d20s with a crit at 20, I've created a convenience function `attack.normal`:

```haskell
-- Player rolled a 12 on a d20 that crits on 20 against an AC of 15
attack.normal (DieResult 20 12 +5) 15
-- Result is a hit, not a critical, resolved against AC of 15 for a total of 17 with the natural roll of 12.
> AttackResult true false 15 +17 12
```

In D&D you've got _hit dice_ and _damage dice_. So you roll to see if you hit, then you roll to see how much damage (if any) you did. So we don't pipe the attack result into the `damage` rule, we roll again, get a second `DieResult` and pass _that_ to `damage`:

```haskell
-- Rolled a 1d8 with no modifier, producing a 5. Damage type is fire and the enemy is fire resistant, 
-- halving the total damage.
rules.damage (DieResult 8 5 +0) Fire [Resistant Fire]
> DamageResult 2 5 Fire
```

This is great, but if the game is calling all of these rules manually, then it _already knows everything about D&D rules_, making the library abstractions useless beyond rolling dice. 

## Enter the Encounter
The encounter is the multi-round state machine that manages rule evaluation for all combat participants. This presented the most fun design challenge that I never tackled in the Elixir library. How do I let the game submit commands to manage an encounter without the encounter knowing how the game works? How do I fold a full round of combat and somehow interleave player-supplied intent, full resolution, and event emission, all without breaking tier isolation?

This is where I used _abilities_.  There are 2 abilities in the `encounters` namespace:

```haskell
ability encounters.Control where
   commandFor: Participant -> Encounter -> {Control} Command

ability encounters.Reactions where
   reactionTo: Participant -> Trigger -> {Reactions} Optional Command
```

The `Control` ability is the _driver_ through which a creature's (which includes players) decision during its turn comes from. The engine calls `commandFor` in the "fold", and the handler decides what command should be submitted. This decision process allows `IO`, so it can obtain that creature intent from an AI, from a human at a keyboard, from a network socket, etc.

The `Reactions` ability is the _driver_ through which a creature can react to a trigger. Something happened _during another creature's turn_, and this creature is given an opportunity to react to that.

If the engine has different deciders/brains/drivers for different participants, it can dispatch inside the handler functions. It's the _game_'s responsibility to provide handlers for these abilities, and the _engine_'s responsibility to use them at the right time.

At this point I was fine with having the game write these handlers when I realized that I could make it even easier for them. I could write the handlers in the engine itself as wrappers around pure functions. In "functional" language, this is kind of like providing "lift" functions that wrap a regular function in a monadic call.

Before I show the encounter loop, let's take a look at the two non-handler functions, `choose` and `react`.

```haskell
examples.terminal.react :
  Participant -> Trigger ->{IO, Exception} Optional Command
examples.terminal.react reactor = cases
  LeftReach mover _ _ ->
    use Text ++ ==
    who =
      (ParticipantId n) = mover
      n
    match List.head (Participant.attacks reactor) with
      None -> None
      Some w ->
        printLine ""
        printLine
          (who
            ++ " is stepping out of "
            ++ nameOf reactor
            ++ "'s reach. Attack? (y/n)")
        typed = getLine stdIn
        if typed == "y" then Some (Attack (id reactor) mover w NoMastery None)
        else None
```
In this `react` implementation, if a combat participant steps out of range of the source participant, the person controlling creatures at the terminal can choose to perform an opportunity attack. Note that `react` is allowed `IO` and `Exception`, but it doesn't manually write an ability handler. The creature's reaction is captured as a single `Command`.

In the code below (also from the terminal-input example), you see the `choose` function is given the encounter as an input to use to aid in choosing an action. This could be an AI-based decision, a random-based decision, or whatever. In our case, we're presenting the person at the terminal with options (`terminal.options`) and turning their input into a `Command`:

```haskell
examples.terminal.choose : Participant -> Encounter ->{IO, Exception} Command
examples.terminal.choose me e =
  use Nat +
  use Text ++
  menu = terminal.options me e
  show i = cases
    []                 -> ()
    (label, _) +: rest ->
      printLine ("  " ++ Nat.toText i ++ ") " ++ label)
      show (i + 1) rest
  board = cases
    []        -> ()
    p +: rest ->
      marker = if id p === id me then "* " else "  "
      printLine (marker ++ brief p)
      board rest
  ask = do
    use Nat -
    printLine ""
    board (standing e)
    printLine ""
    printLine (nameOf me ++ "'s turn — what do you do?")
    show 1 menu
    typed = getLine stdIn
    match Optional.flatMap (n -> List.at (n - 1) menu) (Text.toNat typed) with
      Some choice -> at2 choice
      None        ->
        printLine "  (pick one of the numbers)"
        ask()
  ask()
```

With the game engine's responsibilities managed (choose and react), we can write the encounter driver loop, which is where all of this abstraction and complexity hiding pays off:

```haskell
examples.terminal.run : '{IO, Exception} ()
examples.terminal.run =
  do
    use Text ++
    loop e =
      if isOver e then
        printLine ""
        match outcomeOf e with
          Some (Won side) -> printLine ("The " ++ side ++ " win.")
          _               -> printLine "Nobody is left standing."
      else
        match whoseTurn e with
          None       -> ()
          Some actor ->
            command = commandFor actor e
            match performWith (event -> say (narrate event)) command e with
              Left rejection ->
                printLine ("  " ++ Rejection.render rejection)
                loop e
              Right next     -> loop next
    
    splitmix 42 do
      reactions.fromFunction react do
        control.fromFunction choose do
          printLine
            "A fighter and a cleric against a Goblin Warrior, fifteen feet apart."
          loop roster
```
The entire loop is ~25 lines. The chain of 3 `do`s should look familiar to folks who've used monads before or have used Unison's abilities (note: abilities aren't monads in the academic sense). `splitmix` wraps what comes next in the ability handler for `Random`, `reactions.fromFunction react` lifts the `react` function into an ability handler for `React`, and `control.fromFunction` lifts the `choose` funciton into an ability handler for `Choose`. 

On the last line, `loop roster` starts the loop with the empty encounter (the `roster` term). Thereafter, turns are recursively processed while passing along the modified encounter as the loop input.

## Content
In addition to the encounter and all the underlying rules, this library is chock full of content. It comes with _all_ of the armor, weapons, creatures, skills, conditions, and spells defined in the SRD. All of this content can then be used by a game to help in character creation and building realistic encounters with all the right stats. You can see the content in the Unison Share library [here](https://share.unison-lang.org/@autodidaddict/srd-5e/code/releases/0.1.0/latest/terms/@5kvrsbmdm16i1r4ik15aed5oq0g991efie40sldr1b59jb1ki934knov4tc56vue2ds114s3rn5f2nl0q6svt6iu4kl9d6n90ndg9r0)

## Summary
When I sat back and looked at what I'd been able to do in Unison versus where the Elixir library was headed, I was really pleased with the Unison implementation. It's strictly typed, has real sum types, and gave me the ability to plug _any game_ into the rules library encounters. I could get player commands from TCP messages, from the terminal, from a cloud queue, or even from messages I get from a hardware device (I need to figure out a way to play D&D over LoRa radios!)

