---
title: How many refactors does it take to remove an if statement?
published: 2026-06-22
draft: false
tags:
  - OCaml
  - Codecrafters
  - Redis
  - Refactoring
  - eio
  - Debugging
description: Refactoring the hard coded pattern matching in my Redis server to an extensible and clean foundation.
---

In my [previous blog post](../debug-io-uring-codecrafters-analysis/), I walked through my analysis process in diagnosing and fixing a socket race condition. In this post, I want to talk about the refactoring I did after that step as preparation for the next stage in the Codecrafters challenge.

If you've ever done test-driven development or even just written unit tests, you're probably familiar with the state where the tests "pass but the code is broken". That's where I left things off in the previous stage. My code is a server that "passes" the PING and concurrency tests by having a single pattern match arm on the hard-coded word `PING`.

The next stage of the Codecrafters challenge is to add support for the Redis `ECHO` command. This isn't really adding a feature so much as it is exposing the duct-tape-and-bubble-gum foundation on which my whole server was built. 

At a moment like this, I'm forced to ask myself if I want to keep working in placeholder/hack mode, or if I want to pause to make some parts of my code correct _for realsies_. Obviously I chose to make it work for real, otherwise I wouldn't be babbling about it in a blog post. 

**What I want to build**: an upgraded project layout that has room to grow, a real parser (though I don't want to deal with lex/yacc yet, if ever), a real encoder instead of doing a string compare on a literal, typed commands, better tests (by _better_ I mean _more than zero_). I also want my new parser to work on streaming bytes and **not** rely on a fully formed string that I can just split on whitespace.

The whole time I'm performing each of these refactoring steps, I need to keep Codecrafters <font color="green">green</font>. Hidden inside my motivation to start supporting real portions of the Redis serialization protocol is that `RESP` is length-prefixed and _binary-safe_. If I were to try and `ECHO` a string to my server that _included_ its own newline characters, I'd blow up the parser. Since `ECHO` needs the argument, even if it has multiple newlines, I obviously have to remove the hackery and shenanigans, perhaps even the hijinks.

Before I get into what I did to make things better, let's review the ugly, shall we?

```ocaml
open Eio.Std

let traceln fmt = traceln ("server: " ^^ fmt)

module Read = Eio.Buf_read
module Write = Eio.Buf_write

let handle_client flow addr =
  traceln "Reading line from %a" Eio.Net.Sockaddr.pp addr;

  let from_client = Read.of_flow flow ~max_size:100 in
  let rec read_loop () =     
    try
      let line = Read.line from_client in    
      let linestr = Printf.sprintf "%S" line in
      
      traceln "Received: '%s'" linestr;
      if linestr = "\"PING\"" then
        Eio.Flow.copy_string "+PONG\r\n" flow
      else
        traceln "not a ping";
      
        
      read_loop ()
    with
    | End_of_file -> 
        traceln "client closed connection.";
        Eio.Flow.close flow
    | exn ->
        traceln "Error handling client: %a" Fmt.exn exn;
        Eio.Flow.close flow
    in
      read_loop()
```

One has to chuckle when they see this:

```ocaml
if linestr = "\"PING\"" then
    Eio.Flow.copy_string "+PONG\r\n" flow
else
    traceln "not a ping";
```

Naturally, this approach is not sustainable. Input parsing, dispatching, and output encoding can't all be in the same function.

The net visible result to the Codecrafters engine will be that my tests still pass and nothing else has changed. Yay.

## Housekeeping
The first thing I did was rearrange everything into a proper project structure. I moved the executable into `src/dune`, created a `resp` library in the `resp` folder, and created a `test` folder. I had no idea what canonical dune-based OCaml projects were supposed to look like. So I went and found some open source OCaml projects and sifted through the structure and then had Claude give me an analysis of the structure and which pieces were personal preference of the maintainer, and which pieces are load-bearing. This was an enlightening activity. 

I made sure that `dune build src/main.exe` would work. Then I added the `alcotest` dependency as a test-only dependency so that I could write my own unit tests that wouldn't interfere with the test scripts used by Codecrafters.

I left the housekeeping phase with a single test that asserted that `true == true`. This is good enough for my scaffolding, because I now have a known good foundation that I can iteratively expand.

Let's take a look at what my `Value` type now looks like to cover `RESP`'s somewhat recursive (maps, arrays, sets, and stacks all take `t` as the type argument) serialization scheme:

```ocaml
type t =
  | Simple_string of string
  | Simple_error of string
  | Integer of int64
  | Bulk_string of string option
  | Array of t list
  | Boolean of bool
  | Double of float
  | Null
  | Map of (t * t) list
  | Set of t list
  | Push of t list
  | Blob_error of string
  | Verbatim_string of {
      format : string;
      value : string;
    }
  | Big_number of string
```

I always feel better after a quick whole-house vacuum.

## A Parser as a Pure State Machine
I imposed a couple of constraints on myself for this part. I wasn't going to use `yacc` or `lex` or any of the heavyweight tools that automatically generate parsers from Backus-Nauer Form language definitions. The Redis RESP format is length-prefixed, which means I can read it as I go without all the heavyweights.

First, I wanted my parser to be "stateless"[^1]. If I was making a DSL, this requirement would probably crush me but it's an advantage here. In my head, a parser is a pure function. It takes context and then some additional buffer of further content. The parser doesn't know or care if that buffer is done accumulating or not.

The parser state can be "waiting for type" or "waiting for length" or "waiting for data" or "done" or "error", etc. In Redis, the type for a bulk string is `$`. This type indicator is immediately followed by the payload length. This means `$4\r\nHIYA\r\n` is a complete bulk string. Note that there's a newline after the length parameter, _which makes parsing this even easier_. I don't want to have to figure out if I just read the number `4` and there are 3 more digits coming for the length or no more digits. It's almost like the people who wrote the spec for `RESP` knew what they were doing!

Here's what's fascinating about my pure state machine parser: _it didn't live more than a commit or two_. The first versions of parsing worked on a low-level type and so when I read data off the wire, I converted it into strings and then passed it through to the parser.

Crucially, this meant that _I owned the buffer_. If I own the buffer then I need to support the suspend and resume behaviors I expect. One of the states could be `Reading_data` with 5 bytes read and 300 more to go. I thought I was being clever with a state machine like this, then I switched my wire reading to use `Eio.Buf_read` directly. This meant that Eio dealt with all of the pause and resume crap, the explicit states of the state machine can be represented as _stack frames_ in the call stack capturing the input parameters as "state".

## Enter the Recursive Descent Parser
So now that I was using `Eio.Buf_read` and I _wasn't_ using an explicit state machine, I could actually write _parser combinators_ that essentially control the chain of parser execution on that stack. It
becomes a far more elegant and extensible solution.

There's a trade-off here in that 6-months-from-now me is likely going to have a few head-scratch moments to rehydrate the context around what the parsers are doing. That said, that moment is better than 
maintaining the alternative and, if I'm honest, forcing future me into one of those moments where I have to stop and analyze a piece of code is a good thing.

If you look at the code below (yes, that's the _entire_ parser module), you can see that the `value` parser is _recursive_. We grab the data type indicator character (I told you the protocol builders were smart), and depending on the protocol and what subset the code supports, it invokes other parsers with the _remainder of the unparsed input_ (kinda like how I was doing it the hard way with the state machine).

```ocaml
module R = Eio.Buf_read

(* Read a header count line (the bytes after a type byte, up to CRLF) as a
   non-negative int. *)
let count ~(what : string) (r : R.t) : int =
  let line = R.line r in
  match int_of_string_opt line with
  | Some n when n >= 0 -> n
  | _ -> failwith (Printf.sprintf "invalid %s length %S" what line)

let rec value (r : R.t) : Value.t =
  match R.any_char r with
  | '$' -> bulk r
  | '*' -> array r
  | c -> failwith (Printf.sprintf "unexpected type byte %C" c)

and bulk (r : R.t) : Value.t =
  let n = count ~what:"bulk string" r in
  let body = R.take n r in
  R.string "\r\n" r;
  Value.Bulk_string (Some body)

and array (r : R.t) : Value.t =
  let n = count ~what:"array" r in
  (* Parse elements left-to-right; the explicit recursion fixes evaluation order
     (unlike List.init, whose order is unspecified). *)
  let rec elements k acc =
    if k = 0 then List.rev acc else elements (k - 1) (value r :: acc)
  in
  Value.Array (elements n [])
```

The `count` function even accepts a friendly `what` argument so that if it fails to read it, it can explain what it failed to read. The `and` syntax here for OCaml modules still leaves me a little confused, but I understand it on a basic level. It's syntactic sugar so that I don't have to repeat the `let ...= ... in` over and over. It's now `let .. = ... in ... and ... and ... and ...`

The tl;dr here - terse and (hopefully) readable syntax lets me produce `Value` data from a buffered input stream.

## The Opposite of Parsing: Encoding
In order to support the `ECHO` requirement, I need to be able to send that command along with its parameter on the wire. I could keep up my current pattern of just hard-coding stuff right into the buffered read loop, but that's ugly.

Encoding performs the opposite job of parsing (aka `decoding`). What I want is a function that takes a `Value.t` and returns a `string` that is suitable for transmission over the wire to a Redis client (or server). It doesn't take much more work to encode the entire `RESP` format as opposed to just a few `Value` variants, so here's the full encoder module:

```ocaml
let rec encode (value : Value.t) : string =
  match value with
  | Simple_string s -> "+" ^ s ^ "\r\n"
  | Simple_error s -> "-" ^ s ^ "\r\n"
  | Integer i -> Printf.sprintf ":%Ld\r\n" i
  | Bulk_string None -> "$-1\r\n"
  | Bulk_string (Some s) -> Printf.sprintf "$%d\r\n%s\r\n" (String.length s) s
  | Null -> "_\r\n"
  | Boolean b -> if b then "#t\r\n" else "#f\r\n"
  | Double f -> Printf.sprintf ",%s\r\n" (encode_double f)
  | Big_number s -> Printf.sprintf "(%s\r\n" s
  | Blob_error s -> Printf.sprintf "!%d\r\n%s\r\n" (String.length s) s
  | Verbatim_string { format; value } ->
    let payload = format ^ ":" ^ value in
    Printf.sprintf "=%d\r\n%s\r\n" (String.length payload) payload
  | Array items -> aggregate '*' (List.length items) items
  | Set items -> aggregate '~' (List.length items) items
  | Push items -> aggregate '>' (List.length items) items
  | Map pairs ->
    let body =
      pairs
      |> List.concat_map (fun (k, v) -> [ encode k; encode v ])
      |> String.concat ""
    in
    Printf.sprintf "%%%d\r\n%s" (List.length pairs) body

(* Aggregate types share a [<prefix><count>\r\n] header followed by their
   elements encoded back-to-back. *)
and aggregate (prefix : char) (count : int) (items : Value.t list) : string =
  Printf.sprintf "%c%d\r\n%s" prefix count
    (items |> List.map encode |> String.concat "")

and encode_double (f : float) : string =
  if Float.is_nan f then "nan"
  else if f = Float.infinity then "inf"
  else if f = Float.neg_infinity then "-inf"
  else Printf.sprintf "%.17g" f
```
The only thing that kind of threw me and I got my AI buddy to explain to me was the `%.17g` format string. This is apparently the exact same format that Redis uses for its floating point precision when representing doubles in RESP values. I try and avoid doubles when encoding and decoding for exactly this kind of situation, so I'd never seen Redis use this format.

I find the initial `match value with` expression to be very pleasing. It's not quite zen, but it's more friendly and relaxing than a similar switch would be in other languages[^2].

## A Sidebar on MLI Files and Public APIs
I wanted to take a second to talk about `mli` files in OCaml. When I first encountered them, I
thought they were really just an encapsulation of some best practices. For instance, maybe someone
had decided that certain types of declarations are more easily found in `mli` files.

Turns out, they're far cooler, far more powerful, and we've seen this pattern before. They are 
first-class compiler-aware entities. They declare the publicly accessible API of that module. If you
don't have an `mli` file, then everything in your `ml` file is visible to other modules. I don't 
like the everything-visible defaults. I prefer starting from nothing visible and slowly narrowing
based on what I need. I might've developed this practice (bad habit?) from re-exporting in Rust and Haskell.

When I was still building the state machine by hand, I had a `partial` type. This was used as the outcome of the state machine advancing. The consumer of the `partial` didn't need access to _any_ of its members. All they needed to do was carry it around like an opaque token. So, in the `mli` file, there was just a line that had `type partial`. Inside the `ml` file with the actual code, there were a number of accessors. This didn't erase the internal members or cause any kind of data deletion, it just controlled the surface area with which clients interact. 

The more I used the `mli` files the more I wanted to know - Do people _really_ enter the same type definition in both the `mli` and the `ml` files as part of their OCaml day-to-day? _Apparently they do_, and _they like it that way_. 

When I thought about it some more, I started to like that pattern. Rather than having a single data type definition that's littered with `pub` and `private` and `protected` and `kinda-obscure` etc, the public and internal definitions are kept clean. The duplication of work is an acceptable tradeoff for OCaml developers.

This reminds me of how `.h` files work in a very unused, esoteric, hardly-ever-deployed language. You might not have heard of it, it's called **C**.

Here's the inside of my `parser.mli` file. The _only_ thing accessible is a function that takes an eio read buffer and returns a `Value.t`. The public API doesn't include functions like `count` or `bulk` or `array`, etc. As I end up supporting more and more types, the payoff of the `mli` file will likely grow exponentially.

```ocaml    
val value : Value.t Eio.Buf_read.parser
```
The same is true of the `encoder.mli` file:

```ocaml
val encode : Value.t -> string
```

That's the only function usable or visible to code outside the encoder module itself.

One parting thought before leaving `mli` files behind. They are a _compiler contract_. If the definition in the module changes in a way that conflicts with the interface, the build fails. Like Rust's borrow checker, this behavior falls squarely in the "trust me, you'll thank me later" category.

## Testing Strategy and Justifying Alcotest
I don't want to spend too much time here because you can read the `alcotest` docs if you want. It's an additional dependency, so I wanted to make sure that adding this to the project was not only worth it, but wouldn't interfere with the `codecrafters test` operation. 

OCaml has a weird (to me) behavior where you can create different executables for your tests. Running the executable is basically running the test. This came in handy because it let me keep all of the Codecrafters plumbing on its own in isolation while I created my own executables for unit tests.

![You guys are writing unit tests?](/images/you_guys_unit_tests.jpg)

With the API getting narrowed through the introduction of the `mli` files, my tests stopped making assertions against internal state and instead validated the behavior of the functions. I think this is a better kind of test anyway as it lets me refactor internals without breaking a test for no good reason.

## Lifting RESP Values into Typed Redis Commands
Soooo I might be a bit of a nerd here but I get a little giddy whenever I get to talk about lifting types into other types. I blame Haskell and pretend like this has nothing to do with my deviant proclivities.

What I want to accomplish here is to convert a (potentially deeply nested) `Value.t` into a _command_. A command is a strongly-typed representation of a command available in the Redis protocol. For example, both `Ping` and `Pong` are commands, as is `ECHO`. I don't want to have to continually deal with the lower level shape of values and bulk strings and arrays of values, etc. If my code needs to take action based on a command, I want it to be a type I can reason about.

Take a look at the code from `command.ml` (there's a corresponding `mli` file with, you guessed it, nothing but the type signature of `of_value`):

```ocaml
type t =
  | Ping
  | Echo of string

let of_value (value : Value.t) : (t, string) result =
  match value with
  | Array (Bulk_string (Some name) :: args) -> (
    match (String.uppercase_ascii name, args) with
    | "PING", _ -> Ok Ping
    | "ECHO", [ Bulk_string (Some msg) ] -> Ok (Echo msg)
    | "ECHO", _ -> Error "ECHO expects one argument"
    | other, _ -> Error (Printf.sprintf "unknown command '%s'" other))
  | _ -> Error "expected a non-empty array of bulk strings"
```

Here `Value.t` is acting almost like an AST. I decompose the AST and, based on the structure, I can create strongly typed commands ripe for pattern matching and dispatch.

## Shipping the ECHO Feature (Finally!)
So now that my detours have taken detours and I've refactored my refactorings, it was actually time to implement the `ECHO` feature. As you may have guessed, all of this prep work made it super easy to add this command to the list of commands supported by my server (below is `server.ml`):

```ocaml
open Eio.Std
open Resp

(* Prefix all trace output with "server: " *)
let traceln fmt = traceln ("server: " ^^ fmt)

module R = Eio.Buf_read

(* Turn a parsed RESP value into a reply value by interpreting it as a typed
   command and dispatching on it. *)
let reply_to (value : Value.t) : Value.t =
  let open Value in
  match Command.of_value value with
  | Ok Ping -> Simple_string "PONG"
  | Ok (Echo msg) -> Bulk_string (Some msg)
  | Error msg -> Simple_error ("ERR " ^ msg)

(* Parse RESP values straight from the connection's buffered reader and reply to
   each. Buf_read buffers bytes across socket reads, so a command split over
   several packets just makes [Parser.value] read again — no manual buffering. *)
let handle_client flow addr =
  traceln "client connected: %a" Eio.Net.Sockaddr.pp addr;
  let from_client = R.of_flow flow ~max_size:(1024 * 1024) in
  let rec loop () =
    if R.at_end_of_input from_client then traceln "client closed connection."
    else begin
      let reply = Encoder.encode (reply_to (Parser.value from_client)) in
      Eio.Flow.copy_string reply flow;
      loop ()
    end
  in
  try loop () with
  | End_of_file -> traceln "client closed connection."
  | Failure msg -> traceln "protocol error, closing: %s" msg
```
The `reply_to` function serves as a dispatcher. It converts a raw value into a command, and then pattern matches on the command. These dispatch to real work (it's a bit hard to tell from the primitive stuff here, but you'll see this when I get to implementing `GET` and `SET` in the next phase).

There's some code here that's burning my eyes. It's the `let reply = ...` line. This is deeply nested parenthetical code, and it's positively begging to be pipelined.

I just don't have the patience or the cognitive spare time to waste on mentally reversing all those calls into something intelligible. So I rewrote that line to:

```ocaml
let reply = from_client |> Parser.value |> reply_to |> Encoder.encode in 
```
What do you think, is the pipelined version more easily read by the humans who come after me (including future me)?

## Another Sidebar: The Clean Shutdown Bug
In the previous blog post on this, I spent a considerable amount of (very rewarding) time tracking down why the Codecrafters testing harness was failing and it ended up being a graceful shutdown race condition. I fixed this, but I noticed that even in the new passing tests, I was seeing errors like `failed to exit in 2 seconds after receiving sigterm` on every test.

Crap. More graceful shutdown problems.

Many rabbit holes were entered. One thing evidence showed was that the local sandbox used by Codecrafters was mangling signal delivery, which made liars out of the local tests. The `Eio.Condition` + `broadcast` wake-up code was not the culprit, despite it looking like an easy target.

The real cause showed up in `eio`'s own documentation (thank you doc writers <3). Executing `run_server ~stop` is _graceful_. It waits for in-flight connections to finish and the tester holds one open. The fix was, as so many are, difficult to find and easy to implement. I swapped `stop` for a `Fiber.first` cancellation so canceled fibers allow the process to exit.

## Wrap-Up
There were a number of good lessons learned in this stage. This is _exactly_ why I love these exercises. I'm not going to learn or _experience_ anything like this by building hello world demos or by letting my coding assistant do all the hard work for me (I did let it tell me how to interpret `/proc/net/tcp` though).

I learned a lot about OCaml project structure and `mli` files. I learned how my standard practice of separation of concerns and responsibility layers made for some clean OCaml code with the `parse -> interpret -> encode -> dispatch/reply` flow. 

I would also say another lesson here is to be willing to let go of your own clever idea when evidence supports it. I created a super fancy state machine that advanced over buffers and I was super proud of myself. When I saw that `eio` buffers did the exact same hard work, just better, I was able to dramatically simplify the code and use recursive descent parsing, which makes me sound _waaay_ smarter than I really am.

I'm also becoming a big fan of small commits. I've been making tiny commits in my history so that I can come back to various inflection or decision points and re-examine the before and after without having to compare two massive codebases instead. Those tiny commits also come in handy when I need to reflect on what I did and why to write a blog post like this.

To recap: I spent many, many hours converting an if statement into a typed protocol pipeline and I loved it. Up next in the Codecrafters challenge is `GET` and `SET`. Here, both of those commands have tons of optional flags and settings, so I'm genuinely worried if I'll be able to write the parsers for them properly. 

Tune in to find out!

---
[^1]: If I'm being pedantic here, I'm considering "accept state as input to the function" as effectively stateless. If it's an input parameter to a function, then it's not "state" in the State-with-a-capital-S way.
[^2]: Your mileage may vary. I also do love Rust and Unison pattern matches.