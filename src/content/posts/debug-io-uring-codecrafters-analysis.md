---
title: Five Wrong Theories and a Socket Table - Debugging eio on CodeCrafters
published: 2026-06-21
draft: false
tags:
  - Codecrafters
  - OCaml
  - eio
  - Redis
  - Debugging
  - Analysis
description: I return to the scene of the crime to re-investigate my "Build your own Redis" OCaml eio submission for Codecrafters
---

Many moons ago, I started a [Codecrafters track on building my own Redis](https://app.codecrafters.io/courses/redis/overview). I love this kind of stuff because not only does it force you into constraints
dictated by the protocol, but you also get to see how the sausage gets made in whatever language you like. I'd started this track in **Haskell** and
then for giggles (or pain, depending) I started a new **OCaml** track.

In this magical day and age where AI assistants write all of our code, it feels even more important to me to keep the code-writing portion of my brain fresh 
and active.

I'd made it through to the _Handle multiple pings_ step pretty easily. Though, when I look at the code now, it seems I've forgotten a great deal of OCaml,
so returning to the scene of the crime here is paying double dividends. The real symptom, which I didn't notice the first time around, was that the failure is never a specific stage. In this run `Handle concurrent clients` passed and `Respond to multiple PINGs` failed; the
  loser is always whichever stage runs second.

If you look at the code samples for this problem, a very, very small portion of them use OCaml's **eio** library. This library is an I/O library
built on top of OCaml's new effect system (which I've posted about once already). I didn't have many existing samples to go on, and I also didn't
want to spoil the adventure by skipping to the answer.

I've recently come across some reading that reminded me that the analysis is just as important as the solution, if not more so. In other words,
the journey is as important as the destination. With this in mind, I figured I would recount my journey through a difficult yet satisfying problem.

I had code that worked against the `redis-cli` tool. As far as `redis-cli` was concerned, I'd made a good enough server to handle multiple 
concurrent `PING` requests. The problem was that when I ran the Codecrafters test harness (via `codecrafters test` or just pushing the code up), I got some 
weird errors. Specifically, the one that I'd given up on a long time ago:

```
Received: "" (no content received) … Expected start of a new RESP2 value
```

When I ran the test harness, the suite passes one stage, then fails the next one. The failing server logs `Running server` but never `Reading line`. The connection never arrives.

## Wrong Theory 1 - SO_REUSEPORT
I've seen this kind of mess before, especially when it comes to test harnesses running concurrent code. Thinking about how the harness restarts between stages, my "starter hypothesis" was that `~reuse_port:true` was causing the incoming connection to route to the previous (dying) process's socket.

To verify this, I dropped `reuse_port` and re-ran the suite. The tests still fail but this was a useful detour. I was able to verify that `codecrafters test`
runs against my local code and not the latest commit. You might think this is a trivial detail, but ask yourself this: _how many times have you spent hours 
debugging something only to find that you weren't testing the code you'd been fixing_? I do this _all the time_ and it's now something I force myself to check,
no matter how silly it seems.

## Wrong Theory 2 - Swallowing Exceptions
Looking through the code I saw that my younger self had used `_exn` as a variable name. This was basically swallowing a server execution exception. Maybe 
I'd discover the culprit if I used and dumped that variable. Adding some debug prints (everyone's first go-to debug tool), I was able to see 
 `listen succeeded` and `Running server` but no exception occurred. Useful information here is that `accept` just sits there and never does a handoff. Now 
 to figure out what's happening there.

## Wrong Theory 3 - Old Process is Still Bound
This is the theory I had when I walked away from this problem all those many months ago. The hypothesis was that a server is still holding onto the port,
and so when the harness starts up the next process, the next one doesn't accept and the old one eventually dies. To verify this, the
plan was to check `/proc/net/tcp`[^1] at the start of the failing stage. 

The result of the experiment was that there was only one `LISTEN` socket, and it was mine. The old process was gone by that point and the only other information
in there was a few `TIME_WAIT` rows left dangling from the previous server.

### Decoding /proc/net/tcp
 
  Every TCP socket in the namespace is one line. The header:

```
  sl  local_address rem_address   st tx_queue:rx_queue tr:tm->when retrnsmt  uid timeout inode  ...
```
Here are two actual rows from the failing stage. The fresh listener and a leftover from the previous stage:

