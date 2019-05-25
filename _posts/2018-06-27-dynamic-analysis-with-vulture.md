---
layout: post
title: "Dynamic code analysis with Vulture"
description: "Vulture now automatically creates whitelists by cross checking it's results with coverage.py. Find out how!"
tags:
- GSoC
- vulture
blog: true
category: blog
---

This is a follow up post of [Why use coverage to find which parts of a python
code were executed?][why] - there we discussed how we stumbled on this plan of
dynamic code analysis with vulture. Here, we talk about the development process
we (the Vulture team) underwent to integrate Vulture with
[coverage.py][coverage] in order to automatically generate a whitelist of
functions which Vulture reports as unused but are actually being used.

<!--more-->

# Overview

The idea was to let [coverage][coverage] do the dynamic analysis and report it's
results to Vulture (through `XML`) which could then be used to cross check
functions being reported as unused by Vulture. It was further decided to present
this functionality to user in the form of a command line flag, eg.:

```
vulture --make-whitelist coverage.xml files/
```

This would print a whitelist containing false positives reported for `files/` to
`stdout`. Let's walk through all of it step by step:

# Dynamic Code Analysis - What is it?

To analyse a program dynamically means to execute the program on a real or
virtual processor and to monitor the process to check which lines are being hit
(i.e. being used). This is in contrast to static analysis where the program is
analysed on the sole basis of source code or some form of code object.

[coverage.py][coverage] is a tool, widely popular in Python community which
allows user to perform code coverage measurements through dynamic code analysis.

# Implementing --make-whitelist

Code was pretty simple - just look up for `hit` switch in the `XML` in the
entire range of the unused function. As soon as you find any line which is used,
print it. ;-)

Here is the `make_whitelist` module from vulture:

```python
# vulture/make_whitelist.py

from __future__ import print_function

from collections import defaultdict
import os.path
from xml.etree import ElementTree as ET

from vulture import utils


def create_namewise_dict(v):
    namewise_unused_funcs = defaultdict(lambda: [])
    for item in v.unused_funcs:
        filename = os.path.normcase(utils.format_path(item.filename))
        namewise_unused_funcs[filename].append(item)
    return namewise_unused_funcs


def make_whitelist(v, xml):
    xpath_file = './packages/package/classes/class'
    with open(xml) as f:
        tree = ET.parse(f)
    files = [node.attrib['filename'] for node in tree.findall(xpath_file)]
    namewise_unused_funcs = create_namewise_dict(v)
    for filename in files:
        xpath = ('./packages/package/classes/class/[@filename="{}"]'
                 '/lines/line[@hits="1"]'.format(filename))
        lines_hit = [int(
            node.attrib['number']) for node in tree.findall(xpath)]
        filename = os.path.normcase(filename)
        unused_funcs = namewise_unused_funcs.get(filename, [])
        if unused_funcs:
            print("# " + filename)
            for item in unused_funcs:
                span = item.first_lineno+1, item.last_lineno+1
                for lineno in range(*span):
                    if lineno in lines_hit:
                        print(item.name)
                        break
            print()
```

## Were there any problems? 

Yes, a lot of them. But, I'm glad that there are a lot of smart people in the
Vulture community. ;-)

Here are some weird & notable issues I would like to document:

### Output all over pytest results

When writing tests, I had to capture the output when running `make_whitelist`,
so I decided to use pytest's [capsys][capsys] fixture. The code was as follows:

```python
def test_create_whitelist(v, tmpdir, capsys):
    code = """\
class Greeter:
    def greet(self):
        print("Hi")
greeter = Greeter()
greet_func = getattr(greeter, "greet")
greet_func()
"""
    expected_output = """\
# {}
greet

"""
    sample = os.path.normcase(str(tmpdir.join("unused_code.py")))
    xml = str(tmpdir.join("coverage.xml"))
    with open(sample, 'w') as f:
        f.write(code)
    subprocess.call(["coverage", "run", sample])
    subprocess.call(["coverage", "xml", "-o", xml])
    v.scavenge([sample])
    make_whitelist(v, xml)
    assert capsys.readouterr().out == expected_output.format(sample)
```

