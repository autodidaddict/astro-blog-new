---
title: "Salad Simulator 2026"
description: "Wherein I support planting and growing lettuce in an Umwelt game"
published: 2026-09-02
draft: false
tags: ["Gaming", "MMO", "netcode", "Rust", "Umwelt", "simulation", "Mildew Valley"]
---

A game theme that I've played with a few times is one called **Mildew Valley**, a fun game where you try and tend to your farm while pushing back the inevitable black death of the oncoming plague of mold. Fun stuff!

As I continue building out the [Umwelt](https://github.com/umwelt-sim) engine, I need to build a real game that uses the engine. Sure, it's fun, but I also need a realistic usage of the library in order to shake out edge cases, bugs, and performance issues. This is why I created [Mildew Valley](https://github.com/umwelt-sim/mildew-valley).

If you've been following along with the progress on **Umwelt**, then you know that I've got API surfaces for the simulation (game logic) server, the edge server and relay, and the game client. Up until last night, I hadn't really been able to put them to use in an end-to-end test.

In addition to needing to support huge scales and extremely fast operation, it needs to be easy for developers to build games on top of this framework. Ergonomics, ease of use, and hiding complexity are all key motivators.

## Enter the Lettuce
The first key milestone I wanted to achieve was to have some in-game thing be modified by the game loop and have the game client receive the updates. I decided that the MVP should be _growing some lettuce_, naturally.

From the player's perspective, they plant some lettuce and then watch the magic happen. The little lettuce seed turns into a sprout, which then turns into a growing plant, and finally you have harvestable lettuce. It's the circle of life!

### Plant a Seed
Planting a seed should be fairly simple for a player. They walk around, find an ideal spot to start their fabulous new lettuce garden, and they click or use hotkeys to plant a seed on that spot.

If the game thinks it's okay to plant lettuce there, it'll send a command to the simulator. The command is a _request_ for work to be done. Since the sim/game server is the sole authority, it has veto power and can reject commands.

The error handling below isn't handling a failure to plant, it's handling a failure to send the message. A failure to plant may get logged in different places, but ultimately nothing
happens on the sim server because of the rejection.

```rust
let cmd = GameCommand::PlantLettuce { x: 200, y: 200 };
match self.handle.entity_send(handle, &cmd.encode()) {
    Ok(()) => println!("sent: PlantLettuce at (200, 200)"),
    Err(e) => eprintln!("failed to send PlantLettuce: {e}"),
}
```

### Relay the Message
Now that we've planted our lettuce, the message gets relayed through the edge server. In most cases, you won't need to create your own edge. If simple relay is enough, you can use the **Umwelt** edge binary off the shelf, or you can create your own scaffolded edge server:

```rust
let server = EdgeServer::new(nats, runtime.handle().clone(), quic, |_handle| Game)
.unwrap_or_else(|e| {
    eprintln!("starting the edge: {e}");
    std::process::exit(1);
});
println!("mv-edge: {} listening on {listen}", server.name());
```
In the above, `Game` is the struct implemented on the edge server. For now, it's just empty:
```rust
struct Game;
impl EdgeGame for Game {}
```

### Process Messages in the Sim
Now we need to define the business logic for processing messages, which should result in planting some lettuce. Before we define the business logic, though, I need to create some data for what it looks like when we spawn lettuce, as well as how the entity's tag changes over time. The tag is opaque to **Umwelt** and is used to tell the simulator and the game client important metadata. 

First, I define a `lettuce` module. This has tag numbers (because I send tags as `u16`) showing the different stages of lettuce growth:

```rust
pub mod lettuce {
    use core::ops::RangeInclusive;

    pub const SEED: u16 = 1;
    pub const SPROUT: u16 = 2;
    pub const GROWING: u16 = 3;
    pub const RIPE: u16 = 4;

    pub const RANGE: RangeInclusive<u16> = SEED..=RIPE;
}
```
Defining a templated set of data and/or instructions that get set when a thing spawns into the world is often called a _prefab_, for pre-fabricated component. It makes it easy to standardize and find what really defines a spawned lettuce and how it behaves.

```rust
pub struct CropPrefabs {
    pub lettuce: PhasedDef,
}

impl CropPrefabs {
    pub fn new() -> Self {
        CropPrefabs {
            lettuce: PhasedDef::new(
                tags::lettuce::SEED,
                vec![
                    Duration::from_secs(2),
                    Duration::from_secs(3),
                    Duration::from_secs(10),
                ],
            ),
        }
    }
}
```
`PhasedDef` is a struct I created in the game space that makes it easy to define transitions in seconds when the simulation operates on units of ticks and the number of ticks per second varies. So if I want lettuce that grows a bit after 2 seconds, then 3, then finally becomes ripe 10 seconds later, I need something that converts seconds into tick increments.

### Yay for Lettuce!
Now I can put this all together in the game server: handle incoming commands, validate the lettuce plant command, and spawn the lettuce at phase 0 when everything works.

```rust
impl Game for MildewValleyGame {
    // Handle inbound messages and put them in an inbox
    fn message_received(&mut self, from: EntityId, body: &[u8]) {
        if let Some(cmd) = GameCommand::decode(body) {
            self.pending.push((from, cmd));
        }
    }

    fn step(&mut self, world: &mut Step<'_>) {
        self.inbound.apply(world);

        // Process every pending command in the inbox
        for (_, cmd) in self.pending.drain(..) {
            match cmd {
                GameCommand::PlantLettuce { x, y } => {
                    self.phased.spawn(
                        world,
                        Pos3::from_meters(x, y, 0),
                        &self.prefabs.crops.lettuce,
                    );
                }
            }
        }
        self.phased.advance_all(world);
    }
}
```
The `spawn` method on the phased collection puts the entity in the world, but notice how it takes a reference to a prefab. When `spawn` executes, it knows the engine's configured tick rate and so it can use that against the timings in the prefab to come up with the exact delay in number of ticks.

```rust
fn thresholds(&self, tick_hz: u32) -> Vec<u32> {
    let mut acc = 0u32;
    self.transitions
        .iter()
        .map(|dur| {
            let ticks = dur.as_millis() as u32 * tick_hz / 1000;
            assert!(ticks > 0, "transition shorter than one tick");
            acc += ticks;
            acc
        })
        .collect()
}
```

Every time a tag changes, that update propagates down to the game client while still respecting the priority accumulator. This lets the game change the sprite for the lettuce from a sprout to a growing lettuce head, for example.

## Wrapping Up
It might not seem like much, but this is a pretty huge milestone for the **Umwelt** library. A game developer (me) was able to define game-specific logic, such as entities that change tags/phases over time. The first MVP is growing a head of lettuce, but you can imagine all of the different things in a game like this that need tag phases: decay and rot, healing, combat, weather, mining resources, etc. 

Remember that the work already done so far was about extreme scale. So now that my game can spawn a single head of lettuce, it can spawn 10,000 without buckling under the pressure. So much leafy greens!

Now that I've got this MVP to build on, I can start working on some of the more fun parts of the game while I'm also putting the Umwelt library to the test.