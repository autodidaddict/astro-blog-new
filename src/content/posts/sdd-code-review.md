---
title: Uncle Kevin's Further Unillustrated Adventures in SDD
published: 2026-05-28
draft: false
tags:
  - Agentic
  - Elixir
  - AI  
  - Spec-Driven Development
  - Claude
description: I continue my spec-driven development journey with constitution and code.
---

In my first post on this subject I talked about how the act of following the SDD process is a great way to increase the predictability of the AI and get much better results than if you'd "vibe coded" directly from a conversation with the agent.

The **clarify** step is especially important. This takes a look at the specification that your prompt produced and does a deep analysis. It attempts to find holes in your specification that can cause ambiguity or even force the coding to stop for human intervention.

In this post I want to discuss my thoughts around the code output of a spec-driven process (again, _not_ vibe coding). There are a couple of pretty zealous camps that divide opinions on agent-generated code. The first camp holds the view that the code is still the most important artifact and it must be reviewed and subject to all of the constraints non-AI-generated code is. The other camp believes that if you follow the process well enough, then you don't even need to review the code anymore.

The first group is the less controversial of the two. Regardless of how code came to be in existence, it must be subject to reviews, linting, analysis, and testing.

The second group can cause fights to break out at conferences or even in coffee shops. The opinion, in a nutshell: **_If the generated code meets your specification, passes linting and format rules, and has generated tests, then humans don't need to look at it_**.

I may draw a lot of ire and ridicule from the community for this, but I believe that this is the same as saying, **_"My self-driving car is working, and its self-diagnosis is green, so I don't need to look at the road."_**

## Testing

When I use Claude and spec-kit, Claude writes all of the unit, integration, and automated smoke tests just like it writes all of the code. I admit that I don't review the tests with as small a magnifying glass as I do the regular code. If all the tests pass, and the smoke test mirrors what I would do manually, and that passes, then I move on to my own manual verification.

Manual verification should be mandatory, but I've even seen people claim that we can skip this. Without manual verification, I would have shipped some absolutely _colossal_ bugs. Since I wrote the spec prompt and I approved the generated spec and plan, then I know enough to be able to go through all of my edge cases manually.

During this phase, I regularly find bugs. I also find bugs that _should_ be covered by the existing unit tests but aren't. These are worse than regular bugs because it's easy to assume that if the test passed, that bug's gone. I'll forget within minutes that a passing test was making the wrong assertions. 

If Claude makes a habit of letting the same types of bugs slip through bad tests, then I'll go into the constitution and add explicit rules for verifying code behavior and writing tests to prevent these. I do **not** just fix the bugs and move on. **_My role as the spec-driven AI developer is to teach the tool how to make my job easier_**. If I clean up after its messes but don't teach it how to avoid the mess, I'm wasting my time and money (tokens, etc).

## Spec-Kit Implement

The implementation phase is where the code is generated. This phase is only ever run after I approve the specification, approve the plan that includes the technical and architectural approach, and approve the list of tasks discovered to complete the implementation. At this point I usually have a pretty high degree of confidence that I'm going to get good code out of my agent.

This phase is iterative. Claude will produce a finished set of code, run unit tests, run integration tests, and run smoke tests. It may also give me some instructions on how to interact with the system for testing. It does this a lot when I don't yet have a UI. I'll then go follow the instructions and verify that the system does what it's supposed to. If everything is good, I'll then go on to smashing the system and hitting all of the edge cases.

This edge case discovery usually turns up a couple bugs that I report to Claude for repair. Depending on the size of the feature, I've gone through this step with no failures and I've also had night where I've spent over 4 hours trying to get something to work the way I want.

### Don't Panic!

I watch the terminal as line after line of code whizzes by and I see little notes from the agent like "Now I'm adjusting the widgets to only work on Tuesdays", etc. Most of this spam is just that and I ignore it. Other times, however, I'll see something that looks terrifying out of the corner of my eye. My first reaction is _"Why the hell are you doing this? Don't you know this is wrong? What the hell??"_. 

The problem is, I've only caught the tiny bits that Claude is spamming out. It's like overhearing half of a phone conversation (assuming people still use phones to make calls) and making a decision based on that.

I used to smash the escape key in a panic, stop Claude dead in its tracks, and then demand it explain itself. _What the hell is going on?_ _What are you doing, you fool?_

