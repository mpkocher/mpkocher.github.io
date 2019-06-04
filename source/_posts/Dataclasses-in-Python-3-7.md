---
title: Dataclasses in Python 3.7
date: 2019-05-22 20:52:43
tags:
   - python
---

TDLR: Initially, I was interested in the `dataclasses` feature that was added to Python 3.7 standard library. However after a closer look, It's not clear to me if `dataclasses` provide enough value. Third-party libraries such as [attrs](https://github.com/python-attrs/attrs) and [pydantic](https://pydantic-docs.helpmanual.io) offer considerably more value. Moreover, by adding `dataclasses` to the standard library, Python now has 3 (or 4) overlapping mechanisms (namedtuple, `typing.NamedTuple`, "classic" classes, and dataclasses) for defining a core concept in the Python programming language. 


In Python 3.7, [dataclasses](https://docs.python.org/3/library/dataclasses.html) were added to the standard library. There's a [terrific talk by Raymond Hettinger](https://www.youtube.com/watch?v=T-TwcmT6Rcw) that is a great introduction to the history and design of `dataclasses`.

A demonstration and comparison of the 3 different models is shown below.

For the demo, I'll use a `Person` data model with an id (int), a name (str), and an optional favorite color (Optional[str]). The `Person` model will also have custom `eq` and `hash` using only the `Person`s `id`. I'm a fan of immutable classes, so the container data model will be `immutable` if possible.

Here's an example using the "classic" class style leveraging Python 3 type annotations.

Nothing particularly interesting for any Python dev. We do have to write a bit of boilerplate in the constructor, however, this also enables some convenience translation layers (e.g., converting datetime as a string to datetime instance). I've omitted the immutability aspect due to the boilerplate that would be added.


```python
from typing import Optional

class Person(object):
    def __init__(self, id:int, name:str, favorite_color:Optional[str] = None):
        self.id = id
        self.name = name
        self.favorite_color = favorite_color

    def __repr__(self):
        return "<{} id:{} name:{}>".format(self.__class__.__name__, self.id, self.name)

    def __hash__(self):
        return hash(self.id)

    def __eq__(self, other):
        if self.__class__ == other.__class__:
            return self.id == other.id
        return False
```

We can also write this using the `typing.NamedTuple` (or an untyped version using the Python 2 style `collections.namedtuple`).


```python
from typing import NamedTuple, Optional

class Person(NamedTuple):
    id:int
    name:str
    favorite_color:Optional[str] = None

    def __hash__(self):
        return hash(self.id)

    def __eq__(self, other):
        if self.__class__ == other.__class__:
            return self.id == other.id
        return False
```

Nothing too interesting here either. This approach does have well understood downsides due to the tuple nature of the underlying design. Specific downsides include index access and comparison operators (e.g., `__lte__`) which leverage the sortablity of tuples that can potentially introduce unexpected behaviors. 


Let's use the new Python 3.7 [dataclasses](https://docs.python.org/3/library/dataclasses.html) approach.


```python
from dataclasses import dataclass, field

@dataclass(frozen=True)
class Person:
    id:int
    name:str = field(hash=False, compare=False)
    favorite_color: Optional[str] = field(default_factory=lambda :None, hash=False, compare=False)
```

The `dataclasses` design is a declarative centric approach yielding a terse and clean interface to define a class in Python. 

It's important to note that **none of the 3 approaches in the standard lib support any mechanism for type checking at runtime**. The `dataclasses` API has a `__post_init` hook that can be used to add any custom validation (typechecking or otherwise).

I think it's useful to dig a bit deeper and understand the development requirements, or conversely, the non-goals of the `dataclasses` API. [PEP-557](https://www.python.org/dev/peps/pep-0557/#rationale) provides some key insights (for context, one of the libraries mention in the quote below is `attrs`).

Here's a few useful snippets from the PEP.

> One main design goal of Data Classes is to support static type checkers. The use of PEP 526 syntax is one example of this, but so is the design of the fields() function and the @dataclass decorator. Due to their very dynamic nature, some of the libraries mentioned above are difficult to use with static type checkers.

It's not clear to me if these "very dynamic" libraries still have an issue with popular static analysis tools such as `mypy` (as of `mypy` 0.57, `attrs` is supported). Nevertheless, it's very clear from the PEP that there is a strong opinion that Python is a dynamic language and adding types is to annotation functions with metadata for static analysis tools. Adding types is not for runtime type checking, nor is to imply that type signatures are required. 

Another useful snippet to provide context.

> Data Classes are not, and are not intended to be, a replacement mechanism for all of the above libraries. But being in the standard library will allow many of the simpler use cases to instead leverage Data Classes. Many of the libraries listed have different feature sets, and will of course continue to exist and prosper.

Historically, Python has a very heavy batteries-included approach to the standard library. To put the size of the standard library into perspective, there's a recent [PEP-594](https://www.python.org/dev/peps/pep-0594/) to remove "dead batteries" from the standard library. The list of packages to remove is quite interesting. This battery-centric included approach motivates questions about the standard library. 

- Does the Python standard library need to be any bigger? 

- Why can't (or shouldn't) `dataclasses` style or functionality be off-loaded to the community to develop libraries (e.g., `attrs` and `traitlets`)? Is a "half" or "minimal" solution really worth adding enough value?

