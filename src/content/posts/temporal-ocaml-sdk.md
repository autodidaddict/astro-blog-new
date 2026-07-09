---
title: Building a Temporal SDK in OCaml
published: 2026-07-09
draft: false
tags:
  - AI
  - Temporal
  - OCaml
description: Teaching myself some more OCaml by creating a clean API for Temporal
---

In my last blog post about Temporal, I wrote about externalizing the "brain" workflows
for the NPCs populating my game, _Agentic Realms_. Learning enough to use the Temporal SDK
wasn't quite deep enough for me, so I invented a project to get me all the way into the deep
dark dungeons below: _writing an OCaml Temporal SDK_.

## About Temporal
The extremely over-simplified description of how Temporal (and therefore its SDK) interacts with your
application is that you create a worker process that _you own_ and _you deploy_. This worker process
polls your Temporal task queue(s). When important things happen like workflow activations, signals,
and query jobs, the appropriate code does its thing. The workflow is _functionally pure_ while all of the side-effects are performed by your _activities_. 

The Temporal server manages the state, event log, and metadata for your workflows. It's easy to think that maybe the Temporal server also lives inside your application process but it is _always_ out of process, whether it's running locally or managed by Temporal themselves. The clean and pure separation of concerns between functionally pure and side-effecting code means that the Temporal server can replay your workflow upon activation all the way up to its "current" spot. This architecture may seem simple enough, but it is a great example of building powerful functionality out of small primitives. 

If you were to look at the code and docs for the Temporal server, you'll find a `gRPC` definition in some `.proto` files. When I first started investigating oh-so-long-ago (last week), I remember thinking, _"dear self, this is just a gRPC service"_. Yeah, and **Elden Ring** is "just a game". I love it when I'm wrong in ways like this, it makes the discovery journey so much more fun.

