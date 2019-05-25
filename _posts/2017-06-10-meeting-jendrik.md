---
layout: post
title: "Meeting Jendrik"
date: 2017-06-10 02:16:57
blog: true
category: blog
description: 'A meet with my mentor, tweaking the VultureBear and my new Laptop. Ahh, perfect.'
tags:
- meeting
- mentor
- Jendrik
---

A meeting with my mentor, tweaking the VultureBear and my new laptop. Ahh, perfect!

<!--more-->

# Coding period kicks off

A quite has happened since I last wrote, most important of which was a meeting with my mentor [@jendrikseipp](https://github.com/jendrikseipp), which by the way went awesome. Thank You! @jendrik

## A meeting with my mentor @jendrikseipp

A little overview - In the starting of the session, we had already discussed
that we would be meeting weekly on Wednesdays at 3ish PM UTC +2:30 on
hangouts, but due to some circumstances we couldn't meet prior June 7.

**Wednesday, 7 June 2017**- I was feeling a little nauseatic, so I decided to take a short nap (which later turned out to be of 4 hours ). When I woke up, it had already been past 7 (in my timezone UTC +5:30) - Oh! I missed the meeting again, I thought, it had been quite a bad day, so this was kind of expected :cry:. But I thought it would be better if I could atleast update the doc which we maintain for our meets. I quickly penned down some points I thought would be better if Jendrik took a look and also drafted an email stating the same.

Just a couple of seconds later, he messaged me, saying if he could call me in 10 mins. I was excited and nervous both at the same time, and replied with a text stating it would be great indeed (Finally). I went upstairs to the rooftop - the only place you *might* get silence in an Indian joint family :stuck_out_tongue_winking_eye:. Waiting meanwhile, I browsed through Jendirk's G+ profile, saw that he is currently studying at University of Basel, Switzerland, probably doing PhD - I read some of the blog posts about the student life over there and immediately fell in love with it.

When I picked up the phone call, to my surprise, it was a video call - I wasn't presentable **at all** (I thought that this would be a voice call). While looking at my screen, I can see there is a handsome guy, dressed up nicely ,in his cabin, and me on the other hand, was somewhere in the dark, and was shabbily dressed perhaps food deprived for a few days (atleast, I imagine myself that way) - I am very sorry @Jendrik, I would be presentable the next time you call me ;P . Quickly, I ran into my room and caught my breath. We exchanged greetings and then Jendrik asked me about how was everything going? To my surprise (I am quite talkative), I couldn't speak anything - I knew exactly what to, but just couldn't - I was quite nervous, but since Jendrik sounded nice and rehearsed, the conversational flow went very well :-). We talked about a wide variety of topics ranging from VultureBear to college life, coala and what not.

To conclude, Jendrik is quite an astonishing guy and that it was a thrilling experience to talk to him. I had a conversation that is amongst the most cherished ones of my life. I am looking forward to our next meeting.

## Tweaking the VultureBear to follow the vulture API

### Overview

* `coala`
    *  `coala` is a smart code linter - It finds problems in your code and even proposes a patch to fix them. It has primary processors called Bears - They contain the logical part whereas the infrastructure is provided by coala.
    * [coala](https://coala.io)


* `vulture`
    * It is a dead code reporter - It reports unused variables, functions, classes, etc.
    * [vulture](https://github.com/jendrikseipp/vulture)

* `VulturBear` is a coala bear which finds dead python code by running vulture over it.



### The Problem

How `VultureBear` currently worked:
* It runs a shell command: `vulture filename1.py filename2.py ...` where filenames is provided by coala.
* It then captures the output.
* Analyses the output through `re` catching every line of result for getting the `filename`, `lineno`, type of entity(function, variable, class, etc.)
* Creates a Result Object

This implementation had several issues:
* It creates a `Popen` instance.
* It uses `re` to analyse the files and their asymptotic complexity may rise exponentially.
* If vulture tweaks the way results are displayed, `VultureBear` won't be able to cope up with this.

### The current implementation

Taking a closer look at vulture, I found that there is a utility provided by vulture which would do just the same thing in a much more cleaner way:
`vulture.Vulture.scavenge(filenames)` and this would store all the relevant info (lineno, filename, etc.) in the `vulture.Item` object.

Implementation of a wrapper around this can be like this:
```python
def _retrieve_unused(filenames):
    """
    :param filenames: List containing paths to all the files
                      which are to be checked by vulture.
    :return: List of Result objects containing information about
             dead code (filename, lineno, message).
    """

    def file_lineno(item):
        return (item.filename.lower(), item.lineno)
    unused = []

    vulture = Vulture()
    vulture.scavenge(filenames)
    for item in sorted(
            vulture.unused_funcs + vulture.unused_imports +
            vulture.unused_props + vulture.unused_vars +
            vulture.unused_attrs, key=file_lineno):
        message = 'Unused {0}: {1}'.format(item.typ, item)
        unused.append(Result.from_values(origin='VultureBear',
                                         message=message,
                                         file=item.filename,
                                         line=item.lineno))

    return unused

```

### A note:

When I first tried this implementation, all the things seemed to be in place, except for this `AttributeError: 'Vulture' object has no attribute 'unused_imports'`.

After half an hour of debugging and scratching my hair, suddenly the `PipRequirement` version of `vulture` caught my eye: `0.10.0` whereas current version of coala is `0.14.0` and then it occured to me - Oh! God! may be this wasn't introduced in this version of vulture, quickly upgraded to latest version and everything worked like a charm.

## My Laptop

I finally managed to buy a new Macbook Air. Yay!
Previously, I was using a Dell Inspiron Mini Netbook - Intel Atom, 2 GB RAM, 10.5 inches screen and a tampered keyboard, and absolutely no battery life.
I am crazy happy. :-)
