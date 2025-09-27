---
title: Building Friendly Services from the Future with Unison
published: 2025-09-27
draft: false
tags:
  - Unison
  - Services
  - Gaming
description: I build some services, APIs, and data code in Unison
---

The **Unison** programming language brands itself as a _friendly language from the future_. While the sentiment here is true, I think Unison is also from the past in that many of the things that Unison does properly are concepts that have long been discussed in computer science but rarely ever implemented. Unison is that language we all _should_ be able to use but not many of us can "for real" (yet).

I've been building the backend for a game (_shocker_, I know) using Unison. I can run this backend locally, but I can also deploy it to _Unison cloud_ by running a simple function in my `ucm` prompt.

My first instinct is to start describing this service at the lowest level of data access, but instead, let's start at the public-facing API.

In this game API, client code can create new objects and move them around the game world. The two API functions I want to show you are `createObject` and `getObjectsAtLocation`. This should show enough of the plumbing that you'll get an idea of what it feels like to build a (micro)service in Unison.

To hold all of the public-facing data types and routes, I put them in the `api` namespace. Here is the `api.createObject` function:

```haskell
api.createObject : AppStorage -> '{Route, Exception, Storage, Remote} ()
api.createObject storage = do
  use Debug trace
  noCapture POST (s "objects")
  createRequest = decodeJson CreateObjectRequest.fromJson
  (CreateObjectRequest rId mId x y name behaviorPath) = createRequest
  realmId = RealmId rId
  mapId = MapId mId
  loc = FullLocation (Coordinate x y) realmId mapId
  behaviorId = BehaviorId realmId mapId behaviorPath
  res = db.createObject storage behaviorId loc name
  ok.text "ok"
```

This code will look pretty alien if you're not used to Unison (or, to a lesser extent, Haskell). `noCapture POST (s "objects")` defines a route where we don't need to extract information from the URL route. Next, I create an instance of `api.CreateObjectRequest` by decoding it from the request body via the `fromJson` function.

Let's take a look at `api.CreateObjectRequest`:

```haskell
type api.CreateObjectRequest
  = { realmId : Text, mapId : Text, x : Nat, y : Nat, name : Text, behaviorPath : Text }
```

