---
title: "Dependency_injection_without_being_weird_about_it"
date: 2023-10-16T14:01:29-05:00
draft: true
---

## The Mysterious Promise of Dependency Injection

For some reason, there are books about dependency injection. I've read some! And scratched my head as I failed repeatedly to Really Get It.

Authors promise some aha-moment that takes you straight to a software Valhalla of noise-free ultra-lucid code; Bugless and irreproachable. But I always feel let down once I'm handed  a libary or [shudders] some kind of XML config.

Even LLMs can't put it concisely. But I was able to extract a definition I'm willing to work off from Perplexity:

> [Dependency injection] is the process of supplying a resource that a given piece of code requires

Now we're getting somewhere. Let's run with that.

## My Extremely Simple (Python-Centric) Definition

When it comes to Python, I tweak the verbiage a bit:

> Dependency injection in Python is using types to write advantageous signatures.

That's it. Write functions, methods, and constructors that tell maintainers a lot and make their work easier.

But what does that mean in practice? Let's start with the signature bit.

## tl;dr: Info, Params, and somtimes Resource

Don't think about dependency injection frameworks or the different forms. Don't even worry about the "injection site". Just reach for these three words and expect to pass them all around your application:

1. `Info`
2. `Params`
4. *and sometimes* `Resource` *–but usually a synonym*

Be suspicious when you *don't* see them, and begin reaping the benefits of DI.

You "see" interfaces with type hints so, **make sure you're annotating every signature,** or else you're not getting the ergonomic benefits of types.

| The Object | Its Usual Purpose                                            |
| ---------- | ------------------------------------------------------------ |
| `Info`     | A record and, frequently, some metadata about it.<br />Use it as a go-to *return value* or *rightmost* argument. |
| `Params`   | Input from outside the application. Like an HTTP request or command line input.<br />Use it as a go-to for your *rightmost* arguments. |
| `Resource` | An interface to a Database, External HTTP Service, File, Queue, or anything "remote".<br />Use these as your *leftmost* arguments. |

## The Anti-Pattern

When it comes to business logic, it's easier to think about what is *not* advantageous.

Let's take this [pseudocode] example of a widely-used function in a social app for bakers.

```python
def get_baker_id(email, password):
    """Provide the numeric ID for some baker."""
    query = f"""
        select id from bakers
        where email = {email} and password = {password};"""
    record = db.exec(query).one()
    return record.id


# many elsewheres...
baker_id = get_baker_id('foo@bar.baz', 'password123')
```

### Clarifying with Types

Since this is shipped, system-critical code, it's probably typed.

```python
def get_baker_id(email: str, password: str) -> int: ...
```

This is the function of signature you're forced to annotate and think *Ugh why have we taken on this chore of adding types to our fun dynamic language?* The typing doesn't feel worth the exercise.

Turns out typing functions like this feels bad because functions like this are bad.

**The issue is that this function and code similar to it is a signature of exclusively primitives.** A second, subtler issue is that `db` is completely absent.

### Primitives do not extend

Answer these questions and quickly understand why `get_baker_id` will extend poorly if it's the primary mechanism for getting the business entity of "Baker."

1. What if you want to support other ways for bakers to identify themselves, such as with a phone number or OTP?
2. How do you feel about altering the implementation of `db`? (Which, is not even clearly a part of this function.)
3. Over time, you realize you *always* need a baker's avatar along with their ID. How should that be implemented?
4. We need another remote resource (in addition to `db`)––let's say a `cache`. Where do we get that?

The answer to all of these is *Spaghetti*.

And the reason is that we are dealing with our *business domain* by handing around  *a trio of primitives.* In an object oriented language? With an increasingly-capable type system? WHY?!?

## Advantageous Python Function Signatures

A dependency-injection-ish version would look like this

```python
def get_baker(db, params):
    """Provide the representation of some baker."""
    query = f"""
        select id from bakers
        where email = {params.email} and password = {params.password};"""
    record = db.exec(query).one()
    return BakerInfo(id=record.id)
```

Now, rather than dealing with scalar primitives...

- we wrap our data dependencies (`email` and `password`) in an object which is *by definition inherently extensible.* 
- we wrap `id` in `params`––now there's room for something like `display_name`.
- we bring `db` in so this function can be portable and fully documented in its signature.

### Clarifying with Types

It's hard to see the benefit of the refactor just yet. Reason being that **you can't see the full specification of this function in its signature.** The intrepid among us will read the whole block into their cavernous working memory and just chug. But I, for one, am busy, stressed, in a hurry, and juggling most of the time.

A quick summary is extremely valuable for someone who reads code professionally. (That's you.)

Here's the signature of the refactored function above:

```python
def get_baker(db: Database, params: BakerGetParams) -> BakerInfo: ...
```

Now, something amazing has happened: Maintainers have gained superpowers:

1. ***Go to definition and hover.***
    Editor support can come off like a petty concern. But without it, functions and methods must be grokked by uncertain means, often leading us on goose-chases throughout large codebases or––in the case of methods not called directly (think an API route)––out of the application entirely and into a documentation trawl.

2. ***A one-liner of documentation.***
    Type systems for dynamic languages are like embedded markup languages for describing functionality. They're wasted on primitives, but highly informative for domain types.

3. ***Type hints with purpose and without tedium.***
    When you say `foo: int` right before writing `foo + 5`, adding types to Python feels dumb. When all your signatures are chock-full of [simple objects](https://pythonbakery.com/bites/passive-data-patisserie/), however, types become useful, information-dense contracts .

## The Pattern

The three words in the heading represent three general object patterns that can replace primitives at least 80% of the time.

Three of the three are already in the example at hand.

```python
def get_baker(db: Database, params: BakerGetParams) -> BakerInfo: ...
```

### Info

Any time you're wanting some *quality* or *attribute* of some *thing*, wrap it in an "info class".

```python
class BakerInfo:
    id: int
    display_name: str
    avatar: bytes
```

Whenever you find callers need more "info", it has a happy home:

```python
class BakerInfo:
    id: int
    display_name: str
    avatar: bytes
    
    @property
    def avatar_kb(self) -> int:
        return len(self.avatar) / 1024
```

I find myself literally using the word "info" very often.

Info is very often a result of some kind, so it's frequently spotted as a return value..

### Params

When you need any kind of *input* like search criteria, indexes, identifiers, paths, etc, you need `Params`.

```python
class BakerGetParams:
    email: str
    password: str
```

Param objects might not have the word "param" in them, to be clear. For example incoming, not-yet-persisted data is a kind of parameter:

```python
class BakerSignup:
    email: str
    password: str
    display_name: str
```

Params are data usually, so put them after resources in the signature. That said, if they aren't being *manipulated* and there's also some kind of `Info` in scope, the `Info` should be last.

### Resource

A resource rarely goes by that name. But it always represents something from the outside world.

Examples would be a file / database / cache / HTTP API / environment variable / queue.

In this case, it's a simple SQL db interface:

```python
class Database:
    def exec(self, sql: str) -> SQLResult:
        ...
        
class SQLResult:
    def one(self) -> Any:
        ...
    # etc
```

[Building strong interfaces at the boundaries of your applications](https://pythonbakery.com/bites/beyond-the-kitchen-door/) should be a topline concern for any application.

I always put resources before any data in function signatures. This facilitates [partial evaluation and/or currying](https://hughfdjackson.com/javascript/why-curry-helps/) if you're into that. 
