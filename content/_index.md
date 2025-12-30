+++ 
draft = false
date = 2025-12-27T20:25:42+01:00
title = "The Blog"
description = "How am I, what am I building and Some Rules"
slug = "the-blog"
authors = ["22peacemaker"]
externalLink = ""
series = []
+++

# The Me and Why

I am Jannis Karpinski, a youth welfare worker from Germany. I have been programming since my youth and have left many projects unfinished (see my Github). For some time now, I have fallen in love with Golang and have been using only this language ever since. At some point, the idea took hold in my mind that I would only share what I wanted to share. Not only on GitHub, but also where it is not immediately visible. I do not want all my data to be sold to data brokers to analyze my behavior and preferences, often without my consent or clear benefit to me. Therefore, only what I want to share should be passed on. Open source and people who voluntarily host software for the public can contribute to this. However, there must also be people who write this software. I want to be one of them!

## What is this Blog About

In this blog, I will describe my journey to build a decentralized voice chat system. It should be secure but also supported by the community. I don't have the resources to host it all myself, which is why, if it ever gets released, only small core systems will be hosted by me and the rest will be hosted by the community. But honestly, I have to get that far first. Let's see if it works out.

## Goals and Structure of the Blog

The blog aims to explain concepts and principles in an understandable way, but also to provide implementation ideas and justify decisions.
It is also intended to serve as a reference for me, so that I can remember everything.
This results in the following sequence:
1. Explanation of the concepts to the point where I understand them.
2. Explanation of why I use this way.
3. Step-by-step implementation.

his is not a basic course on Go or cryptography. Principles that I myself have not understood may be explained, and I will take a closer look at principles that I myself have not understood. My aim is to remove any stumbling blocks.

I cannot share every single line here, so I will refer back to the networking repo again and again. You can look up everything there.

## What Could Happen
I am not showing you the most optimal way to write something, nor the “correct” way to implement something. I am convinced that such a single correct way does not exist—true to the saying, “All roads lead to Rome.” (At least that’s how we say it in Germany.)

As I said, programming is my hobby, and even I might someday run out of time or motivation. That is not my goal, just a realistic assessment (I’m a pedagogue, after all).

The code may of course also contain errors or open up critical security vulnerabilities. In that case, feel free to contact me or even contribute to the code yourself. Feel free!

## Why This Approach?

There are three very simple reasons for this:

1. I am just learning all this myself, so in a way we are learning together.
2. I want to make it simple, take the magic out of it.
3. Everyone should have the opportunity to build it themselves.

## Why This Project?

This is also relatively simple. You keep coming across two names when you want a voice server: TeamSpeak or Discord. Behind both are companies that want to make money. They just do it in different ways—TeamSpeak sells servers, Discord sells data.

If you then say, “Fine, I’ll just use TeamSpeak,” you quickly run into the fact that TeamSpeak is simply missing a few features. Since all of this somehow annoys me, I want to try building it myself. Even if that seems impossible, my guiding principle is: “Then you just haven’t tinkered with it enough yet.”


I don’t even have to say that we’re using UDP sockets — not because it’s secret, but because it should become obvious once we break things down.