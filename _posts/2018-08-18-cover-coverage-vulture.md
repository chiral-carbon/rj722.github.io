---
layout: post
title: "Do we really need to cover coverage with Vulture?"
tags:
- vulture
blog: true
category: blog
---

The team behind Vulture (a tool used for detecting unused Python code) decided
not to integrate it with coverage (a tool for measuring code coverage of Python
programs). Read why!

<!--more-->

## `coverage` - wow, so accurate - we need it...?

When this phase kicked in, I was still wrapping my head around coverage - My
plan was to get coverage integrated with Vulture, which would allow users to
"transfer" the results from coverage to Vulture so that the false positives were
automatically detected and thereby supressed. It sounded so neat and moreso
doable (using an interminnent xml file) and so naturally, I just quickly got
down to nuts and bolts and started a Pull Request. But, alongst those splendid
colors of awesome functionality, Jendrik comes in and talks a bunch on why we
shouldn't do it. I could extrapolate the following reasons:
* We already created an easier, dynamic and robust way to create and manage
  whitelists (`--make-whitelist`) which shall eliminate the need of having 10
  different things for dealing with false positives.
* Coverage is a tool for dynamic analysis (which requires your code to be
  actually run) and is therefore slow, but gives much more accurate results.
  And, if we already have results from coverage, why would we then need Vulture
  for??
* Vulture is supposed to be a static analysis tool.
* Vulture would've no longer been independent of external modules.

But, still it struck me as a little odd at that time because I thought that the
functionality was optional and if someone didn't want it, he would simply just
not use it - simple. By now, may be you've judged that that this was the
_"feature syndrome"_ talking (The more features we have, the more usable we are)
and yes, you're right. Luckily, Jendrik foresaw this early and redirected me
towards [http://neugierig.org/software/blog/2018/07/options](this blog post)
which explains why it's actually toxic for anything to have more "options" and
how it was an expensive process in terms of time spent on writing, documenting
and maintaining it.

I'm very thankful of Jendrik and proud of the fact that we've still managed to
keep the workflow involved when using `Vulture` as simple as it could get. :-)
