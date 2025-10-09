---
title: "Making a MUD out of Monad Stacks"
published: 2025-06-25
draft: false
#author: "Kevin Hoffman"
description: "An experiment to isolate mud side effects from pure functions"
tags:
  - MUDs
  - Monads
  - Haskell
  - FP
---

For those of you not old enough to have experienced them, [MUDs](https://en.wikipedia.org/wiki/Multi-user_dungeon) are "Multi-User Dungeons" (or Dimensions, depending on your preference). The main thing you need to know about them is that there was a single server that played host to large (for their time) numbers of online players via the `telnet` protocol.

The way these servers work is pretty simple: they accept incoming TCP connections on a known port like `3000`. Each one of these new connections spins up a loop that sends messages to the player and accepts commands from them. Players then type things like `kill tomato` to attack the sketchy looking tomato standing nearby.

In a MUD written in an imperative language, side-effects and business logic would be impossibly entangled. Take the following code snippet:

```c
target->set_hp(target->get_hp() - 100);
write("You dealt 100 points of damage!");
```

Here, the code is mutating the hitpoints of the target and then using a function like `write` to send text directly to the current player's socket.

A couple nights ago, I was still feeling the effects of anesthesia from an outpatient procedure and I was bizarrely motivated to see if I could set up the framework for an elegant way to code a MUD by capturing side effects with Monad transformer stacks in Haskell. I know, some people make TikTok videos of themselves making poor decisions but I go back to MUD building. It's sad and I accept that.

In idiomatic Haskell, you'll often see a `runXXX` function that accepts the necessary initial state and parameters and accepts code in a `do` block as input. These "monad runners" make all of the functions of the monad type available to any code running "inside" the monad, even though the function chains don't actually see the input parameters. Much of the "implicit" magic here relies on partial application/currying, but it's fine if you want to just think of it as magic/sorcery.

My theory was that I could write a `runMUDCommand` monad runner every time we accept a new connection from a client. This tail-recursive function (no loops here!) would be running inside the monad, so any of the code it calls can, in theory, write to the player's socket, mutate local or global state, etc.

The first thing I needed was some state to weave through the monad runner:

```haskell
data CommandState = CommandState
  { clientSocket :: Socket
  , playerName :: String
  , playerList :: TVar (Map String Socket)
  }
```
This state will be available to anything running inside the monad. So let's take a look at my monad:

```haskell
newtype MUDCommand = MUDCommand { unMUD :: StateT CommandState IO a }
  deriving (Functor, Applicative, Monad, MonadIO, MonadState CommandState)
```

Here I'm getting a _lot_ of work done by deriving the monad hierarchy (`Functor` -> `Applicative` -> `Monad`). The short version of what's going on here is that the `unMUD` field is storing a `StateT` monad transformer. With the monad type in hand, I created the `runMUDCommand` function:

```haskell
runMUDCommand :: CommandState -> MUDCommand a -> IO a
runMUDCommand st action = evalStateT (unMUD action) st
```

With these in place, I can "simply" start writing functions that are available to _any_ function that runs within the monad. For example, writing to the player's socket:

```haskell
rawWrite :: String -> MUDCommand ()
rawWrite msg = do
  sock <- gets clientSocket
  liftIO $ NSB.sendAll sock (B.pack msg)

writeLine :: String -> MUDCommand ()
writeLine s = rawWrite (s ++ "\r\n")
```

Now we can hopefully reap the benefits of this architecture. We should be able to write pure functions that call functions like `writeLine` as side-effects. The prototype that I built the other night lets players log in (no storage), get commands echoed back to them, and even use `/tell` to send a message to another connected player.

Let's take a look at my `main`:

```haskell
main :: IO ()
main = withSocketsDo $ do
  hSetBuffering stdout NoBuffering
  playerMap <- newTVarIO Map.empty
  addr <- resolve "3000"
  sock <- open addr
  putStrLn "Server running on port 3000"
  acceptLoop sock playerMap
```

The `<-` left arrows are (I'm taking huge liberties here to avoid getting into complex details) "monadic assignments". It's pulling a value from a monadic function call and storing it in a variable and then moving on to the next statement in the `do` list without propagating it. This is syntactic sugar for chaining Haskell's oh-so-fun `>>` and `>>=` operators.

The `acceptLoop` function is where the magic happens. I won't dump the whole thing here, but I'll show where this function uses `runMUDCommand` to illustrate my entire goal for writing this sample.

After a player successfully connects inside the accept loop, I create a new instance of `CommandState` and then use `runMUDCommand` to add the player to the _global_ player list. This is subtle but powerful - the command state is _local_ to the function running inside the monad, but because we're using a `TVar`, it's basically a local pointer to a global atomically wrapped value.

```haskell
let st = CommandState conn name playersTVar
runMUDCommand st $ do
  pl <- gets playerList
  liftIO $ atomically $ modifyTVar' pl (Map.insert name conn)
```

And now the guts of the input handling loop:

```haskell
-- this gets run before the loop
runMUDCommand st $ do
  writeLine $ "Welcome, " ++ name ++ " to Kevin's delusional universe!"

let loop = do
  msg <- recvLine conn
  if null msg
    then disconnect
    else do
      keepGoing <- runMUDcommand $ st handleCommand msg
      if keepGoing then loop else disconnect
  
  disconnect = do
    putStrLn (name ++ " disconnected.")
    runMUDCommand st $ do
      pl <- gets playerList
      liftIO $ atomically $ modifyTVar' pl (Map.delete name)
    close conn

loop
```

Hopefully you're seeing the pattern now. Any time we want to run a function and give it the ability to interact with the game and with the player, we just run that function "inside" the `MUDCommand` monad via `runMUDCommand`. In the last code sample, you can see me using `runMUDCommand` to add and remove players from the global connection list. 

The `handleCommand` function is essentially where the rabbit hole starts. This is where we (hopefully) will have functions that handle player commands like `"wield can opener"` and `"attack mouse"`.

`handleCommand` does a split and then invokes a separate `handleSlashCommand` function. This is where commands like `/tell`, `/who`, and `/quit` are defined.

```haskell
handleSlashCommand :: String -> MUDCommand Bool
handleSlashCommand input = case words input of
  ["who"] -> do
    pl <- gets playerList
    players <- liftIO $ atomically $ readTVar pl
    writeLine "Connected players:"
    mapM_ writeLine (Map.keys players)
    return True

```

Another pattern that is emerging is that the function's type signature says it's returning a type of `MUDCommand Bool` (remember `MUDCommand` actually takes a type parameter `a`, similar to generics in other languages) but you don't actually see any code that constructs a new `MUDCommand`. Here we have `return True` and that somehow remains _within_ the `MUDCommand` monad.

This is the key to creating a (theoretically) elegant and extensible library of MUD functions. Any function that you want to run inside the `MUDCommand` monad just needs to indicate that in its type signature because the monad itself isn't represented as a variable as it is "higher" than the function itself.

Assuming I decide to spend more time on this between now and my next outpatient procedure, my plan for the next step is:

* Optionally add wizard commands to the monad stack so that the type system enforces access to those commands. A player without the wizard transformer in the stack will fall through to the bottom error handler while a player with that wizard transformer will support commands like `/summon` or `/boot` etc. Core goal: support wizard and non-wizard commands without using an `if` or `case` expression.
* Add a logger to the stack that adds the current player's name to the log emission, e.g. `[bob] booted user 'alice' from the game`.

Stay tuned, I'll either continue with this or I'll drop it like I do all my other side projects!