```
   0: 0100007F:18EB 00000000:0000 0A 00000000:00000000 00:00000000 00000000  0  0 1635131 ...
   1: 0100007F:C6D8 0100007F:18EB 06 00000000:00000000 03:0000176F 00000000  0  0       0 ...
     └local──┘ └p─┘ └remote─┘ └p─┘ └┘                  └┴timer──┘             └inode┘
```
Of course this is all complete gibberish so I had to decode everything up to inode (after the inode is all kernel internal goop)
```
  ┌───────────────┬──────────────────┬──────────────────┬───────────────────────────────┐
  │     Field     │ Row 0 (listener) │ Row 1 (leftover) │            Meaning            │
  ├───────────────┼──────────────────┼──────────────────┼───────────────────────────────┤
  │ local_address │ 0100007F:18EB    │ 0100007F:C6D8    │ local IP:port (hex)           │
  ├───────────────┼──────────────────┼──────────────────┼───────────────────────────────┤
  │ rem_address   │ 00000000:0000    │ 0100007F:18EB    │ remote IP:port (hex)          │
  ├───────────────┼──────────────────┼──────────────────┼───────────────────────────────┤
  │ st            │ 0A = LISTEN      │ 06 = TIME_WAIT   │ connection state              │
  ├───────────────┼──────────────────┼──────────────────┼───────────────────────────────┤
  │ tr:tm->when   │ 00:00000000      │ 03:0000176F      │ active timer : ticks left     │
  ├───────────────┼──────────────────┼──────────────────┼───────────────────────────────┤
  │ inode         │ 1635131          │ 0                │ socket inode (→ /proc/PID/fd) │
  └───────────────┴──────────────────┴──────────────────┴───────────────────────────────┘

  Decoding an address — 0100007F:18EB:
  - IP is a 32-bit value in little-endian byte order: 01 00 00 7F → reversed → 7F.00.00.01 = 127.0.0.1.
  - Port is plain hex: 0x18EB = 6379. (So C6D8 = 50904, an ephemeral client port.)

  State codes (st):

  01 ESTABLISHED   05 FIN_WAIT2    09 LAST_ACK
  02 SYN_SENT      06 TIME_WAIT    0A LISTEN
  03 SYN_RECV      07 CLOSE        0B CLOSING
  04 FIN_WAIT1     08 CLOSE_WAIT   0C NEW_SYN_RECV
```

## Wrong Theory 4 - Eio Isn't Draining its Accept Queue
Related to the previous theory is the idea that maybe `eio` isn't cleaning up after itself quickly. If this is the case, then maybe the connection
is going to the right place, but `accept` doesn't return it along with control. To check this theory, I created a watcher fiber[^2] that dumped the 
socket table every 200ms while `accept` is blocking and a manually coded `accept` loop with logging. 

The result: the event loop is healthy (watcher keeps firing), and the tester's connection never appears in the `/proc/net/tcp` table at all; not even `SYN_RECV`. 
It's starting to look less like this is an `eio` issue because the connection itself at the kernel level is never making it to my process.

## Wrong Theory 5 - The Old Process Takes Too Long to Die via SIGTERM
The test harness gets rid of the previous server process via `SIGTERM` (⚠️ _assumption_). So the
hypothesis here was that the old `eio`-based process loiters after `SIGTERM`, hanging around long
enough that its listener is still up when the next stage's client dials in. The experiment to verify
this was to use Docker + Linux (this is what `codecrafters test` uses; I was testing on my Mac),
measure how long the listener survives teardown via `/proc/net/tcp`, and test both backends.

There are two different backends available for `eio` on Linux: `io_uring` and `posix`. The `io_uring`
backend held onto its listener for **4-6ms** after `SIGTERM`; the `posix` backend was gone in around
**1ms**. Under both, the process was effectively dead almost immediately.

So the theory was wrong. Nothing _loiters_. There's no hung process, no listener camped on the port
for any meaningful length of time. `io_uring` clings to its listener several times longer than `posix`, 
and that few-millisecond gap turned out to be the whole enchilada.

## Log Comparisons
Comparing the passing test logs and the failing test logs showed an interesting _ordering_ symptom. For the passing test suite, I saw the `listen` and then a 
client `Connected`. For the failing test suite, I saw `Connected` then `Sent bytes` and _then_ `listen`. At this point there's another fact worth keeping 
top of mind: when I used the `Lwt` library instead of `eio`, I didn't have any errors. It worked just fine. _The foot is agame, Watson!_.

