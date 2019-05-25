---
layout: post
title: "Why use coverage to find which parts of a python code were executed?"
description: "Case Study - How we decided that coverage was the best option for detecting false positives in vulture."
tags:
- vulture
- GSoC
blog: true
category: blog
---

In this post, I'll walk you through the decision making process the team behind
Vulture underwent to come up with a way to deal with false positives in it's
results.

<!--more-->

Let's first start with a brief introduction of
[Vulture](https://github.com/jendrikseipp/vulture):

## Vulture

As the name suggests, vulture helps find [dead
code](https://en.wikipedia.org/wiki/Dead_code) for Python programs. There are
many reasons for dead code ending up in a project. The most common is
refactoring, but another is misspellings, which are only detected at runtime for
dynamic languages. Finding and removing dead code allows to keep the code base
clean and reduces bugs.

Vulture can detect unused imports, variables, attributes, functions, methods,
properties and classes. Other than these, code after return statements and
checking for a Boolean False (eg. `if False:` or `while 0:`) can also be
detected.

### Using vulture

Vulture is a standard Python package, that is installed with `pip`:

```
(venv) $ pip install vulture
```

Let us say that you have the following program (say `program.py`) on which you
want to perform analysis:

```py
# program.py
import os

def false_positive_function():
    """
    A function which is vital for your application to function, but Vulture
    reports it as unused.
    """
    pass

def hello_world():
    message = "Hello World"
    print("Hello World")


def main():
    hello_world()

if __name__ == '__main__':
    main()
```

Analysing the program with vulture is as simple as running the following
command:

```
(venv) $ vulture program.py
```

which would produce the following output (on `vulture 0.26`):

```
program.py:2: unused import 'os' (90% confidence)
program.py:4: unused function 'false_positive_function' (60% confidence)
program.py:12: unused variable 'message' (60% confidence)
```

As you can see, along with every result, vulture also reports a confidence
value - which is a measure of how sure vulture is about that part of code being
unused. This output can be made even more meaningful with the help of flags like
`--min-confidence` and `--sort-by-size`. Read more about them
[here](https://github.com/jendrikseipp/vulture/tree/master/README.rst)

Owing to Python's dynamic nature, Vulture is likely to miss some dead code.
Also, code which is only implicitly used is reported unused, such as overloading
a parent class method, overriding methods of C/C++ extensions, etc.

Some other examples where vulture may report "useful" code as unused:
- **API Endpoints** - They are meant for users and are not employed to any use
  directly in the source code, therefore confusing vulture.
- **ORM Schema** - Again, they aren't used by program's source code directly.

### Handling false positives

One way to prevent Vulture from reporting false positives is to explicitly use
the code anyway. > WHAT! - Are you telling me to run my already used code?

Worry not! - no need to actually call the code - If you create a mocking class
with attributes, name of whose exactly match the name of the unused code
(variables, functions, classes, anything which can be unused and has a name),
you can very cleverly fool Vulture into believing that that part of the code is
being used (because Vulture keeps only track of the names parsed from the AST).
This is known as "Whitelisting" and since it is a fairly common practice to
create such a class for mocking objects, we already ship it with Vulture.

Let me show how you can create your own whitelists with our previous example.
(Remeber we had a `false_positive_function` - let's whitelist it.)

Note that I am calling this file `whitelist_program.py` because it's a good idea
to start the name with `whitelist` just so you know what that file does, but
there is no such compulsion - You can call it whatever you want.

```py
# whitelist_program.py
from vulture.whitelist_utils import Whitelist

awesome_whitelist = Whitelist()

Whitelist.false_positive_function
```

Now, let's run Vulture using the following command:
```
(venv) $ vulture program.py whitelist_program.py
```

And hurray, output does not contain the false positive function.
```
program.py:2: unused import 'os' (90% confidence)
program.py:12: unused variable 'message' (60% confidence)
```

**How did that work?**

Since you also passed the whitelist file along with the file to be analysed,
vulture created `ast`'s for both of them and while parsing those trees, Vulture
created a common `set` for storing the names of used and defined objects. Since
the name of the false positive function occurs in both of them, it is therefore
not treated as unused.

A thing you may find interesting and  noteworthy about the `Whitelist` class is that  it does absolutely "nothing". It's current implementation is as follows:
```py
class Whitelist:
    """
    Helper class that allows mocking Python objects.
    Use it to create whitelist files that are not only syntactically
    correct, but can also be executed.
    """
    def __getattr__(self, _):
        pass
```

Now, since whitelists are so extensively used, Vulture already comes [loaded for
some popular libraries like `sys`, `collections`, `unittest`,
etc.](https://github.com/jendrikseipp/vulture/tree/master/vulture/whitelists/) -
These whitelists are automatically "activated" as soon as the user imports that
library. The developers at Vulture are working hard (gives a pat on his back) to
ship even more of these. Guess what, you can add one for your library, or just
open an issue and we would create one for you. PR's are more than welcome! :-)

**Some other ways of dealing with false poisitives:**
- Mark unused variables by starting them with an "`_`". (e.g., `_x, y =
  get_pos()`)
- Use different files for API endpoints, ORM, etc. and exclude them with the
  help of `--exclude` flag.

You can find more information about [vulture in it's
documentation](https://github.com/jendrikseipp/vulture/tree/master/README.rst).


## What more does vulture want?

As we saw earlier, the results reported by vulture sometimes contain false
positives. We want to be able to develop such a system which should be able to
detect wether or not the result given by vulture is a false positive.

Please note that this discussion originally occurred on
[jendrikseipp/vulture/#109](https://github.com/jendrikseipp/vulture/issues/109)
and this post is going to be a translation with some insight on how we finally
came up with a decision.

### Approach 1 - User employed regular expressions

Vulture has different methods for parsing different kind of nodes in an `ast`.
So for example, if Vulture encounters a function definition, it triggers the
`visitFunctionDef` method for parsing that node and similarly for classes,
variables, etc. Now, in those methods, we can easily insert a check to see if a
name matches with any of the regex then we should ignore that construct.

The implementation for this was very easy, but there was a whole new problem -
How do we present this functionality to the user?

All the inputs in Vulture are supplied through command line arguments and there
aren't any config files or variables (because that would be an overkill for such
a simple tool). Although passing regex through a cli argument is possibe,
seeking that it would be a non trivial task to write such a command and that
there would be way too many different permutations of "lists" of regex for all
the different types of constructs (functions, classes, variables, methods,
properties, etc.), we soon dropped the idea.

### Approach 2 - Using XML output from `coverage.py`

Now, since we were restricted to minimum user interaction, Jendrik came up with
this brilliant idea of automatically generating an initial whitelist and then
letting user adapt it to her needs. This lead us to think that we should let
user run [`coverage.py`](http://coverage.readthedocs.io/) on their code base and
export the XML output which could then be consumed by Vulture to detect which
lines are actually used.

So, we tried a basic prototype. We took a sample file, let's say `ab.py`:

```py
def a(f):
    print(f)

@a
def b():
    pass
```

Running `coverage.py`, the following XML output was observed:

```xml
<?xml version="1.0" ?>
<coverage branch-rate="0" line-rate="0.75" timestamp="1524751600489" version="4.3.4">
    <!-- Generated by coverage.py: https://coverage.readthedocs.io -->
    <!-- Based on https://raw.githubusercontent.com/cobertura/web/f0366e5e2cf18f111cbd61fc34ef720a6584ba02/htdocs/xml/coverage-03.dtd -->
    <sources>
        <source>/Users/rahuljha/Documents/test_everything_here</source>
    </sources>
    <packages>
        <package branch-rate="0" complexity="0" line-rate="0.75" name=".">
            <classes>
                <class branch-rate="0" complexity="0" filename="ab.py" line-rate="0.75" name="ab.py">
                    <methods/>
                    <lines>
                        <line hits="1" number="1"/>
                        <line hits="1" number="2"/>
                        <line hits="1" number="4"/>
                        <line hits="0" number="6"/>
                    </lines>
                </class>
            </classes>
        </package>
    </packages>
</coverage>
```

Now, as you may have observed, the problem with this was that the line numbers
were not accurate (function `b` starts at line number 6, but according to the
report, line only line number 6 is unused) and no information about the name of
the unused code was provided. So, we discraded this idea and decided to go yet
more bare bones and to write a tracer.

### Approach 3 - Writing a `tracer`

A tracer would allow us to keep track of what is going on in the current Python
process, but after trying to develop a simple protoype, I knew that it wasn't a
trivial task. Also, Abdeali who have had previous experience working with such
"technology" held the following opinion:

> If vulture starts to maintain tracer and so on it is starting to go towards
> non-static analysis which I am not sure you guys want to maintain as it can
> have a bunch of issues :/

So, in this moment of confusion, we decided to ask Ned Batchelder, the guy
behind `coverage.py` himself. Let me quote him:

> You will need a trace function. Writing your own doesn't have to be
complicated, though you are right there are details like subprocesses that can
be a pain. Writing a file tracer plugin could be a way to piggy-back on
coverage.py. Getting function names wouldn't be hard, since you can inspect the
frame object in the trace function to get what you need. Variables are harder.
You've noted that coverage.py reports line numbers, but then you say it's hard
to go from line numbers to the information you need. But you already are doing
AST analysis. Couldn't you use coverage.py's line numbers to index into the AST
you already have, and then use your AST expertise from there?

So, we were back to square one (well, square two in this case) - Use output from
`coverage.xml`

### Back to approach 2 - Use XML output from `coverage.py`

We quickly discovered that `<line hits="1">` was merely a binary switch
indicative of whether or not that line was called and contained no information
about "how many times". It was important because it would always be "1" for the
first line of function (It is used when the program is initialised to store the
name of the function in memory) and therefore we couldn't use this switch as an
indication of whether or not the function was actually used. This also explained
why the line numbers were off when we inspected the output earlier.

But, this gave us a workable idea: If Vulture says that a function is unused, we
can check whether this is really the case by checking whether the next line is
marked as unused in the coverage.py XML file. The only exception would be a
function defined in a single line, such as:

```py
def true(): return True
```

which aren't very common nor very useful and therefore can be neglected.

And that's how my dear friends we decided to use `coverage.py` to detect if code
was actually used or not. Although, it would only enable vulture to detect
unused "functions" (and properties, maybe) and not classes or variable, but it
would still be a very useful feature.

P.S. I am excited to work with the team on this feature. It is also one of the
goals of my Google Summer of Code project.

