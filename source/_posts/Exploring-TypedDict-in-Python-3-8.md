---
title: Exploring TypedDict in Python 3.8
date: 2019-09-19 22:13:08
tags:
  - python
  - mypy
---

This post will explore the new `TypedDict` feature in Python and explore leveraging `TypedDict` combined with the static analysis tool [`mypy`](https://github.com/python/mypy) to improve the robustness of your Python code.

## PEP-589

`TypedDict` was proposed in [PEP-589](https://www.python.org/dev/peps/pep-0589/) and accepted in Python 3.8.

A few key quotes from PEP-589 can provide context and motivation for the problem that `TypedDict` is attempting to address.

> This PEP proposes a type constructor typing.TypedDict to support the use case where a dictionary object has a specific set of string keys, each with a value of a specific type.

> More generally, representing pure data objects using only Python primitive types such as dictionaries, strings and lists has had certain appeal. They are are easy to serialize and deserialize even when not using JSON. They trivially support various useful operations with no extra effort, including pretty-printing (through str() and the pprint module), iteration, and equality comparisons.

This particular section of the PEP is interesting and suggests that `TypedDict` can be particularly useful for retrofitting legacy code (with type annotations). 

> Dataclasses are a more recent alternative to solve this use case, but there is still a lot of existing code that was written before dataclasses became available, especially in large existing codebases where type hinting and checking has proven to be helpful. Unlike dictionary objects, dataclasses don't directly support JSON serialization, though there is a third-party package that implements it

The reference implementation was defined in [mypy_extensions](https://github.com/python/mypy_extensions) and can be installed in Python 3.7 (e.g., `pip install mypy_extensions`), or using `typing.TypedDict` in Python 3.8. 

These following examples are run with `mypy` 0.711 and examples shown below can be obtained from [this gist](https://gist.github.com/mpkocher/e123aa0613242715717d291f9df7afdd).


## Motivation: Dictionary-Mania

Here's a common example where a type-checking tool (e.g., `mypy`) won't be able to help you catch type errors in your code.

```python
def example_0() -> int:
    """Simple Example of Using raw dict and how mypy won't catch
    these errors with the keys
    """

    m = dict(name='Star Wars', year=1234)

    # mypy will NOT catch this error
    t = m['name'] + 100
```

However, with `TypedDict`, you can define this a structural-typing-ish interface to `dict` for a specific data model.

Using Python < 3.8 will require `from mypy_extensions import TypedDict` whereas, Python >= 3.8 will require `from typing import TypedDict`.

Let's create a simple `Movie` data model example and explore how `mypy` can be used to help catch type errors.

## Example 1: Basic Usage of TypedDict

```python
class Movie(TypedDict):
    name: str
    year: int


def example_01():
    m = Movie(name='Star Wars', year=1977)
    # or
    m2:Movie = dict(name='Star Wars', year=1977)

    # the type checker will catch this
    n = m['name'] + 100
```


To enable runnable code that **purposely has errors** that can be caught by `mypy`, let's define a helper function to require a specific `Exception` type to be raised. 

```python
import logging
log = logging.getLogger(__name__)


def log_expected_error(ex, fx):
    raised_error = False
    try:
        return fx()
    except ex as e:
        raised_error = True
        log.info(f"Got Expected error `{e}` of type {ex} from {fx.__name__}")
    finally:
        if not raised_error:
            log.error(f"Expected {fx} to raise {ex}")

```

## Example 2: Exploring Mutating and Assignment of TypedDicts

Let's mutate the `Movie` `TypedDict` instance and explore how `mypy` can catch type errors during assignment.

```python
def example_02() -> int:
    m = Movie(name='Star Wars', year=1977)
    log.info(m)

    # mypy will catch this
    m['name'] = 11111

    def f() -> int:
        m['year'] = m['year'] + 'asfdsasdf'
        return 0

    log_expected_error(TypeError, f)

    # Use dict methods to mutate
    # Note, current verison of mypy is confused
    # by this and generates `"Movie" has no attribute "clear"`
    m.clear()

    def f2() -> int:
        # mypy won't catch this KeyError from .clear()
        return m['year'] + 100

    log_expected_error(KeyError, f2)

    # Can we mix Movie and a raw dict?
    d2 = dict(extras=True, alpha=1234, name=12345, year='1978')

    # mypy will raise TypeError here
    m.update(d2)

    log.info(m)

    # Update a Movie with a Movie
    m2 = Movie(name='Star Wars', year=1977)
    new_m = Movie(name='Movie Title', year=1234)

    # Both of these are proper Movie TypedDict
    # hence, no mypy type error
    m.update(new_m)
    log.info(m2)
```


There's a few interesting items to note.

- `mypy` will catch assignment errors 
- The current version of `mypy` will get a bit confused with `dict` methods, such as `.clear()`. Moreover, `.clear()` will also yield `KeyError`s (related, see `total=False` keyword of the `TypedDict`)
- `mypy` will only allow merging dicts that are the same type. You can't mix `TypedDict` and a raw dict without `mypy` raising an issue


## Example #3: TypedDicts total Keyword Argument

There's a `total` keyword to the `TypedDict` that communicates that the dict does not need to be completely well formed. This is particularly interesting in how the `mypy` interpets the types. 

For example, `X` with `alpha`, `beta` and `gamma` as `int`s, will be

```python
class X(TypedDict, total=False):
    alpha:int
    beta:int
    gamma:int

x:X = dict()
x['alpha'] = 1
x['beta'] = 2
x['gamma'] = 3
```

Lets dive deeper using a variation of the previously defined `Movie` example using `total=False` to explore how `mypy` interprets the 'incomplete' data model.


```python
class Movie2(TypedDict, total=False):
    name:str
    year:int
    release_year: Optional[int]


def example_03() -> int:
    """
    Explore with defining an 'incomplete' Movie data model and how
    None/Null checking works with mypy
    """

    m = Movie2(name='Star Wars')
    log.info(m)

    def f() -> int:
        # mypy will catch this
        m['name'] = 1234
        return 0

    # Use dict methods to mutate
    # mypy is confused by this. The error is:
    # `"Movie" has no attribute "clear"`
    m.clear()

    def f2() -> int:
        # mypy doesn't catch this NPE
        # I don't think it treats the type
        # as Optional[int]
        m['year'] = m['year'] + 100
        return 0

    log_expected_error(KeyError, f2)

    # Explicit test with release_year which
    # is fundamentally Optional[int]

    def f3() -> int:
        # mypy WILL catch this NPE
        m['release_year'] = m['release_year'] + 100
        return 0

    log_expected_error(KeyError, f3)


    # This works as expected and
    m2 = Movie2(name='Star Wars', release_year=2049)

    # This works as expected and mypy won't raise an error
    if m2['release_year'] is not None:
        t = m2['release_year'] + 10

```


Finally, let's explore how `isinstance` works with `TypedDict`

## Example 4: TypedDict and isinstance

```python
def example_04() -> int:
    """Testing isinstance"""

    m = Movie(name='Movie', year=1234)


    def f() -> int:
        is_true = isinstance(m, dict)
        return 0


    # This is a bit unexpected that this
    # will raise an exception at runtime
    # ` Cannot use isinstance() with a TypedDict type`
    def f2() -> int:
        is_true = isinstance(m, Movie)
        return 0

    log_expected_error(TypeError, f2)
```

The important item to note here is that you can NOT use `isinstance` with `TypedDict`. Python will raise a runtime error of `TypeError`. Specifically the error you'll see is show below.

```
TypeError: TypedDict does not support instance and class checks
```

# Summary

- `TypedDict` + `mypy` can be valuable to help catch type errors in Python and can help with heavy dictionary-centric interfaces in your application or library
- `TypedDict` can be used in Python 3.7 using `mypy_extensions` package
- `TypedDict` can be used in Python 2.7 using `mypy_extensions` and the 2.7 'namedtuple-esque' syntax style (e.g., `Movie = TypedDict('Movie', {'title':str, year:int})`)
- Using the `total=False` keyword to `TypedDict` can introduce large wholes the static typechecking process yielding `KeyError`s. The keyword `total=False` should be used judiciously (if at all)
- `isinstance` should **not** be used with `TypedDict` as it will raise a runtime `TypeError` exception
- Be mindful when using `TypeDict` methods such as `clear()`
- `TypeDict` introduces a new (somewhat) competing data modeling alternative to [dataclasses](https://docs.python.org/3/library/dataclasses.html), [typing.NamedTuple](https://docs.python.org/3/library/typing.html#typing.NamedTuple), "classic" classes and third-party libraries, such as [pydantic](https://github.com/samuelcolvin/pydantic) and [attrs](https://github.com/python-attrs/attrs). It's not completely clear to me how all these different competing data model abstractions models are going to age gracefully


I believe `TypedDict` can be a value tool to help improve clarity of interfaces, specifically in legacy code that is a bit **dictionary-mania** heavy. However, for new code, I would suggest avoid using `TypedDict` in favor of the thin data models, such as [pydantic](https://github.com/samuelcolvin/pydantic) and [attrs](https://github.com/python-attrs/attrs). 

Best to you and your Python-ing. 

P.S. A runnable form of the code used in the post can be found in this [gist](https://gist.github.com/mpkocher/e123aa0613242715717d291f9df7afdd).