The game details don't much matter here. What's important is that this is a Unison structural type that defines the attributes of the new object to be created, including its location and a string describing the behavior (I'll talk more about that in a different game-related post).

Unlike Rust I can't just drop a `serde` attribute on the type and magically get a JSON decoder. It's not much more code in Unison, however:

```haskell
api.CreateObjectRequest.fromJson : '{Decoder} CreateObjectRequest
api.CreateObjectRequest.fromJson = do   
  use Decoder nat text
  use object at!
  r = at! "realmId" text
  m = at! "mapId" text
  x = at! "x" nat
  y = at! "y" nat
  name = at! "name" text
  p = at! "behaviorPath" text
  CreateObjectRequest r m x y name p
```

Here we're using `at!` to grab fields out of the JSON body and then the function returns a `CreateObjectRequest` instance built from the JSON-extracted values.

If you look back at the definition for `api.createObject` you'll see that it boils down to preparing for and invoking a single function: `db.createObject`. It might seem like a bit more ceremony than I need, but I firmly believe in keeping the `api` types separate from the `db` types, even if they look identical at the start of the project. Trust me, it pays huge dividends as the complexity grows.

Now let's look at `db.createObject`, which creates a new object in `Storage`:

```haskell
db.createObject :
  AppStorage
  -> BehaviorId
  -> FullLocation
  -> Text
  ->{Exception, Storage, Remote} ObjectSummaryRow
db.createObject storage behaviorId location name =
  use OrderedTable.write tx
  timeStamp = instantToOffsetDateTime()
  locationObjectsTable = AppStorage.locationObjectsTable storage
  objectsTable = AppStorage.objectsTable storage
  objectLocationsTable = AppStorage.objectLocationsTable storage
  transact (AppStorage.database storage) do
    instanceId = InstanceId (UUID.toText v4.new())
    row = ObjectSummaryRow behaviorId instanceId name
    detail = ObjectDetailRow behaviorId instanceId name 0
    tx locationObjectsTable (location, instanceId) row
    tx objectsTable instanceId detail
    tx objectLocationsTable instanceId location
    row
```

There are really just 3 lines here of code that makes changes to storage:

```haskell
tx locationObjectsTable (location, instanceId) row
tx objectsTable instanceId detail
tx objectLocationsTable instanceId location
```

Here, `tx` is `OrderedTable.write`. An [OrderedTable](https://share.unison-lang.org/@unison/website/code/main/latest/types/@omchgqo2nr441sv8olbiqpl998cjk3f09920r4gpahgn88rcafc6vr4n0v0l7sfiir1dc4c91nrrfgu9h8ac63i56i75c8kdv7malu8) can be (over-simplification warning) thought of as key-value stores or columnar stores. These are very different than data stores with classical rows, columns, tables, and SQL queries.

Important here is that when creating a single object, there are actually 3 writes to 3 different ordered tables:

* A write to the objects table, which maps instance IDs to object _details_
* A write to the object locations table, which maps instance IDs to their locations within the world
* A write to the location objects table, an inverse of the previous, which maps locations to a list of objects _at that location_

The latter two tables are sort of like materialized views. We can't assume that we have the ability to scan and filter keys, so instead we create views that support queries we know the game is going to need: _where is this object?_ and _what objects are at this location?_.

Now let's take a look at the query functions to get object details and get a list of objects at a location:

```haskell
db.getObjectDetail : AppStorage -> InstanceId ->{Exception, Storage, Remote} ObjectDetailRow
db.getObjectDetail storage instanceId =
  objectsTable = AppStorage.objectsTable storage
  OrderedTable.read objectsTable instanceId

db.getObjectsAtLocation :
  AppStorage -> FullLocation ->{Exception, Storage, Remote} [ObjectSummaryRow]
db.getObjectsAtLocation storage location =
  locationObjectsTable = AppStorage.locationObjectsTable storage
  resultStream = rangeClosed.prefix locationObjectsTable prefixOrdering location location
  objects = Stream.map at2 resultStream
  Stream.toList objects
```

In the first function, we just use `OrderedTable.read` to pull up the value by key. In the second, we use the `rangeClosed.prefix` function and supply it with the same parameter twice for the start and end of the range, `location`. This `rangePrefix` function is super powerful in that if my ordered table is using a tuple as a key, I can query for all the items in my table that have a key tuple where the first element is my target. It's not quite like having a full `KEYS` function like you do in Redis, but it's still pretty powerful.

Every single piece of data I/O in this microservice is faciliated by the `Storage` ability. This lets my functions stay mostly pure and use a function that provides the `Storage` ability to deal with the implementation.

One thing I love about abilities over monads is that abilities automatically come with an interface (or typeclass) style declaration. I don't need to go out of my way to invent something that shows all of the functions available like I have to do with monads.

Let's take a look at the functions that are available to any function I write that uses the `Storage` ability (you can do this yourself in `ucm` by typing `view Storage`)

```haskell
ability Storage where
  tryTransact :
    Database -> '{Transaction, Exception, Random, Batch} a ->{Storage} Either Failure a
  tryBatchRead : Database -> '{Exception, Batch} a ->{Storage} Either Failure a
```

This isn't quite as self-explanatory as some other abilities. This ability is actually _composed_ of other abilities like `Transaction` and `Batch` and `Random`. It has two functions: `tryTransact` and `tryBatchRead`. Most of the functions we use actually come from the `OrderedTable` namespace, which in turn are wrappers around `Storage` and consumers of the `Database` type.

The `Transaction` ability looks like this (you should recognize the `write` and `read` functions from earlier):

```haskell
ability Storage.Transaction where
  write.tx : Table k v -> k -> v ->{Transaction} ()
  tryRead.tx : Table k v -> k ->{Transaction} Optional v
  delete.tx : Table k v -> k ->{Transaction} ()
```

At no point in this code do we see _how_ the ordered table is implemented. There's no connection string, no choice of ODBC provider, not even a whiff of Redis or Cassandra or Etcd or ... you get the idea. What's even more powerful about this is that when I'm running my service locally, `Storage` is provided by something appropriate for a local, ephemeral, testing environment. But when I'm running fully deployed in [Unison Cloud](https://www.unison.cloud/), then I know that my service is backed by multiple nodes worth of distributed data storage with a relatively high SLA (depending on what I'm paying for, etc).

Another thing we don't see is a choice of implementation for an HTTP server. Instead, we're writing functions that use the `Route` ability. This again frees us from annoying implementation details and lets us declare our application functionality. This also means testing is _unbelivably_ easy, because any function that requires an ability can take _any_ provider of that ability, such as a tester/mock.

This is why Unison is a friendly language from the future. It's fun to use, easy to learn, and has such a low cognitive overhead that I often feel happy or "zen" just because I'm using Unison. So far, I've found that Unison has the lowest _impedance mismatch_ between what I want and what I write for code.

I _strongly_ recommend that you go through the Unison cloud tutorial and build your own microblogging service. You won't really appreciate how good this experience is until you've done something that you've done before in other languages.

