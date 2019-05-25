---
layout: post
title: "The story of Dead Code, Vulture and scavenging"
description: "Vulture Vulture, Chop Chop Dead code - Hoard no more, throw away the unused"
tags:
- vulture
blog: true
category: blog
---

It isn't uncommon for software developers to encounter some code that they had
written in the past and reflecting on it - the most common reaction would
probably be "It must be the most horrible thing I wrote". But sometimes, there's
that aha moment where you find something and you are instantly gratified and
proud of yourself, "Oh, this is so beautiful, no wonder it took so many
sleepless nights". However glamorous it may sound, but it is indeed a difficult
task to write and maintain such code, and this is where automatic tools come in
to the picture. Let's discuss about one such tool - [Vulture][vulture], which
helps discover unused stuff in Python code.

So, today we present to you the voodoo which throws out unused code.

<!--more-->

## The obvious TODO - Find things which aren't used and remove them

<!--I am Rahul Jha. I study Electronics Engineering at Aligarh Muslim
University.--> <!--Also, and more importantly, I am the youngest person on the
face of earth to--> <!--have spoken at so many technical conferences,
_obviously_ aside from the people--> <!--who have spoken at more conferences
than me.-->

Before, we let ourselves dive in the nitty gritty details of [Vulture][vulture],
let me first tell you a story from my childhood.

> Being a young, fearless and curious lad I was, whenever I used to visit
> someone's house, I was very interested in some of the stuff they had in their,
> ahmm, store room - All those cool **worn out** CFL Bulbs, game boards,
> extension chords and all other kinds of circuitry you can imagine and just so
> it happened, people would readily give me any of those items I asked for. And
> guess what, I used to take away **all** of that with me (my mom was kind
> enough to let me play with it, obviously as long as she didn't see all of it
> :D). There were two main reasons I used to collect these 'things' - One, I
> wanted to understand how such wonders could actually happen, and second and
> more importantly, I thought that if I become a circuit wizard some day, I
> might need some of this 'equipment' in my toolbox. Now, while learning this
> wizardry, of course, I had to deal with new tools as well, so I bought some
> things for myself, and more and more. On one fine day, it so happened that I
> came up with this wonderful idea to create a local fax machine (a different
> story for a different day), and I desperately needed a couple of (working)
> bulbs to fit into it, but I couldn't find one in my stash - none of them would
> light up, although I knew that I bought a few earlier and had them somewhere
> here. That was a painful experience and that's when it hit me - How important
> it is not to keep things which aren't/can't being used anymore or at the very
> least, storing them away. Anyways, I had a hard time testing every piece and
> dumping the ones which couldn't be used anymore.

> PS: It later turned out that I didn't actually had any bulbs, the ones' I had
> bought were already martyred in my brother's quest to see what would happen to
> the filament if we crack the glass open, genius it was!

Later, when I found Vulture, I realised the worth of a tool which could do such
fumigation for me.

## Introduction

By now, it must've been pretty clear what exactly does Vulture do - it cleans up
stuff, chops dead code. But, this statement begs the question that what exactly
is dead code?

### What is dead code?

In simple terms, dead code is that code which is never ever run by your program.
Some examples of dead code:

* A variable/function/class which is defined, but is never utilised by your
  program.
* Code after return statement in a function/method.
* Code inside an `if False:` statement. (:facepalm:)

### Why on holy Earth would I write dead code in my own software?

> For the same reason you introduce bugs in it ;-)

Dead code isn't intentionally introduced into the source code, instead it just
crawls in through tiny sieves of carelessness and lack of attention, which is
completely humane. Some of the most common causes of dead code are:

* **Refactoring** - Many a times, programmers delete some legacy code which used
  to call a function (or use an import) and rewrite it in a different manner.
  Thus, eliminating the need of those old functions/imports. Now, these unused
  functions/imports are dead.
* **Misspellings** - Was it `increase_temperature` or `temperatur_increase`???
  Hmmm, let's go with `increase_temperature` - If both of these names are
  already defined in your codebase, and you used the wrong one, not only have
  you introduced dead code, but also a bug into your program. Luckily, Vulture
  would help you catch those bugs.
* **During debugging** - As humans, we tend to be lazy and that is why instead
  of using debugger checkpoints, we tend to use debug strings like `"Reached
  here"` and `"Here2"` and obscurely placed `return` & `break` statements. If
  not paid attention, these changes can be committed and would again introduce
  bugs alongside dead code.

By now, it must be evident that having dead code is **evil** - it causes bugs,
confusion for newcomers and unnecessarily increases your program size and that
you should absolutely remove it. It also helps improve maintainability and code
quality of your program.

In the next post, I would continue with how to install Vulture and how to use
it.

[vulture]: https://github.com/jendrikseipp/vulture
