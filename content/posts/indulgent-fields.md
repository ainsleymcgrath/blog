---
title: "Indulgent Use of Dataclass Fields"
date: 2022-10-10T08:23:15-05:00
---

## Dataclasses

[Dataclasses](https://docs.python.org/3/library/dataclasses.html) are one of the best (semi) recent Python features. Maybe one of the best features of Python overall.

Dataclasses allow for the *specification* of fields and arguments with a compact syntax.

```python
class Sandwich:
    bread: str
    inches: int = 12
```

Emphasis on *specification* because above, a `str` type hint is used to *specify* that `bread` is a required attribute and a required `__init__` argument for `Sandwich`. Declaring `inches` with a hint of `int` *specifies* that that attribute is optional and has a default.

Type hints and default values are the two main "field specifiers" in the dataclasses library.

**Let's explore some maybe-not-obvious ways to use those.**

But first, background:

## Dataclasses are a a formalism

With the introduction of [PEP 681](https://peps.python.org/pep-0681/), Python formalized the idea of dataclass-ness.

`builtins.dataclass` is not the first or only means of creating Python types with convenient hint-based syntax. Before it came [attrs](https://github.com/python-attrs/attrs) and [SQLAlchemy](https://www.sqlalchemy.org). [Pydantic](https://pydantic-docs.helpmanual.io) and others not far behind.

PEP 681 acknowledges that this is a pattern that editors and type-checkers must know about. And all of the above means of dataclass creation come with ways to specify fields *beyond* standard hints and defaults.

The "pattern" basically says:

- The class fields of a dataclass-ish type are used to generate an `__init__` 
- **Those fields have their behavior modified in a number of ways.**

## `builtins.dataclasses.field`

This [factory function](https://docs.python.org/3/library/dataclasses.html#dataclasses.field) produces [`Field` objects](https://docs.python.org/3/library/dataclasses.html#dataclasses.Field), which can be used to specify more fine-grained behaviors of fields and defaults.

**`field` is the original "field specifier"** which PEP 681 defines as:

> *[a tool to] describe attributes of individual fields that a static type checker must be aware of, such as whether a default value is provided for the field.*

Commonly used ones include:

- `init=False` tells Python that a field should not be included in the generated `__init__` for a dataclass, but it should remain a public attribute.
- `default` tells Python the default value of a field in an alternative way.
- `default_factory` takes a [nullary](https://en.wiktionary.org/wiki/nullary) function for *lazily*  comupuiting a default at initialization.

## Recipe 1: Getting defaults from external resources

I'll use two simple examples of "external resources": environment variables and config files.

### Default values from the environment

I've written this many times as a way to replicate the behavior of Pydantic's splendid `BaseSettings` [feature](https://pydantic-docs.helpmanual.io/usage/settings/) in projects where I don't need that entire library.

**Write a function that can take an env var name and produce a `Field` with a `default_factory` that will load it.** 

```python
import os
from dataclasses import field


def env_field_specifier(var_name: str):
    
    def default_factory():
        return os.environ[var_name]
    
    return field(default_factory=default_factory)    
```

Use it like so:

```python
from dataclasses import dataclass


@dataclass
class Settings:
    token: str = env_field_specifier("AUTH_TOKEN")
    

settings = Settings()  # default gets read from env
Settings(token='123abc')  # or pass as normal
```

### Eagerly parse a file

It's good practice to deserialize things as quickly as possible. JSON configurations are very common, so turning them into complex objects is a good way to document them and improve ergonomics.

This object can represent some JSON config:

```python
from dataclasses import dataclass


@dataclass
class JsonConfig:
    version: str
```

And we can have a specific field specifier responsible for creating it:

```python
import json
from dataclasses import field
from pathlib import Path


def json_config_field_specifier(path):
    with Path(str).open() as file:
	    content = json.load(file)
    return field(default_factory=lambda: JsonConfig(**content))
```

Now, there's a place for all JSON configs:

```python
@dataclass
class JsonConfigRepo:
    config: JsonConfig = json_config_field_specifier('~/config.json')
```

## Recipe 2: More inspectable classes

`dataclasses` comes with a set of utility functions. [`fields()`](https://docs.python.org/3/library/dataclasses.html?highlight=dataclasses#dataclasses.field) can be particularly useful.

The `fields` function must be called on a dataclass instance or type. It raises a `TypeError` otherwise. 

It returns a list of `Field` objects, which have access to things like:

- `name`, the property name the `Field` was assigned to
- `default_factory`, the function passed (or not passed) to `field`
- `metadata`, which I'll use in this example

Leveraging field descriptors along with `fields` can help to create powerful utilities for working with your dataclasses.

### Mark classes for special treatment

It's common to use dataclasses as [passive data structures](https://pythonbakery.com/bites/passive-data-patisserie/) or record objects.

In the case of record objects, maybe you export to [msgpack](https://msgpack.org) using [`msgpack-python`](https://github.com/msgpack/msgpack-python).

```python
from dataclasses import asdict
import msgpack


def dataclass_msgpack(record):
    data = asdict(record)
    return msgpack.packb(data)
```

It would be useful to exclude certain properties when serializing to msgpack, maybe to conserve size.

```python
from dataclasses import dataclass


@dataclass
class Record:
    id: str
    very_long_internal_id: str  # would love to exlude this
```

This can be accomplished with a field descriptor that passes `metadata` for `dataclass_msgpack` to inspect.

```python
from dataclasses import dataclass


def no_msgpack(**kwargs):
    # downstream, only check for the presence of the key
    kwargs["metadata"] = {"no_msgpack": object()}
    return field(**kwargs)
```

Change the `Record` class to use the descriptor.

```python
@dataclass
class Record:
    id: str
    very_long_internal_id: str  = no_msgpack()
```

And augment the `dataclass_msgpack` function to use the new metadata.

```python
from dataclasses import asdict, fields
import msgpack


def dataclass_msgpack(record):
    # inspect the `Field` objects' metadata
	exclude_keys = {f.name for f in fields(record) if 'no_msgpack' in f.metadata}
    data = {k: v for k, v in asdict(record).items() if k not in exclude_keys}
    return msgpack.packb(data)
```

Now any dataclass implementation that may end up getting serialized can easily exclude attributes.

Any time you're acting externally on dataclass instances, `fields` and `Field.metadata` can be useful.

## More to learn

The dataclasses [module](https://docs.python.org/3/library/dataclasses.html?highlight=dataclasses#module-dataclasses) is not particularly large, but it does contain a fantastic amount of functionality.

By embracing the `dataclasses` fully, many problems can be solved concisely and elegantly, sometimes in surprising ways.

