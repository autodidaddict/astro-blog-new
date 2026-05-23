---
title: You Got Your NATS in my Akka!
published: 2026-05-23
draft: false
tags:
  - NATS
  - Akka
description: Exposing an Akka Services as a NATS Microservice
---

Some of you may be old enough to remember the Reese's Peanut Butter cup commercials where one guy complains about chocolate in his peanut butter, and the other guy complains about peanut butter in his chocolate. The moral, as much as one can be found in a commercial, is that they're both right and things are better together.

This (yeah, I know it's a stretch) applies to both Akka and NATS.

The Akka SDK has support for a number of different endpoint types. You can expose your services over HTTP or gRPC, and you can even stream data bi-directionally. These endpoints can use an Akka `ComponentClient` to talk to other components like entities, views, workflows, and most recently, agents.

On the other hand, NATS has a standard protocol for discovering and communicating with microservices. Any service you expose over NATS this way can be discovered via `nats micro ls` and you can make requests of that service by sending messages on the service's subject.

This is where I thought it would be great to combine the two. What if I could create an Akka service that has entities and views and agents, but advertise and expose it as a NATS microservice.

One way to implement this would have been to fork the Akka SDK, but it actually turned out to be much easier than that. I created an annotation, `@NatsMicroService` that can be applied to any standard class in an Akka service project. The presence of this annotation automatically exposes a NATS microservice according to the metadata in the annotation. Every function you want to expose as part of the NATS service simply needs the `@NatsSubject` annotation. 

Take a look at implementing a simple echo service as a class in an Akka service. The subjects you use not only support individual tokens within the subject, but also broader wildcards. You can also get access to the token and other request metadata from inside the microservice function.

```java
  @NatsMicroService(
    name = "echo-service",
    version = "1.0.0",
    description = "Echo NATS micro-service sample")
public class EchoNatsService {
  @NatsSubject(value = "echo.upper", 
               description = "Uppercases the request payload")
  public byte[] upper(byte[] payload) {
    return new String(payload, StandardCharsets.UTF_8)
        .toUpperCase(Locale.ROOT)
        .getBytes(StandardCharsets.UTF_8);
  }

  @NatsSubject(value = "echo.repeat.{count}", 
               description = "Repeats the payload {count} times")
  public byte[] repeat(NatsRequest request) {
    int count = Integer.parseInt(request.token("count"));
    return new String(request.payload(), StandardCharsets.UTF_8)
        .repeat(count)
        .getBytes(StandardCharsets.UTF_8);
  }

  @NatsSubject(value = "echo.subject.>", 
               description = "Echoes the concrete request subject")
  public byte[] whichSubject(NatsRequest request) {
    return request.concreteSubject().getBytes(StandardCharsets.UTF_8);
  }

  @NatsSubject(value = "echo.greet.{name}", 
               description = "Greets the {name} wildcard token")
  public byte[] greet(NatsRequest request) {
    String name = request.token("name");
    return ("Hello, " + name + "!").getBytes(StandardCharsets.UTF_8);
  }

  /**
   * Always throws an ordinary exception, 
   * demonstrating that an unhandled failure becomes a
   * NATS-native error with the generic code {@code 500}.
   */
  @NatsSubject(value = "echo.fail", 
               description = "Always fails with an unhandled exception")
  public byte[] fail(byte[] payload) {
    throw new IllegalStateException("handler failed on purpose");
  }

  /**
   * Always rejects the request, demonstrating an explicit
   * rejection with a developer-chosen error code.
   */
  @NatsSubject(value = "echo.reject", 
               description = "Always rejects with error code 400")
  public byte[] reject(byte[] payload) {
    throw new NatsHandlerException(400, "request rejected by handler");
  }
}
```

When you run this Akka service, you'll see each of these operations in the `nats micro info` results. One thing to notice here is that this class is just a regular class, it's not an actual Akka component. Since every Akka service needs at least one Akka component, you'll need to add a dummy one to your project to make it compile.

With this in place, I can use standard NATS microservice functions to query my Akka views, send commands to my Akka entities, trigger workflows, and even invoke agents. 

Take a look at this code that makes use of the auto-injected `ComponentClient`:

```java
@NatsMicroService(
    name = "counter-service",
    version = "1.0.0",
    description = "Demonstrates calling an Akka component from a NATS handler")
public class CounterNatsService {

  private final ComponentClient componentClient;

  public CounterNatsService(ComponentClient componentClient) {
    this.componentClient = componentClient;
  }

  @NatsSubject(
      value = "counter.increment.{id}",
      description = "Increments counter {id} via the Akka ComponentClient")
  public byte[] increment(NatsRequest request) {
    String id = request.token("id");
    int value =
        componentClient
          .forKeyValueEntity(id)
          .method(CounterEntity::increment)
          .invoke();
          
    return ("counter " + id + " = " + value).getBytes(StandardCharsets.UTF_8);
  }
}
```

There's NATS in my Akka and I love it!

p.s. If you want the source code for the `@NatsMicroService` annotation and its corresponding `@NatsSubject` annotation, take a look at my sample repository here: [akka-nats-endpoints](https://github.com/autodidaddict/akka-nats-endpoints/tree/main).