## Show us the Code, Nerd
As I write this, Temporal has [official support](https://docs.temporal.io/encyclopedia/temporal-sdks) for a bunch of languages, including the usual suspects like TypeScript, Ruby, Java, Go, Rust, .NET (_Captain Pedantic says, "I know, not a language"_). Kind of a bad look that there's no COBOL support or GW-BASIC. I looked for a language that didn't have an official or unofficial SDK and, to my happy surprise, there wasn't one for OCaml. I've been teaching myself OCaml lately by [building a Redis server](https://github.com/autodidaddict/ocaml-redis) using the CodeCrafters challenge. 

## Workflows, Activities, and Signals
If you're interested in Temporal then you should [check out the encyclopedia](https://docs.temporal.io/workflows) and then come back here and read on. I'm likely not going to do any of the subjects justice since my aim in this post is to show some cool OCaml code.

A **workflow** should occupy the same mental model as a pure function. It is deterministic. Every time you invoke `run` (or whatever your SDK calls it. From now on, just assume that's a commentary on every function name I use), your code must produce the same output for a given _state_.

When you want your workflow to do anything non-deterministic (if you want to sound smart at parties, you should call this **_stochastic_** or **_probabilistic_**), you'll use an _**activity**_. Making outbound requests to LLMs are obvious uses for activities, but some less obvious ones are simple things like generating a random number or reading from a "wall" clock. If it can cause your run function to be unpredictable, it must be an activity.

_**Signals**_ are how you poke and prod the workflows. A signal can advance a workflow, and the workflow's signal handler is allowed to mutate internal state because Temporal (and the SDKs) guarantee that signal handling is single-threaded (party prowess booster: call this **_non-reentrant_**).

If you want to get super nerdy about this, you can think of it as a distributed version of the _async/await_ pattern. And that fantastic segue is what gets me to my OCaml SDK.

## Writing a Temporal Worker in OCaml
You all know I love a [good event sourcing story](https://pragprog.com/titles/khpes/real-world-event-sourcing/), and Temporal is no exception. It's event-sourced internally and I use event-sourcing in the OCaml implementation of the SDK. Remember when I said I was pleasantly surprised that these SDKs weren't just simple gRPC clients? No? Well, keep reading anyway.

From the SDKs I looked at when trying to figure out what "idiomatic Temporal" looked like, the prevailing pattern is for developers to declare their activities as functions, declare their workflows as functions, and declare the signals to which the workflow responds as strongly typed structures. In Rust, this is done through macro expansion and the `#[run]` tag. Python uses something like `@workflow` and so on.

I'll start with the smallest primitive and then work my way to the top. I've built an eCommerce sample with a number of representative workflows to show how to use the SDK. The lowest level primitive is the _signal_, and I have 2 of them that I use for an approval workflow. They're defined as follows:

```ocaml
let approve = Signal.define ~name:"approve" Codec.unit
let reject = Signal.define ~name:"reject" Codec.string
```

I don't know about you, but I like how clean this looks. I readily admit that could be "this is the language I'm currently learning so it's pretty" bias.

Here I've got an `approve` signal that carries an empty (`unit`) payload. The `reject` signal carries a `string`, which is the reason for the rejection. I'll use the approval workflow to illustrate defining a workflow because it doesn't use activities, so we can add complexity progressively.

This is the approval workflow:

```ocaml
let approval_workflow =
  Workflow.define ~name:"ApprovalWorkflow" ~input:Codec.string ~output:Codec.string
  @@ fun ctx (subject : string) ->
  let decision = ref None in
  on_signal ctx approve (fun () -> decision := Some "approved");
  on_signal ctx reject (fun reason -> decision := Some ("rejected: " ^ reason));
  wait_condition ctx (fun () -> !decision <> None);
  Printf.sprintf "%s -> %s" subject (Option.get !decision)
```

The `approval_workflow` is defined to accept a string as input and it returns a string as an output. Remember that workflows are "just" pure functions, so they receive and return values as such. If you're not familiar with the syntax, the `@@ fun ctx (subject : string) ->` syntax uses a lower-precedence operator to avoid unnecessary indentation, nesting, and superfluous parentheses. This also shows that the workflow's input is the `subject`.

First, I use a ref to hold a mutable option. Then, I use `on_signal ctx approve ...` to declare the function to be invoked when the workflow receives the approve signal. Note that all I'm doing in that function body is mutating (via `:=`) the `decision` ref. It's all just clean functional code, the closures all work, and I don't have to do anything that doesn't look like "regular OCaml".

Next, `wait_condition` will, as the name implies, wait at that exact line in the workflow until the condition function returns true. Because I've done some of the hard work, and the Temporal server has done even more of the hard work, this gets to look like a vanilla boolean-returning lambda.

This code doesn't have any references to running in a distributed system, to event sourcing, or to the fact that both my worker process and the Temporal server could go down and come back up and would _properly_ and _without duplication_, pick up where it left off. This is the kind of abstraction I love: abstraction with _actual measurable benefit_ rather than abstraction for its own sake.

Okay, let's kick it up a notch and show a workflow that uses _activities_.

```ocaml
let return_workflow =
  Workflow.define ~name:"ReturnWorkflow" ~input:Return_request.codec
    ~output:Codec.string
  @@ fun ctx (r : Return_request.t) ->
  let rma = execute_activity ctx Activities.authorize_return r.return_order in
  let restocked = execute_activity ctx Activities.restock_inventory r.return_items in
  let refund =
    execute_activity ctx Activities.refund_payment (r.return_charge, r.return_amount)
  in
  Printf.sprintf "%s: restocked %d item(s), refunded %s" rma restocked refund
```

This workflow uses a few other features of the SDK. The first you see is the use of a `Codec` for the input. I put a lot of thought into how to support developers who want to benefit from OCaml's type system while also keeping their lives easy.

![Thinky think thunk](/images/think.gif)

There is a `@@deriving` feature you can use on your OCaml types that will automatically make them round-trip various wire formats, including JSON which is what this SDK does. The problem is that to do so, it requires the `ppx` language feature. This means the developer using this SDK would have to include all of the Jane Street stuff _just_ to get 0-code JSON round trips. I decided the trade-off wasn't worth it, so I kept `ppx` out and developers write their own Codecs like so:

```ocaml
module Return_request = struct
  type t = {
    return_order : string;
    return_charge : string;
    return_amount : int;
    return_items : Line_item.t list;
  }

  let codec : t Codec.t =
    Codec.json
      ~encode:(fun (r : t) ->
        `Assoc
          [ ("return_order", `String r.return_order);
            ("return_charge", `String r.return_charge);
            ("return_amount", `Int r.return_amount);
            ("return_items", `List (List.map Line_item.to_json r.return_items)) ])
      ~decode:(fun j ->
        let open Yojson.Safe.Util in
        { return_order = j |> member "return_order" |> to_string;
          return_charge = j |> member "return_charge" |> to_string;
          return_amount = j |> member "return_amount" |> to_int;
          return_items =
            j |> member "return_items" |> to_list |> List.map Line_item.of_json })
end
```
I'm not going to hide or sugar-coat this. This is the ugliest code you'll have to write as a consumer of this OCaml SDK (without using extras).

If you, as the consumer of the SDK, choose to use the `ppx`/Jane Street ecosystem, then you can definitely do so and then you won't have to manually define Codecs. The stance I took was that I didn't want my SDK to _force_ you to take those dependencies. That should be your choice.

An easy build is a happy build. A happy build is a happy dev. 

## Launch it
Now that we've defined our workflows and activities and signal handlers, we're ready to launch the application. As I mentioned in my last post on Temporal, you don't ship them your code and they don't run it. Your code runs wherever and however you want, and it connects to a Temporal server for its core functionality.

In OCaml, I need the equivalent of a `main` function (which is `let () = ...`) to run the worker. When I define the worker and start it, the SDK takes care of keeping the server up, handling interrupts, restarting, etc. As an application developer (and not SDK contributor), you have very little to fuss over.

Take a look at this main function. I love how easy it is to tell _exactly_ which components are being assembled and managed in the worker:

```ocaml
let env_or name default = try Sys.getenv name with Not_found -> default

let () =
  Eio_main.run @@ fun env ->
  let target = env_or "TEMPORAL_TARGET" "http://localhost:7233" in
  let task_queue = env_or "TEMPORAL_TASK_QUEUE" "ecommerce" in
  Eio.traceln "connecting to %s ..." target;
  let client = Client.connect env ~target in
  let worker =
    let open Worker in
    let open Activities in
    let open Workflows in
    create client ~task_queue
    |> register_activity validate_order
    |> register_activity charge_payment
    |> register_activity reserve_inventory
    |> register_activity request_shipment
    |> register_activity pick_and_pack
    |> register_activity dispatch_carrier
    |> register_activity confirm_delivery
    |> register_activity authorize_return
    |> register_activity restock_inventory
    |> register_activity refund_payment
    |> register_workflow order_workflow
    |> register_workflow shipment_workflow
    |> register_workflow return_workflow
    |> register_workflow countdown_workflow
    |> register_workflow approval_workflow
  in
  Worker.run worker
```
I'm leveraging OCaml's human-friendly syntax here (I do love me some `|>` pipelining). It reads pretty easily that this code is creating
a client that is pulling from the given `task_queue`, registering a bunch of activities, and registering a bunch of workflows. Each of these workflows have all the resilience and scaling guarantees that Temporal provides without the developer having to deal with ugly bits of distributed
systems and event sourcing code.

So there you go, [that's my SDK](https://github.com/autodidaddict/temporal-ocaml)! It doesn't yet support query jobs but that's just a function of time for me. I plan on adding it if I get some spare evenings this week. I hope you like it and got inspired to go tinker with OCaml or Temporal or both.

---
**_AI disclosure_**: _I used Claude for some of the SDK work. Specifically, Claude saved my life when I was trying to set up the magic incantation that lets OCaml use my C wrapper around the Rust C FFI from the official Rust SDK's Temporal core. Nobody has time to figure that out by hand._

_Claude does not write any of my prose. Only I can butcher the English language this way._