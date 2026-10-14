---
title: "Learning to let go in the age of AI"
description: "As developers using AI, learning to let go of things we used to hold sacred."
published: 2026-10-14
draft: false
tags: ["AI", "patterns", "Engineering", "Vibe coding"]
---

I've been using an AI-assisted development workflow for quite some time now. In the early days when the experience and results were terrible, I still forced myself to try so that I could learn from the experience and pass that experience on to others.

My first real moments when I thought an AI-supported development loop could be useful was when I was exploring spec-driven development. While you can informally build any kind of spec and let your agent loose on it, I was specifically using **spec-kit** because I wanted to see how that formalization of the flow worked.

I've been seeing a number of sentiments showing up in the hype cycle and on "the socials" that have forced me to re-examine some of my long-held positions. For example:

* _"I don't manually write code anymore"_
* _"I don't even look at the code output anymore"_
* _"I don't manually review pull requests anymore"_

My knee-jerk, gut reaction to this is _**do you people even care about quality??**_ How is it possible for folks to ship code to production that a human didn't review and a human didn't write? That's insanity right?

Right?

Let's take a look at why I reacted this way. Throughout history, we've written code so that it is clear and concise; so that it's easily read and maintained. We do this to be nice to the people who come after (including ourselves). We do this so that we can write tests and make sure they work and so that when something fails, we can troubleshoot it. Quality and cleanliness have been synonymous with best practices and professionalism for as long as I can remember.

_But does it still matter?_

Just yesterday I was in a Claude session where "we" were building a feature on my side-project (a game called **NO CARRIER**). I meticulously described all the requirements, specifications, and invariants that needed to hold in the generated output. I took a quick glance at the generated code, and I found something terrible within 10 seconds.

_"That's it, then. AI generates crap code so we need to police it_," one of my inner voices exclaimed.

The `main.go` file had a bunch of logic in it that didn't have anything to do with the command-line binary. It had code in it that was a near-duplicate of code already written in `/internal/...`. I've been trained by isolationist monks in the mountains of Tibet to reject violations of the Single Responsibility Principle (SRP). This code was doing too much and I needed to fix that.

This kind of "leakage" of business logic code into the binary wrapper usually triggers a comment on a PR where someone tells me to refactor. But _why_? Why does it matter that the `main.go` file is too big and difficult for a human to grok?

It matters because the development loop is designed around the _old_ premise that a human needs to be able to quickly and easily read `main.go` and add command-line switches and miscellaneous stuff without interfering with the regular library code. But what if humans aren't reading `main.go`?

If the only consumer of that file is an AI agent, then it doesn't matter what it looks like. If the AI can figure out what's going on and how to make changes, is that enough? Do we _care_? This is where things get hard for me.

I **_care_**. I care _a lot_. I take pride in the code I write, I take pride in how reliable it is, how easy it is to maintain, how easy it is to read and troubleshoot. I take this all personally, and I apply the "camping rule" when I write new code: _leave the code base a better place than when you arrived_. 

The idea of an AI generating code that is reviewed by another AI that is then possibly deployed by yet another AI terrifies me. From my workstation to production without a human in that loop? _Yikes_.

Recently I've been asking myself why that terrifies me so much. What is really causing my fear? That's when I realized that my fear is actually what makes me resist this AI-native flow. We're afraid of our code, and we're afraid of what might happen when it is unleashed on the world.

I've learned to conquer that fear by _releasing more often_. By automating things and releasing more frequently, I have to build code that I trust. It forces a rigor that we don't always apply, especially in big companies where releases might only happen 2-3 times per year.

Setting aside my fears, here's what I care about:

* Does the app do what it's supposed to?
* Does it violate any of its requirements?
* Does it meet its performance requirements? Has it regressed?
* Do all of the unit and integration tests pass?
    * Do I agree with all of the things being asserted in these tests?
* Does the code pass lint and static analysis checks?
* Does it pass security audits?
* Does it crash or fail in acceptance test or staging environments?
* Does the PR pass the AI-run code review that is applying our constitution rules, style guidelines, and other core principles?

If the code does what it's supposed to, it does it as fast as it's supposed to and as reliably as it's supposed to, and it deploys properly and holds up to stress and fuzz testing and benchmarks... do I still care what the code looks like?

Even something as low-level as not aligning data structures to the right boundaries and causing a false sharing issue in the processor cache can be picked up and fixed when some of the above bullets fail their checks. **_Assuming we're measuring the right things_**. We still need to declare what success and failure look like, and we _cannot abdicate that responsibility to the AI_.

All of the bullet points above can be measured and verified by an AI agent, inside or outside of an automation/CI pipeline. If we can vet that all of these conditions are met, does it really matter whether the `main.go` file is poorly designed and violates some best practices for modular coding?

Put another way: _If we can automate confidence measuring in human-generated code, then we can do the same for AI-generated code_.

## Summary
Everyone is afraid of being left behind by whatever the current AI hype train is pushing. It's exhausting trying to stay up to date with things and choosing where I devote my learning and experimentation time. 

Increasingly, I find myself feeling like the old change-resistant codger who yells at a cloud to resist change rather than because he has real reasons to object.

I still write code by hand because I love it. But when I need to ship a new feature in a week, if my pipeline gives me confidence, then I have confidence that putting AI-generated code in that pipeline will still result in an application that meets my customers' needs, even if `main.go` is a plate of spaghetti.

What's your stance on this issue? Hit me up on the interwebs and we can discuss.