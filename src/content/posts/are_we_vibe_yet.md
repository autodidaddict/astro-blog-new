---
title: Are we vibe yet?
published: 2025-08-04
draft: false
tags:
  - Haskell
  - AI
  - Vibe coding
  - Monads
  - MUDs
description: A measured experiment with AI-assisted development 
---

Last week I used my vacation (_don't judge me!_) time to do an experiment in AI-assisted coding. Developers have a saying where you haven't truly used a new technology until you've done so _in anger_.

Here are the parameters of my experiment:

* Develop AI-first/assistant-first (as much as possible)
* Build solution using a language in which I don't think natively - _Haskell_
* Create a [MUD](../monad_mud/), making as much progress as I could
* Experiment was time-boxed at 1 week
* Used Junie Ultimate inside IntelliJ, no Haskell plugins
* Occasional use of ChatGPT to compare and correct

I'd started stewing on the idea of using a monad stack to expose game functionality to otherwise pure Haskell functions. Without AI's help, I'd managed to get far enough to have built an "echo" server over TCP--the classic sockets "hello world"

## First Impressions
The hardest part of this experiment was figuring out which one of the hojillion available AI assistants I was going to use. I'd already gotten my "Junie Pro" subscription for a discount when I bought the IntelliJ IDE, so that seemed like a natural fit. 

I had ChatGPT to use in a browser tab as a backup in case I needed it.

## Think More and Token Quotas
During each day when I was working on this, I probably spent the equivalent of a full day's worth of work had I not been on vacation. This gave me a pretty good level set and fair comparison between work output and--what was probably more important to me--mental impact and fatigue.

Using Junie Pro, I burned through the entire month's worth of token quota at the end of my first day of AI-assisted coding. I had to upgrade to "Ultimate" in order to get enough quota to put in a full day's work with the AI assistant.

I can't find anything in my Junie subscription that tells me what my current usage is, but I'm 100% certain that I've been using more tokens than my subscription fee covers at a cost level. In other words, I'm pretty sure IntelliJ/Junie is losing money on me, despite the price of my subscription.

Some _anecdotal_ evidence is that on Thursday (day 4 of experiment), the Junie panel in IntelliJ stopped leaving the "think more" option checked. It now explicitly _unchecked_ that box after every request I made. The difference in response quality between "think more" and "don't think more" was night and day. The "think more" responses were usable and I was able to commit code from them, while the "don't think more" responses were confused, hallucination-filled, incoherent mashups of steaming trash.

I don't know what kind of developer IntelliJ thinks won't notice the difference between the cheap thinking and the real thinking, but they can't possibly be building anything even remotely complex.

## Your Scaffolding is your Context
Before I started building with AI, I had a decent setup of recommended modules. In fact, I got this recommended layout of modules from a prior ChatGPT discussion. My `main.hs` file had the entry point as well as the setup for the usual socket app: a _bind_, _listen_, _accept_, _spawn loop_ pattern.

It was in the spawned loop where all of the real important code would go, especially the monad runner(s).

I continually added features and refactored throughout the week. At the end of the experiment, this is the `Main` module I had:

```haskell
-- | Main entry point
main :: IO ()
main = withSocketsDo $ do
  hSetBuffering stdout NoBuffering
  let config = defaultConfig
  playerMap <- newTVarIO Map.empty
  objectsMap <- newTVarIO preloadRooms
  dummyState <- createDummyCommandState config playerMap objectsMap
  scriptMapResult <- loadPrototypeList dummyState (Map.keys preloadRooms)
  scriptMap <- case scriptMapResult of
    Left err -> do
      putStrLn $ "Error loading scripts: " ++ err
      exitFailure
    Right sm -> newTVarIO sm
  

  addr <- resolve (serverPort config)
  sock <- open addr (serverBacklog config)
  putStrLn $ "Server running on port " ++ serverPort config
  acceptLoop sock playerMap objectsMap scriptMap config clientHandler
```

You can see that, throughout the week, I went from having nothing but a scaffolded TCP chat server to having a robust system of rooms, in-game objects, and even behavior scripting. You can't see it in the `Main` module, but all the behavior scripting is in `Lua`, and it was all done via AI assistance.

Having the core scaffolding in place, including placeholder modules with decent documentation comments had a noticeable impact on the quality of the code generated from my requests.

## Core Features
Probably the single most important thing that the MUD server does is accept player input. The details of getting data from a socket aren't really the important thing. However, the AI assistant did properly give me all the code I needed to emit the right `telnet` control codes to hide user input during password prompt, as well as perform all of the necessary player loading and saving and password hashing. 

Enabling player login and the first command input was done in the first four hours of day 1.

Here's the current command handler (without the telnet-specific stuff):

```haskell
-- | Handle a command from the user
handleCommand :: T.Text -> GameM Bool
handleCommand msg
  | T.null msg = return True
  | T.head msg == '/' = handleSlashCommand (T.drop 1 msg)
  | T.head msg == '@' = do
      isWizard <- amWizard
      if isWizard
        then handleWizardCommand (T.drop 1 msg)
        else do
          writeLine "You don't have permission to use wizard commands."
          return True
  | otherwise = do
      handled <- handleMortalCommand msg
      if handled
        then return True
        else do
          writeLine "unknown command"
          return True
```

The first thing that hits me about this function is how _clean_ it is. This started its life as a single hard-coded reply that just told the player what they typed. As I added more command flavors, this function continued to stay clean, eventually reaching its current state.

It's pretty clear that the game supports slash (`/`), wizard (`@`), and mortal commands. The most important thing here is the `GameM` type, which is my game monad. All game-related functionality from persistence to the global object map is all done through this monad. 

When a script for an object wants to send a message to a player, that bubbles out of the Lua script and ultimately calls a function within the game monad.

Now let's take a look at one of the command handlers:

```haskell
-- | Registry of all available wizard commands
wizardCommandRegistry :: CommandRegistry
wizardCommandRegistry = 
  let emptyRegistry = Map.empty
      hereCmd = Command { cmdHandler = cmdHere, cmdHelp = "Display information about your current environment and list objects in the room", cmdPrimary = "" }
      allObjectsCmd = Command { cmdHandler = cmdAllObjects, cmdHelp = "Display a list of all object references in the global object map", cmdPrimary = "" }
      teleportCmd = Command { cmdHandler = cmdTeleport, cmdHelp = "Teleport to a target location", cmdPrimary = "" }
      helpCmd = Command { cmdHandler = cmdHelpHandler, cmdHelp = "Display help for available wizard commands", cmdPrimary = "" }
  in registerCommand ["here", "where"] hereCmd $
     registerCommand ["allobjects", "objects", "objs"] allObjectsCmd $
     registerCommand ["teleport", "tp", "goto"] teleportCmd $
     registerCommand ["help"] helpCmd emptyRegistry
```

There is an abstraction being used here called a `CommandRegistry`. This abstraction and all of the stuff you see in this function came from the AI assistant. The success{{< sidenote "sn-cmdregistry" >}}There is a `cmdPrimary` field that I need to get the AI to remove. Getting it to handle the concept of command aliases took several hours and dozenas of prompts. This took longer than I hoped because I didn't notice the "think more" box had unchecked itself.{{< /sidenote >}}
or failure of these abstractions all comes down to the quality and precision of my prompts. 

Before moving on to the prompting, here's a snippet of code that is invoked _by user-supplied Lua scripts_ to send a message to a player:

```haskell
case Map.lookup target players of
    Just (sock, _) -> do
	let formattedMsg = unPlayerName from <> " tells you: " <> messageText <> "\r\n"
        liftIO $ NSB.sendAll sock $ TE.encodeUtf8 formattedMsg
        return True
    Nothing ->
        return False
```

The `unPlayerName` function caught me by surprise, but it turns out Junie was correct in recommending this name as it's a pretty idiomatic naming convention for newtypes like `PlayerName`.

## Make Detailed, Specific Requests
Precision in communication with the AI assistant is crucial to the success of any AI-supported development life cycle.

Here's one of my prompts from my conversations during this experiment (my workdays typically had several dozen top-level prompts and many of those spawned long-running conversations) :

```
The code now has support for mortal commands. 
I've added a call to handleMortalCommand to the handleCommand
function in the Input module. This code is incorrect in 
that when a mortal command is supplied it complains 
about an invalid command.

I want command processing to check for commands of 
the following types in the following order:

1. / prefix commands
2. @ prefix commands
3. Mortal (no prefix) commands

If there is no command match for any of the above,
output "Unknown command: {player input}"
```

Note the tone here. I'm explaining to the assistant _exactly_ what I want. I'm phrasing my needs the way I might list detailed requirements for an issue that I assigned to a junior developer.

I can give it the logic that I want performed without having to resort to writing the conditions/pattern matching myself. More importantly, by specifying things at this level, the assistant is free to use idiomatic Haskell in the solution--something that I'm not likely to get right on the first try because I'm not (yet) a native Haskell thinker.

The **key fact** here is that I've _already done the design_. I've done the thinking to determine my requirements. I've done the dilligence that I would do in order to create a "good first issue" for an open source contributor that empowers someone to solve this issue in my absence.

Your AI assistant is a junior developer that does not operate well (or at all) in the absence of guard rails. They need explicit direction and guidance. The only thing they bring to the table is an ability to crank out vast amounts of _syntax_ in a short period of time.

So you're ready to hand off a task to your AI assistant when you can _precisely_ describe what you need and how you need/want it done.

You _**absolutely cannot**_ have the assistant design and architect solutions at any kind of high level. AIs _DO NOT_ think, they predict and infer. As the developer and designer, _you_ are responsible for providing enough information for the model to infer the correct before and after state of your code.


## Git Commit your Prompts
This may be one of the most important pieces of developer loop advice that I can give. A lesson learned the hard way is that rolling back or declining a change may not actually entire undo a thing. Or worse, it will undo more than you wanted and put you back in a failing state.

This is actually two pieces of advice in one. First: _only make one change per prompt session_. It's really easy to fall into the (arguably bad) habit of asking for lots of changes in a single prompt. Keep your changes small and discrete. This not only makes them easier to roll back, but more importantly, it makes the changes easier for you to _understand_ and _verify_.

Trust me when I say that rejecting your assistant's suggestions is a regular part of the new AI-supported iteration loop.

Another habit I formed is to _git commit my prompts_. While Junie (and other assistants) maintain the prompt history, one thing they _don't_ do (well) is maintain the correlation between your prompt and the code changed. 

What I do is when I've made an AI-generated or supported change, I will add the prompt directly to the commit message. Use `git commit` without supplying the usual `-m` parameter. This will bring up your default editor to create the commit message.{{< sidenote "sn-commits" >}}Most IDEs/editors have support or plugins for committing multi-line messages.{{< /sidenote >}}

As usual, the subject of the commit message is the first line. Leave a blank line, and then past the prompt that produced the code change exactly as you typed it.

This not only lets people on your team know that this commit was prompt-supported{{< sidenote "sn-shame" >}}There is no shame in this. AI-supported development is here now and isn't going anywhere.{{< /sidenote >}}, but it also lets everyone see _what_ prompt you used. 

There's many benefits for this. First, it lets your team learn from their prompts and the code produced. Additionally, you can now use your own git commit history as context in subsequent prompts to learn from and improve this dev loop.

## AI Lies
Junie (and all other assistants that I know of) are dirty, low-down, filthy liars. They will lie to your face and feel no remorse like a deranged serial killer. Junie would routinely tell me "all files in the project compiled successfully" and "all tests have passed". 

I would then drop into my own terminal session and `cabal build` would fail with dozens or even hundreds of errors. The same would happen for `cabal test` even when the project compiled.

Then I would spend sometimes dozens more interactions in that same session where I would paste in the compilation errors and tell it to fix them. It would then take multiple attempts, but it almost always produced something that worked.{{< sidenote "sn-worked" >}}Though this also frequently required subsequent refactoring to clean up.{{< /sidenote >}}

I don't know _why_ it lies so explicitly like this, but it does. If an assistant says it has run tests, don't trust them (see the "never trust" section).

Do not believe a word that comes out of their chats.

## Using Library Dependencies
If you need to use a 3rd party in your code, Junie was usually very good at working with that. It would take the time to go query online documentation and samples and then apply that context to the questions I asked.

Herein lies the problem: _Junie was only as good as the documentation for that library_.

This turned into an infuriating problem while I was trying to add support for Lua scripts. I was using the `HsLua` library. There was a drastic change in the API/SDK between old versions and version `2.4+`. 

I had to repeatedly "remind" Junie that I needed syntax from the 2.4+ versions and _not_ from the old versions. Part of the problem -- the documentation for that library contained references to the "old style" even in the newest versions of the docs. Since I didn't know the difference in syntax off the top of my head (otherwise I wouldn't need AI to help me), I couldn't vet the assistant's code with as much precision as it needed.

