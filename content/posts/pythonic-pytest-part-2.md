---
title: "Pythonic Pytest Part 2"
date: 2022-05-14T14:02:53-05:00
draft: true
---

# Pythonic Pytest Part 2: The Parametrization Mantra 🧘🏽‍♂️

In a few words, "parametrization" in pytest means *"Configuring one test to run many times, with many inputs, as if it were many tests."* 

**Parametrization is a silver bullet in many cases for facilitating exhaustive testing.**

:information_source: If you want to try these examples out, Part 1 offers some brief setup guidance.

# 🤔 Why parametrize?

1. Parametrization literally turns many tests into one. For this reason, **it can facilitate the DRY principle.**
2. **Parametrization helps enforce “one way to do things” within your own code.** Since functionality can only vary based on inputs/outputs (rather than, for example, a particular sequence of method calls).
3. **Once you’re set up, it is simply faster and easier to write new tests.** It becomes trivial to start churning out edge cases and codify *“Aha!”* moments.

Good candidates for parametrization include:

- Low-level functions that do math or other arithmetic
- Validation code
- Predicates

# 🙅🏽‍♂️ Why not?

1. Parametrization can be more trouble than it’s worth when **the thing under test has a complex interface or behaviors** that can’t be expressed imperatively.
2. The style that parametrization calls for **can become unwieldy and hard to read** very fast when inputs are large or involved. Parametrizing *is absolutely* more complex than not.
3. Sometimes there are simply **not that many permutations of input/output** to be worth abstracting.

Poor candidates for parametrization include:

- Methods/functions that aren’t always called the same way
- Objects with lots of properties that you want to observe in lots of ways
- Functions / objects with large signatures

# 👀 How Do I Do It?

We’ll do it with an example! Here’s the function we’ll test.

This is an example of the “lowish-level” code that is generally great to unit test: A straightforward iteration recipe.

```python
def take_up_to_n(n, it):
		"""Return as many members of the iterable as specified by `n`.
		If n >= len(it), return all the elements."""
		_it = iter(it)  # guarantee given object can be next()'d
		out = []
		for _ in range(n):
				try:
						out.append(next(_it))
				except StopIteration:
						# oops! out of elements
						return out

		return out
```

Here are some plain ol’ tests to prove it out:

```python
	def test_take_n_less_than_length():
    assert take_up_to_n(2, ["!", "?", "..."]) == ["!", "?"], (
        "With sequence of len < n, return list of first n elements"
    )

def test_take_n_greater_than_length():
    assert take_up_to_n(5, [1, 2, 3]) == [1, 2, 3], (
        "With sequence of len > n, return list of all the elements"
    )

def test_take_n_empty_sequence():
    assert take(1000, []) == [], (
        "With an empty sequence, an empty sequence is returned"
		)

def test_take_n_zero():
    assert take(0, [object(), 0, "123"]) == [], (
       "With n=0, an empty sequence is returned"
		)
```

Great! Some documented, tested code.

Let’s zoom out.

## 🪘 The Mantra

All 4 of the tests above look the same: There are 5 parts present in all of them:

1. The function: In this case, `take`. We’ll call it `interface`.
2. Its arguments: an integer and a sequence. We’ll call that `given`.
3. The result of the function call: In this case, it’s anything to the right of `==`. We’ll call that `expected`.
4. The expected return: In this case, anything to the right of `==`. We’ll call that `actual`.
5. A string message after `assert`, describing what the function should do for that test. We’ll call that `should`.

So the recipe is:

> **Given** some input the feature **should** behave some way and meet this **expected** condition once we get its **actual** result.

In Python, that’s:

```python
actual = interface(given)
assert expected == actual, should
```

If your functionality can be expressed this way, congratulations! You’ve got some highly testable code.

## 👯‍♀️ Parametrization as a sibling of fixtures

