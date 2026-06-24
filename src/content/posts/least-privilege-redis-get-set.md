---
title: An OCaml Redis and the Principle of Least Privilege
published: 2026-06-24
draft: false
tags:
  - OCaml
  - Codecrafters
  - Redis
  - eio
  - Capabilities
description: I implemented a least-privilege data store and used the robust type system to make bad data unrepresentable.
---

In my [previous blog post](../from-ping-to-proper-ocaml-redis/), I talked about taking a step back to refactor
and upgrade my code to remove short-term hacks before proceeding. In this post, I want to go through my journey adding support for both `SET` and `GET` to my OCaml Redis clone.

## Let the Type Carry the Rules
As I cracked my knuckles, mentally prepping to implement [the GET/SET Codecrafters stage](https://app.codecrafters.io/courses/redis/stages/la7) (you need to log in to see the stage contents), I was already excited.

I've modeled enough problem domains and data in Haskell and other functional languages like Scala to know that the power is in the type system. If I go through the rigor of modeling things properly, the type system can prevent entire classes of data corruption because corrupt states _physically can't exist_.

So that was my plan here. I need to add the `GET` and `SET` variants to my `Command.t`.

The first thing I did was add a new `Set` variant to `Command`. The Redis `SET` command has a number of optional flags as well as parameters to those optional flags. I started to freak out about how I was
going to do the difficult read-ahead parsing on the wire to support `SET`, but I put that worry behind me. I added `set_options` as a record type and used union types for `existence` and `expiry` that force
the compiler to enforce mutually exclusive variants.

Here's my `command.mli` signature:

```ocaml
(* SET key value [NX | XX] [GET] [EX s | PX ms | EXAT s | PXAT ms | KEEPTTL] *)
type existence =
  | Always  (** no NX/XX supplied *)
  | If_not_exists  (** NX *)
  | If_exists  (** XX *)

(** SET's expiry option. One field, so EX/PX/EXAT/PXAT/KEEPTTL are mutually
    exclusive by construction. EX/PX are relative durations; EXAT/PXAT are
    absolute Unix timestamps — keep them distinct here and resolve to an
    absolute deadline at apply time. *)
type expiry =
  | Expire_seconds of int  (** EX, seconds, relative *)
  | Expire_millis of int  (** PX, milliseconds, relative *)
  | Expire_at_seconds of int  (** EXAT, Unix time, seconds *)
  | Expire_at_millis of int  (** PXAT, Unix time, milliseconds *)
  | Keep_ttl  (** KEEPTTL - retain existing ttl*)

type set_options = {
  key : string;
  value : string;
  existence : existence;
  get : bool;
  expiry : expiry option;
}

(** A strongly typed client command, lifted from a raw {!Value.t}.

    Clients send commands as RESP arrays of bulk strings, e.g.
    [*1\r\n$4\r\nPING\r\n]. {!of_value} turns that wire shape into a typed
    command so dispatch can match on [Ping] / [Echo] rather than on array
    structure. *)
type t = Ping | Echo of string | Get of string | Set of set_options
```
There's also an important design decision here. I _could have_ set variants in the `expiry` type that match the wire protocol, so `Ex` and `Px` and `Exat`, etc. But there's no need to do that. Letters don't cost anything in my code. If this is a type I'm going to have to reason about later, then I want to do myself and everyone else a favor by naming things _clearly_. 

The `Expire_seconds` variant gives everyone all the context they need while `Ex` tells the reader nothing.

### Reducing the Code and Consolidating Magic Strings
I braced myself for figuring out how to deal with options and read-ahead in the wire parser when I realized _I didn't need to do any of that hard work_. I'm already parsing all the data on the wire into a `Value.t`. That gives me an AST I can use _destructuring pattern matches_ on. I don't need read-ahead when my `Value` is already fully populated with all of the tokens received.

I wrote a function to lift the right `expiry` variant from a `Value`, and all four of the keywords were quoted strings in place:

```ocaml
(* Build the expiry variant for an already-recognised keyword. [kw] is one of
   the four matched in [parse_set], so the final arm is unreachable. *)
let expiry_of (kw : string) (n : int) : expiry =
  match kw with
  | "EX" -> Expire_seconds n
  | "PX" -> Expire_millis n
  | "EXAT" -> Expire_at_seconds n
  | "PXAT" -> Expire_at_millis n
  | _ -> assert false
```

And then there was another set of match arms with the exact same string literals in `parse_set`'s recursive `go` function (lines 10 through 14 are particularly gross):

```ocaml
| Bulk_string (Some tok) :: rest -> (
    (* Normalise the keyword once, then dispatch on (keyword, remaining tokens)
       so "EX with an argument" and "EX with nothing after it" fall out as two
       distinct arms. *)
    match (String.uppercase_ascii tok, rest) with
    | "NX", rest -> with_existence acc If_not_exists rest
    | "XX", rest -> with_existence acc If_exists rest
    | "GET", rest -> go { acc with get = true } rest
    | "KEEPTTL", rest -> with_expiry acc Keep_ttl rest
    | ("EX" | "PX" | "EXAT" | "PXAT") as kw, Bulk_string (Some n) :: rest -> (
        match positive_int n with
        | Some n -> with_expiry acc (expiry_of kw n) rest
        | None -> Error (Printf.sprintf "SET: invalid expire time in '%s'" kw))
    | ("EX" | "PX" | "EXAT" | "PXAT") as kw, _ ->
        Error (Printf.sprintf "SET: %s requires an integer argument" kw)
    | other, _ -> Error (Printf.sprintf "SET: unsupported option '%s'" other))
```
When I looked at this code, two things bothered me. First, there was the `assert false`. This was basically a hack to get rid of the exhaustiveness check failure on the match in `expiry_of`. Secondly, the same string literals were being used in multiple places. My procedural go-to solution here would be to create some constants for each of the key words.

_BUT_, I wasn't going to settle for just a constant. I knew there had to be a cleaner and more idiomatic OCaml-y way to do this. The OCaml-y way to do this was to combine recognition and construction into a single function that returns a partially-applied function.

```ocaml
(* Recognise an expiry keyword, returning the constructor that still needs its
   numeric argument (or None for any non-expiry token). Folding recognition and
   construction together keeps the keyword set in exactly one place — and leaves
   no unreachable [assert false]. *)
let expiry_kw : string -> (int -> expiry) option = function
  | "EX" -> Some (fun n -> Expire_seconds n)
  | "PX" -> Some (fun n -> Expire_millis n)
  | "EXAT" -> Some (fun n -> Expire_at_seconds n)
  | "PXAT" -> Some (fun n -> Expire_at_millis n)
  | _ -> None
```

And now in the set parse, I _create a partially applied function_ (`expiry_kw kw`) and, depending on the results, I invoke it (by calling `(make n)`):

```ocaml
... other match stuff ...

| kw -> (
    (* Anything else must be EX/PX/EXAT/PXAT taking the next token as a
        positive integer; expiry_kw returns None for unknown keywords. *)
    match (expiry_kw kw, rest) with
    | Some make, Bulk_string (Some n) :: rest -> (
        match positive_int n with
        | Some n -> with_expiry acc (make n) rest
        | None -> Error (Printf.sprintf "SET: invalid expire time in '%s'" kw))
    | Some _, _ ->
        Error (Printf.sprintf "SET: %s requires an integer argument" kw)
    | None, _ ->
        Error (Printf.sprintf "SET: unsupported option '%s'" kw)))
```
This reduction was more than just a refactor toward cleanliness. It accomplished a few important things:

* The `assert false` is gone. The matches that do remain are provably exhaustive and I don't have to use an "I know what I'm doing" hack.
* The set of expiration keywords is only stated once. 
* The dispatcher match is one dimension smaller. Instead of matching on the tuple `(keyword, rest)` with duplicated keywords it matches just a single string.

## The Store and the Capability Model
Now that I can produce `Set` variants of `Command`, I need to actually implement the get and set functionality. Again I get worried and have a bit of anxiety. _How am I going to store data in memory and avoid concurrency problems and not use globals_? In Haskell I would probably have wrapped the serve calls with a monad. In Unison I would use the built-in `Store` ability and just instantiate my own store before running the server.

I thought about using OCaml 5 effects here but it turns out that not everything is a nail while holding that hammer. I actually lose quite a bit of power and functionality if I treat the store like an effect here, including some type safety and future-proofing. 

I went with simply using a `Hashtbl` that I hid behind an abstract type called `Store`. I learned about how to make abstract types in my previous post on this topic and their power is slowly starting to click.

But surely I have to worry about concurrency and conflicting updates! Nope. Embracing the `eio` concurrency model, there's a single domain with co-operative scheduling. This means that a non-yielding read-update-write operation is atomic _without a lock_. This is actually how Redis's own single-threaded execution works.

Next it came time to figure out how to inject the store into the functions that need to interact with it. This is where the capability model comes in. A capability is _an unforgeable reference that combines designation and authority_. This is what smart people call "no ambient authority" and removes a category of bugs/exploits called the [confused deputy](https://en.wikipedia.org/wiki/Confused_deputy_problem).

This definitely makes me think about Unison _abilities_. The idea is that a function can't do something that it hasn't been delegated the authority to do. In Haskell, you can give the `IO` ability as a single, giant monolith. In Unison, you can explicitly control what a function can do _and_ that control appears at the type level in function signatures. OCaml is (as far as I can tell) somewhere in the middle.

The discipline of delegated authority shows up as function parameters, but the honor system means there's nothing stopping that function from "cheating" and fabricating an instance of some authority on the fly. This is where the abstract type becomes critical: _If a function takes a reference to an abstract type designed as a capability, the function cannot create a new reference, it can only take that authority via **delegation**_.

In other words, if I want my function to be able to use a store capability, I will pass an _opaque_ reference to the `Store` type, and the Store type can only be received as a reference.

This kind of thing is a lot to take in. If you're as fascinated by it as I am, check out the references section at the end of this blog post.

### I'm not so Clever

Freshly armed with shiny new knowledge and techniques, I had an absolutely _brilliant_ idea. The store made perfect sense, but what if _the clock was a capability_?! So clever it's mindblowing, right? I'll pass an opaque clock type into the command dispatch and handling routines. I'll know right at the call site that these functions need a clock. As a bonus, my tests can use a mock clock capability that I can use to set fixed and predictable times.

I'll be famous! I will make millions! I can finally be an influencer! This idea is going to turn me into a ... wait. _Damn_.

There is a downside here. Everything that needs a clock now has to know about my clock type. This means the clock type needs to be exposed like it was a library and used explicitly in other modules. Propagating a
dependency is grunt work, but it's not unusual. The thing that should've stood out to me immediately is that _my store doesn't actually need a clock_. 

To figure out if something has expired, the only thing I need is the number representing "now". If "now" is greater than or equal to the key's expiration time, then I remove the item. So I can refactor all of my clever code, remove a bunch of dependencies, and just pass an integer value for `now`. This works with my tests and simplifies them as well.

Simpler is better.

## Proving it Works

To prove this all works, I got to use the `redis-cli` more than ever before. My server finally supports `GET` and `SET`, so I ran a script of `redis-cli` commands with various options of `SET` and used `GET` to verify expected values.

Once this all worked cleanly, I was then able to run the Codecrafters test. Since that passed, I submitted the work and Codecrafters automatically passed and moved on from the expiration stage because my code started out by passing the expiration tests! Two birds with one stone!

## Wrapping Up

This is yet another lengthy blog post for something that should be fairly minor. There are a couple
of key points worth making. First, this wasn't just about getting `GET` and `SET` to pass. I had to decide 
how I was going to store the data for the KevRedis database and the threading and concurrency model for that. I learned more about capabilities and idiomatic OCaml. I learned how to clean and reduce functions and pattern matches and that there's a ton of future work that will benefit from being able to lift `Value`s into `Command`s rather than combining parsing and lexing and dispatch.

Next up in the Codecrafters challenge is persisting the database in the exact format Redis uses. That's going to be a metric crapton of fun because one of the tests should be starting a real Redis database with my KevRedis data file!

---

## References
Here are a list of links you can follow if you want to go deep down the rabbit hole on the concepts of protection, ambient authority, capability models, and so on.

- Morris, James H., Jr. "Protection in Programming Languages." *Communications of the ACM* 16, no. 1 (1973): 15–21. https://doi.org/10.1145/361932.361937
- Noble, James, Andrew P. Black, Kim B. Bruce, Michael Homer, and Mark S. Miller. "Abstract and Concrete Data Types vs Object Capabilities." In *Principled Software Development*, Springer, 2018. https://link.springer.com/chapter/10.1007/978-3-319-98047-8_14
- Stiegler, Marc, and Mark Miller. "How Emily Tamed the Caml." HP Laboratories, tech report HPL-2006-116, 2006. https://www.hpl.hp.com/techreports/2006/HPL-2006-116.html
- Miller, Mark S., Ka-Ping Yee, and Jonathan Shapiro. "Capability Myths Demolished." Technical Report SRL2003-02, Johns Hopkins University, 2003. https://papers.agoric.com/papers/capability-myths-demolished/full-text/
- Miller, Mark S. "Robust Composition: Towards a Unified Approach to Access Control and Concurrency Control." PhD thesis, Johns Hopkins University, 2006. http://www.erights.org/talks/thesis/
- Eio contributors. "Eio Design Rationale." ocaml-multicore/eio, `doc/rationale.md`. https://github.com/ocaml-multicore/eio/blob/main/doc/rationale.md
- Dennis, Jack B., and Earl C. Van Horn. "Programming Semantics for Multiprogrammed Computations." *Communications of the ACM* 9, no. 3 (1966): 143–155.
- Hardy, Norm. "The Confused Deputy (or why capabilities might have been invented)." *ACM SIGOPS Operating Systems Review* 22, no. 4 (1988): 36–38.
