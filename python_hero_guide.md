# 🐍 Python 0-to-Hero: A Practical Guide for Developers

> A guide for writing Python like a senior backend dev — clean scripts, typed code, decorators, Pydantic, and production FastAPI services (DB, auth, background jobs).
> **Main references:** [docs.python.org](https://www.python.org/doc/) for the language, [fastapi.tiangolo.com](https://fastapi.tiangolo.com/) for the backend half.
> **Current stack this guide targets:** Python 3.13/3.14, FastAPI 0.13x, Pydantic v2, SQLAlchemy 2.0 (async), `uv` for packaging, `ruff` for linting.

---

## Who this is for, and how it's organized

This guide assumes you already know how to program — you understand what a variable, a loop, and a function *are* from prior experience, in whatever language that was. It **never explains the "what."** It focuses on two things instead:

1. **Why Python's approach works the way it does** — explained simply, with a light analogy where one actually helps.
2. **How a senior dev actually writes it** — idiomatic, typed, production code, not textbook toy examples.

Each module ends with practice exercises — attempt them before moving on; that's where the concepts actually stick.

---

## 🗺️ The Full Roadmap

| # | Module | What it covers |
|---|--------|-----------------|
| 0 | Foundations | Variables & mutability, EAFP/LBYL, concurrency, REPL, duck typing |
| 1 | Syntax Fast-Pass | Unpacking, comprehensions, generators, context managers, walrus |
| 2 | Data Structures & Idioms | dict/set/tuple tricks, `collections`, slicing, `dataclasses`, `Enum` |
| 3 | Functions Deep-Dive | closures, decorators, `functools`, iterators, generators, context managers |
| 4 | OOP That Doesn't Suck | classes, dunder methods, properties, inheritance, composition, ABCs, dataclasses, Pydantic bridge |
| 5 | Exceptions & File Handling | exceptions, custom errors, file I/O, `pathlib` |
| 6 | Modules & Packages | imports, packages, `__init__.py`, `__main__`, `uv`, project layout |
| 7 | Advanced Python | iterators/generators review, descriptors, metaclasses, typing, pattern matching, `asyncio` |
| 8 | Standard Library | `itertools`, `functools`, `pathlib`, `datetime`, `collections`, `heapq`, `bisect`, `statistics`, `json`, `csv`, `sqlite3`, `logging`, `typing` |
| 9 | Testing & Tooling | `pytest`, `unittest`, mocking, Ruff, Black, mypy/pyright, debugging |
| 10 | Pythonic Best Practices | idioms, performance, pitfalls, standard-library patterns, interview patterns |
| — | **FastAPI — separate notebook** | routing, Pydantic integration, dependencies, database, auth, background work, production |
| — | **Capstone — separate project** | real application tying the Python foundations together |

Modules are designed to be read in order, but each one stands reasonably well on its own if you want to jump straight to, say, Pydantic or FastAPI Auth.

---

## Module 0 — Foundations: How Python Thinks

### 1. Variables are labels, not boxes

A variable name in Python works like a label stuck onto a value, not a box that holds the value inside it. When you write `b = a`, you didn't copy anything into a new box — you stuck a second label onto the exact same object.

```python
a = [1, 2, 3]
b = a          # b is a second sticky note on the SAME list
b.append(4)
print(a)       # [1, 2, 3, 4] — a changed too, even though you never touched "a"
print(a is b)  # True — same object, two names
```

This single fact — names are references, not containers — explains most of the "why did my data change when I didn't touch that variable" bugs you'll hit early on.

### 2. Mutable vs. immutable data

Some Python objects are **mutable** (you can change them in place): `list`, `dict`, `set`, and your own custom objects by default.
Some are **immutable** (any "change" actually creates a brand new object): `tuple`, `str`, `frozenset`, `int`, and `@dataclass(frozen=True)` if you opt in.

```python
name = "ada"
name.upper()      # returns a NEW string "ADA" — name itself is still "ada"
print(name)        # ada

nums = [1, 2, 3]
nums.append(4)      # mutates the SAME list in place
print(nums)          # [1, 2, 3, 4]
```

**The one habit to build immediately:** for every function you write or call, ask "does this mutate its input, or return a new value?" Python will not protect you from finding out the hard way — it's on you to keep track of which of your objects change in place and which don't.

### 3. Exceptions as control flow (EAFP)

Python's dominant style is **EAFP** — "Easier to Ask Forgiveness than Permission." You attempt an operation and catch the exception if it fails, rather than checking every precondition up front (called **LBYL** — "Look Before You Leap," the way you'd inspect a locked door's hinges and keyhole from the hallway before ever touching the handle). EAFP just turns the handle — if it's locked, you deal with that *then*.

```python
# LBYL style — checking first
if "age" in data and isinstance(data["age"], int):
    process(data["age"])

# EAFP style — Python's preferred idiom
try:
    process(data["age"])
except (KeyError, TypeError):
    handle_missing_or_bad_age()
```

Both work. But idiomatic Python — and most of the standard library — leans EAFP. Get comfortable reading and writing `try`/`except` as a normal, non-exceptional part of everyday logic, not just for genuine emergencies.

### 4. Concurrency: the GIL, threads, processes, and async

This is the concept that surprises the most developers moving to Python, regardless of what they came from. Picture an office with ten employees (threads) but only **one bathroom key** for the whole floor: even though all ten people are "at work" simultaneously, only one can actually be in the bathroom — executing Python bytecode — at any given instant. That single key is the **GIL** — the Global Interpreter Lock.

Practical consequence:

- **CPU-bound work** (crunching numbers, image processing) — threads won't actually run in parallel because of the GIL. You need `multiprocessing`, which spins up entirely separate processes (separate offices, each with their own bathroom key).
- **I/O-bound work** (waiting on a database, an HTTP call, a file read) — the GIL is released while waiting, so `threading` or, better, `asyncio` gives you real concurrency, because nobody's fighting over the key while everyone's just waiting on external replies.

`asyncio` is a single-threaded event loop that juggles many waiting tasks cooperatively — it's the tool you'll reach for constantly once we get into FastAPI (Module 7 and Module 10+), since API servers spend most of their time waiting on databases and network calls, not crunching numbers.

### 5. The tooling landscape

Python's packaging story historically fragmented: `pip` + `venv` (the built-in, bare-bones combo), `poetry`, `pipenv`, and `conda` all compete for the same job — managing dependencies and virtual environments. There isn't one single blessed tool the way some ecosystems have.

In 2026, **`uv`** (a fast, Rust-based tool) has become the converging standard — it replaces `pip`, `venv`, and most of what `poetry` does, and it's what this guide uses starting in Module 8.

### 6. The REPL

Run `python` in a terminal for a basic interactive shell, or install `ipython` for a much nicer one — tab-completion, better history, and `object?` to pull up inline docs.

### 7. Duck typing

Python cares about *what an object can do*, not *what it's declared to be*. If an object has a `.read()` method, it can be used anywhere something "file-like" is expected — no matter its actual class. This is called **duck typing**: "if it walks like a duck and quacks like a duck, treat it like a duck." Module 5 covers `Protocol`, which lets you express this formally with type hints.

### Quick gut-check exercise (attempt before reading further)

Predict the output, then run it to check yourself:

```python
def add_item(cart, item):
    cart.append(item)
    return cart

my_cart = ["apple"]
new_cart = add_item(my_cart, "banana")

print(my_cart)   # what prints here?
print(new_cart)  # and here?
print(my_cart is new_cart)  # and here?
```

*(This aliasing behavior surprises almost everyone the first time they hit it. Once you can explain why all three lines print what they print, Module 0 is done.)*

---

## Module 1 — Syntax Fast-Pass & Pythonic Idioms

This module skips anything as basic as what a loop or an `if` statement is, and covers only the syntax and idioms that are genuinely distinctive to Python.

### 0. Control flow syntax at a glance

Python uses indentation (not braces) to mark blocks, and colons to open them. Quick reference:

```python
# if / elif / else
if x > 10:
    print("big")
elif x > 0:
    print("small")
else:
    print("non-positive")

# ternary (conditional expression)
label = "big" if x > 10 else "small"

# while
count = 0
while count < 5:
    count += 1          # no ++ in Python
    if count == 3:
        continue         # skip to next iteration
    if count == 4:
        break             # exit the loop early
else:
    # runs only if the loop finished WITHOUT hitting `break`
    print("loop completed normally")

# for — always iterates over something iterable, there's no C-style for(;;)
for i in range(5):        # 0,1,2,3,4
    print(i)

for index, value in enumerate(["a", "b", "c"]):
    print(index, value)

for key, value in {"a": 1, "b": 2}.items():
    print(key, value)

# match / case — structural pattern matching (3.10+), Python's answer to a switch statement
match command:
    case "start":
        do_start()
    case "stop" | "halt":
        do_stop()
    case ("move", direction):
        do_move(direction)
    case _:
        do_default()
```

Two habits worth flagging early: the `for...else` / `while...else` clause (the `else` runs only if the loop was never `break`-ed out of — most languages don't have this), and the fact that Python's `for` only ever iterates over an iterable — there's no classic `for (i=0; i<n; i++)` form, `range()` is how you simulate it.

### 1. Unpacking

```python
a, b = 1, 2

# "rest" capture — grab the first item, bundle the remainder
first, *rest = [1, 2, 3, 4]
print(first, rest)  # 1 [2, 3, 4]

# swapping — no temp variable needed
a, b = b, a

# unpacking in function calls
def move(x, y): ...
point = (3, 4)
move(*point)  # same as move(3, 4)
```

Note: unpacking happens at assignment or at the call site — Python doesn't let you destructure directly inside a function's parameter list the way some languages allow matching on structured arguments.

### 2. Comprehensions

```python
result = [x * 2 for x in numbers if x * 2 > 5]

# dict comprehension
squares = {x: x**2 for x in range(5)}

# set comprehension
uniques = {x % 3 for x in range(10)}
```

A comprehension builds a whole list, dict, or set in one expression instead of a multi-line loop with repeated `.append()` calls — it reads almost like a sentence: *"give me `x * 2`, for every `x` in the list, but only when `x * 2` is bigger than 5."*

If a comprehension needs more than one `if` or one nested `for`, stop — write a real loop instead. Unreadable comprehensions are a classic Python code-smell.

### 3. Generators

```python
def read_big_file(path):
    with open(path) as f:
        for line in f:
            yield line.strip()

# nothing runs yet — this is lazy
lines = read_big_file("huge.log")
first_five = [next(lines) for _ in range(5)]
```

A normal function is **eager**: `return` computes the whole result up front and hands it all back at once, like a vending machine that dumps every snack into your bag the moment you press the button. A generator (`yield`) is **lazy** instead: it hands you one value at a time, only when you ask for the next one via `next()` — the machine makes each snack fresh rather than dispensing its whole stock in one go.

Any function with `yield` in it becomes a generator — calling it doesn't run the body immediately, it just returns an iterator that runs the body piece by piece as you pull values from it.

### 4. Context managers (`with`)

```python
with open("file.txt") as f:
    data = f.read()
# file is guaranteed closed here, even if read() raised an exception
```

`with` guarantees that no matter what happens inside the block — even an exception — the cleanup step runs afterward. It's used constantly in Python: files, database connections, locks, and HTTP sessions all lean on this pattern rather than reserving it for rare edge cases.

You'll write your own context managers once we hit database sessions in Module 11 — it's one function decorated with `@contextmanager`, or one class implementing `__enter__`/`__exit__`.

### 5. The walrus operator `:=`

```python
# instead of:
data = fetch()
if data:
    process(data)

# you can inline the assignment:
if data := fetch():
    process(data)
```

Small, but you'll see it constantly in modern (post-3.8) production code and inside comprehensions where you want to avoid computing something twice.

### 6. `*args` and `**kwargs` (quick preview — full depth in Module 3)

```python
def log(*args, **kwargs):
    print(args)    # tuple of positional args
    print(kwargs)  # dict of keyword args

log(1, 2, name="Ada")
# (1, 2)
# {'name': 'Ada'}
```

This is how Python functions accept "whatever you throw at me" — a variable number of positional and keyword arguments — which becomes essential once you see how FastAPI and decorators lean on it.

### Practice exercises

1. Write a generator `even_numbers(limit)` that lazily yields even numbers up to `limit`, and prove it's lazy by showing it doesn't compute anything until you call `next()` on it.
2. Write one comprehension that takes a list of dicts `[{"name": "a", "age": 17}, {"name": "b", "age": 22}, ...]` and returns just the names of people 18 or older.
3. Using unpacking (no indexing with `[0]`, `[1]`), write a function `describe_point(point)` that takes a 3-tuple `(x, y, z)` and returns a formatted string — capture `x, y` and let `z` be captured via a rest-pattern for a 4th+ optional dimension.

---

*Next up: Module 2 — Data Structures & Idioms.*

---

## Module 2 — Data Structures & Idioms

### Slicing

```python
data = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]

data[2:5]      # [2, 3, 4]        — start:stop
data[:3]       # [0, 1, 2]        — omit start = from the beginning
data[7:]       # [7, 8, 9]        — omit stop = to the end
data[::2]      # [0, 2, 4, 6, 8]  — step
data[::-1]     # reversed copy
data[-3:]      # [7, 8, 9]        — negative index counts from the end
```

Slicing always returns a **new** list — it never mutates the original, and it never raises an `IndexError` even if the range is out of bounds (`data[100:200]` just returns `[]`).

### Tuples vs. lists vs. sets vs. dicts — when to reach for each

- **`list`** — ordered, mutable, allows duplicates. Default choice for a sequence you'll modify.
- **`tuple`** — ordered, immutable. Use for fixed-size records (a coordinate, a row) or anything you want to guarantee won't be mutated — and note tuples are hashable, so they can be dict keys or set members, unlike lists.
- **`set`** — unordered, unique elements, O(1) membership checks. Reach for a set the moment you're about to write `if x in some_list` in a hot path — `in` on a list is O(n), `in` on a set is O(1).
- **`dict`** — ordered (as of 3.7+, insertion order is guaranteed), key-value mapping. The default tool for anything you'd reach for a hash map for.

### The `collections` module — the toolbox senior devs actually use

```python
from collections import Counter, defaultdict, namedtuple, deque

# Counter — frequency counting in one line
Counter("mississippi")
# Counter({'i': 4, 's': 4, 'p': 2, 'm': 1})

# defaultdict — no more `if key not in d: d[key] = []`
grouped = defaultdict(list)
for name, age in [("a", 1), ("b", 2), ("a", 3)]:
    grouped[name].append(age)
# {'a': [1, 3], 'b': [2]}

# namedtuple — a lightweight, immutable, self-documenting record
Point = namedtuple("Point", ["x", "y"])
p = Point(3, 4)
print(p.x, p.y)   # 3 4

# deque — O(1) appends/pops from BOTH ends (a plain list is O(n) on the left end)
q = deque([1, 2, 3])
q.appendleft(0)
q.pop()
```

### `dataclasses` — the modern way to write a data-holding class

```python
from dataclasses import dataclass, field

@dataclass
class User:
    name: str
    age: int
    tags: list[str] = field(default_factory=list)   # never use a mutable default directly — see Module 3

u = User(name="Ada", age=30)
print(u)                 # User(name='Ada', age=30, tags=[])
print(u == User(name="Ada", age=30, tags=[]))  # True — __eq__ generated for you

@dataclass(frozen=True)
class ImmutablePoint:
    x: int
    y: int
```

`@dataclass` auto-generates `__init__`, `__repr__`, and `__eq__` based on the fields you declare — no more hand-writing boilerplate constructors. `frozen=True` makes instances immutable (an assignment after creation raises an error).

### `Enum` — named constants with real type safety

```python
from enum import Enum, auto

class Status(Enum):
    PENDING = auto()
    ACTIVE = auto()
    CLOSED = auto()

def handle(status: Status):
    if status is Status.ACTIVE:
        ...
```

Prefer `Enum` over a handful of loose string or int constants — it's self-documenting, and type checkers (Module 5) will catch a typo'd status you'd never catch with bare strings.

### Practice exercises

1. Given `words = ["apple", "banana", "apple", "cherry", "banana", "apple"]`, use `Counter` to find the most common word and its count in one line.
2. Rewrite this without a `defaultdict`, then with one, and compare: group a list of `(department, employee)` tuples into a dict of `department -> [employees]`.
3. Define a `frozen` dataclass `Money` with `amount: float` and `currency: str`, then write a function that adds two `Money` instances only if their currencies match, raising a `ValueError` otherwise.

---

## Module 3 — Functions Deep-Dive

### The mutable default argument trap

```python
# DON'T do this:
def add_item(item, cart=[]):   # the list is created ONCE, at function definition time
    cart.append(item)
    return cart

add_item("apple")   # ['apple']
add_item("banana")  # ['apple', 'banana']  <- surprise! same list reused across calls

# DO this instead:
def add_item(item, cart=None):
    if cart is None:
        cart = []
    cart.append(item)
    return cart
```

This is one of Python's most infamous gotchas: default argument values are evaluated **once**, when the function is defined — not on every call. Any mutable default (`[]`, `{}`, a custom object) becomes a shared, persistent trap across calls.

### `*args` and `**kwargs`, properly

```python
def summarize(*args: int, **kwargs: str) -> None:
    print("positional:", args)     # tuple
    print("keyword:", kwargs)      # dict

summarize(1, 2, 3, unit="kg", source="scale")

# unpacking the other direction — spreading a collection INTO a call
values = [1, 2, 3]
options = {"unit": "kg"}
summarize(*values, **options)

# keyword-only arguments — force callers to name them (great for readability)
def resize(image, *, width, height):
    ...
resize(img, width=100, height=200)   # OK
resize(img, 100, 200)                 # TypeError — width/height MUST be named

# positional-only arguments (3.8+) — force callers NOT to name them
def divide(a, b, /):
    return a / b
```

### Closures

```python
def make_counter():
    count = 0
    def increment():
        nonlocal count      # without this, `count += 1` below would raise UnboundLocalError
        count += 1
        return count
    return increment

counter = make_counter()
print(counter())  # 1
print(counter())  # 2
```

A closure is an inner function that "remembers" variables from the scope it was defined in, even after that outer function has returned. `nonlocal` is required any time the inner function needs to *reassign* (not just read) a variable from the enclosing scope.

### Decorators

A decorator is a function that takes a function and returns a (usually wrapped) function — it's how Python lets you attach reusable behavior (timing, logging, auth checks, caching) around a function without touching its internals.

```python
import functools
import time

def timed(func):
    @functools.wraps(func)          # preserves func's name/docstring — skip this and debugging gets painful
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = func(*args, **kwargs)
        print(f"{func.__name__} took {time.perf_counter() - start:.4f}s")
        return result
    return wrapper

@timed
def slow_add(a, b):
    time.sleep(1)
    return a + b

slow_add(2, 3)   # prints timing, then returns 5
```

`@timed` above `def slow_add` is just sugar for `slow_add = timed(slow_add)`.

**Decorators that take arguments** need one extra layer of nesting:

```python
def retry(times):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(times):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if attempt == times - 1:
                        raise
                    print(f"retrying after: {e}")
        return wrapper
    return decorator

@retry(times=3)
def flaky_call():
    ...
```

### `functools` — the standard library's decorator toolbox

```python
from functools import lru_cache, partial, reduce, singledispatch

@lru_cache(maxsize=None)      # memoization in one line
def fib(n):
    return n if n < 2 else fib(n - 1) + fib(n - 2)

double = partial(lambda x, y: x * y, y=2)   # pre-fill an argument
double(5)   # 10

reduce(lambda acc, x: acc + x, [1, 2, 3, 4], 0)   # 10

@singledispatch     # different implementations based on the type of the first argument
def render(value):
    raise NotImplementedError

@render.register
def _(value: int):
    return f"int: {value}"

@render.register
def _(value: str):
    return f"str: {value}"
```

### Practice exercises

1. Write a decorator `@log_calls` that prints the function name and its arguments every time the decorated function is called, then returns the function's result unchanged.
2. Write a closure-based `make_multiplier(n)` that returns a function multiplying its input by `n`. Create `double = make_multiplier(2)` and `triple = make_multiplier(3)` and show they don't interfere with each other.
3. Fix this buggy function using what you learned about mutable defaults: `def add_tag(tag, tags=[]): tags.append(tag); return tags`.

---

## Module 4 — OOP That Doesn't Suck

### The basics, fast

```python
class Account:
    interest_rate = 0.02          # class attribute — shared by all instances

    def __init__(self, owner: str, balance: float = 0):
        self.owner = owner        # instance attribute — unique per instance
        self.balance = balance

    def deposit(self, amount: float) -> None:
        self.balance += amount
```

`self` is just the instance — Python passes it explicitly as the first parameter of every instance method (there's no implicit `this`).

### Dunder (magic) methods — how you plug into Python's own syntax

```python
class Money:
    def __init__(self, amount, currency):
        self.amount, self.currency = amount, currency

    def __repr__(self):                        # what you see in a REPL / debugger
        return f"Money({self.amount}, {self.currency!r})"

    def __eq__(self, other):                   # powers ==
        return (self.amount, self.currency) == (other.amount, other.currency)

    def __add__(self, other):                  # powers +
        if self.currency != other.currency:
            raise ValueError("currency mismatch")
        return Money(self.amount + other.amount, self.currency)

    def __len__(self):                          # powers len(obj)
        ...

    def __iter__(self):                          # powers `for x in obj`
        ...
```

Implementing `__repr__`, `__eq__`, and whichever operators make sense (`__add__`, `__lt__`, etc.) is what makes a custom class feel like a first-class Python citizen instead of a bag of attributes.

### `@property`, `@staticmethod`, `@classmethod`

```python
class Circle:
    def __init__(self, radius):
        self._radius = radius

    @property
    def area(self):                 # accessed like an attribute: circle.area, no parens
        return 3.14159 * self._radius ** 2

    @property
    def radius(self):
        return self._radius

    @radius.setter
    def radius(self, value):
        if value < 0:
            raise ValueError("radius can't be negative")
        self._radius = value

    @staticmethod
    def unit_circle():              # doesn't need self or cls — just namespaced under the class
        return Circle(1)

    @classmethod
    def from_diameter(cls, diameter):   # alternate constructor — gets the class, not an instance
        return cls(diameter / 2)
```

`@property` is how you add validation or computed values behind attribute-style access (`circle.radius = -5` runs the setter and can reject it) without forcing every caller to use `get_radius()`/`set_radius()` methods.

### Composition over inheritance

```python
# Inheritance — "IS-A" relationship
class Vehicle:
    def move(self): ...
class Car(Vehicle):
    ...

# Composition — "HAS-A" relationship, usually more flexible
class Engine:
    def start(self): ...

class Car:
    def __init__(self):
        self.engine = Engine()      # Car HAS an Engine, isn't forced into an Engine hierarchy
    def start(self):
        self.engine.start()
```

Default to composition. Reach for inheritance only when there's a genuine "is-a" relationship and you want to share behavior across a real hierarchy — deep inheritance chains are a common source of fragile, hard-to-follow Python codebases.

### Abstract base classes — enforcing a contract

```python
from abc import ABC, abstractmethod

class PaymentProcessor(ABC):
    @abstractmethod
    def charge(self, amount: float) -> bool:
        ...

class StripeProcessor(PaymentProcessor):
    def charge(self, amount: float) -> bool:
        ...   # must implement this, or instantiation raises TypeError
```

Use an `ABC` when you want to guarantee every subclass implements a given method — this becomes especially useful once you're wiring up interchangeable services (e.g. swappable payment providers, storage backends) in Module 10+.

### `super()` vs inheritance syntax

These solve two different problems.

```python
class Dog(Animal):
    ...
```

establishes inheritance: `Dog` is an `Animal`.

```python
super().__init__(name)
```

uses the inherited implementation, usually to extend the parent's behavior.

```python
class Animal:
    def __init__(self, name):
        self.name = name


class Dog(Animal):
    def __init__(self, name, breed):
        super().__init__(name)
        self.breed = breed
```

Prefer `super()` over directly calling `Animal.__init__(self, ...)` because it follows Python's method resolution order and behaves correctly with cooperative multiple inheritance.

> **Mental model:** inheritance says *what you inherit*; `super()` lets you access the inherited implementation.

### `property`: getter vs setter

`@property` creates the getter:

```python
class Product:
    def __init__(self, price):
        self._price = price

    @property
    def price(self):
        return self._price
```

Now:

```python
product.price
```

calls the getter.

A setter is optional:

```python
@price.setter
def price(self, value):
    if value < 0:
        raise ValueError("price cannot be negative")
    self._price = value
```

Without the setter, the property is effectively read-only. Setters are useful when assignment needs validation or transformation.

### `dataclass` vs Pydantic model

Dataclasses are primarily for **internal application data**:

```python
from dataclasses import dataclass

@dataclass
class Product:
    name: str
    price: float
```

Pydantic models are especially useful at **data boundaries**, where incoming data needs runtime validation/parsing:

```python
from pydantic import BaseModel

class ProductInput(BaseModel):
    name: str
    price: float
```

A useful rule:

```text
trusted/internal data  → dataclass
external/untrusted data → Pydantic
```

Pydantic can also be used internally when its validation/serialization behavior is useful; this is a guideline, not a hard rule.

### Real-world OOP example — Banking

Model a simple banking domain.

```python
class Account:
    def __init__(self, account_number, owner, balance=0):
        self.account_number = account_number
        self.owner = owner
        self._balance = balance

    @property
    def balance(self):
        return self._balance

    def deposit(self, amount):
        if amount <= 0:
            raise ValueError("deposit must be positive")
        self._balance += amount

    def withdraw(self, amount):
        if amount <= 0:
            raise ValueError("withdrawal must be positive")
        if amount > self._balance:
            raise ValueError("insufficient funds")
        self._balance -= amount
```

The important design ideas are more important than the exact implementation:

- balance is controlled by the object rather than freely mutated by callers
- `deposit()` and `withdraw()` enforce domain rules
- `@property` exposes read access without exposing unrestricted write access
- exceptions represent invalid domain operations

You can extend the model with:

```text
SavingsAccount
CheckingAccount
Bank
Transaction
```

but only introduce inheritance when the relationship is genuinely **IS-A**. A `Bank` having accounts is composition, not inheritance.

### OOP exercise — Banking System

Design a small banking system.

Requirements:

- A `Bank` manages many accounts.
- Accounts have owners and balances.
- Customers can deposit and withdraw.
- Withdrawals must reject insufficient funds.
- Transfers move money between two accounts.
- Every transfer should produce a transaction record.
- Account balances should not be directly writable.
- Support at least two account types with different rules.

Decide:

1. Which classes do you need?
2. Where does composition make sense?
3. Where, if anywhere, does inheritance make sense?
4. Which attributes should be properties?
5. Which operations should raise exceptions?
6. What should be a dataclass?
7. Where would an `Enum` make sense?
8. Which behavior belongs on `Bank` versus `Account`?
9. How would you test transfers and failed withdrawals?

### OOP exercise — Inventory System

This is the second real-world foundation exercise.

Model:

- `Product`
- `Category`
- `Supplier`
- `Inventory`

A product should have:

```text
id
name
sku
price
quantity
category
supplier
```

Decide:

1. Which things should be classes?
2. Which relationships are **IS-A** and which are **HAS-A**?
3. Which behavior belongs on `Product`?
4. Which behavior belongs on `Inventory`?
5. Which objects are internal domain models?
6. Which incoming data would be better represented by Pydantic?
7. Which lookups need a dictionary?
8. Where would validation and exceptions live?

This model will later become the conceptual foundation for the separate FastAPI project, but the domain model should remain understandable without FastAPI.

---

### Practice exercises

1. Write a `Vector` class supporting `__add__`, `__sub__`, `__eq__`, and `__repr__`, so that `Vector(1, 2) + Vector(3, 4) == Vector(4, 6)` is `True`.
2. Add a `@property` called `magnitude` to your `Vector` class that computes `(x**2 + y**2) ** 0.5`.
3. Define an abstract `Shape` base class with an abstract `area()` method, then implement `Rectangle` and `Circle` subclasses.
4. Complete the **Banking System** exercise above, focusing on domain rules and class responsibilities.
5. Complete the **Inventory System** exercise above, focusing on modelling and choosing between regular classes, dataclasses, and Pydantic models. supporting `__add__`, `__sub__`, `__eq__`, and `__repr__`, so that `Vector(1, 2) + Vector(3, 4) == Vector(4, 6)` is `True`.
2. Add a `@property` called `magnitude` to your `Vector` class that computes `(x**2 + y**2) ** 0.5`.
3. Define an abstract `Shape` base class with an abstract `area()` method, then implement `Rectangle` and `Circle` subclasses.

---

## Module 5 — Typing Like a Senior Dev

### Basic annotations

```python
def greet(name: str, times: int = 1) -> str:
    return (name + " ") * times

age: int = 30
scores: list[int] = [90, 85, 77]           # modern syntax (3.9+) — no need for `List` from typing
lookup: dict[str, int] = {"a": 1}
```

Type hints are **not enforced at runtime** by Python itself — they're documentation plus fuel for external tools (`mypy`, `pyright`) and editors. `def greet(name: str)` will happily accept `greet(42)` at runtime; the type checker is what catches that before you ship it.

### Optional and Union

```python
from typing import Optional, Union

def find_user(user_id: int) -> Optional[dict]:      # same as `dict | None`
    ...

def parse(value: Union[int, str]) -> int:            # same as `int | str` (3.10+)
    ...

# modern syntax (3.10+) — prefer this over typing.Optional/Union
def find_user(user_id: int) -> dict | None:
    ...
```

### Generics — writing a function/class that works across types, safely

```python
from typing import TypeVar, Generic

T = TypeVar("T")

class Stack(Generic[T]):
    def __init__(self) -> None:
        self._items: list[T] = []

    def push(self, item: T) -> None:
        self._items.append(item)

    def pop(self) -> T:
        return self._items.pop()

int_stack: Stack[int] = Stack()
int_stack.push(5)
```

### `Protocol` — the formal version of duck typing

```python
from typing import Protocol

class SupportsRead(Protocol):
    def read(self, size: int = -1) -> bytes: ...

def load(source: SupportsRead) -> bytes:
    return source.read()
```

Unlike an `ABC`, a class doesn't need to explicitly inherit from a `Protocol` to satisfy it — if it has a matching `read()` method, it type-checks. This is structural typing: the formal expression of "if it walks like a duck."

### Running a type checker

```bash
pip install mypy
mypy your_module.py

# or, faster and increasingly the default choice in 2026:
pip install pyright
pyright your_module.py
```

Type checking is a **static** analysis step — it runs separately from your program, catching mismatches before you ever execute the code. Wiring it into CI is standard on any production Python codebase.

### `TYPE_CHECKING` — avoiding circular imports for type-only hints

```python
from typing import TYPE_CHECKING
if TYPE_CHECKING:
    from .models import User    # only imported for type checkers, never at runtime

def process(user: "User") -> None:   # quoted "forward reference" since User isn't imported at runtime
    ...
```

### Practice exercises

1. Add full type hints to your `Vector` class from Module 4.
2. Write a generic `Box[T]` class with a `value: T` and a method `get() -> T`.
3. Define a `Comparable` `Protocol` requiring `__lt__`, then write a generic `smallest(items: list[T]) -> T` function constrained to types implementing it.

---

## Module 6 — Pydantic v2

### 1. Why Pydantic — validating the boundary

A type hint (Module 5) is a promise nobody checks. Pydantic is the checkpoint that actually enforces it — every request body, config file, or queue message coming from *outside* your program gets inspected and coerced into a known shape before your business logic ever touches it, the way a border checkpoint verifies a passport before waving someone through.

```python
from pydantic import BaseModel

class User(BaseModel):
    name: str
    age: int
    email: str | None = None

u = User(name="Ada", age="30")     # "30" (str) coerced to 30 (int)
print(u.age, type(u.age))          # 30 <class 'int'>

User(name="Ada", age="not a number")   # raises ValidationError
```

The key difference from a plain type hint: Pydantic validates and coerces **at runtime**, every time a model is built.

### 2. `Field()` — constraints and metadata

```python
from pydantic import BaseModel, Field

class Product(BaseModel):
    name: str = Field(min_length=1, max_length=100)
    price: float = Field(gt=0)
    tags: list[str] = Field(default_factory=list)   # same mutable-default rule as Module 3 — use a factory
    sku: str = Field(alias="SKU")                     # accept a differently-named key on input
```

### 3. Validators — rules a type hint can't express

```python
from pydantic import BaseModel, field_validator, model_validator

class SignupForm(BaseModel):
    password: str
    password_confirm: str

    @field_validator("password")
    @classmethod
    def password_strength(cls, v: str) -> str:
        if len(v) < 8:
            raise ValueError("password must be at least 8 characters")
        return v

    @model_validator(mode="after")
    def passwords_match(self) -> "SignupForm":
        if self.password != self.password_confirm:
            raise ValueError("passwords don't match")
        return self
```

`field_validator` checks one field in isolation; `model_validator(mode="after")` runs once every field is already built, for rules that span multiple fields.

### 4. Nested models

```python
class Address(BaseModel):
    city: str
    country: str

class User(BaseModel):
    name: str
    address: Address

u = User(name="Ada", address={"city": "Nairobi", "country": "KE"})
print(u.address.city)   # Nairobi — the nested dict was parsed into an Address automatically
```

### 5. Serialization — going back out

```python
u.model_dump()                        # -> plain dict
u.model_dump_json()                   # -> JSON string
u.model_dump(exclude={"password"})    # drop sensitive fields on the way out
```

`model_dump()` / `model_dump_json()` are the v2 names — if you see `.dict()` or `.json()` in older code or tutorials, that's v1.

### 6. `pydantic-settings` — typed config from the environment

```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    database_url: str
    debug: bool = False

    model_config = {"env_file": ".env"}

settings = Settings()   # reads DATABASE_URL / DEBUG from the environment or .env
```

This replaces scattered `os.environ.get(...)` calls with one typed object — a missing or malformed env var fails loudly at startup instead of silently blowing up three requests into production.

### 7. `model_config` / `ConfigDict` — the one you'll use constantly

```python
from pydantic import BaseModel, ConfigDict

class UserOut(BaseModel):
    model_config = ConfigDict(from_attributes=True)   # build directly from an object's attributes, not just dicts
    id: int
    name: str

# UserOut.model_validate(some_sqlalchemy_user_instance)
```

`from_attributes=True` (the v2 rename of v1's `orm_mode`) is what lets you build a Pydantic model straight from a SQLAlchemy object — you'll reach for this constantly once Module 11 wires up the database.

### Practice exercises

1. Build an `Order` model with `items: list[str]` and `total: float = Field(gt=0)`, plus a `model_validator` that raises if `items` is empty.
2. Write a `Settings` class using `pydantic-settings` that reads `API_KEY` from the environment with no default, so startup fails immediately if it's missing.
3. Given nested `Author` and `Book` models, write a function that returns a `Book` instance's `model_dump_json(exclude={"author"})` — then note in one line why you'd want to strip nested data like that on the way out.

---

---

## Module 5 — Exceptions & File Handling

### `try / except / else / finally`

Catch exceptions you can meaningfully handle.

```python
try:
    value = int(user_input)
except ValueError:
    print("Invalid number")
else:
    print(f"Got {value}")
finally:
    print("cleanup")
```

- `try` — operation that may fail
- `except` — recovery/handling
- `else` — success path
- `finally` — cleanup that must happen either way

### Raising exceptions

```python
def withdraw(balance, amount):
    if amount > balance:
        raise ValueError("insufficient funds")
    return balance - amount
```

Raise specific exceptions when the caller needs to react differently.

### Custom exceptions

```python
class InsufficientFundsError(Exception):
    pass
```

Custom exceptions are useful for domain-specific failures.

### Exception chaining

```python
try:
    config = load_config()
except OSError as exc:
    raise ConfigError("could not load configuration") from exc
```

The original cause remains available for debugging.

### File I/O

Use a context manager:

```python
with open("data.txt") as file:
    content = file.read()
```

For large files, iterate lazily:

```python
with open("huge.log") as file:
    for line in file:
        process(line)
```

Common modes:

```text
r   read
w   write, replacing existing content
a   append
rb  read binary
wb  write binary
```

### `pathlib`

```python
from pathlib import Path

path = Path("data") / "users.json"

path.exists()
path.is_file()
path.is_dir()
path.read_text()
path.write_text()
path.mkdir()
```

Use `Path` rather than manually joining path strings.

---

## Module 6 — Modules & Packages

### Modules

A module is a Python file.

```python
# helpers.py
def slugify(value):
    ...
```

```python
from helpers import slugify
```

Prefer explicit imports over wildcard imports.

### Packages and `__init__.py`

A package groups related modules.

```text
inventory/
    __init__.py
    models.py
    services.py
```

`__init__.py` can expose selected names or configure the package, but it can also be empty.

### `__name__ == "__main__"`

```python
def main():
    print("running")


if __name__ == "__main__":
    main()
```

This runs `main()` when the file is executed directly, not when it is imported.

### Virtual environments and `uv`

The project workflow used in this guide is based on `uv`.

```bash
uv init
uv add fastapi
uv add pytest
uv run python app.py
uv run pytest
```

Typical files:

```text
pyproject.toml
uv.lock
.venv/
```

`pip` remains worth knowing:

```bash
pip install requests
```

but use the project's declared dependency workflow rather than manually maintaining environments.

### Project organisation

A sensible application can start with:

```text
inventory-app/
├── pyproject.toml
├── uv.lock
├── README.md
├── src/
│   └── inventory/
│       ├── __init__.py
│       ├── models.py
│       ├── services.py
│       ├── repositories.py
│       └── main.py
└── tests/
```

The important principle is separation of responsibilities, not memorising one directory tree.

---

## Module 7 — Advanced Python

### Iterators, iterables and generators

Already covered earlier. Keep the mental model:

```text
iterable → can produce an iterator
iterator → produces values with __next__()
generator → convenient lazy iterator
```

### Descriptors

Descriptors control attribute access through:

```python
__get__
__set__
__delete__
```

`property` is built on descriptor machinery.

Descriptors appear underneath frameworks and ORMs, but you usually consume them rather than implement them.

### Metaclasses

A metaclass controls how classes are created.

```python
class Meta(type):
    pass


class MyClass(metaclass=Meta):
    pass
```

Normally:

```text
class → creates instances
metaclass → creates classes
```

`type` is the default metaclass.

Metaclasses are powerful but uncommon in application code. Prefer decorators, registries, class methods, or `__init_subclass__` when they solve the problem more simply.

### Advanced typing

```python
from typing import TypeVar, Protocol

T = TypeVar("T")


def first(items: list[T]) -> T:
    return items[0]
```

Protocols provide structural typing:

```python
class SupportsRead(Protocol):
    def read(self, size: int = -1) -> bytes:
        ...
```

A class does not need to inherit from the protocol to satisfy it for static type checking.

### Pattern matching

```python
match command:
    case {"action": "create", "name": name}:
        create(name)
    case {"action": "delete", "id": id}:
        delete(id)
    case _:
        raise ValueError("unknown command")
```

Use it when matching structure makes the code clearer.

### `asyncio`

```python
import asyncio


async def fetch():
    await asyncio.sleep(1)
    return "done"
```

```python
result = asyncio.run(fetch())
```

Concurrent I/O:

```python
results = await asyncio.gather(
    fetch_a(),
    fetch_b(),
)
```

`await` gives control back while an operation waits. Async is particularly useful for I/O-heavy applications; it does not automatically speed up CPU-bound Python code.

---

## Module 8 — Standard Library

Before adding a dependency, check whether the standard library already solves the problem.

### `itertools`

Useful lazy tools:

```python
chain()
islice()
product()
permutations()
combinations()
groupby()
```

### `functools`

Useful tools:

```text
wraps()
partial()
cache()
lru_cache()
singledispatch()
reduce()
```

Use `wraps` in decorators, `partial` to pre-fill arguments, and caching when repeated deterministic work is expensive.

### `pathlib`

Covered in Module 5.

### `datetime`

```python
from datetime import datetime, timezone

now = datetime.now(timezone.utc)
```

Prefer timezone-aware datetimes for real-world timestamps.

### `collections`

Important tools:

```text
Counter
defaultdict
deque
ChainMap
namedtuple
```

Choose based on the operations required.

### `heapq`

A min-heap:

```python
import heapq

heapq.heappush(heap, item)
heapq.heappop(heap)
```

Useful for priority queues and top-N problems.

### `bisect`

```python
from bisect import insort

values = [1, 3, 5]
insort(values, 4)
```

Finding an insertion position is efficient; inserting into a list still shifts later elements.

### `statistics`

```python
from statistics import mean, median

mean(values)
median(values)
```

### `json`

```python
import json

text = json.dumps(data)
data = json.loads(text)
```

For files:

```python
with open("data.json") as file:
    data = json.load(file)
```

### `csv`

`DictReader` is particularly useful for large CSVs:

```python
import csv

with open("users.csv", newline="") as file:
    reader = csv.DictReader(file)

    for row in reader:
        process(row)
```

### `sqlite3`

SQLite provides an embedded relational database.

```python
import sqlite3

with sqlite3.connect("app.db") as connection:
    connection.execute(
        "SELECT * FROM users WHERE id = ?",
        (user_id,),
    )
```

Always parameterize SQL values.

### `logging`

```python
import logging

logger = logging.getLogger(__name__)

logger.info("Application started")
logger.warning("Unexpected state")
logger.error("Operation failed")
```

Common levels:

```text
DEBUG
INFO
WARNING
ERROR
CRITICAL
```

Use logging for application diagnostics instead of scattering `print()` statements through production code.

### `typing`

Modern Python commonly uses:

```python
str | None
list[str]
dict[str, int]
```

Also know:

```text
Any
Literal
TypeVar
Protocol
TypedDict
Callable
Iterable
Iterator
```

---

## Module 9 — Testing & Tooling

### pytest

```python
def add(a, b):
    return a + b
```

```python
def test_add():
    assert add(2, 3) == 5
```

Run with:

```bash
uv run pytest
```

Test behavior rather than implementation details.

### Parametrization

```python
import pytest


@pytest.mark.parametrize(
    "a,b,expected",
    [
        (1, 2, 3),
        (10, 5, 15),
        (-1, 1, 0),
    ],
)
def test_add(a, b, expected):
    assert add(a, b) == expected
```

### `unittest`

Know the standard-library framework:

```python
import unittest


class TestAdd(unittest.TestCase):
    def test_add(self):
        self.assertEqual(add(2, 3), 5)
```

For new projects, `pytest` is often simpler.

### Mocking

Mock external dependencies when a real dependency would make a unit test slow, unreliable, or impossible.

```python
from unittest.mock import Mock

gateway = Mock()
gateway.charge.return_value = True
```

Do not mock everything. Prefer real behavior where practical.

### Ruff

```bash
uv run ruff check .
```

Use it to catch common code-quality problems.

### Black

Black is an opinionated formatter:

```bash
black .
```

If your project uses Ruff's formatter instead, choose one formatting workflow rather than running competing formatters.

### mypy / pyright

Static type checking catches type mismatches without executing the program.

```bash
mypy .
```

or:

```bash
pyright .
```

Wire type checking into CI once a project becomes substantial.

### Debugging

Use the simplest tool appropriate to the problem:

```text
logs → reproduce → inspect → breakpoint() → profiler
```

Python's built-in debugger starts with:

```python
breakpoint()
```

---

## Module 10 — Pythonic Best Practices

### Common idioms

Prefer:

```python
for index, item in enumerate(items):
    ...
```

over manual counters.

Use:

```python
for name, score in zip(names, scores):
    ...
```

Use:

```python
if users:
    ...
```

rather than `if len(users) > 0`.

Use `any()` and `all()` where they express the intent clearly.

### EAFP vs LBYL

Python often favors:

```python
try:
    value = data[key]
except KeyError:
    ...
```

over checking every condition first.

Use whichever style makes the behavior clearer.

### Performance

Measure before optimizing.

High-value improvements often come from:

- choosing the right data structure
- avoiding unnecessary copies
- streaming large datasets
- moving repeated work outside loops
- using database queries appropriately
- caching genuinely repeated expensive work

Remember common average-case costs:

```text
list membership      O(n)
set membership       O(1)
dict lookup          O(1)
deque append/popleft O(1)
heap push/pop        O(log n)
```

### Common pitfalls

Watch for:

- mutable default arguments
- shallow vs deep copies
- late-binding closures
- modifying collections while iterating
- broad exception handling
- accidentally materializing generators
- naive datetime handling
- shared mutable class attributes
- circular imports
- unnecessary inheritance

### Thinking in Python

When solving a problem, ask:

1. What is the data?
2. What operations do I need?
3. Which data structure makes those operations cheap?
4. Do I need all the data in memory?
5. Can the standard library solve this?
6. Where should validation happen?
7. Where should side effects happen?
8. What should happen when something fails?

The goal is not to use the most Python features. It is to choose the simplest construct that expresses the intent clearly.

### Standard-library-first mindset

Before adding a dependency, check:

```text
collections
itertools
functools
pathlib
datetime
json
csv
sqlite3
logging
```

### Common interview patterns

Frequency counting:

```python
from collections import Counter
Counter(values)
```

Grouping:

```python
from collections import defaultdict

groups = defaultdict(list)
```

Lookup:

```python
lookup = {item["id"]: item for item in items}
```

Deduplication:

```python
unique = set(values)
```

Top N:

```python
counter.most_common(10)
```

Queue:

```python
from collections import deque
queue = deque()
```

Priority queue:

```python
import heapq
heapq.heappush(heap, item)
```

Lazy processing:

```python
def process(items):
    for item in items:
        yield transform(item)
```

Recognize the underlying problem rather than memorizing recipes.

---

## Final Review Exercise — Python Foundations

Before moving to the separate FastAPI notebook, use the inventory and banking exercises to review the book as a whole.

For each system, be able to explain:

- why each class exists
- where composition is preferable to inheritance
- where a property protects an invariant
- where an Enum is appropriate
- where a dataclass fits
- where Pydantic belongs at a boundary
- which exceptions represent domain failures
- which data structures support the required operations
- how the code would be tested
- how the project would be organised

The separate FastAPI project will build on these decisions rather than replace them.

