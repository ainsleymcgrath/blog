---
title: "Pythonic Pytest Part 1: Fixtures"
date: 2021-06-11
draft: false
---

# Pythonic Pytest Part 1: Fixtures

Pytest is a wonderful testing library. And like most things worth learning, there are some unfamiliar, novel concepts that come with it. Or at least they were unfamiliar to me when I picked it up!

# Getting Started

Let's do the standard stuff. Head into a directory and create a virtual environment.

```jsx
mkdir learn-pytest
cd learn-pytest
touch test_tutorial.py

python -m venv learn-pytest
pip install pytest
```

`pytest` is a python library and an executable. With no arguments, the executable will run any python files and functions prefixed with `test_`.

Everything in this post will be runnable with no dependencies except `pytest`. I encourage you to copy/paste these code samples into a file pytest can see, like `test_tutorial.py`.

So, to begin, here’s a simple test suite:

# The Simplest Possible Test Suite

The following is a complete unit test for a a silly function:

```python
# the function under test
def is_bread(thing: str) -> bool:
		"""Predicate: Determine if incoming `thing` is bread."""
    return thing == "bread"

# the test
def test_is_bread():
    thing = "satisfaction"
    result = is_bread(thing)

    assert result is False, (
        "When passed a non-bread "
        "function returns False"
    )
```

This runs as-is with the `pytest` command. Easy as pie, and it passes! Only a few things to note:

1. Any function prefixed with `test_` in any file or module prefixed with `test_` will be executed by pytest automatically. That behavior is configurable, but I like defaults.
2. When writing pytest assertions, it's encouraged to use the bare `assert` keyword in the form: `assert <condition>: Any, <message>: str`.

As I learned it, the "message" should describe the desired outcome of the test, _not_ the failure condition. The test name should be good and descriptive too but for my money, it's all about a thorough assertion message. They're great as a form of documentation inside of your test suites. The message also propagates into the test failure output inside of the `AssertionError`, so it's helpful to read that output as saying _"Hey, this is what was supposed to happen, and it didnt! :("_

Edit the test above to fail by saying `assert result is 999` or something definitely untrue. You'll see helpful output when you run `pytest`.

Ok, with that sorted, there's one major thing you need to know before you can effectively write professional-grade tests.

# Fixtures: The Special Sauce

Pytest comes with a concept that took me a good while to wrap my head all the way around: "fixtures."

Fixtures" are the solution to a problem that many developers have become callous to: **Fixtures allow you to lift setup, teardown, input, and even expected output of tests out of the bodies of test suites.**

Once more: Pytest fixtures exist to create and destroy any thing or state a test needs in order to run.

Here's a re-write of the test above with a fixture.

```python
import pytest

# the function under test
def is_bread(thing: str) -> bool:
    return thing == "bread"

# using this decorator 'registers' the function as a fixture
@pytest.fixture
def non_bread() -> str:
		return "friendship"

# when a test function has an argument with the same name as some fixture,
# it 'requests' the fixture's return value for use
def test_is_bread(non_bread):
    assert is_bread(non_bread) is False, (
        "When passed a non-bread "
        "function returns False"
    )
```

In the case of our test depending only on a simple string, this example is kind of overkill. We'll do some more legit examples in a moment

For now, I think the main things are to remember that:

1. The _input or state needed_ for a function, class, or other API under test will generally be at home in a fixture.
2. As a soft-goal, pytest fixtures help to optimize test suites for brevity.
3. As a starting point when beginning to test, think “A test has a fixture.”

# Setup, Teardown, and Dependency Injection All at Once

Let's make the above test more complicated and realistic with a database. In this case, we’ll make it so `is_bread` relies on a database connection. Sqlite is built into python, so we’ll use that for examples.

```python
from sqlite import Connection

def is_bread(connection: Connection, thing_name: str) -> bool:
    cursor = connection.execute(
				# in sqlite, `?` is used as a placeholder
        "select is_bread from things where name = ?", (thing_name,)
    )
    (result,) = cursor.fetchone()
    return result == 1  # sqlite uses 1/0 for True/False
```

Now, the function presupposes the existence of a database table called `things` with a column `is_bread` and a column `name`. This "application code" now relies on the behavior of the persistent storage mechanism. In this case, that's SQLite. Regardless, our tests will be incomplete without including the behavior of the database. So how do we do it?

Now fixtures can really shine.

```python
import pytest

@pytest.fixture
def testconn() -> Connection:
    # this is setup
    connection = connect(":memory:")
    connection.execute("create table things (name text, is_bread bool);")
    connection.executemany(
        "insert into things values (?,?)",
        [("bread", True), ("mirth", False), ("croissant", True)],
    )

    # this is dependency provisioning
    yield connection

    # this is teardown
    connection.execute("drop table things")
    connection.close()

def test_is_bread(testconn: Connection):
    assert is_bread(testconn, "bread") is True, "Bread is bread"
    assert is_bread(testconn, "mirth") is False, "Mirth is not bread"
```

Now that we know about fixtures in general, we can quickly glean the gist of this test.

The signature of the test tests tell us the names of whatever objects or preconditions they rely on. In this case, we need a `Connection`. We can go check the implementation of `is_bread` to see why and how it's used.

A `pytest` fixture can be referenced in one or many tests. My rule of thumb (although like all my rules, it's frequently broken) is that in general, **_a_ test has _a_ fixture\***.\* We affectionately refer to these referenced-only-once fixture as "soul mate fixtures" at Amper.

# To be continued... but what did we learn?

More importantly, _why_?

These examples are contrived, but let's revisit some points from earlier:

Here are my introductory tenets of `pytest`:

1. The _input or state needed_ for a function, class, or other API under test will generally be at home in a fixture.
2. As a soft-goal, pytest fixtures help to optimize test suites for brevity.
3. As a starting point when beginning to test, think “A test has a fixture.”

And ultimately, the reason why is for low-boilerplate tests that are super quick to read. If the complexity is lifted out of test bodies, they can read more like specification and tell the story of your features, rather than the story of how you got them into a testable state. 🐍

**In my next piece** we'll look at more advanced fixture use cases, and the second silver bullet of `pytest`: parametrization.