Takeaway: If you're making a library for other people to use in the AI "age", make sure you hide no longer supported syntax from the most easily accessible documentation you have. This _especially_ applies to the built-in code docs that get produced through package managers like Hackage for Haskell and **crates.io** for Rust, etc.

## Never Trust, Always Verify
It may be a hard pill to swallow, but **_if you cannot verify the correctness of the code coming from your AI assistant, you have no business using an AI assistant in the first place_**.

I'm not a Haskell expert. In fact, I'm not all that much above mid-level in Haskell. However, I know _just_ enough to be able to see when the assistant has made a horrible mistake. For the rest, I can be precise in my tests (discussed in an upcoming section) and have those help catch the most glaring errors.

Other times I can use a different model to examine small sections of code for correctness, precision, and idiomatic style.

## Split Plans and Executions for Complex Tasks
Sometimes I don't know ahead of time what the precise breakdown of tasks will be to build a feature. If what I want is going to affect a broad cross-section of code and potentially involve multiple changes affecting the same bit of code for multiple reasons, then I want to use my assistant as a _planner_.

Junie has two different modes: _ask_ and _code_. When I need one of these plans to break down my needs into smaller, more digestible chunks, I'll use _ask_ mode (and make sure the "think more" button is checked). 

Junie does a great job of breaking down a change into sub-tasks. If I'm not happy with the breakdown, I can have a longer interactive session where I progressively supply more context and refinement until it's come up with a plan that I approve.