- Does Python standard lib need yet another competing mechanism to define a class or data container? 

- Is all of these packages in the standard lib really that useful in practice? For example, is the [CSV parser](https://docs.python.org/3/library/csv.html) used when `pandas.read_csv` exists? 

- Do these features in the standard library bog down the Python core team?

A recent talk at Python Language Summit in May of 2019, "[Batteries included, But They're Leaking](http://pyfound.blogspot.com/2019/05/amber-brown-batteries-included-but.html)" by [Amber Brown](https://twitter.com/hawkieowl) brought up some contentious ideas on the current state of the standard library (in practice, that ship has sailed many moons ago and I'm not sure there's a lot of constructive discussion that can be had on this topic).

I don't really have any solid answers to any of these questions. Two of the most popular and Python libs, `requests` and `numpy` are both outside of the standard library (for good reasons that may be orthogonal to motivations of adding `dataclasses` to the standard library) and are thriving. 

Without hooks at a per `field` level validation hooks among other features, I'm struggling to find the usefulness of `dataclasses` for Python developers, particularly in the context of current polished third-party libraries, such as `attrs`, `pydantic`, or `traitlets`. 

For comparison, let's take a look at the third-party libraries that have inspired `dataclasses`.

# Third Party Libraries

Let's take a quick look into two of the third-party libraries, [attrs](https://github.com/python-attrs/attrs) and [pydantic](https://pydantic-docs.helpmanual.io) that inspired `dataclasses`. I believe both of these libraries are supported by `mypy`.

## Attrs

Similar to the `dataclasses` approach, the types in `attrs' `are only used as annotations (e.g., `__annotations__`) and are not enforced at runtime. However, the general validation mechanism trivially enables runtime type validation. For the purposes of this example, let's also add custom validation on the `name` of the `Person` as well as adding type checking.


```python
from typing import Optional

import attr
from attr.validators import instance_of as vof


def is_positive_nonzero(instance, attribute, value):
    if value < 1:
        raise ValueError(f"Value {value} must be >0")


def check_non_empty_str(inst, attribute, value):
    if not value:
        raise ValueError("name must be a non-empty string")


@attr.s(auto_attribs=True, frozen=True)
class Person:
    id:int = attr.ib(validator=[vof(int), is_positive_nonzero])
    name:str = attr.ib(validator=[vof(str), check_non_empty_str])
    favorite_color: Optional[str] = attr.ib(default=None, validator=[vof((str, type(None)))])

    def __hash__(self):
        return hash(self.id)

    def __eq__(self, other):
        if self.__class__ == other.__class__:
            return self.id == other.id
        return False
```

The [abstract of Raymond Hettinger's talk from Pycon](https://www.youtube.com/watch?v=T-TwcmT6Rcw) has an interesting summary of `dataclasses`.

>Dataclasses are shown to be the next step in a progression of data aggregation tools: tuple, dict, simple class, bunch recipe, named tuples, records, attrs, and then dataclasses. Each builds upon the one that came before, adding expressiveness at the expense of complexity.

I'm not sure I completely agree. The `dataclasses` implementation looks closer to `attrs`-lite, than the "next step of progression". 

## Pydantic

Another alternative is [pydantic](https://pydantic-docs.helpmanual.io). This is a bit more opinionated design. It also has a nice `Schema` abstraction to communicate core metadata on the fields as well as first class support for serialization hooks. The `pydantic` library also has a `dataclasses` wrapper layer that can be accessed via `pydantic.dataclasses`.

Here's an example of defining our `Person` data model.

```python
from typing import Optional
import pydantic
from pydantic import BaseModel, validator, PositiveInt


class Person(BaseModel):
    class Config:
        allow_mutation = False
        validate_all = True 

    id:PositiveInt
    name:str
    favorite_color: Optional[str] = None

    @validator('name')
    def validate_name(cls, v):
        if not v:
            raise ValueError("Name can't be an empty")
        return v

    def __hash__(self):
        return hash(self.id)

    def __eq__(self, other):
        if self.__class__ == other.__class__:
            return self.id == other.id
        return False
```

Overall, I like the general style and approach, however, it does have a few quarks. Specifically the keyword only usage as well as unexpected casting behavior of `int`s to `str`s.

The `pydantic` API also supports rich metadata that could be useful for generating commandline interfaces for a given schema data model and emitted JSONSchema.


```python
from pydantic import BaseModel, validator, ValidationError, Schema

class Person(BaseModel):
    class Config:
        allow_mutation = False

    # '...' means the value is required.
    id: PositiveInt = Schema(..., title="User Id")
    name:str = Schema(..., title="User name")
    favorite_color:Optional[str] = Schema(None,
                                          title="User Favorite Color",
                                          description="Favorite Color. Provided as case sensitive english spelling")

     @validator('name')
    def validate_name(cls, v):
        if not v:
            raise ValueError("Name can't be an empty")
        return v

    def __hash__(self):
        return hash(self.id)

    def __eq__(self, other):
        if self.__class__ == other.__class__:
            return self.id == other.id
        return False

#Person.schema() will emit the JSONSchema as a dict.
```


# Summary And Conclusion

- `dataclasses` offers a terse syntax for defining a class or data container that has type annotations using a code generation approach
- `dataclasses` `field` metadata can be used to define defaults, communicate which fields should be used in `eq` or `hash`, `lte`, etc...
- `dataclasses` has a `__post_init` hook that can be used for validation
- `dataclasses` By design does not do type validation. It only adds `__annotation__` to the data container for static analyzers to consume, such as `mypy`
- Since `dataclasses` is now in the standard lib, this means feature enhancement, bug fixes, and backwards compatibility are now coupled the official Python release process
- [Raymond's Pycon](https://www.youtube.com/watch?v=T-TwcmT6Rcw) talk mentions the end-to-end develop time on `dataclasses` was 200+ hrs

Initially, I was a intrigued by the addition of `dataclasses` to the standard library. However, after a deeper dive into the `dataclasses`, it's not clear to me that these are particularly useful for Python developers. I believe third-party solutions such as `attrs` or `pydantic` might be a better fit due to their validation hooks and richer feature sets. It will be interesting to see the adoption of `dataclasses` by both the Python core as well as third-party developers.

For a deeper look and comparison of the 3 (or 4) models to define a class or data container in Python, please consult this [Notebook in this Gist](https://gist.github.com/mpkocher/4482e9315c64241f442b1e5f8783316d)

Best on all your Python-ing!