Counterintuitively, this is usually a _bad_ idea. I've found that when I stop Claude in the middle of a thing to demand an explanation, it is noticeably worse at finishing that task than it was starting it. This is a small sample size and only my own experience, but I no longer hit the panic button unless I'm 100% sure we need to stop the train immediately.

Instead what I do is wait until the next pause in work and I'll prompt something like, _"Can you clarify the following assumptions and questions for me and include justification? I saw (something) in your logs that I need more detail on...(assumptions/questions)". Claude will then give me rationale for everything it was doing, tie it to a particular task and the part of the plan from which the task came.

If you're not familiar with the concept of [Chesterton's fence](https://fs.blog/chestertons-fence/), I highly recommend you take a quick detour to read about it. The short version is _"don't take a fence down unless you know why it was put up."_

We don't always stop to ask why a thing is the way it is, we routinely apply our own limited perspective to what we see. This urges us to make bad, uninformed decisions. I've failed this test a million times, and when I first started doing spec-driven with Claude, I did it a lot because I simply didn't _trust_ the agent to do what I wanted the way I wanted.

So rather than risk poisoning the context or degrading behavior, I'll soak in what I've seen and wait for the next stopping point. Then I will ask Claude to give me what I need to reassure me that it's done the right thing (or prove that it's been doing the wrong thing).

There have been a lot of _"Oh yeah, that's right. Huh, I hadn't thought of that"_ moments this way. Claude was actually following my spec and dealing with things that I hadn't yet considered.

## Code Analysis

There's a "polish" step encoded into the spec-kit instructions. It takes a look at the code that was produced for the current feature and checks it for standard opportunities for refactoring or simplifying or clarifying. I routinely see it find groups of code that can be refactored for better reuse among the multiple modules produced over time. It's just like a regular human programmer in this regard: we might not see these groupings until after we've written all the code.

I do something extra here that some might find to be a waste of tokens. After Claude has given the thumbs-up on completing the polish phase in this session/feature, I ask a _completely ignorant_ agent to do an analysis of the code. I deliberately want an agent that doesn't have any context from the conversation to analyse the code. 

This "LLM as judge" agent isn't even given the spec. I _just_ ask it to analyze the code. For my current project (**Agentic Realms**), I've had to include specific instructions to look for code that will cause problems when running in a multi-node cluster, race condition issues, bad database performance, anti-patterns, and poor use of `GenServer`s.


## Constitution

In the context of spec-driven development, a **constitution** is a core rulebook that dictates _how_ software gets built across the entire project. It serves as the immutable architectural governance layer that both AI agents and human developers must follow at all times.

Most of the people I've spoken to about using constitutions don't really pay it much attention. For Akka projects created using `specify`, we automatically include a constitution that captures rules and best practices developed by the Akka team over the years. For my project, I think I actually left the constutition untouched for my first few features.

Lately I've been adding a number of rules to my Agentic Realms (Elixir/Phoenix) constitution because (here comes the controversial bit) I've been **_reviewing the code_** and have found a number of very, very, very-very bad things. I've included my list of [unbreakable rules of event sourcing](https://pragprog.com/titles/khpes/real-world-event-sourcing/) as well as a handful of others that arose specifically from terrible code:

* It wrote code that directly manipulated the persisted state of an aggregate without use of command or event. This kind of code is easy to miss, especially if you're assuming Claude won't do this kind of thing.
* It accessed the wall clock for computation in an event handler.
* It routinely rewrites huge portions of code that implement already existing things. It rewrote (poorly) the CSS functionality that comes from `columns: reverse`.
* It misdiagnosed something as a race condition and spent 2 hours trying to fix, when the problem stemmed from a far-too-tightly-coupled string assertion in a test on HTML output.
* It sent every single character a user typed to the server for a full Phoenix component round-trip for no reason whatsoever. It couldn't even tell me why it had done this. The text input field wasn't a live update or an autocomplete, it was just a box that waits for the user to click submit or hit enter.
* It kept writing (over and over and over ...) code that would break if a piece of JSON de-serialized into something that had a non-existent atom in it.
* It constantly made `GenServer` and process and messaging mistakes that would go unnoticed when testing locally and would fail when on `n`-node clusters.


## Wrap-Up

After all this blabbering, what I hope you take away from this post is that _you own the policing of AI generated code_. The AI, no matter how advanced or how good your prompts and specs are, is going to make mistakes. It's our job to spot those mistakes before our customers do.

