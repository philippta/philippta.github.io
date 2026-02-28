---
title: "The Missing Piece of AI Discourse is Trust"
date: 2026-02-28T10:00:00+01:00
draft: false
tags: ["ai"]
summary: "The shift from manual coding to AI generation isn't just about correctness. It's about who owns the reasoning and carries the burden of trust."
---

Recently I listened to a Software Unscripted podcast episode titled the "Broken AI Discourse with Steve Klabnik"

In that episode they focused on the point that it technically should not matter who or what produced the code, if it's correct. 

While I do agree that code should be treated as is, I believe there is a much deeper concern that wasn't talked about, which is trust.

As code reviewers, what we're seeing is the final output (code) that a coding agent or LLM produced. What's lost in the process is the reasoning of why it came up with the code and what trade offs or considerations it made along the way.

When I review code that was written by a coworker by hand, I do know who wrote it and I judge it differently based on the experience and contextual knowledge they  have. I have to make these assumptions because I'm not always deeply familiar with the problem they are solving. If my coworker has a good track record of making changes, I can use that as a proxy for my judgements.

These principles fall short if the code was fully written by an AI model. The AI inherently has less context about our whole codebase and our product's architecture than someone with years of experience working on it. It can't anticipate the broader side effects of the change, which is a big part of code reviewing.

So in the end the true author is the one who can justify the logic, regardless of what tool generated the text.

If the author of a piece of code meticulously reasoned through it, with or without the help of AI, I can assume it's trustworthy. If it was generated without any prior verification, all trust goes away, leaving me as a code reviewer with the burden of reasoning through it.

So until we have solved the problem of AI producing potentially bogus code, I suggest we prompt the AI to add resoning details and add those to our pull requests.