But, the output from `coverage` and `scavenge` (in verbose mode) would pollute
`capsys.readouterr().out`, so I tried disabling capsys from recording while
executing these commmands, therefore tried this:

```python
# ...
with capsys.disabled():
    subprocess.call(["coverage", "run", sample])
    subprocess.call(["coverage", "xml", "-o", xml])
    v.scavenge([sample])
make_whitelist(v, xml)
assert capsys.readouterr().out == expected_output.format(sample)
```

The tests now passed, `\o/`, but the output we just omitted would now show up
while running the tests, thus ruining pytest's output when running tests, just
like this:

```
============================= test session starts ==============================
platform linux -- Python 3.5.5, pytest-3.6.2, py-1.5.3, pluggy-0.6.0
rootdir: /home/travis/build/RJ722/vulture, inifile: setup.cfg
plugins: cov-2.5.1
collected 156 items                                                            
tests/test_conditions.py ..............                                  [  8%]
tests/test_confidence.py .......                                         [ 13%]
tests/test_errors.py ....                                                [ 16%]
tests/test_format_strings.py ......                                      [ 19%]
tests/test_imports.py .............                                      [ 28%]
tests/test_item.py .                                                     [ 28%]
tests/test_make_whitelist.py Hi
Scanning: /tmp/pytest-of-travis/pytest-0/test_create_whitelist0/unused_code.py
1 Module(body=[ClassDef(name='Greeter', bases=[], keywords=[], body=[FunctionDef(name='greet', args=arguments(args=[arg(arg='self', 
etc...
etc...
1 Load() class Greeter:
.                                           [ 29%]
tests/test_scavenging.py ...........................................     [ 57%]
tests/test_script.py ...........                                         [ 64%]
tests/test_size.py ..............................                        [ 83%]
tests/test_sorting.py .                                                  [ 83%]
tests/test_unreachable.py .........................                      [100%]
```

