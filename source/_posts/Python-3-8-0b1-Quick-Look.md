---
title: Python 3.8.0b1 Quick Look
date: 2019-06-06 18:04:36
tags:
  - python
---

On June 4th 2019, [Python 3.8.0b1 was released](https://github.com/python/cpython/releases/tag/v3.8.0b1). The official [changelog is here](https://docs.python.org/3.8/whatsnew/changelog.html#changelog). 

There are two interesting syntactic changes/features that were added which I believe are useful to explore in a bit of depth. Specifically, the new "walrus"' `:=` operator and the new Positional-Only function parameter features.

## Walrus

First, the "walrus" expression operator (`:=`) defined in [PEP-572](https://www.python.org/dev/peps/pep-0572/)

>...naming sub-parts of a large expression can assist an interactive debugger, providing useful display hooks and partial results. Without a way to capture sub-expressions inline, this would require refactoring of the original code; with assignment expressions, this merely requires the insertion of a few name := markers. Removing the need to refactor reduces the likelihood that the code be inadvertently changed as part of debugging (a common cause of Heisenbugs), and is easier to dictate to another programmer.

A (contrived) example using Python 3.8.0b1 built from source "3.8.0b1+ (heads/3.8:23f41a64ea)"


```python
>>> xs = list(range(0, 10))
>>> xs
[0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
>>> [(x, y) for x in xs if (y := x * 2) < 5]
[(0, 0), (1, 2), (2, 4)]
```

The idea is to use an expression based approach to remove unnecessary chatter and potential bugs of storing local state.

Another simple example:

```python
>>> # Python 3.8
>>> import re
>>> rx = re.compile(r'([A-z]*)\.([A-z]*)')
>>> def g(first, last): return f"First: {first} Last: {last}"
>>> names = ['ralph', 'steve.smith']
>>> for name in names:
...     if (match := rx.match(name)): print(g(*match.groups()))
First: steve Last: smith
```

As a side note, many of these "None-ish" based examples in the PEP (somewhat mechanically) look like a `map`, `flatMap`, `'foreach'` on `Option[T]` cases in Scala. 

Python doesn't really do this well due to its inside out nature of composing maps/filter/generators (versus a left to right model). Nevertheless, here's the example. 

```python
>>> def processor(sx): return rx.match(sx)
>>> def not_none(x): return x is not None
>>> def printer(x): print(g(*x.groups()))
>>> _ = list(map(printer, filter(not_none, map(processor, names))))
First: steve Last: smith
```

The "Exceptional cases" are [worth investigating in more detail](https://www.python.org/dev/peps/pep-0572/#exceptional-cases). There's several cases where "Valid, though probably confusing" is used.

For example:

```python
y := f(x)  # INVALID
(y := f(x))  # Valid, though not recommended
```

The "walrus" operator can also be used in function definitions. 

```python
def foo(answer = p := 42): return "" # INVALID
def foo(answer=(p := 42)):  return "" # Valid, though not great style
```


## Positional Only Args

The other interesting feature added to Python 3.8 is Positional-Only arguments in function definitions.

For as long as I can recall, Python has had this fundamental feature (or bug) on how functions or methods are called. 

For example, 

```python
def f(x, y=1234): return x + y
>>> f(1)
1235
>>> f(1, 2)
3
>>> f(x=1, y=2)
3
>>> f(y=1, x=2)
3
```

Often this isn't really a big deal, however, it can leak local variable names as part of the public interface. Small renaming, can break interfaces. It's also not clear what should be a keyword only argument or a positional only argument with a default. For example, simply changing `f(x, y=1234)` to `f(n, y=1234)` can potentially break the interface depending on the call "style".

I've worked with a few developers over the years that viewed this as a feature and thought that this style made the API calls more explicit. For example:

```python
def compute(alpha, beta, gamma):
    return 0

compute(alpha=90, gamma=80, beta=120)
```

I never really liked this looseness of positional vs keyword and would try to discourage it's use. Regardless, it can be argued this is a feature of the language (at least in <= 3.7). I would guess that many Python developers are also leveraging the unpacking dict style as well.

```python
d = dict(alpha=90, gamma=70, beta=120)
compute(**d)
```

In Python 3, function definitions using Keyword-Only was added (see [PEP-3102](https://www.python.org/dev/peps/pep-3102/) from 2006) using the `*`. All arguments to the right of the `*` are Keyword-Only arguments. 


```python
def f(a, b, *, c=1234):
    return a + b + c
```

However, this still leaves a fundamental issue with clearly defining args. There are three cases: Positional-Only, Positional or Keyword, and Keyword-Only. PEP-3102 only solves the Keyword-Only case and doesn't address the other two cases. 

Hence in Python < 3.8, there's still a fundamental ambiguity when defining a function and how it can be called (I'll leave it up to the reader to decide if this is a bug or a feature).

For example:

```python

def f(a, b=1, *, c=1234):
    return a + b + c

f(1, 2, c=3)
f(a=1, b=2, c=3)

```


Starting with Python 3.8.0, a Positional-Only parameters mechanism was added. The details are captured in [PEP-570](https://www.python.org/dev/peps/pep-0570/)

Similar to the `*` delimiter in Python 3.0.0 for Keyword-Only args), a `/` delimiter was added to clearly delineate Positional-Only (or conversely Keyword-Only args) in function of method definitions. This makes the three cases of function arguments unambigious in how they should be called.

Here's a few examples:


```python
def f0(a, b, c=1234, /):
    # a, b, c are only positional args. 
    return a + b + c

def f1(a, b, /, c=1234):
    # a, b must be a positional,
    # b can be positional or keyword
    return a + b + c

def f2(a, b, *, c=1234):
    # a, b can be keyword or positional
    # but c MUST be keyword
    return a + b + c

def f3(a, b, /, *, c=1234):
    # a, b only positional
    # c only keyword
    # e.g., # f3(1, 2, c=3)
    return a + b + c
```

Combining the `/` and `*` with the type annotations yeilds:

```python
def f4(a:int, b:int, /, *, c:int=1234):
    # this can only be called as
    # f4(1, 2, c=3)
    return a + b + c

```

We can dive a bit deeper and inspect the function signature via [inspect](https://docs.python.org/3.8/library/inspect.html).

```python
import inspect
def extractor(f):
    sx = inspect.signature(f)
    return [(v.name, v.kind, v.default) for v in sx.parameters.values()]
```

Let's inspect each example:

```python
def pf(f):
    print(f"Function {f.__name__} with type annotations {f.__annotations__}")
    print(extractor(f))
```

Running in the REPL yeilds:

```python
>>> funcs = (f0, f1, f2, f2, f2, f4)
>>> _ = list(map(pf, funcs))
Function f0 with type annotations {}
[('a', <_ParameterKind.POSITIONAL_ONLY: 0>, <class 'inspect._empty'>), ('b', <_ParameterKind.POSITIONAL_ONLY: 0>, <class 'inspect._empty'>), ('c', <_ParameterKind.POSITIONAL_ONLY: 0>, 1234)]
Function f1 with type annotations {}
[('a', <_ParameterKind.POSITIONAL_ONLY: 0>, <class 'inspect._empty'>), ('b', <_ParameterKind.POSITIONAL_ONLY: 0>, <class 'inspect._empty'>), ('c', <_ParameterKind.POSITIONAL_OR_KEYWORD: 1>, 1234)]
Function f2 with type annotations {}
[('a', <_ParameterKind.POSITIONAL_OR_KEYWORD: 1>, <class 'inspect._empty'>), ('b', <_ParameterKind.POSITIONAL_OR_KEYWORD: 1>, <class 'inspect._empty'>), ('c', <_ParameterKind.KEYWORD_ONLY: 3>, 1234)]
Function f2 with type annotations {}
[('a', <_ParameterKind.POSITIONAL_OR_KEYWORD: 1>, <class 'inspect._empty'>), ('b', <_ParameterKind.POSITIONAL_OR_KEYWORD: 1>, <class 'inspect._empty'>), ('c', <_ParameterKind.KEYWORD_ONLY: 3>, 1234)]
Function f2 with type annotations {}
[('a', <_ParameterKind.POSITIONAL_OR_KEYWORD: 1>, <class 'inspect._empty'>), ('b', <_ParameterKind.POSITIONAL_OR_KEYWORD: 1>, <class 'inspect._empty'>), ('c', <_ParameterKind.KEYWORD_ONLY: 3>, 1234)]
Function f4 with type annotations {'a': <class 'int'>, 'b': <class 'int'>, 'c': <class 'int'>}
[('a', <_ParameterKind.POSITIONAL_ONLY: 0>, <class 'inspect._empty'>), ('b', <_ParameterKind.POSITIONAL_ONLY: 0>, <class 'inspect._empty'>), ('c', <_ParameterKind.KEYWORD_ONLY: 3>, 1234)]
```

Note, you can use [bind](https://docs.python.org/3.8/library/inspect.html#inspect.Signature.bind) to call the func (this must also adhere to the correct function definition of arg and keywords in the function of interest).

Also, it's worth noting that both scipy and numpy have been using this `/` [style in the docs](https://docs.scipy.org/doc/scipy/reference/generated/scipy.special.gammaln.html) for some time. 


## When Should I Start Adopting these Features?

If you're a library developer that has packages on [PyPi](https://pypi.org), it might not be clear when it's "safe" to start leveraging these features. I was only able to find one source of Python 3 adoption and as a result, I'm only able outline a **very crude model**.

On December 23, 2016, Python 3.6 was officially released. In the Fall of 2018, [JetBrains release the Python Developer Survey](https://www.jetbrains.com/research/python-developers-survey-2018/#python-3-adoption) which contains the Python 2/3 breakdown, as well as the breakdown of different versions within Python 3. As of Fall 2018, 54% of Python 3 developers were using Python 3.6.x. Therefore, using this very crude model, if you assume that the rate of adoption of 3.6 and 3.8 are the same and if the minimum threshold of adoption of 3.8 is 54%, then you'll need to wait approximately 2 years before starting to leverage these 3.8 features. 

{% asset_img "jetbrains_python_survey_2018.png" %}

When you do plan to leverage these 3.8 specific features and pushing a package to the Python Package Index, then I would suggest clearly defining the Python version in the `setup.py`. For more details, see the official packaging [docs](python_requires='>3.5.2') for more details.

```python
# setup.py
setup(python_requires='>=3.8')
```

# Summary and Conclusion 

- Python 3.8 added the "walrus" operator `:=` that enables results of expressions to be used
- It's recommended reading the [Exceptional Cases](https://www.python.org/dev/peps/pep-0572/#exceptional-cases) for better understanding of where to (and to not) use the `:=` operator.
- Python 3.8 added Positional-Only function definitions using the `/` delimeter
- There are changes in the standard lib to communicate. It's not clear how locked down or backward compatible the interfaces were changes. Here's a random example of the function signature of [functools.partial](https://docs.python.org/3.8/library/functools.html#functools.partial) being updated to use `/`.
- Migrating to 3.8 might involve potentially breaking API changes based on the usage of `/` in the Python 3.8 standard lib
- For core library authors of libs on pypi, I would recommend  using the crude approximation (described above) of approximately 2 years away from being able to adopt the new 3.8 features
- For `mypy` users, you might want to make sure you investigate the supported versions of Python 3.8 ([More Details](https://github.com/python/mypy/issues/6545) on the compatiblity matrix)

I understand the general motivation to solve core friction points or ambiguities at the language level, however, the new syntatic changes are a little too noisy for my tastes. Specifically the Positional-Only `/` combined with the `*` and type annotations. Regardless, the (Python 3.8) ship has sailed long ago. I would encourage the Python community to periodically track and provide feedback on the current [PEP](https://www.python.org/dev/peps/)s to help guide the evolution of the Python programming language. And finally, Python 3.8.0 (beta and future 3.8 RCs) bugs should be filed to https://bugs.python.org . 


Best to you and your Python-ing!

## Further Reading

- [Full Python 3.8.0b1](https://www.python.org/downloads/release/python-380b1/) release notes
- [Hacker News](https://news.ycombinator.com/item?id=19690152) discussion on Positional-Only feature
- [Alexander Hutner](https://medium.com/hultner/try-out-walrus-operator-in-python-3-8-d030ce0ce601) covering walrus feature
- [Bug Tracker](https://bugs.python.org)
- [Python 3.8 ChangeLog](https://docs.python.org/3.8/whatsnew/changelog.html#changelog)
- [Python PEP](https://www.python.org/dev/peps/)s


P.S. A reminder that the PSF has a [Q2 2019 fundraiser](https://www.python.org/psf/donations/2019-q2-drive/) that ends June 30th.
