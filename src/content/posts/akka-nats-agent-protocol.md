---
title: You Got Your Akka in My NATS - Synadia Agent Protocol
published: 2026-05-24
draft: false
tags:
  - Synadia
  - NATS
  - Agentic
  - AI  
description: I build on the NATS Microservice annotation to expose Synadia Agents
---

In my [previous post](/posts/nats-in-my-akka/), I talked about creating a `@NatsMicroservice` annotation that allowed me to expose an Akka service as a NATS microservice. Once I saw how well that worked, I got a crazy idea.

If you've been following the new agentic AI trends, then you know that one big area of work is in figuring out how agents can communicate with each other. We also need a good way of talking to agents from other code, but in a standardized way. There's also protocols for exposing tool selection functionality to LLMs.

This has given us the A2A (agent-2-agent) communication protocol, the ACP (agent communication protocol), MCP (model context protocol). Now we have the **[Synadia Agents Protocol](https://github.com/synadia-ai/synadia-agent-sdk-docs)**, which describes a way to communicate with agents over a NATS connection.

This can be super handy if you're already an enterprise that's using NATS for intra- and inter- project communication. 

The SAP (not sure if Synadia really wants me to call their protocol "SAP"?) actually sits on top of the NATS Microservice protocol. All agents are to advertise themselves as part of the `agents` service. 

By leveraging this simple mechanism, communicating with agents over NATS automatically inherits the rich NATS security systems, topic filtration and restriction systems, and service and topic discovery.

Akka agents don't automatically bring their own endpoints. It's up to the developer to choose what kind of endpoint (MCP, gRPC, HTTP, etc) they want to use to expose the agent and what security model they want around it.

I've added another small Java annotation that makes it brain-dead simple to automatically expose simple functionality for an agent.

In the following code, I've created a tiny little Synadia Agents Protocol wrapper around my agent (I used a bunch of carriage returns here to bump my Lines of Code count):

```java
@SynadiaAgent(agent = "echo", 
              owner = "acme", 
              name = "echo-1", 
              version = "1.0.0")
public class EchoSynadiaAgent {

  /** Echoes the caller's prompt back, prefixed with {@code echo:}. */
  @PromptHandler
  public String handle(PromptRequest request) {
    return "echo: " + request.prompt();
  }
}
```

That's it! You don't need to do anything else. 

Both the Microservice and the Synadia Agent annotations support having the Akka `ComponentClient` injected at construction time so you can interact with all of your existing Akka SDK components. You can use this component client to talk to an Akka agent, an autonomous agent, entities, views, or workflows.

Once you've exposed your Akka code to NATS, the possibilities are endless. I keep imagining the edge scenarios where I can get messages from hardware devices all the way out in the field through NATS and into my application.

Don't forget that NATS has installations _in space_. That's right, _Martians can invoke my Akka services!!_. (Martians are using NATS, right? Right?)

You can find and use both of my Java annotations in the [public repo](https://github.com/autodidaddict/akka-nats-endpoints). 

If you've been thinking, _"I've got this great Akka service, but I want to hook it into my other apps via NATS"_, then these blog posts have been for you.

Enjoy!