Now, after scratching my head for a while, when I was on the brink of giving up,
I pinged Grand Master **[The-Compiler][the-compiler]** for help {% sidenote
'sn-id-florian' 'I think that I underrepresented the enthusiasm and humility
with which [Florian][the-compiler] helped me. He even asked me to ask for help
sooner rather than later to prevent confusion. A huge shoutout to him - Thank You
so much!' %} . In a matter of seconds, he inspected the code and the output and
pointed out that whenever `capsys.disabled()` is used, the output is printed to
`stdout` immediately. Aha!

He suggested the following:

```python
# ...
subprocess.call(["coverage", "run", sample])
subprocess.call(["coverage", "xml", "-o", xml])
v.scavenge([sample])
capsys.readouterr()  # Flush output from coverage run
make_whitelist(v, xml)
assert capsys.readouterr().out == expected_output.format(sample)
```

Hooray! `\o/` Thank You Florian! `:-)`


### How to pass around fixtures?

Observing the test case, it came to me that most of the code could be reused and
thereby I decided to write a helper function and soon came up with the following
implementation:

```python
def check_whitelist(code, expected_output, v, tmpdir, capsys):
    sample = os.path.normcase(str(tmpdir.join("unused_code.py")))
    xml = str(tmpdir.join("coverage.xml"))
    with open(sample, 'w') as f:
        f.write(code)
    subprocess.call(["coverage", "run", sample])
    subprocess.call(["coverage", "xml", "-o", xml])
    v.scavenge([sample])
    capsys.readouterr()  # Flush output from coverage run
    make_whitelist(v, xml)
    assert capsys.readouterr().out == expected_output.format(sample)


def test_create_whitelist(v, tmpdir, capsys):
    code = """\
class Greeter:
    def greet(self):
        print("Hi")
greeter = Greeter()
greet_func = getattr(greeter, "greet")
greet_func()
"""
    expected_output = """\
# {}
greet

"""
    check_whitelist(code, expected_output, v, tmpdir, capsys)
```

The problem with this implementation was that with every test case, all the
fixture objects (`v`, `tmpdir`, `capsys`, etc.) needed to be juggled around to
the helper function - there had to be a better way, I thought and started
digging things around but soon, I was down the rabbit hole.

Once again, I went to [The-Compiler][the-compiler] for rescue - He said that
this problem was not uncommon and pointed me to check out the following
solutions:
- Creating a class for test cases - Particularly helpful when having a lot of
test cases. eg.
[qutebrowser/qutebrowser/tests/unit/javascript/conftest.py][conftest]
- Defining an inner function inside a fixture, and then using that fixture
instead of calling the function directly. eg.
[qutebrowser/qutebrowser/tests/unit/browser/webkit/network/test_filescheme.py#L125-L153][inner-func]


Since the former option "looked" like a lot of work, I decided to go with the
inner function and quickly came up with the following:

```python
@pytest.fixture
def check_whitelist(code, expected_output):
    def check(v, tmpdir, capsys):
        sample = os.path.normcase(str(tmpdir.join("unused_code.py")))
        xml = str(tmpdir.join("coverage.xml"))
        with open(sample, 'w') as f:
            f.write(code)
        subprocess.call(["coverage", "run", sample])
        subprocess.call(["coverage", "xml", "-o", xml])
        v.scavenge([sample])
        capsys.readouterr()  # Flush output from coverage run
        make_whitelist(v, xml)
        assert capsys.readouterr().out == expected_output.format(sample)
    return check


def test_create_whitelist(check_whitelist):
    code = """\
class Greeter:
    def greet(self):
        print("Hi")
greeter = Greeter()
greet_func = getattr(greeter, "greet")
greet_func()
"""
    expected_output = """\
# {}
greet

"""
    check_whitelist(code, expected_output)
```

and soon the tests were green and I was busy finding a bottle of Champagne for
celebrations when I receive a message from [The-Compiler][the-compiler] saying
that it isn't supposed to work that way, did it do the job? and myself while
being drunk in the divine green flavours of [travis](https://travis-ci.org)
replied: "It did :-p". Next, he tells me that the test running successfully was
a false positive - Behind the hoods, it wasn't running at all. :-(

Due to the way, pytest's fixtures work, `check_whitelist` - the argument to
`test_create_whitelist` wasn't the function `check_whitelist` declared outside
and that it was a mere representation and when calling it from within the test,
pytest would intelligently call the inner function. Let's see the final code:

```python
@pytest.fixture
def check_whitelist(v, tmpdir, capsys):
    def check(code, expected_output):
        sample = os.path.normcase(str(tmpdir.join("unused_code.py")))
        xml = str(tmpdir.join("coverage.xml"))
        with open(sample, 'w') as f:
            f.write(code)
        subprocess.call(["coverage", "run", sample])
        subprocess.call(["coverage", "xml", "-o", xml])
        v.scavenge([sample])
        capsys.readouterr()  # Flush output from coverage run
        make_whitelist(v, xml)
        assert capsys.readouterr().out == expected_output.format(sample)
    return check


def test_create_whitelist(check_whitelist):
    code = """\
class Greeter:
    def greet(self):
        print("Hi")
greeter = Greeter()
greet_func = getattr(greeter, "greet")
greet_func()
"""
    expected_output = """\
# {}
greet

"""
    check_whitelist(code, expected_output)
```

Despite being deceptive at first (especially given the mismatch in the arguments
and parameters to `check_whitelist`), the resulting test case still provides a
neat and robust hack.

Again, Thank You so much Florian! :-)

### MS Windows

_As usual_, Microsoft Windows was being a pain in the ass - which in the end
(after 6 hours of swearing and 44 appveyor builds - I do not have access to a
Windows machine) was a result of non normalised (wrong slashes and case
sensitive) paths, _as usual_. :-p

At one point, I was thinking of dropping support for Windows, but thanks to
[Jendrik][jendrik] - I could finally find my way out.


[why]: https://rj722.github.io/blog/2018/05/19/why-use-coverage-to-find-which-python-code-is-run/
[coverage]: https://github.com/nedbat/coveragepy
[capsys]: https://docs.pytest.org/en/latest/capture.html#accessing-captured-output-from-a-test-function
[the-compiler]: https://github.com/the-compiler
[conftest]: https://github.com/qutebrowser/qutebrowser/blob/master/tests/unit/javascript/conftest.py
[inner-func]: https://github.com/qutebrowser/qutebrowser/blob/master/tests/unit/browser/webkit/network/test_filescheme.py#L125-L153
[jendrik]: https://github.com/jendrikseipp