## Potential Root Cause - The io_uring Shutdown Window
The `io_uring` backend's teardown _quiesce_ keeps the listening socket completing handshakes for a few milliseconds after `SIGTERM`. Codecrafters test 
hands off stages quickly enough that the next tester's dial is in that window on the dying server and gets dropped on exit. Libraries that close
 the fd synchronously on process exit (`Lwt`, plain Unix sockets, `eio`'s own posix backend) have no async-teardown window while `io_uring` does.

## The Backend Sledgehammer
There was a blunt force trauma fix for this problem. I simply included `Unix.putenv "EIO_BACKEND" "posix"` at the top of my code and, like magic, it worked.
This little setting forces `eio` to use the `posix` backend and therefore clean up its sockets synchronously upon exit. This did the job and my tests
passed but it felt, well, _sledgehammery_.

## The Backend Rubber Mallet
Wanting to try something a little less brute force than forcing the backend usage, I decided to try and implement a graceful shutdown. This 
would give me a chance to close the listener before teardown, forcing the next dial to get rejected and cause a retry, which ultimately connects
the two ends like lovers running at each other across a field in slow motion.

My first attempt at graceful shutdown resolved the "stop promise" inside the signal handler. This worked, but according to the `eio` docs, `Promise.resolve`
is not **signal-safe**. I might never run into an edge case where this lack of signal safety burns me, but this is one of those things where you know
the fire is going to be _massive_ if it ever does. 

The idiomatic `eio` way to deal with this was to use an `Eio.Condition`. The handler only does signal-safe broadcasting, and then a _daemon fiber_ does
the rest of the work. So this is what I ended up with.  Where `Promise.resolve` is not signal-safe, `Eio.Condition.broadcast` _is_.  The machinery
here is similar to a `tokio::Notify` in Rust or `sync.NewCond()` in Go.

```ocaml
open Eio.Std

let addr = `Tcp (Eio.Net.Ipaddr.V4.loopback, 6379)

let () =
  Eio_main.run @@ fun env ->
  Switch.run @@ fun sw ->
  let socket =
    Eio.Net.listen (Eio.Stdenv.net env) ~sw ~reuse_addr:true ~backlog:128 addr
  in  
  let shutdown = Eio.Condition.create () in
  let signalled = Atomic.make false in
  let on_signal _ =
    Atomic.set signalled true;
    Eio.Condition.broadcast shutdown
  in
  Sys.set_signal Sys.sigterm (Signal_handle on_signal);
  Sys.set_signal Sys.sigint (Signal_handle on_signal);
  let stop, set_stop = Promise.create () in
  Fiber.fork_daemon ~sw (fun () ->
      Eio.Condition.loop_no_mutex shutdown (fun () ->
          if Atomic.get signalled then Some () else None);
      Promise.resolve set_stop ();
      `Stop_daemon);
  Eio.Net.run_server ~stop socket Server.handle_client
    ~on_error:(fun ex -> traceln "connection error: %a" Fmt.exn ex)
    ~max_connections:1000
```
In this code, the `on_signal` handler toggles the atomic `signalled` to `true` and then uses `broadcast` to poke the fiber daemon. When this 
daemon reacts to the poke, it resolves the stop promise. One thing I still find difficult about this code is there's no explicit cleanup
anywhere, there's the signal handler, the broadcast->fiber daemon call, and then in the daemon code we just see a promise resolution and `Stop_daemon`.

The answer is that `eio` ties resources to a `switch`, similar to a `using` block in .NET. In this case, the `~sw` switch owns the socket, so
when `run_server` returns and `Switch.run` exits, the switch automatically closes the listener. I'm sure if I was more used to OCaml, this would
feel more natural and less like the "implicit magic" I generally dislike.

## Conclusions
I took a couple of important things away from this exercise. The first is to observe the kernel instead of guessing about the runtime. I didn't realize
I had more tools available to me to do this than before. There's no shame in asking your AI coding buddy to give you a list of techniques to observe
the information you need.

Another takeaway is more of a re-affirmation. If something works locally but fails in CI, it's rarely ever a real logic bug and is more likely a timing
problem or a race condition. 

Maybe the most important one for me was that fix can look like the right answer but be wrong for subtle reasons
you may not discover until it's too late. I try and force myself to check the code docs for whatever code I add as part of a fix. 

Also - `eio` really shouldn't behave this way. It should "just work" with the same logic that worked with `Lwt`. I'll have to stew on whether this is something that
warrants an issue filed against `eio`.

---
[^1]: Holy crap! I had no idea I was allowed to use this as a debugging tool live. I'm definitely "that guy" who has been doing things the hard way for decades.
[^2]: Claude helped me write the watcher fiber. I hadn't even used the fiber system before this. 
