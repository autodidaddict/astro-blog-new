---
title: Fast, Cheap Agent Decisions
published: 2026-09-23
draft: false
tags:
  - AI
  - Agentic
  - Akka
  - LLM
  - Jev
  - SystemOne
description: Using TypeSafe’s Jev with Akka agents and AI gateway
canonical: https://akka.io/blog/fast-cheap-agent-decisions
---

_This post was originally published on the [Akka blog](https://akka.io/blog/fast-cheap-agent-decisions)._

Jev has only been available to the public for about a week (unveiled September 15th, 2026), but it’s already created a ton of excitement and discussion within the AI community. [TypeSafe](https://typesafe.ai) made Jev available through limited, early access at the same time they disclosed their \$40 million seed round.

Let’s strip away the hype and talk about what Jev is, and what it isn’t. When our agentic code submits a prompt to an LLM, it can include a description of available tools, context, and conversation history. The typical pattern is to stream text into a model and we get the answer streamed back out.

If we want more structured replies, we can give these LLMs a schema and tell them that their output must conform to that schema. This structured output can be anything from data queried from a customer’s account to a set of product recommendations or an itinerary for a vacation based on weather forecasts and traveler preferences.

Sometimes this output schema is extremely focused, like asking specific questions. Is there PII exposed anywhere in this text? Is the sentiment in the text negative or positive? Does the supplied context refer to a tech support issue or is it an account query? Narrowing down the potential output of an LLM to these extremely focused questions dramatically increases accuracy and reduces hallucinations.

However, using a general-purpose LLM to answer these questions is extremely inefficient. While you could technically use a jumbo jet at runway taxi speed to commute from home to the office, it’s slow, inefficient, and costly. Overkill has a cost and many of us are using all-purpose LLMs when we could be using something more focused and efficient.

This is where Jev comes in. It’s a special kind of model with its own interface style. It doesn’t stream text bi-directionally like LLMs. Instead, you ask it questions based on some state and it gives you answers and their probability scores.

## Performance and cost figures TypeSafe published for Jev

TypeSafe published these figures with the Jev launch on September 15, 2026. All of them come from TypeSafe’s own evaluations and demos.

| Claim | Jev | Comparison, as TypeSafe states it |
| --- | --- | --- |
| End-to-end response time | 70 ms to 500 ms | 3 s to 329 s for frontier models |
| Speed on System One queries | 40x to 200x faster | The same level of frontier intelligence |
| Workflow evaluation, time per task | 0.114 s | 8.566 s for LLMs, 193.6x slower |
| Workflow evaluation, cost per task | \$0.000081 | \$0.013880 for LLMs, 444.6x more expensive |
| Accuracy, average of four workflows | 67.8% at \$0.0004 per workflow | GPT-5.6 Terra 67.9% at \$0.0304; Claude Sonnet 5 67.8% at \$0.1174 |
| Input token price | \$0.042 per million (\$42 per billion) | \$0.20 to \$10 per million |
| Input price against Claude Fable 5.1 | 238x lower | \$10 per million |
| Output token price | Free | About 5x the input price |
| Hallucinations and type errors | Zero, guaranteed by the output schema | LLMs hallucinate and make type errors |
| Confidence | A calibrated probability with every answer | Overconfident and inconsistent, even when prompted for confidence |
| Doom demo | 10 queries per second for about \$7 per hour | None given |

TypeSafe states that the 193.6x and 444.6x figures are on the higher end of real-world gains. The zero-hallucination figure follows from the output schema and was not measured. The LLM error rates TypeSafe compares against come from OpenRouter traffic. Sources: [TypeSafe launch post](https://typesafe.ai/blog/introducing-system-one-models-and-jev), [typesafe.ai](https://typesafe.ai), [evals.typesafe.ai](https://evals.typesafe.ai/).

![Figure 1. Accuracy against cost per workflow. Every model runs the same workflow code. Reference answers are the average of GPT-6 Astra and Claude Fable 5.1 at high thinking.](/images/jev-accuracy-vs-cost.svg)

![Figure 2. List price per million input and output tokens.](/images/jev-input-token-prices.svg)

Let’s take a look at a classic use case for the kind of interrogations Jev makes fast and cheap. The user’s original prompt looks like this:

> "Hi, I ordered the blue jacket on the 9th and it still hasn't shipped. I've emailed twice already. If it's not out the door by Friday I want a refund. Order #48213."

Using a traditional LLM, we might be able to get some actionable routing or planning decisions out of this. What we really want are some answers to discrete questions, and then our agent can take the right actions based on those answers with high confidence.

| Question | Allowed Answers | Jev’s Answer | Probability |
| --- | --- | --- | --- |
| What is this about? | **Shipping**, returns, billing, product question, other | **Shipping** | 0.94 |
| How is the customer feeling? | Calm, **frustrated**, angry | **Frustrated** | 0.81 |
| Is this a repeat contact? | **Yes**, no | **Yes** | 0.97 |
| Is the customer threatening to leave or dispute? | **Yes**, no | **Yes** | 0.88 |
| Does this need a human, or can automation handle it? | **Human**, automation | **Human** | 0.76 |
| Which queue? | Tier 1, tier 2, **retention** | **Retention** | 0.71 |
| Is there an order number in the message? | **Yes**, no | **Yes** | 0.99 |
| Priority | Low, medium, **high** | **High** | 0.83 |

Jev does one thing: answer questions. It does this quickly and cheaply. If we want to interrogate a state based on a user’s input, then this kind of Q&A is the right optimization. Take a look at the question, “Is there an order number in the message?”. Jev-style interactions don’t let us extract the order number from the message. Instead, we can only ask questions with a fixed number of potential responses. That limited inference target area is what makes this kind of fast and cheap interrogation possible.

A pretty popular pattern is to interrogate the state and message and, based on the answers we get, we decide whether we need to make a full (relatively slow and costly) call to an LLM. We might use this to extract an order ID only if our interrogation is confident there is one in the message. If we are going somewhere on a nice day, we can choose to ride a bike, otherwise, we can use the massive transport truck.

## Asking questions in Akka

In the [Akka SDK](https://akka.io/platform/sdk), interaction with models is managed via an effects API. Sending a system prompt, user prompt, and other context to a model and getting a stream of text back is I/O and not a “pure” function. The same applies to asking Jev (or technically anything that talks SystemOne, which isn’t yet a standardized protocol) questions, though talking to Jev is always synchronous.

Here’s how you can easily use the effects API (available to the public soon in an upcoming SDK release) to have your agent ask Jev questions:

```java
public Effect<Judgment> triage(String ticket) {
  return effects()
    .judgment()
    .state(ticket)
    .question("route", Question.choice("Which team should handle this?")
        .option("billing", "Payments, invoicing, refunds")
        .option("technical", "Bugs, outages, integrations"))
    .question("severity", Question.score("How severe is this?",
        "Low", "Medium", "High", "Critical"))
    .question("urgent", Question.yesNo("Does this need a reply today?",
        "Time-sensitive", "Can wait"))
    .thenReply();
}
```

Here the `Judgment` class from the Akka SDK contains the answers and their probabilities.

This is incredibly powerful on its own, but it’s really a game-changer when combined with the rest of Akka’s agentic arsenal, including durable workflows, autonomous agents, and composed agentic hierarchies. Using fast and cheap triage on user input can save the LLM use only for when it’s really needed, increasing performance and decreasing cost.

Just as with LLM interactions, the model provider settings are provided through regular configuration properties at runtime.

## Using Jev to make routing decisions in the Akka AI Gateway

If you’re running your Akka projects on Akka’s automated operations infrastructure with [Akka Optimize](https://akka.io/platform/optimize), then you have access to Jev through our AI gateway. Akka’s AI gateway can route requests either to the API-based Jev hosted at [Typesafe.ai’s portal](https://typesafe.ai), or, you can self-host an OpenJev open-weight model within Akka’s inference engine for local inference.

One of the many things this gateway does is provide traffic routing for LLM interactions.

![Figure 3. The path of an agent's judgment request through the Akka AI Gateway to a SystemOne API.](/images/jev-akka-gateway-routing.svg)

What is routing if not asking the question, “Where should this request go?” It’s possible to use a full LLM for routing decisions, but that’s often overkill. A popular alternative is to use a locally hosted SLM to make those decisions. When you leverage an SLM to aide in routing decisions, you can start to make routing decisions based upon the substance and the content of the AI message itself, in other words routing based upon the semantic context of the prompt.

Akka’s gateway can also use semantic routing to quickly route based on use case. This uses embeddings and doesn’t involve an LLM at all. If you want to route based on interrogation of the message and its content, then you can use Jev to triage your AI message traffic.

Simply supply questions and answer possibilities to the router and choose the destination backend based on those answers and their probabilities. The answers, chosen backend, and probabilities will all be logged for compliance and governance just like everything else that goes through our AI gateway.

Adding in new support for Jev, here are some of the different ways you can route AI traffic in an Akka environment (these are fictional examples):

- **Static routing.** A prompt under 300 tokens and no tool definitions attached goes to Claude Haiku 4.5; anything with attached documents over 20k tokens goes to Claude Sonnet 5; everything else defaults to Sonnet 5 with Opus 5 only when the request carries a `reasoning: high` hint.
- **Tool guardrail.** An agent loop asks Jev "which tool should run next?" as a choice over the registered tool names, and only invokes the full LLM to author the tool arguments once Jev has picked the tool with confidence above 0.85. Below that threshold, the LLM makes the full decision itself. This happens before traffic gets to the gateway, which can make further decisions based on cost, quota, etc.
- **Fast Q&A routing.** Jev scores an incoming support ticket on urgency (1 to 5) and the router sends 4 and 5 to a Sonnet-backed agent with escalation tools, and 1 to 3 to a Haiku-backed responder with a canned-answer retrieval index.
- **Sovereign routing.** A tenant with an EU data residency clause routes to Vertex AI in europe-west4 and is never allowed to touch the US-hosted direct API, enforced by policy rather than by the caller.

## Summary

Jev is an incredibly new product, yet there are already a number of open source alternatives that all communicate using TypeSafe’s SystemOne concept. This specific interaction type is new, but wanting to interrogate AI input and context isn’t.

Incorporating structured questions and answers into your agentic applications has the potential to make things more predictable, more reliable, faster, and even cheaper. We’d love to hear what you’re building and how this kind of optimized Q&A fits into your strategy.
