---
title: "Unison Abilities & OCaml Effect Handlers"
published: 2025-04-07
draft: false
description: "Exploring Unison Abilities and OCaml Effect Handlers"
tags:
  - Unison
  - OCaml
---
_Gettin' thunky with functional programming_. 

**_Disclaimer_**: _Consider the code in this post to be pseudocode. I'm using snippets I pulled from documentation and samples to do my comparison. I didn't run this code._

One of the things you hear all the time about functional programming is that it's _pure_. I really don't like this word but it's stuck and that's the term most people know. I'd rather think of it as "free of side-effects" or, as the math folks might say, _referentially transparent_. I'll just continue to use _pure_ for pragmatic reasons despite my objections.

Let's start with a simple example. In the world of FP, if you see:
```ocaml
add 2 2
// or
2 + 2
```
Then you are _guaranteed_ that absolutely nothing else happened while that function was executing. No I/O took place, no logging, no additional work at all. You're guaranteed that no matter how many times you call that function, you will always get the same result for the same parameters. This kind of "purity" is what makes FP so desirable and historically more easily tested than imperative programming.

It's easy to see the benefit of a function remaining pure when it's a simple calculation. You don't want random things to happen when you add 2 numbers. But what about a more complex scenario? Let's say you have a function that calculates the shipping cost for a particular order. The order can be an immutable parameter to the function, but now you want the function to use certain rates and values from an in-memory cache.

You could fetch whatever values you need from the cache prior to calling `calculate_shippping` and that's a perfectly valid scenario. But what if you don't know which values you need from the cache until you're in the middle of doing the calculations because those keys are dynamically derived? There's a million ways to do mocks and shims and proxies to make testable side effects, but I think a lot of them clutter up the code and remove some of the elegance and clarity of syntax that we want with FP.

In Java or C# or countless other languages, you might deal with this problem with interfaces, or traits from Rust. In functional languages like Haskell, I might solve this problem with a `monad`.

Unison and OCaml both have a really cool way of dealing with this. I've talked about Unison's [abilities](https://kevinhoffman.blog/post/exploring_unison_abilities/) before, and OCaml added support for something called _effect handlers_ in version 5.0 (2022).

Let's take a look at what it might look like to create an ability-consuming function in Unison:

```haskell
calculate_shipping : Order -> {ShippingCache} Float
calculate_shipping order =
  let Order { region, total } = order
  use ShippingCache
  let rate = get region
  rate * total
```
Here the call site is decorated with an annotation indicating that this function _requires_ an implementation of the `ShippingCache` ability in order to function. This means we are free to supply a test/mock implementation or a real one and the function should then behave deterministically in test, even though it has a side-effect.

You would then invoke `calculate_shipping` with code like this:

```haskell

shippingCost = ShippingCache.inMemory rates (calculate_shipping order)
```
Here the implementation of the `ShippingCache` ability is `ShippingCache.inMemory`, which could be used for dev or test, while we might construct a "real" version of the cache like `ShippingCache.cassandra config`.

Let's look at a Unison implementation of the shipping cache _ability_:
```haskell
ShippingCache.inMemory : Map String Float -> '{ShippingCache} a -> a
ShippingCache.inMemory rates computation =
  handle computation with
    case get region -> k ->
      let rate = Map.get region rates |> Option.withDefault 0.001
      k rate
```
This is a function that takes a map from `String` to `Float` as its inital parameter, and then returns an _ability handler_. It naively uses a rate of .1% if none can be found. Here `k` is a continuation or a _thunk_. Think of an effect handler this way: to use an effect, the calling function suspends and calls the effect with parameters as well as a thunk, which is basically a pointer to the next line of code in the caller. The effect then does its work, and then _calls the thunk_ with the original parameters and the new value (in this case, a shipping rate).

The code is designed to make it look like you're synchronously calling the `get` function but you're really suspending the caller, passing the suspension to the handler, and then the handler is calling the next line from the caller. It's like inserting a detour in the original code flow and not a request/response the way the code appears.

I fell in love with this notion when I originally saw it in Unison. The syntax in Unison is based on [this paper](https://arxiv.org/pdf/1611.09259) from 2017, referred to as the _"Frank language"_. OCaml added the concept of _effect handlers_ to the language in 2022 with version 5.0.

Let's see how we might write the `calculate_shipping` function in OCaml:

```ocaml
let calculate_shipping (order : Order.t) : float =
  let region = Order.region order in
  let rate = Order.rate in
  perform (GetShippingRate region) * rate
```
Something worth pointing out here is that the `calculate_shipping` function doesn't advertise the required effect handlers visibly at its call site like the Unison version does. While you'll get a compilation failure attempting to use this function without the appropriate handler, it's a bit less self-documenting.

Recall that the Unison version of invoking the calculate shipping function with the in-memory cache looks like this:

```haskell
shippingCost = ShippingCache.inMemory rates (calculate_shipping order)
```
Where the OCaml equivalent feels a bit more monadic in that it doesn't attempt to hide the "function wrapping" nature of using the handler:

```ocaml
let result = run_with_in_memory_cache cache (fun () -> calculate_shipping order) in
```
Here we're using a helper function called `run_with_in_memory_cache` where the `run_` prefix is common in both monadic and OCaml syntax. You'll often see library functions in Haskell like `run_with_nats` or `run_with_sql` that take an instance of an effect handler and then an anonymous function to run, e.g. `run_with_postgres (new_postgres config) ...`

This OCaml runner function is a wrapper that uses `match f...` to run the function until it hits an effect request:

```ocaml
let run_with_in_memory_cache (cache : float StringMap.t) (f : unit -> 'a) : 'a =
  match f () with
  | result -> result
  | effect (GetShippingRate region) k ->
      let rate =
        match StringMap.find_opt region cache with
        | Some r -> r
        | None -> 0.0
      in continue k rate
```
If the execution of the lambda here produces a result, then this function returns the result. If it results in an `effect`, then we provide a handler for it. Note the `continue k` at the bottom of the effect match arm, which looks a lot like many ability handlers in Unison, which uses the `handle k with...` syntax.

These new effect handlers form the foundation of the more modern OCaml asynchronous library, `eio`, which is an effectful I/O library.

Both OCaml and Unison require you to do some nesting if you want to provide multiple effect handlers to a single function execution, though I think Unison's syntax is a bit cleaner. I'm no Unison expert and I know next to nothing about OCaml, but I find Unison's ability syntax a bit more concise. I'm betting that one could easily create some abstractions or types in OCaml that would get us close to the clarity of the Unison syntax, I just don't know what that would look like.

In both cases, the concept of effect handlers is a beautiful, powerful, and usually underrated power you can use with your pure functional code base. So if you've got a spare minute, go play with effect handlers in whatever language you like, because chances are a number of your favorite higher-level libraries are implemented with them under the hood.