Then I can either YOLO it and tell it to implement the entire plan or, what I do more often, is to tell it to "implement step 1" and then "implement step 2", etc.

The takeaway here is that the "ask" mode is better at long-term planning than the "code" mode. Don't overlook this and make sure you ask your assistant for plans as often as you like. It really has made a huge difference in code quality and correctness.


## Use Explicit Test Assertions and Requirements
Contrary to popular belief you can't simply tell your assistant to "write tests for this feature". Well, I suppose you _can_, but it's not a good idea. Most tests produced this way will pass 100% of the time but not actually prove anything useful.

When you're asking the assistant to generate tests, give a specific list of assertions that must be true or false before generating. I found that this frequently revealed bugs in the AI-generated code that I would've missed because _I lack the depth of knowledge of the codebase that someone would who wrote it without AI_.

Generative AI is _fantastic_ at producing test data. It can generate sample data in a matter of seconds that might have taken me all day to produce. More importantly, I no longer suffer the cognitive drain of that kind of busywork. _The AI does my busywork so I am not physically and mentally exhausted at the end of the day_.

## Conclusion - Are we Vibe Yet?
**Yes**, for the narrow definition of `vibe` that refers to using an AI assistant to _meet your specific demands_ without using it to produce architectures and designs from thin air. Vibe coding does not abdicate your responsibility as a developer who owns a code base. 

In fact, you have _more_ responsibility because you not only own the design and architecture, but you own the code produced by the AI. _You can't blame the AI for allowing bad code to make its way to production_--that's on us.
TBD