Remember [fixtures](https://ainsleymcgrath.com/pythonic-pytest-part-1-fixtures/#fixtures-the-special-sauce) and the way they’re injected? Here’s one example from Part 1:

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

Above 👆 notice that the return value of function decorated with `@pytest.fixture` is injected into `test_is_bread` by its *name.* Pytest gets the values of fixtures by finding decorated functions whose names match the test function’s arguments.

## 🥊 A Parametrized Test in Action

`**@pytest.mark.parametrize` also injects values by name and takes position into consideration as well.**

The decorator, `@pytest.mark.parametrize`, requires 2 positional arguments:

1. A string of comma-separated names that will be injected into the test’s signature
2. An iterable of iterables (I favor a list of tuples) that will be passed on as function arguments.

**Here’s the parametrized version of the tests above:**

```python
import pytest

@pytest.mark.parametrize(
    "given, expected, should",
    [
        (
            (2, ["!", "?", "..."]),
            ["!", "?"],
            "With a sequence of len < n, return list of first n elements",
        ),
        (
            (5, [1, 2, 3]),
            [1, 2, 3],
            "With a sequence of len > n, return list of all the elements",
        ),
        (
            (1000, []),
            [],
            "With an empty sequence, return an empty sequence",
        ),
        (
            (0, [object(), 0, "123"]),
            [],
            "With n=0, return an empty sequence",
        ),
    ],
)
def test_take_up_to_n(given, expected, should):
    n, it = given
    actual = take_up_to_n(n, it)

    assert expected == actual, should
```

**The string given as the first argument matches the decorated function’s signature.**

**The second is a set of three-tuples that are essentially unpacked into the signature, in order.**

The execution of the first test essentially does this:

```python
given, expected, should = (
    (2, ["!", "?", "..."]),
    ["!", "?"],
    "With a sequence of len < n, return list of first n elements",
)

test_take_up_to_n(given, expected, should)
```

Then the second test unpacks the value of the second tuple:

```python
given, expected, should = (
    (5, [1, 2, 3]),
    [1, 2, 3],
    "With a sequence of len > n, return list of all the elements",
)

test_take_up_to_n(given, expected, should)
```

And so on.

# 🔙 Back to the Mantra

Let’s look at one of those cases *one* more time.

```python
given, expected, should = (
    (2, ["!", "?", "..."]),
    ["!", "?"],
    "With a sequence of len < n, return list of first n elements",
)
```

This follows the structure cited earlier in a concrete way.

> **Given** some input the feature **should** behave some way and meet this **expected** condition once we get its **actual** result.

Imagine it as a (pseudo) Python type.

```python
Tuple[
	Given["function inputs"]
	Expected["desired return value"]
	Should["prose description of the test"]
]
```

This cognitive model is based on [Eric Elliott’s RITEWay testing philosophy](https://medium.com/javascript-scene/tdd-the-rite-way-53c9b46f45e3). 

At [Amper](https://www.amper.xyz), we have hundreds of tests parametrized in this exact fashion with these *exact* arguments. The amount of time, energy, and boilerplate saved is already great, but the ability to extend test suites with further cases is an even better long-term benefit.

**In my experience, this pattern does not wear out until you are testing very complex conditional logic or functionality that relies on complex state/inputs.** For example, raising different errors conditionally or observing different object properties during different states doesn’t jive with the very homogenous nature of the `given, should, expected` test structure.

# 💁‍♂️ Part 2.1: Fixtures Can Do It Too!

**The same way you can configure a test to run many times with different inputs, the same can be done with [fixtures](https://ainsleymcgrath.com/pythonic-pytest-part-1-fixtures/).**

Without getting into too much detail, it's achieved with a keyword-argument called 🥁 `params`, given to `@pytest.fixture`. 

`params` receives an iterable, but in contrast to `@pytest.mark.parametrize`, they can contain anything.

*Every single fixture* can refer to a [built-in pytest fixture called `request`](https://docs.pytest.org/en/6.2.x/reference.html#request). This object contains a ton of data about the current test run. When you pass `params`, the injected value on each test run is bound to `request.param`.

```python
@pytest.fixture(params=[{'apple', 'papaya'}, 'banana', None])
def stocked_pantry(request):
  	# it's often helpful to assign the value of `request.param` for clarity
  	thing_in_pantry = request.param
```

This fixture will be set up in 3 different ways, resulting in all downstream dependent tests running 3 times.

On the first run, the value is a `set`, on the second it's a `str`, and on the last, it's `None`.

I chose those values to illustrate the fact that **parameters given to fixtures are injected as-is**, unlike parametrized tests, which unpack them into arguments.

## 🧐 Why Parametrize Fixtures?

When The Mantra stops being applicable, you should start thinking about it.

**The Mantra "stops being applicable" when your functionality stops acting like pure functions:**

- Maybe you're testing behavior at a high level where input and output aren't the core issue.

- Or you want something to behave the same in many different external conditions.

That second example is easy to demo.

## 🔬 An Example of Fixture Parametrization

Let's say we have a class that reads from the environment on `__init__`.

I would call this "high-ish" level code, as opposed to the "lowish" level pure function from before.

```python
import os


def is_valid_username_var(name: str) -> bool:
    name = name.lower()
    return "username" in name or "user_name" in name


class SystemUserInfo:
    """This class will look for any env var containing 'username'
    or `user_name` to use for its own `username` property."""

    def __init__(self):
        sentinel = object()
        var_names = (name for name in os.environ if is_valid_username(name))
        username_var = next(matching_vars, sentinel)

        self.username = (
            "Nameless User" if username_var is sentinel else os.environ[username_var]
        )
```

For our test, we want to make sure the class finds the username via a valid environment variable.

So, we'll use a fixture (as intended) to set up some external state.

```python
DEFAULT_VALID_USER_NAME = "Ainsley"


@pytest.fixture(
    params=["USERNAME", "THE_USER_NAME", "SUPERDUPERUSERNAME", "__USER_NAME__"]
)
def set_user_env_var_valid(monkeypatch, request):
    env_var_name: str = request.param
    # use the builtin monkeypatch fixture to manipulate the env
    monkeymatch.setenv(env_var_name, DEFAULT_VALID_USER_NAME)


def test_system_user_valid_names(set_user_env_var_valid):
    info = SystemUserInfo()
    assert info.username == DEFAULT_VALID_USER_NAME
```

This test will run 4 times with each different environment config.

Since everything is derived from data external to the caller, `given, should, expected` isn't applicable. Still, we can reap the benefits of exhaustive testing.

The same recipe applies to the path of an incorrectly-configured environment:

```python
@pytest.fixture(params=["USER___NAME", "USER_NOMBRE", "MY_NAME", "NAME"])
def set_user_env_var_invalid(monkeypatch, request):
    env_var_name: str = request.param
    # use the builtin monkeypatch fixture to manipulate the env
    monkeymatch.setenv(env_var_name, "This won't work.")


def test_system_user_invalid_names(set_user_env_var_valid):
    info = SystemUserInfo()
    assert info.username == "Nameless User"
```

## ⏩ Forwarding Values

In the examples we just did, we tested that the *exact* same result occurred based on certain conditions.

In the test for valid usernames, maybe we actually don't want to use the same default name over and over. Let's actually use it in the test!

A slight alteration to our fixture can make that happen: Let's revisit the parametrization style from the start of this post and use tuples.

```python
@pytest.fixture(
    params=[
        ("USERNAME", "Bob"),
        ("THE_USER_NAME", "Rue"),
        ("SUPERDUPERUSERNAME", "Elias"),
        ("__USER_NAME__", "Anika"),
    ]
)
def set_user_env_var_valid(monkeypatch, request):
    # now, `param` is a tuple!
    env_var_name, env_var_value = request.param
    monkeymatch.setenv(env_var_name, env_var_value)
    # by returning a value, we can 'forward it' to the test
    return env_var_value


def test_system_user_valid_names(set_user_env_var_valid):
    expected_name = set_user_env_var_valid  # alias for coherence
    info = SystemUserInfo()
    assert info.username == expected_name
```

A few things to note here:

- While we parametrized the fixture with two inputs, only one is forwarded to the test.
  - This is arguably a point for isolating test setup and execution
  - You could also write it off as unnecessary indirection
- This opens the door to more ways to express complexity but does come at the expense of difficult naming and aliasing.
- This can get out of hand rapidly. Luckily, Part 3 will arm us with the tools we need to carry on with expressive, minimal-boilerplate, exhaustive testing.

# 🥴 Recap

What did we learn?

1. Pytest tests can easily be configured to run many times with various inputs.
2. **The mantra of `given, expected, should` is applicable to *lots* of code.** Using that mental framework of thinking will take you *very* far.
3. When your tests go beyond the scope of pure input and output, consider parametrizing fixtures.
4. **Overall, fixtures exist to optimize your tests for exhaustiveness and extensibility.**

Lift your thinking away from implementation, and think of the way it behaves. Then, your code will grow more testable and more parametrize-able.

# 🔮 To be Continued...

Sometimes, we "fall off the edge of the world". The mantra loses its ring, cases grow hard to grok, and our fixtures get confusing. What do we do?

**In Part 3** I'll cover my idea of "lucidity" in tests as a powerful tool for wrangling out-of-control complexity in exhaustive tests.

