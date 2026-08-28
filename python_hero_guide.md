# 🐍 Python 0-to-Hero: A Practical Guide for Developers

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

Python's packaging story historically fragmented: `pip` + `venv` (the built-in, bare-bones combo), `poetry`, `pipenv`, and `conda` all compete for the same job ie: managing dependencies and virtual environments. There isn't one single blessed tool the way some ecosystems have.

In 2026, **`uv`** (a fast, Rust-based tool) has become the converging standard — it replaces `pip`, `venv`, and most of what `poetry` does.

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

Two habits worth flagging early: the `for...else` / `while...else` clause (the `else` runs only if the loop was never `break`-ed out of), and the fact that Python's `for` only ever iterates over an iterable; there's no classic `for (i=0; i<n; i++)` form, `range()` is how you simulate it.

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

Note: unpacking happens at assignment or at the call site.

### 2. Comprehensions

```python
result = [x * 2 for x in numbers if x * 2 > 5]

# dict comprehension
squares = {x: x**2 for x in range(5)}

# set comprehension
uniques = {x % 3 for x in range(10)}
```

A comprehension builds a whole list, dict, or set in one expression instead of a multi-line loop with repeated `.append()` calls it reads almost like a sentence: *"give me `x * 2`, for every `x` in the list, but only when `x * 2` is bigger than 5."*

If a comprehension needs more than one `if` or one nested `for`, stop — write a real loop instead. Unreadable comprehensions are a classic Python code-smell.

### 3. Generators

### 3. Generators (A Quick Introduction)

Most beginners learn generators by memorizing one rule:

> Replace `return` with `yield`.

That works, but it doesn't explain **why generators exist**.

Think about watching a movie on Netflix.

Netflix doesn't download the entire movie before you press Play. It sends you small pieces as you need them. If it downloaded the whole movie first, you'd spend a long time waiting, and your device would need enough storage for the entire file.

Generators work the same way.

Instead of creating every value up front, they produce one value at a time, only when someone asks for the next one.

```python
def count_to_three():
    yield 1
    yield 2
    yield 3

numbers = count_to_three()

print(next(numbers))
print(next(numbers))
print(next(numbers))
```

Output

```text
1
2
3
```

Notice something interesting.

Calling `count_to_three()` doesn't execute the function immediately. Instead, Python returns a generator object that remembers where execution should begin.

Each call to `next()` runs the function until it reaches the next `yield`. At that point, Python pauses the function and remembers everything about its current state:

- local variables
- current line number
- loop position
- function arguments

The next call to `next()` continues exactly where execution stopped.

This is called **lazy evaluation**.

A normal function computes everything immediately.

A generator computes values only when they're needed.

For small collections, the difference isn't noticeable.

```python
numbers = [1, 2, 3, 4, 5]
```

But imagine processing a log file with 50 million lines.

Building a list means reading all 50 million lines into memory before your program can start working.

```python
lines = [line for line in open("server.log")]
```

A generator reads only one line at a time.

```python
def read_log(path):
    with open(path) as file:
        for line in file:
            yield line
```

Your program can begin processing immediately without waiting for the entire file to load.

Generators are ideal for

- reading large files
- streaming API responses
- database records
- infinite sequences
- pipelines where data is processed step by step

They are usually **not** the right choice when

- you need random access using indexes
- you need the length of the collection
- you'll iterate over the data many times
- the entire dataset is already small enough to fit comfortably in memory

A generator can usually be consumed only once.

```python
numbers = (x for x in range(5))

for n in numbers:
    print(n)

for n in numbers:
    print(n)
```

The second loop prints nothing because the generator has already been exhausted.

In this module, the important takeaway is simple:

- Lists store all values immediately.
- Generators produce values only when needed.
- This makes generators much more memory efficient for large or continuous streams of data.

In Module 6, we'll look behind the scenes and learn how generators are built on top of Python's iterator protocol, why `for` loops work with them automatically, how `yield` actually pauses a function, and when generators outperform lists in real applications.

### 4. Context managers (`with`)

```python
with open("file.txt") as f:
    data = f.read()
# file is guaranteed closed here, even if read() raised an exception
```

`with` guarantees that no matter what happens inside the block (even an exception ), the cleanup step runs afterward. It's used constantly in Python: files, database connections, locks, and HTTP sessions all lean on this pattern rather than reserving it for rare edge cases.

You can write own context managers wirh a function decorated @contextmanager or one class implementing `__enter__`/`__exit__`.

### 5. The walrus operator `:=`
It allows you to assign a value to a variable inside an expression

```python
# instead of:
data = fetch()
if data:
    process(data)

# you can inline the assignment:
if data := fetch():
    process(data)

# In a while loop
while True:
    user_input = input("Enter data (or 'quit'): ")
    if user_input == "quit":
        break
    print(f"Processing {user_input}")

# with :=
while (user_input := input("Enter data (or 'quit'): ")) != "quit":
    print(f"Processing {user_input}")

# List comprehensions
# Calculates f(x) once, saves it to 'value'
results = [value for x in data if (value := f(x)) > 0]

```

#### Syntax Rules & Style Tips:
- Use Parentheses: The walrus operator has low operator precedence. Always wrap the assignment expression in parentheses when combining it with operators like if or while conditions.
- Do Not Overuse: The official PEP 572 documentation warns against using the walrus operator if it harms readability. Use it to reduce complexity, not to make your code look clever.
- Invalid Names: You cannot use the walrus operator for plain global variable assignments where a regular statement is expected (e.g., x := 5 on its own line will throw a syntax error in many contexts).

### 6. `*args` and `**kwargs` (quick preview )

```python
def log(*args, **kwargs):
    print(args)    # tuple of positional args
    print(kwargs)  # dict of keyword args

log(1, 2, name="Ada")
# (1, 2)
# {'name': 'Ada'}
```

This is how Python functions accept "whatever you throw at me".

### Practice exercises

1. Write a generator `even_numbers(limit)` that lazily yields even numbers up to `limit`, and prove it's lazy by showing it doesn't compute anything until you call `next()` on it.

```python
def even_nums_gen(limit):
    for n in range(0, limit):
        yield n
```

2. Write one comprehension that takes a list of dicts `[{"name": "a", "age": 17}, {"name": "b", "age": 22}, ...]` and returns just the names of people 18 or older.

```python
data = [{"name": "a", "age": 17}, {"name": "b", "age": 22}, {"name": "c", "age": 18}, {"name": "d", "age": 17} ]
legal = [person["name"] for person in data if person["age"] >= 18]

```

3. Using unpacking (no indexing with `[0]`, `[1]`), write a function `describe_point(point)` that takes a 3-tuple `(x, y, z)` and returns a formatted string — capture `x, y` and let `z` be captured via a rest-pattern for a 4th+ optional dimension.

```python
def describe_point(point):
    x, y, z, *rest = point
    if rest:
        return f"x={x}, y={y}, z={z}, extra={rest}"
    return f"x={x}, y={y}, z={z}"

print(describe_point((1, 2, 3)))
print(describe_point((1, 2, 3, 4)))
print(describe_point((1, 2, 3, 4, 5)))
```

---

# Module 2 — Python Data Structures

Data structures determine how your data is stored, accessed, and modified. Choosing the right one makes code simpler, faster, and easier to maintain.

Python has many data structures, but most programs rely on just four built-in types:

- `list`
- `tuple`
- `dict`
- `set`

Each one is designed for a different purpose. Before learning their methods, it's important to understand when to use each.

| Type | Ordered | Mutable | Duplicates | Best Use |
|------|---------|---------|------------|----------|
| `list` | ✓ | ✓ | ✓ | General collections |
| `tuple` | ✓ | ✗ | ✓ | Fixed records |
| `dict` | ✓ | ✓ | Keys only | Fast key-value lookup |
| `set` | ✗ | ✓ | ✗ | Unique values |

---

## Lists

Lists are mutable sequences. They're Python's default collection type and are used to store ordered groups of objects.

```python
fruits = ["apple", "banana", "orange"]
```

Lists preserve insertion order, allow duplicate values, and can store different object types.

```python
items = ["Alice", 25, True]
```

Although this is allowed, storing related values of different types in a list is usually a sign that another data structure, such as a tuple or dataclass, would be a better choice.

### Creating Lists

```python
numbers = [1, 2, 3]

empty = []

letters = list("python")

zeros = [0] * 5
```

### Accessing Items

Lists are indexed from zero.

```python
letters = ["a", "b", "c"]

letters[0]      # "a"
letters[-1]     # "c"
```

Negative indexes count from the end.

Trying to access an index that doesn't exist raises an `IndexError`.

```python
letters[10]
```

### Updating Lists

Lists can be modified after creation.

```python
numbers = [1, 2, 3]

numbers[1] = 20

print(numbers)
```

```
[1, 20, 3]
```

### Common Operations

```python
numbers.append(4)

numbers.extend([5, 6])

numbers.insert(0, 0)

numbers.remove(3)

numbers.pop()

numbers.clear()
```

### Iterating

Iterate over the objects directly whenever possible.

```python
for number in numbers:
    print(number)
```

If you also need the index, use `enumerate()`.

```python
for index, number in enumerate(numbers):
    print(index, number)
```

Avoid using `range(len(...))` unless you specifically need index arithmetic.

### Membership Testing

Use the `in` operator to check whether an item exists.

```python
if "apple" in fruits:
    print("Found")
```

### Copying Lists

Assignment creates another reference to the same list.

```python
a = [1, 2, 3]

b = a
```

Modifying `b` also changes `a`.

```python
b.append(4)

print(a)
```

```
[1, 2, 3, 4]
```

To create a new list, make a copy.

```python
b = a.copy()

# or

b = a[:]
```

Both create a shallow copy.

### Sorting

Sort a list in place.

```python
numbers.sort()
```

Return a sorted copy without modifying the original.

```python
sorted(numbers)
```

Sort using a custom key.

```python
users.sort(key=lambda user: user.age)
```

### Performance

| Operation | Average Complexity |
|-----------|-------------------|
| Index | O(1) |
| Append | O(1) |
| Pop (end) | O(1) |
| Insert (front) | O(n) |
| Remove | O(n) |
| Search (`in`) | O(n) |

Lists are optimized for adding and removing items at the end. Frequent insertions at the beginning should use `collections.deque` instead.

### Quick Reference

| Method | Description |
|---------|-------------|
| `append(x)` | Add one item |
| `extend(iterable)` | Add multiple items |
| `insert(i, x)` | Insert at an index |
| `remove(x)` | Remove first matching value |
| `pop([i])` | Remove and return an item |
| `clear()` | Remove all items |
| `copy()` | Return a shallow copy |
| `sort()` | Sort in place |
| `reverse()` | Reverse the list |

### Shallow vs Deep Copy

Sometimes copying an object doesn't copy everything inside it.

A **shallow copy** copies only the outer object, while a **deep copy** copies the outer object and everything nested inside it.

```python
import copy

original = [[1], [2]]

shallow = original.copy()
deep = copy.deepcopy(original)
```

Suppose we modify the first nested list.

```python
shallow[0].append(99)

print(original)
print(shallow)
print(deep)
```

Output

```text
[[1, 99], [2]]
[[1, 99], [2]]
[[1], [2]]
```

The shallow copy shares the same nested lists as the original, so changes to nested objects are reflected in both.

The deep copy creates completely independent objects, so changes don't affect the original.

> **Rule of thumb:** Use a **shallow copy** for flat collections. Use a **deep copy** for nested collections that should not share data.

| Copy Type | Copies Outer Object | Copies Nested Objects |
|-----------|---------------------|-----------------------|
| Shallow (`copy()` or `copy.copy()`) | ✓ | ✗ |
| Deep (`copy.deepcopy()`) | ✓ | ✓ |

### When to Use

```python
numbers = [1, 2, 3]

copy = numbers.copy()
```

A shallow copy is enough because the list contains only immutable values.

```python
company = {
    "employees": [
        {"name": "Alice"},
        {"name": "Bob"}
    ]
}

copy = copy.deepcopy(company)
```

Use a deep copy when nested lists, dictionaries, sets, or custom objects should be completely independent.

---

## Tuples

Tuples are immutable sequences. Once created, their contents cannot be changed.

```python
point = (10, 20)
```

Like lists, tuples preserve order and support indexing and slicing.

```python
point[0]      # 10
point[-1]     # 20
```

Unlike lists, tuples cannot be modified.

```python
point[0] = 15
```

```text
TypeError: 'tuple' object does not support item assignment
```

### Why Use a Tuple?

Use a tuple when the values belong together and shouldn't change.

Examples include:

- Coordinates
- RGB colors
- Database rows
- Dates
- Function return values

```python
location = (1.2921, 36.8219)

color = (255, 255, 255)
```

A tuple communicates intent to other developers:

> "This data is fixed."

### Creating Tuples

```python
empty = ()

numbers = (1, 2, 3)

single = (1,)      # The comma is required

letters = tuple("python")
```

Without the comma, Python treats the value as the object itself.

```python
single = (1)

print(type(single))
```

```text
<class 'int'>
```

### Packing and Unpacking

Python automatically packs multiple values into a tuple.

```python
point = 10, 20
```

This is equivalent to

```python
point = (10, 20)
```

Unpacking assigns each value to a variable.

```python
x, y = point
```

Extended unpacking captures the remaining values.

```python
point = (1, 2, 3, 4)

x, y, *rest = point

print(rest)
```

```text
[3, 4]
```

This is useful when the number of values isn't fixed.

### Returning Multiple Values

Python functions often return multiple values.

```python
def divide(a, b):
    return a // b, a % b

quotient, remainder = divide(10, 3)
```

Although it looks like multiple values are returned, Python is actually returning a tuple.

### Named Tuples

A regular tuple stores values by position.

```python
user = ("Alice", 25)
```

Remembering what each position means becomes difficult as the tuple grows.

`namedtuple` gives each position a name.

```python
from collections import namedtuple

User = namedtuple("User", ["name", "age"])

user = User("Alice", 25)

print(user.name)
```

Named tuples are lightweight, immutable, and more readable than plain tuples.

For most new code, `dataclass` is often a better choice. We'll cover that later in this module.

### Performance

Tuples are generally:

- slightly smaller than lists
- slightly faster to iterate
- hashable (if all their elements are hashable)

Because they're immutable, tuples can be used as dictionary keys and set elements.

```python
point = (10, 20)

locations = {
    point: "Nairobi"
}
```

A list cannot be used as a dictionary key.

```python
locations = {
    [10, 20]: "Nairobi"
}
```

```text
TypeError: unhashable type: 'list'
```

### Quick Reference

| Feature | List | Tuple |
|---------|------|-------|
| Ordered | ✓ | ✓ |
| Mutable | ✓ | ✗ |
| Allows duplicates | ✓ | ✓ |
| Supports indexing | ✓ | ✓ |
| Can be dict key | ✗ | ✓* |

\* Only if every element inside the tuple is hashable.

### When to Use

Use a **list** when the collection changes.

Use a **tuple** when the values represent a fixed record.

```python
shopping_cart = ["Laptop", "Mouse"]

location = (1.2921, 36.8219)
```

---

## Dictionaries

A dictionary stores data as **key-value pairs**. Each key is unique and maps to a value.

```python
user = {
    "name": "Alice",
    "age": 25,
    "active": True
}
```

Unlike lists, which use numeric indexes, dictionaries use keys to access values.

```python
print(user["name"])
```

```text
Alice
```

### Why Use a Dictionary?

Use a dictionary when you need to associate one piece of data with another.

Some common examples include:

- User profiles
- Configuration settings
- API responses
- JSON data
- Counting and grouping data

If you find yourself searching a list to locate an object by a unique value, a dictionary is usually a better choice.

### Creating Dictionaries

Using curly braces:

```python
user = {
    "name": "Alice",
    "age": 25
}
```

Using the `dict()` constructor:

```python
user = dict(name="Alice", age=25)
```

Creating an empty dictionary:

```python
user = {}
```

### Accessing Values

Retrieve a value using its key.

```python
print(user["name"])
```

Accessing a missing key raises a `KeyError`.

```python
print(user["email"])
```

```text
KeyError: 'email'
```

If the key might not exist, use `get()`.

```python
email = user.get("email")
```
You can also provide a default value.

```python
email = user.get("email", "Not provided")
```

Using `get()` avoids checking if the key exists before reading it.
Use `get()` when the key is optional

### Adding and Updating Values

Assigning to a new key adds it to the dictionary.

```python
user["email"] = "alice@example.com"
```

Assigning to an existing key updates its value.

```python
user["age"] = 26
```

### Removing Values

Remove a key and return its value.

```python
email = user.pop("email")
```

Remove the last inserted key-value pair.

```python
user.popitem()
```

Delete a specific key.

```python
del user["age"]
```

Remove everything.

```python
user.clear()
```

### Checking if a Key Exists

Use the `in` operator.

```python
if "name" in user:
    print("Found")
```

Avoid checking values when you only care about the key.

```python
# Good
"name" in user

# Avoid
"name" in user.keys()
```

### Iterating Over Dictionaries

Loop through the keys.

```python
for key in user:
    print(key)
```

Loop through the values.

```python
for value in user.values():
    print(value)
```

Loop through both keys and values.

```python
for key, value in user.items():
    print(key, value)
```

`items()` is the most common approach when both are needed.

### Dictionary Views

The methods `keys()`, `values()`, and `items()` return **view objects**.

```python
user.keys()

user.values()

user.items()
```

Views reflect changes made to the dictionary.

```python
user = {"name": "Alice"}

keys = user.keys()

user["age"] = 25

print(keys)
```

```text
dict_keys(['name', 'age'])
```

### Dictionary Comprehensions

Dictionary comprehensions create dictionaries from an iterable.

```python
squares = {
    x: x ** 2
    for x in range(5)
}
```

Output

```python
{
    0: 0,
    1: 1,
    2: 4,
    3: 9,
    4: 16
}
```

They follow the same pattern as list comprehensions.

```python
{
    key: value
    for ...
    if ...
}
```

### Merging Dictionaries

Create a new dictionary using the merge operator.

```python
defaults = {"theme": "light"}

user = {"language": "en"}

settings = defaults | user
```

Update an existing dictionary.

```python
defaults |= user
```

You can also use `update()`.

```python
defaults.update(user)
```

### Nested Dictionaries

Dictionaries can store any object, including other dictionaries.

```python
user = {
    "name": "Alice",
    "address": {
        "city": "Nairobi",
        "country": "Kenya"
    }
}
```

Access nested values by chaining keys.

```python
user["address"]["city"]
```

### Key Requirements

Dictionary keys must be **hashable**.

This means the key must be immutable.

Common hashable types include:

- `str`
- `int`
- `float`
- `bool`
- `tuple` (if all its elements are hashable)

Lists, dictionaries, and sets cannot be used as keys.

```python
data = {
    [1, 2]: "numbers"
}
```

```text
TypeError: unhashable type: 'list'
```

### Performance

| Operation | Average Complexity |
|-----------|-------------------|
| Lookup | O(1) |
| Insert | O(1) |
| Update | O(1) |
| Delete | O(1) |
| Membership (`in`) | O(1) |

These operations are fast because dictionaries are implemented as **hash tables**.

You don't need to understand the implementation details to use dictionaries effectively, but it's useful to remember that dictionary operations are generally much faster than searching through a list.

### Common Methods

| Method | Description |
|---------|-------------|
| `get(key)` | Return a value or a default |
| `setdefault(key, default)` | Return the value for a key, creating it if needed |
| `update(other)` | Merge another mapping |
| `pop(key)` | Remove and return a value |
| `popitem()` | Remove the last inserted item |
| `keys()` | Return a view of the keys |
| `values()` | Return a view of the values |
| `items()` | Return key-value pairs |
| `clear()` | Remove all items |
| `copy()` | Return a shallow copy |

### When to Use

Use a dictionary when:

- Each item has a unique identifier.
- You need fast lookups.
- Data is naturally represented as key-value pairs.
- You're working with JSON or API responses.

Examples include:

```python
user = {
    "id": 1,
    "name": "Alice"
}

config = {
    "debug": True,
    "port": 8000
}

scores = {
    "Alice": 95,
    "Bob": 87
}
```

---

## Sets

A set is an unordered collection of unique values.

```python
numbers = {1, 2, 3, 4}
```

Unlike lists, sets:

- don't allow duplicates
- aren't indexed
- are optimized for fast membership testing

### Creating Sets

```python
numbers = {1, 2, 3}

empty = set()      # Not {}
```

`{}` creates an empty dictionary, not a set.

Remove duplicates from a collection:

```python
numbers = [1, 2, 2, 3, 3, 4]

unique = set(numbers)

print(unique)
```

```
{1, 2, 3, 4}
```

### Adding and Removing Values

```python
fruits = {"apple", "banana"}

fruits.add("orange")

fruits.remove("banana")

fruits.discard("grape")      # No error if missing

item = fruits.pop()
```

`remove()` raises a `KeyError` if the value doesn't exist.

`discard()` does nothing if it's missing.

### Membership Testing

Checking whether an item exists is one of the main reasons to use a set.

```python
if "apple" in fruits:
    print("Found")
```

Membership checks in a set are much faster than searching through a list, especially for large collections.

### Set Operations

Sets support common mathematical operations.

Union combines values from both sets.

```python
a = {1, 2, 3}
b = {3, 4, 5}

a | b
```

```
{1, 2, 3, 4, 5}
```

Intersection returns values found in both.

```python
a & b
```

```
{3}
```

Difference returns values only in the left set.

```python
a - b
```

```
{1, 2}
```

Symmetric difference returns values found in either set, but not both.

```python
a ^ b
```

```
{1, 2, 4, 5}
```

The equivalent methods are:

```python
a.union(b)

a.intersection(b)

a.difference(b)

a.symmetric_difference(b)
```

### Frozen Sets

A `frozenset` is an immutable version of a set.

```python
permissions = frozenset({"read", "write"})
```

Because it's immutable, it can be used as a dictionary key or as an element inside another set.

### Quick Reference

| Method | Description |
|---------|-------------|
| `add(x)` | Add a value |
| `remove(x)` | Remove a value |
| `discard(x)` | Remove if present |
| `pop()` | Remove and return an arbitrary value |
| `clear()` | Remove all values |
| `copy()` | Return a shallow copy |

### When to Use

Use a set when:

- values must be unique
- you need fast membership checks
- you're comparing two collections
- you're removing duplicates

Examples:

```python
visited = set()

allowed_extensions = {"jpg", "png", "gif"}

unique_users = set(user_ids)
```

---

## Slicing

Slicing returns a portion of a sequence.

```python
sequence[start:stop:step]
```

- `start` is inclusive.
- `stop` is exclusive.
- `step` defaults to `1`.

```python
numbers = [0, 1, 2, 3, 4]

numbers[1:4]
```

```
[1, 2, 3]
```

If `start` is omitted, slicing begins from the start.

```python
numbers[:3]
```

```
[0, 1, 2]
```

If `stop` is omitted, slicing continues to the end.

```python
numbers[2:]
```

```
[2, 3, 4]
```

### Using Steps

The third value controls how many elements to skip.

```python
numbers[::2]
```

```
[0, 2, 4]
```

A negative step traverses the sequence in reverse.

```python
numbers[::-1]
```

```
[4, 3, 2, 1, 0]
```

### Negative Indexes

Negative indexes count from the end.

```python
letters = ["a", "b", "c", "d"]

letters[-1]
```

```
'd'
```

They can also be used when slicing.

```python
letters[-2:]
```

```
['c', 'd']
```

### Copying a Sequence

A full slice creates a shallow copy.

```python
copy = numbers[:]
```

This is equivalent to:

```python
copy = numbers.copy()
```

for lists.

### Slice Assignment

Slices can also replace multiple values at once.

```python
numbers = [1, 2, 3, 4]

numbers[1:3] = [20, 30]

print(numbers)
```

```
[1, 20, 30, 4]
```

You can also remove a range of values.

```python
del numbers[1:3]
```

### Quick Reference

| Expression | Result |
|------------|--------|
| `a[:]` | Copy the sequence |
| `a[:n]` | First `n` items |
| `a[n:]` | Everything after `n` |
| `a[:-1]` | Everything except the last item |
| `a[-1]` | Last item |
| `a[::2]` | Every second item |
| `a[::-1]` | Reverse the sequence |

---

## The `collections` Module

The `collections` module provides specialized container types that solve common problems more efficiently than the built-in data structures.

Import only what you need.

```python
from collections import Counter, defaultdict, deque
```

### Counter

A `Counter` counts how many times each value appears in an iterable.

```python
from collections import Counter

letters = Counter("mississippi")

print(letters)
```

```
Counter({'i': 4, 's': 4, 'p': 2, 'm': 1})
```

Get the count for a specific value.

```python
letters["i"]
```

Find the most common values.

```python
letters.most_common(2)
```

```
[('i', 4), ('s', 4)]
```

## `Counter.most_common()` and Ties

`Counter.most_common(n)` returns the `n` most frequent items.

```python
from collections import Counter

logs = ["INFO", "ERROR", "INFO", "WARNING", "ERROR"]

counts = Counter(logs)

print(counts.most_common(2))
```

Output

```python
[('INFO', 2), ('ERROR', 2)]
```

### A Note on Ties

`most_common()` sorts items by frequency, but **if multiple items have the same count, you should not rely on their order**.

If your application requires a deterministic order (for example, alphabetical order when counts are equal), sort the results yourself.

```python
from collections import Counter

counts = Counter(logs)

result = sorted(
    counts.items(),
    key=lambda item: (-item[1], item[0])
)

print(result)
```

Output

```python
[
    ('ERROR', 2),
    ('INFO', 2),
    ('WARNING', 1),
]
```

Here:

- `-item[1]` sorts by count in descending order.
- `item[0]` breaks ties alphabetically.

> **Best Practice**
>
> `Counter.most_common()` is ideal for quick summaries and analytics. In production systems where the order of tied items matters (e.g., reports, leaderboards, APIs, or tests), define an explicit tie-breaking rule using `sorted()`.

A `Counter` is often cleaner than manually counting values with a dictionary.

---

### defaultdict

A `defaultdict` is like a normal dictionary, but it automatically creates a default value when you access a missing key.

```python
from collections import defaultdict

scores = defaultdict(list)
```

The `list` tells Python:

> "If a key doesn't exist yet, create an empty list."

Now you can add values without first checking if the key exists.

```python
scores["Alice"].append(90)
scores["Alice"].append(95)

print(scores)
```

```text
defaultdict(<class 'list'>, {'Alice': [90, 95]})
```

With a normal dictionary, this would raise a `KeyError`.

```python
scores = {}

scores["Alice"].append(90)
```

Instead, you'd have to create the list yourself.

```python
scores = {}

if "Alice" not in scores:
    scores["Alice"] = []

scores["Alice"].append(90)
```

`defaultdict` does this automatically.

### Common Default Values

Empty list

```python
from collections import defaultdict

groups = defaultdict(list)
```

Useful for grouping values.

Empty set

```python
unique = defaultdict(set)
```

Useful when values should be unique.

Zero

```python
counts = defaultdict(int)

counts["apple"] += 1
counts["apple"] += 1

print(counts)
```

```text
defaultdict(<class 'int'>, {'apple': 2})
```

Since `int()` returns `0`, counting becomes simple.

`defaultdict(int) starts as an empty dict, {}. The int you pass in isn't run right away; it's just stored as "the recipe to use if a key is ever missing." Nothing gets pre-filled.`

### When to Use

Use `defaultdict` when values need to be created automatically, such as:

- Grouping records
- Counting values
- Building lookup tables

It removes the need to check whether a key already exists before using it.

---

### deque

A `deque` (double-ended queue) is optimized for adding and removing items from both ends.

```python
from collections import deque

queue = deque([1, 2, 3])

queue.append(4)

queue.appendleft(0)
```

Remove items.

```python
queue.pop()

queue.popleft()
```

Unlike lists, these operations remain efficient even for large collections.

Use a `deque` when implementing:

- Queues
- Stacks
- Breadth-first search (BFS)
- Sliding windows

---

### namedtuple

A `namedtuple` is a tuple whose fields can be accessed by name instead of position.

```python
from collections import namedtuple

User = namedtuple("User", ["name", "age"])

user = User("Alice", 25)

print(user.name)
```

```
Alice
```

A regular tuple:

```python
user = ("Alice", 25)

print(user[0])
```

A named tuple:

```python
user.name
```

New code often uses `dataclass` instead because it's more flexible, but you'll still see `namedtuple` in existing codebases.

---

### ChainMap

A `ChainMap` combines multiple dictionaries into a single view.

```python
from collections import ChainMap

defaults = {"theme": "light"}
user = {"theme": "dark"}

settings = ChainMap(user, defaults)

print(settings["theme"])
```

```
dark
```

Keys are searched from left to right until a match is found.

`ChainMap` is useful when working with layered configuration, such as application defaults, environment variables, and user settings.

Use ChainMap when you want to search multiple dictionaries as if they were one, while preserving their priority and without copying or merging them. This is why it's commonly used for configuration, layered settings, and fallback values.

#### `ChainMap` vs `|=` vs `dict.update()`

- **`ChainMap`** → Links dictionaries **without copying**; changes to the original dictionaries are reflected.
- **`|=`** → **Merges and updates** a dictionary in-place by copying values from another dictionary.
- **`dict.update()`** → Does essentially the same thing as `|=`; it updates a dictionary in-place with another dictionary's values.

**In short:** `ChainMap` keeps dictionaries **linked and separate**, while `|=` and `update()` **copy values into one dictionary**.

### Choosing the Right Tool

| Type | Best Used For |
|------|---------------|
| `Counter` | Counting occurrences |
| `defaultdict` | Grouping values |
| `deque` | Queues and stacks |
| `namedtuple` | Lightweight immutable records |
| `ChainMap` | Combining multiple dictionaries |


## Data Classes

A data class is a class designed primarily to store data.

Instead of writing boilerplate code such as constructors and object representations yourself, Python can generate it automatically using the `@dataclass` decorator.

```python
from dataclasses import dataclass

@dataclass
class User:
    name: str
    age: int
```

Creating an object is the same as calling a function.

```python
user = User("Alice", 25)

print(user)
```

```text
User(name='Alice', age=25)
```

Without `@dataclass`, you'd have to write the constructor yourself.

```python
class User:
    def __init__(self, name, age):
        self.name = name
        self.age = age

user = User("Alice", 25)
```

For simple data containers, a data class is shorter, cleaner, and easier to read.

> **Note**
>
> The `@dataclass` syntax is called a **decorator**. For now, think of it as a feature that adds extra functionality to a class.

### Default Values

Fields can have default values.

```python
from dataclasses import dataclass

@dataclass
class User:
    name: str
    active: bool = True
```

```python
user = User("Alice")

print(user.active)
```

```text
True
```

### Mutable Defaults

Avoid using mutable objects as default values.

```python
from dataclasses import dataclass

@dataclass
class Team:
    members: list = []      # Avoid
```

All instances would share the same list.

Use `field(default_factory=...)` instead.

```python
from dataclasses import dataclass, field

@dataclass
class Team:
    members: list = field(default_factory=list)
```

Each instance now gets its own list.

### When to Use

Use a data class when an object mainly stores data.

Examples include:

- Users
- Products
- Orders
- Configuration
- API request and response models

Data classes reduce boilerplate while keeping your code readable.

--- 

## Enumerations (`Enum`)

An enumeration, or **enum**, is a collection of named constant values.

Instead of using strings or numbers directly in your code, an enum gives those values meaningful names.

```python
from enum import Enum

class Status(Enum):
    PENDING = "pending"
    APPROVED = "approved"
    REJECTED = "rejected"
```

Access enum members using dot notation.

```python
status = Status.APPROVED
```

### Why Use an Enum?

Enums make code more readable and help prevent invalid values.

Instead of writing:

```python
status = "approved"
```

use:

```python
status = Status.APPROVED
```

This makes it clear that `status` can only be one of the defined values.

### Accessing the Name and Value

Each enum member has a `name` and a `value`.

```python
print(Status.APPROVED.name)
```

```text
APPROVED
```

```python
print(Status.APPROVED.value)
```

```text
approved
```

### Comparing Enum Members

Compare enum members directly.

```python
if status == Status.APPROVED:
    print("Request approved.")
```

Avoid comparing the underlying values unless necessary.

```python
if status.value == "approved":
    ...
```

Comparing enum members is clearer and less error-prone.

### Using Enums in Functions

Enums work well with type hints.

```python
from enum import Enum

class Status(Enum):
    ACTIVE = "active"
    INACTIVE = "inactive"


def handle(status: Status):
    if status is Status.ACTIVE:
        print("User is active.")
```

The type hint makes it clear that `handle()` expects a `Status`, not just any string.

Passing a string still works at runtime unless you validate types, but type checkers such as MyPy and IDEs can detect mistakes before your code runs.

### Enum or String?

Instead of writing:

```python
def handle(status: str):
    if status == "active":
        ...
```

prefer:

```python
def handle(status: Status):
    if status is Status.ACTIVE:
        ...
```

Using an enum:

- documents the allowed values
- reduces typos
- improves auto-completion in IDEs
- works well with type checkers

### `==` vs `is`

You'll often see enums compared using `is`.

```python
if status is Status.ACTIVE:
    ...
```

Since each enum member is a singleton (only one instance exists), both `is` and `==` work.

```python
status is Status.ACTIVE    # True

status == Status.ACTIVE    # True
```

Using `is` emphasizes that you're comparing the enum member itself, not just its value.

### Iterating Over an Enum

Loop through all members.

```python
for status in Status:
    print(status.name, status.value)
```

Output

```text
PENDING pending
APPROVED approved
REJECTED rejected
```

### Creating an Enum from a Value

You can create an enum member from one of its values.

```python
status = Status("approved")

print(status)
```

```text
Status.APPROVED
```

If the value doesn't exist, Python raises a `ValueError`.

### Common Use Cases

Enums are useful whenever a value should come from a fixed set of options.

Examples include:

- Order status
- User roles
- HTTP methods
- Days of the week
- Card suits
- Log levels

Example:

```python
from enum import Enum

class HttpMethod(Enum):
    GET = "GET"
    POST = "POST"
    PUT = "PUT"
    DELETE = "DELETE"


def request(method: HttpMethod):
    ...
```

### Advanced Enum Types

Python also provides specialized enum classes for specific use cases.

- `IntEnum` – Enum members behave like integers.
- `StrEnum` *(Python 3.11+)* – Enum members behave like strings.
- `Flag` – Combine multiple flags using bitwise operators.
- `IntFlag` – A `Flag` whose members are also integers.

These are less common than `Enum` but are useful when working with numeric constants, strings, or bit flags.

### Quick Reference

| Member | Description |
|--------|-------------|
| `.name` | Member name |
| `.value` | Underlying value |
| `Enum(value)` | Create a member from a value |
| `for member in Enum` | Iterate over all members |

### When to Use

Use an enum when:

- a variable can only have a fixed set of values
- you want to avoid magic strings or numbers
- you want clearer code and better type hints

> **Developer Note**
>
> If you find yourself comparing the same strings repeatedly, such as `"active"`, `"pending"`, or `"approved"`, consider replacing them with an enum.

---

## Truth Value Testing (Truthiness)

In Python, every object has a truth value. This allows objects to be used directly in conditions without explicitly comparing them to `True` or `False`.

For example, instead of writing:

```python
if len(items) > 0:
    print("Items found")
```

Python programmers usually write:

```python
if items:
    print("Items found")
```

### Falsy Values

The following values evaluate to `False`:

- `False`
- `None`
- `0`
- `0.0`
- `0`
- `""` (empty string)
- `[]` (empty list)
- `()` (empty tuple)
- `{}` (empty dictionary)
- `set()` (empty set)
- `range(0)`

Everything else is considered truthy unless the object defines its own truth value.

### Examples

```python
if []:
    print("Won't run")

if [1]:
    print("Runs")
```

```python
if "":
    print("Won't run")

if "Python":
    print("Runs")
```

```python
if None:
    print("Won't run")
```

### Checking for `None`

Use `is`, not `==`, when checking for `None`.

`None` is a singleton, meaning only one instance ever exists, so identity comparison is the semantically correct check. 
`==` calls `__eq__`, which a class can override; a poorly implemented `__eq__` could make `obj == None` return `True` even when `obj` isn't `None`. 
`is` bypasses that and is also slightly faster, since it's a pointer comparison with no method dispatch.

```python
value = None

if value is None:
    print("No value")
```

Likewise, use `is not None` when checking that a value exists.

```python
if value is not None:
    print(value)
```

### Truthiness in Practice

Instead of:

```python
if len(users) == 0:
    ...
```

prefer:

```python
if not users:
    ...
```

Instead of:

```python
if len(users) > 0:
    ...
```

prefer:

```python
if users:
    ...
```

These forms are shorter and are the idiomatic way to write Python.

### Beware of `None`

Sometimes an empty value and `None` mean different things.

```python
name = ""

if not name:
    print("Falsy")
```

This also evaluates to `True` when:

```python
name = None
```

If you specifically want to check for `None`, write:

```python
if name is None:
    ...
```

### Quick Reference

| Expression | Result |
|------------|--------|
| `if items:` | Collection is not empty |
| `if not items:` | Collection is empty |
| `if value is None:` | Value is `None` |
| `if value is not None:` | Value is not `None` |

> **Developer Note**
>
> In Python, prefer relying on an object's truth value instead of checking its length or comparing it to `True` or `False`.S

---

## Pythonic Idioms

Python often provides simpler, more readable ways to solve common problems. Learning these idioms will help your code look and feel like idiomatic Python.

### Iterate Over Values

Instead of iterating over indexes, iterate over the values directly.

```python
# Avoid
for i in range(len(names)):
    print(names[i])
```

```python
# Preferred
for name in names:
    print(name)
```

Use `range(len(...))` only when you genuinely need the index.

---

### Iterate With an Index

When you need both the index and the value, use `enumerate()`.

```python
names = ["Alice", "Bob", "Charlie"]

for index, name in enumerate(names):
    print(index, name)
```

You can specify the starting index.

```python
for index, name in enumerate(names, start=1):
    print(index, name)
```

---

### Iterate Over Multiple Iterables

Use `zip()` to iterate over multiple iterables at the same time.

```python
names = ["Alice", "Bob"]
scores = [90, 85]

for name, score in zip(names, scores):
    print(name, score)
```

`zip()` stops when the shortest iterable is exhausted.

---

### Swapping Variables

Python supports swapping values without a temporary variable.

```python
a = 10
b = 20

a, b = b, a
```

---

### Membership Testing

Use the `in` operator instead of looping manually.

```python
if "Alice" in names:
    print("Found")
```

Instead of:

```python
found = False

for name in names:
    if name == "Alice":
        found = True
        break
```

---

### Dictionary Lookups

Use `get()` when a key might not exist.

```python
theme = config.get("theme", "light")
```

If the key is required, use indexing.

```python
user_id = payload["id"]
```

---

### List Comprehensions

List comprehensions provide a concise way to create lists.

Instead of:

```python
squares = []

for number in range(5):
    squares.append(number ** 2)
```

write:

```python
squares = [number ** 2 for number in range(5)]
```

You can also filter values.

```python
evens = [number for number in range(10) if number % 2 == 0]
```

Use list comprehensions for simple transformations. If the logic becomes difficult to read, use a regular `for` loop instead.

---

### Dictionary Comprehensions

Create dictionaries in a single expression.

```python
squares = {
    number: number ** 2
    for number in range(5)
}
```

---

### Set Comprehensions

Create sets using the same syntax.

```python
lengths = {
    len(word)
    for word in ["apple", "pear", "banana"]
}
```

---

# Python Comprehensions: Multiple `for` Clauses

Sometimes a comprehension can have **more than one `for`**.

Think of it like **going through boxes, then going through the things inside each box**.

```python
{
    resp["path"]
    for stat, resps in by_status_code.items()
    if stat >= 400
    for resp in resps
}
```

### Read it like a story

> For each **status group**,
> if the status is an error,
> go through each **response inside that group**,
> and take its `path`.

The longer version makes it easier to see:

```python
error_endpoints = set()

for stat, resps in by_status_code.items():
    if stat >= 400:
        for resp in resps:
            error_endpoints.add(resp["path"])
```

The comprehension is just a shorter way of writing the same thing.

### The pattern

```python
{
    value
    for outer_item in outer_collection
    if condition
    for inner_item in inner_collection
}
```

Think:

```text
📦 For each box
   ↓
   🔍 If the box is interesting
   ↓
   📋 Look at each thing inside the box
   ↓
   ➡️ Take the value you want
```

**Key idea:** the second `for` lets you **loop inside the thing you just found**.

---

### `any()` and `all()`

Use `any()` to check if at least one value is truthy.

```python
numbers = [0, 0, 5]

any(numbers)
```

```
True
```

Use `all()` to check if every value is truthy.

```python
numbers = [1, 2, 3]

all(numbers)
```

```
True
```

These functions are commonly used with generator expressions.

```python
any(score > 90 for score in scores)
```

---

### Sorting

Use `sorted()` when you need a new sorted collection.

```python
sorted_names = sorted(names)
```

Use `sort()` when you want to modify the existing list.

```python
names.sort()
```

Sort using a custom key.

```python
users.sort(key=lambda user: user.age)
```

---

### Unpacking

Unpacking makes working with sequences more readable.

```python
point = (10, 20)

x, y = point
```

Capture remaining values using `*`.

```python
numbers = [1, 2, 3, 4]

first, *rest = numbers
```

---

### Ignoring Values

Use `_` for values you don't need.

```python
x, _, z = (1, 2, 3)
```

This signals that the value is intentionally ignored.

---

### Chained Comparisons

Python supports comparing multiple values in a single expression.

Instead of:

```python
if x >= 0 and x <= 100:
    ...
```

write:

```python
if 0 <= x <= 100:
    ...
```

---

### Quick Reference

| Instead of | Prefer |
|------------|---------|
| `range(len(items))` | `for item in items` |
| Manual counter | `enumerate()` |
| Manual pairing | `zip()` |
| Manual search | `in` |
| Manual list building | List comprehension |
| Temporary variable swap | `a, b = b, a` |
| `len(items) == 0` | `not items` |
| `len(items) > 0` | `items` |
| `dict[key]` (optional key) | `dict.get(key)` |
| Manual sorting copy | `sorted()` |

---

# Module Summary

In this module, you learned Python's core data structures and the idioms used to work with them efficiently.

You should now be comfortable choosing the right data structure for a problem, understanding their trade-offs, and writing more idiomatic Python.

---

# Choosing the Right Data Structure

| If you need... | Use |
|---------------|-----|
| An ordered collection that can change | `list` |
| A fixed collection of values | `tuple` |
| Fast key-value lookups | `dict` |
| Unique values | `set` |
| Count occurrences | `Counter` |
| Automatically create missing values | `defaultdict` |
| A queue or stack | `deque` |
| A lightweight data object | `dataclass` |
| A fixed set of constants | `Enum` |

When in doubt:

- Use a **list** for general collections.
- Use a **dictionary** when data is naturally key-value.
- Use a **set** when uniqueness matters.
- Use a **tuple** when values should not change.

The specialized types in `collections` solve specific problems and should be used when they simplify your code.

---

# Performance Cheat Sheet

Average time complexity.

| Operation | `list` | `tuple` | `dict` | `set` |
|-----------|:------:|:-------:|:------:|:-----:|
| Index | O(1) | O(1) | — | — |
| Append | O(1) | — | — | O(1) |
| Insert (front) | O(n) | — | — | — |
| Remove | O(n) | — | O(1) | O(1) |
| Lookup by key | — | — | O(1) | O(1) |
| Membership (`in`) | O(n) | O(n) | O(1) | O(1) |

> **Note**
>
> Big O describes how an operation scales as the amount of data grows. It is a useful guideline, not an exact measure of runtime.

---

# Best Practices

- Choose the simplest data structure that solves the problem.
- Prefer readability over clever code.
- Use dictionaries for lookups instead of searching lists.
- Use sets for membership checks and removing duplicates.
- Use tuples to represent fixed records.
- Use `dataclass` for objects whose primary purpose is storing data.
- Use `Enum` instead of magic strings or numbers.
- Prefer Python's built-in functions and idioms over manual implementations.

---

# Exercises

## 1 — Employee Directory

Given the following data:

```python
employees = [
    {"id": 1, "name": "Alice", "department": "Engineering"},
    {"id": 2, "name": "Bob", "department": "Finance"},
    {"id": 3, "name": "Charlie", "department": "Engineering"},
    {"id": 4, "name": "Diana", "department": "HR"},
]
```

1. Build a lookup table keyed by employee ID.

```python
emp_by_id = {
    employee["id"]: employee 
    for employee in employees
}

```
2. Group employees by department.

```python
from collections import defaultdict

by_dept = defaultdict(list)
for employee in employees:
    by_dept[employee["department"]].append(employee)

```

3. Return the names of all engineers sorted alphabetically.
```python
engineers = sorted(
    employee["name"]
    for employee in by_dept["Engineering"]
)
```

---

## 2 — Log Analysis

Given a server log:

```python
logs = [
    "INFO",
    "ERROR",
    "INFO",
    "WARNING",
    "ERROR",
    "ERROR",
    "INFO",
]
```

Using the appropriate data structures:

- Count each log level.

```python
from collections import Counter

counts = Counter(logs)

print(counts)
```
- Determine the most common log level.
```python
Counter(logs).most_common(2)
```

- Return all unique log levels.
```python
set(logs)
```

Do not manually count occurrences.

---

## 3 — API Response Processing

Given:

```python
responses = [
    {"status": 200, "path": "/users"},
    {"status": 404, "path": "/users/100"},
    {"status": 200, "path": "/orders"},
    {"status": 500, "path": "/payments"},
    {"status": 200, "path": "/products"},
]
```

Build a summary that includes:

- requests per status code
- percentage of successful requests
- unique endpoints
- endpoints that returned errors

```python
from collections import defaultdict

by_status_code = defaultdict(list)

for resp in responses:
    by_status_code[resp["status"]].append(resp)

requests_per_status = {
    status: len(resps)
    for status, resps in by_status_code.items()
}

success_percentage = len(by_status_code[200]) / len(responses) * 100

unique_endpoints = {
    "/" + resp["path"].split("/")[1]
    for resp in responses
}

error_endpoints = {
    resp["path"]
    for status, resps in by_status_code.items()
    if status >= 400
    for resp in resps
}
```

---

## 4 — Access Control

Create an `Enum` representing user roles.

```text
ADMIN
EDITOR
VIEWER
```

Then create a `dataclass` representing a user.

Write a function:

```python
def can_delete(user):
    ...
```

Only administrators should be allowed to delete resources.

---

```python
from enum import Enum
from dataclasses import dataclass

class Roles(Enum):
    ADMIN = "admin"
    EDITOR = "editor"
    VIEWER = "viewer"

@dataclass
class User:
    name: str
    role: Roles

def can_delete(user: User):
    return user.role is Roles.ADMIN


user = User("Alice", Roles.VIEWER)
print(can_delete(user))
```

## 5 — Configuration Merge

Applications often load configuration from multiple sources.

Given:

```python
defaults = {
    "host": "localhost",
    "port": 8000,
    "debug": False,
}

environment = {
    "debug": True,
}

user = {
    "port": 9000,
}
```

Merge the configurations so that:

```
user > environment > defaults
```

Then explain why a dictionary is a better choice than a list for this problem.

```python
ChainMap(user, environment, defaults)
```

---

## 6 — Cache Design

Design an in-memory cache for recently viewed products.

Requirements:

- Fast lookup by product ID.
- Prevent duplicate entries.
- Keep only the 100 most recent items.
- Preserve viewing order.

**Questions**

1. Which data structures would you use?
2. Why?
3. Which operations should be O(1)?
4. Would a `list`, `set`, `dict`, `deque`, or a combination work best?

Focus on the design rather than the implementation.

Combine a dict for O(1) lookups and uniqueness with a deque(maxlen=100) to maintain viewing order and automatically evict the oldest item. Each data structure addresses a different requirement.
---

## 7— Event Processing Pipeline

You're processing millions of application events.

Each event contains:

```python
{
    "id": "...",
    "timestamp": "...",
    "user": "...",
    "type": "...",
}
```

Design a solution that can:

- quickly detect duplicate events
- count events by type
- group events by user
- retrieve an event by ID
- process events in arrival order

Choose the appropriate Python data structures for each requirement and justify your decisions.

---

## Challenge

You're asked to review the following code.

```python
users = []

for user in data:
    found = False

    for existing in users:
        if existing["id"] == user["id"]:
            found = True
            break

    if not found:
        users.append(user)
```

Refactor it to use more appropriate data structures and Pythonic idioms.

Your solution should be:

- simpler
- more readable
- more efficient

Explain the reasoning behind your design choices.

---

# Module 3 — Functions & Functional Programming

## Functions

Functions are first-class objects in Python. They can be assigned to variables, passed as arguments, returned from other functions, and stored in collections.

```python
def greet(name: str) -> str:
    return f"Hello, {name}!"
```

Python supports optional type hints for parameters and return values.

```python
def divide(a: float, b: float) -> float:
    return a / b
```

Type hints improve readability, IDE support, and static analysis tools such as MyPy, but they are not enforced at runtime.

Every function returns a value. If execution reaches the end of a function without a `return` statement, Python returns `None`.

```python
def log(message):
    print(message)

result = log("Hello")

print(result)
```

```text
Hello
None
```

Functions should have a single responsibility and return values instead of printing them whenever possible.

---

## Parameter Types

Python supports several kinds of parameters.

### Positional Parameters

Arguments are matched by position.

```python
def power(base, exponent):
    return base ** exponent

power(2, 3)
```

---

### Keyword Arguments

Arguments can also be passed by name.

```python
power(exponent=3, base=2)
```

Keyword arguments improve readability and allow arguments to be supplied in any order.

---

### Default Parameters

Parameters may define default values.

```python
def greet(name, greeting="Hello"):
    return f"{greeting}, {name}!"
```

```python
greet("Alice")
greet("Alice", "Hi")
```

---

### Mutable Default Arguments

Default arguments are evaluated once, when the function is defined.

```python
def add(item, items=[]):
    items.append(item)
    return items
```

```python
print(add(1))
print(add(2))
```

```text
[1]
[1, 2]
```

The same list is reused across function calls.

Use `None` instead.

```python
def add(item, items=None):
    if items is None:
        items = []

    items.append(item)
    return items
```

---

### Positional-Only Parameters

Use `/` to indicate parameters that can only be passed positionally.

```python
def divide(a, b, /):
    return a / b
```

```python
divide(10, 2)
```

```python
divide(a=10, b=2)
```

```
TypeError
```

This is mainly used by library authors to preserve API compatibility.

---

### Keyword-Only Parameters

Use `*` to indicate parameters that must be passed by name.

The * on its own is a divider in the parameter list — everything after it can only be passed by name (keyword=value), never by position.

```python
def connect(host, *, timeout=30):
    ...
```

```python
connect("localhost", timeout=10)
connect("localhost", 10) # TypeError
```

This improves readability when functions accept many optional parameters.

---

### Variable Positional Arguments (`*args`)

`*args` collects extra positional arguments into a tuple.

```python
def average(*numbers):
    return sum(numbers) / len(numbers)
```

```python
average(1, 2, 3, 4)
```

```
2.5
```

---

### Variable Keyword Arguments (`**kwargs`)

`**kwargs` collects extra keyword arguments into a dictionary.

```python
def build_user(**attributes):
    return attributes
```

```python
build_user(name="Alice", age=25)
```

```
{'name': 'Alice', 'age': 25}
```

---

### Argument Unpacking

Sequences can be unpacked into positional arguments.

```python
numbers = (2, 3)

power(*numbers)
```

Dictionaries can be unpacked into keyword arguments.

```python
options = {
    "base": 2,
    "exponent": 3
}

power(**options)
```

---

## Variable Scope (LEGB)

Python resolves variable names using the LEGB rule.

- **Local** – Current function.
- **Enclosing** – Outer function.
- **Global** – Module scope.
- **Built-in** – Python built-ins.

```python
name = "Global"

def outer():
    name = "Outer"

    def inner():
        name = "Inner"
        print(name)

    inner()

outer()
```

Python searches each scope in order until the name is found.

Use `global` to modify a module-level variable.

```python
count = 0

def increment():
    global count
    count += 1
```

Use `nonlocal` to modify a variable from an enclosing function.

```python
def counter():
    value = 0

    def increment():
        nonlocal value
        value += 1
        return value

    return increment
```

 ## Best practices

1. **Prefer explicit data flow over shared state.** Pass values in as arguments, return the new value out. A function that depends on and mutates a global is harder to test and reason about than one that just takes inputs and returns outputs.
2. **If you have `global` mutable state you keep updating from multiple functions, use a class instead.** What `global counter; counter += 1` is really asking for is usually `self.counter += 1` on some object — instance attributes don't need a keyword to rebind.
3. **Don't use `global` just to avoid a `return`.** If a function computes a value, return it; don't stash it in a global as a side channel.
4. **`nonlocal` for closures is fine and common** — it's the standard way to build a stateful function factory without a class. Don't feel like you're doing something wrong reaching for it there.
5. **A lot of `global` statements in one file is a smell** — usually points at code that should be encapsulated in a class or module-level singleton object instead of loose top-level variables.

**Rule of thumb:** `nonlocal` shows up now and then in legitimate closures; `global` should be rare enough that each use is a deliberate, named exception, not a habit.

---

## Lambda Functions

A lambda is a small anonymous function consisting of a single expression.

```python
square = lambda x: x * x
```

Equivalent to:

```python
def square(x):
    return x * x
```

Lambdas are commonly used for short callbacks.

```python
users.sort(key=lambda user: user.age)
```

Prefer a regular function if the logic is more than a simple expression.

---

## Closures

A closure is an inner function that remembers variables from its enclosing scope, even after the outer function has returned.

```python
def multiplier(factor):
    def multiply(number):
        return number * factor

    return multiply

double = multiplier(2)

print(double(5))
```

```
10
```

Closures are commonly used to build function factories and decorators.

---

## Decorators

A decorator is a function that takes another function, adds or modifies its behavior, and returns a new function.

```python
@decorator
def greet():
    ...
```

is equivalent to:

```python
def greet():
    ...

greet = decorator(greet)
```

The `@` syntax is simply syntactic sugar.

---

## Why Decorators Work

Decorators are possible because functions are **first-class objects** in Python. They can be:

- assigned to variables
- passed as arguments
- returned from other functions

```python
def greet():
    print("Hello")

say_hello = greet

say_hello()
```

Output

```text
Hello
```

---

## Returning Functions

A function can create and return another function.

```python
def outer():

    def inner():
        print("Hello")

    return inner


greet = outer()

greet()
```

Output

```text
Hello
```

Notice that `outer()` doesn't print anything itself. It returns `inner`, which is then assigned to `greet`.

This is the foundation of decorators.

---

## Closures

The returned function can still access variables from the outer function.

```python
def outer(name):

    def inner():
        print(f"Hello {name}")

    return inner


greet = outer("Alice")

greet()
```

Output

```text
Hello Alice
```

Even after `outer()` has finished executing, `inner()` remembers `name`.

This is called a **closure**.

---

## Building a Decorator

A decorator wraps another function.

```python
def log(func):

    def wrapper():
        print("Calling function...")
        func()
        print("Done.")

    return wrapper
```

Apply it manually:

```python
def greet():
    print("Hello")

greet = log(greet)

greet()
```

Output

```text
Calling function...
Hello
Done.
```

The original function still runs, but now additional behavior has been added.

---

## Using `@`

The previous example is more commonly written as:

```python
def log(func):

    def wrapper():
        print("Calling function...")
        func()
        print("Done.")

    return wrapper


@log
def greet():
    print("Hello")


greet()
```

Python performs:

```python
greet = log(greet)
```

behind the scenes.

---

## Accepting Any Arguments

Most decorators should work with any function signature.

```python
from functools import wraps

def log(func):

    @wraps(func)
    def wrapper(*args, **kwargs):
        print(f"Calling {func.__name__}")
        return func(*args, **kwargs)

    return wrapper
```

- `*args` accepts positional arguments.
- `**kwargs` accepts keyword arguments.
- `return` preserves the original function's return value.


---

## Preserving Metadata

Without `functools.wraps`, the wrapper replaces information about the original function.

```python
print(greet.__name__)
```

Without `@wraps`:

```text
wrapper
```

With `@wraps`:

```text
greet
```

`@wraps` also preserves the function's docstring and other metadata.

Use it in almost every decorator.

---

## Decorators with Arguments

Sometimes the decorator itself needs configuration.

```python
from functools import wraps

def repeat(times):

    def decorator(func):

        @wraps(func)
        def wrapper(*args, **kwargs):
            for _ in range(times):
                func(*args, **kwargs)

        return wrapper

    return decorator
```

Usage:

```python
@repeat(3)
def greet():
    print("Hello")
```

Output

```text
Hello
Hello
Hello
```

Notice there are now three functions:

1. `repeat()` receives the decorator arguments.
2. `decorator()` receives the function.
3. `wrapper()` executes the wrapped function.

---

### Stacking Multiple Decorators

##### Syntax

​```python
@decorator_one
@decorator_two
@decorator_three
def my_function():
    pass
​```

Equivalent to:

​```python
my_function = decorator_one(decorator_two(decorator_three(my_function)))
​```

## Order

- **Wrapping** happens bottom-up at definition time (closest decorator to the function wraps first).
- **Execution** happens outside-in at call time (topmost decorator's wrapper runs first, calls inward).

## Example

​```python
def bold(func):
    def wrapper(*args, **kwargs):
        return f"<b>{func(*args, **kwargs)}</b>"
    return wrapper

def italic(func):
    def wrapper(*args, **kwargs):
        return f"<i>{func(*args, **kwargs)}</i>"
    return wrapper

@bold
@italic
def greet(name):
    return f"Hello, {name}"

print(greet("Nyakio"))
# <b><i>Hello, Nyakio</i></b>
​```

Trace: `italic` wraps `greet` first → `bold` wraps that result. Calling `greet(...)` runs `bold`'s wrapper first, which calls into `italic`'s wrapper, which calls the real `greet`.

## Why order matters

Not cosmetic — changes behavior. E.g. `@app.route` + `@login_required`:
- auth check before routing logic, or
- routing logic before auth check

...depending on which is on top.

---

## Common Built-in Decorators

```python
@property
```

Expose a method as an attribute.

```python
@staticmethod
```

Method that doesn't use `self` or `cls`.

```python
@classmethod
```

Receives the class (`cls`) instead of an instance (`self`).

```python
@functools.cache
```

Caches function results indefinitely.

```python
@functools.lru_cache
```

Caches the most recently used function results.

These are covered in more detail in later modules.

---

## Mental Model

Think of a decorator as wrapping a function inside another function.

```text
greet()
    │
    ▼
wrapper()
    │
    ├── Before
    ├── greet()
    └── After
```

The original function doesn't change. Instead, Python replaces the function reference with the wrapper returned by the decorator.

---

## Higher-Order Functions

A higher-order function is a function that accepts another function, returns a function, or both.

```python
def apply(fn, value):
    return fn(value)

apply(abs, -5)
```

Decorators, callbacks, and functions such as `map()` and `filter()` are all examples of higher-order functions.

---

## Function Documentation

Use docstrings to document public functions.

```python
def square(number):
    """Return the square of a number."""
    return number * number
```

Docstrings should describe:

- what the function does
- its parameters
- its return value
- exceptions or side effects, if any

---

## Best Practices

- Keep functions focused on a single responsibility.
- Prefer returning values over printing them.
- Use descriptive function names.
- Add type hints to public APIs.
- Avoid mutable default arguments.
- Prefer keyword-only parameters for optional configuration.
- Use lambdas only for simple expressions.
- Document reusable functions with docstrings.

--- 

# Iteration & Lazy Evaluation

Iteration is the process of visiting each item in a collection.

Python provides a consistent iteration model that works across lists, tuples, dictionaries, sets, files, generators, and many other objects.

When you write:

```python
for item in collection:
    print(item)
```

Python isn't looping over the collection directly. Instead, it asks the collection for an **iterator**, then repeatedly requests the next value until there are no more values.

Understanding this model makes generators, comprehensions, and many built-in functions much easier to understand.

---

## Iterable vs Iterator

Although the terms are often used interchangeably, they are different.

### Iterable

An **iterable** is any object that can produce an iterator.

Examples include:

- `list`
- `tuple`
- `dict`
- `set`
- `str`
- `range`
- files
- generators

You can iterate over an iterable multiple times.

```python
numbers = [1, 2, 3]

for number in numbers:
    print(number)
```

---

### Iterator

An **iterator** is an object that produces one value at a time.

Unlike an iterable, an iterator remembers where it currently is.

Once exhausted, it cannot restart.

> An iterable is a data container or source that you can loop over, while an iterator is the stateful agent that actually fetches elements from that source one at a time

---

## `iter()`

Use `iter()` to create an iterator from an iterable.

```python
numbers = [1, 2, 3]

iterator = iter(numbers)
```

Now `iterator` keeps track of the current position.

---

## `next()`

Use `next()` to request the next value.

```python
numbers = [1, 2, 3]

iterator = iter(numbers)

print(next(iterator))
print(next(iterator))
print(next(iterator))
```

Output

```text
1
2
3
```

Requesting another value raises an exception.

```python
next(iterator)
```

```text
StopIteration
```

This is exactly how a `for` loop knows when to stop.

---

## What a `for` Loop Does

This code:

```python
for item in numbers:
    print(item)
```

behaves roughly like this:

```python
iterator = iter(numbers)

while True:
    try:
        item = next(iterator)
        print(item)
    except StopIteration:
        break
```

The `for` loop is simply a cleaner way of working with iterators.

---

## Custom Iterators

You can create your own iterator by implementing two methods:

- `__iter__()`
- `__next__()`

```python
class Count:

    def __init__(self, stop):
        self.current = 0
        self.stop = stop

    def __iter__(self):
        return self

    def __next__(self):
        if self.current >= self.stop:
            raise StopIteration

        self.current += 1
        return self.current
```

```python
for number in Count(3):
    print(number)
```

Output

```text
1
2
3
```

Writing custom iterators is uncommon because generators provide a simpler solution.

---

# Generators

A generator is a simpler way to create an iterator.

Instead of implementing `__iter__()` and `__next__()`, use the `yield` keyword.

```python
def count(stop):
    current = 1

    while current <= stop:
        yield current
        current += 1
```

```python
for number in count(3):
    print(number)
```

Output

```text
1
2
3
```

Whenever Python reaches `yield`, it:

- returns the current value
- pauses the function
- remembers its state

The next call resumes execution immediately after the `yield`.

---

## `yield` vs `return`

`return` ends a function.

```python
def example():
    return 1
```

`yield` pauses a function.

```python
def example():
    yield 1
    yield 2
    yield 3
```

Each call to `next()` resumes execution until the next `yield`.

---

## Generator Expressions

Generator expressions are the lazy equivalent of list comprehensions.

List comprehension:

```python
squares = [x * x for x in range(5)]
```

Generator expression:

```python
squares = (x * x for x in range(5))
```

Notice the parentheses instead of square brackets.

Generator expressions produce values only when needed.

---

# Generators vs Lists

This is one of the most important design choices you'll make in Python.

### Lists

Lists create every value immediately.

```python
numbers = [x for x in range(1_000_000)]
```

Advantages:

- Fast random access.
- Can be iterated multiple times.
- Support indexing and slicing.

Disadvantages:

- Every value is stored in memory.
- Large datasets can consume significant memory.

---

### Generators

Generators produce values only when requested.

```python
numbers = (x for x in range(1_000_000))
```

Advantages:

- Low memory usage.
- Suitable for large datasets.
- Can represent infinite sequences.
- Begin producing results immediately.

Disadvantages:

- Can only be consumed once.
- Do not support indexing or slicing.
- Values are not stored after being consumed.

---

## When to Use a Generator

Use a generator when:

- processing large files
- reading API responses incrementally
- streaming database records
- working with pipelines
- producing infinite sequences
- processing data one item at a time

Example:

```python
with open("large_file.txt") as file:
    for line in file:
        process(line)
```

The file is read one line at a time instead of loading the entire file into memory.

---

## When to Use a List

Use a list when you need:

- indexing
- slicing
- multiple passes over the data
- random access
- the complete collection in memory

---

## Generator Pipelines

Generators work well together.

```python
numbers = (x for x in range(100))

evens = (x for x in numbers if x % 2 == 0)

squares = (x * x for x in evens)

print(sum(squares))
```

Each stage processes values lazily without creating intermediate lists.

---

## Best Practices

- Prefer generators for large or streaming datasets.
- Prefer lists when random access or repeated iteration is required.
- Use generator expressions with functions such as `sum()`, `any()`, `all()`, `min()`, and `max()`.
- Use a generator function (`yield`) when the logic is more complex than a single expression.
- Don't convert a generator to a list unless you actually need all the values.

---

## Developer Note

A useful way to think about the difference:

- A **list** is a container that already holds all its values.
- A **generator** is a recipe that produces values only when asked.

Both can be used in a `for` loop, but only the generator computes each value on demand.

---
# Functional Programming

Python includes several built-in utilities for applying functions to data. Although they come from functional programming, they're commonly used in everyday Python code.

---

## `map()`

`map()` applies a function to every item in an iterable.

```python
numbers = [1, 2, 3, 4]

result = map(lambda x: x * 2, numbers)

print(list(result))
```

```text
[2, 4, 6, 8]
```

The equivalent list comprehension is:

```python
result = [x * 2 for x in numbers]
```

For simple transformations, list comprehensions are generally more readable.

Use `map()` when an existing function naturally describes the transformation.

```python
names = ["alice", "bob", "charlie"]

print(list(map(str.title, names)))
```

---

## `filter()`

`filter()` keeps only the items for which a function returns `True`.

```python
numbers = [1, 2, 3, 4, 5, 6]

evens = filter(lambda x: x % 2 == 0, numbers)

print(list(evens))
```

```text
[2, 4, 6]
```

The equivalent comprehension is:

```python
evens = [x for x in numbers if x % 2 == 0]
```

List comprehensions are often preferred because the condition is easier to read.

---

## `reduce()`

Unlike `map()` and `filter()`, `reduce()` is part of the `functools` module.

It repeatedly combines values into a single result.

```python
from functools import reduce

numbers = [1, 2, 3, 4]

total = reduce(lambda a, b: a + b, numbers)

print(total)
```

```text
10
```

This works by repeatedly applying the function.

```text
((1 + 2) + 3) + 4
```

Many common uses of `reduce()` have simpler built-in alternatives.

```python
sum(numbers)
max(numbers)
min(numbers)
all(numbers)
any(numbers)
```

Prefer these built-ins when available.

---

# The `functools` Module

`functools` provides utilities for working with functions.

---

## `partial()`

`partial()` creates a new function by fixing some of the arguments of an existing function.

```python
from functools import partial

def power(base, exponent):
    return base ** exponent

square = partial(power, exponent=2)

print(square(5))
```

```text
25
```

This is useful when repeatedly calling a function with the same arguments.

---

## `lru_cache`

`lru_cache` caches previous function results.

```python
from functools import lru_cache

@lru_cache
def fibonacci(n):
    if n < 2:
        return n

    return fibonacci(n - 1) + fibonacci(n - 2)
```

Repeated calls with the same arguments return the cached result instead of recomputing it.

This can dramatically improve performance for expensive or recursive functions.

```python
from functools import lru_cache
import time

@lru_cache(maxsize=128)
def get_user_data(user_id):
    # Simulate an expensive network request or database query
    time.sleep(2) 
    return {"user_id": user_id, "status": "Active"}

# First call: Takes 2 seconds (Cache Miss)
print(get_user_data(42))

# Second call: Returns instantly! (Cache Hit)
print(get_user_data(42))

```

---

## `cached_property`

`cached_property` computes a value once and stores the result.

```python
from functools import cached_property

class Report:

    @cached_property
    def data(self):
        print("Loading...")
        return load_data()
```

The first access computes the value.

Subsequent accesses reuse the cached result.

--- 

### lru_cache vs cached_property

This is useful for expensive calculations that do not change during an object's lifetime.
| | `cached_property` | `lru_cache` |
|---|---|---|
| Where it lives | one instance | the function itself, shared everywhere it's called |
| Takes arguments? | no (just `self`) | yes — different args = different cache slots |
| Eviction | none, until you delete it | oldest entries drop once `maxsize` is hit |
| Use on | a class property with no inputs | any function/method with hashable inputs |

**Rule of thumb:** `cached_property` for "this object's expensive attribute, computed once." `lru_cache` for "this function, memoized across many different inputs."

**Gotcha:** `lru_cache` on a method caches keyed on `(self, args)` — since it holds onto `self` to build that key, it can keep instances alive longer than expected (a small memory leak risk). `cached_property` doesn't have that problem since the cache lives *on* the instance.

---

## `singledispatch`

`singledispatch` allows a function to behave differently based on the type of its first argument.

```python
from functools import singledispatch

@singledispatch
def describe(value):
    print("Unknown type")


@describe.register
def _(value: int):
    print("Integer")


@describe.register
def _(value: str):
    print("String")
```

```python
describe(42)
describe("hello")
```

```text
Integer
String
```

This provides a clean alternative to long chains of `isinstance()` checks.

---

# Choosing the Right Tool

| Task | Recommended Tool |
|------|------------------|
| Transform every item | `map()` or a list comprehension |
| Filter items | `filter()` or a list comprehension |
| Reduce to one value | `sum()`, `max()`, `min()`, `any()`, `all()`, or `reduce()` |
| Preconfigure a function | `partial()` |
| Cache expensive results | `lru_cache` |
| Cache computed properties | `cached_property` |
| Dispatch by type | `singledispatch` |

---

# Best Practices

- Prefer comprehensions over `map()` and `filter()` when they improve readability.
- Prefer built-in functions such as `sum()` and `max()` over `reduce()` when possible.
- Use `partial()` to avoid repeatedly passing the same arguments.
- Cache only functions that are deterministic (the same input always produces the same output).
- Use `singledispatch` instead of long `if isinstance(...)` chains when behavior depends on type.

> **Developer Note**
>
> Python supports functional programming, but it is not a purely functional language. In practice, you'll often combine functional tools with Python's imperative and object-oriented features. The goal is readable, maintainable code—not using functional constructs everywhere.

---

# The `itertools` Module

The `itertools` module provides fast, memory-efficient tools for working with iterators.

Unlike lists, most `itertools` functions produce values lazily. This makes them ideal for processing large datasets and building iteration pipelines.

```python
from itertools import count
```

---

## `count()`

`count()` generates an infinite sequence of numbers.

```python
from itertools import count

counter = count(start=1)

print(next(counter))
print(next(counter))
print(next(counter))
```

```text
1
2
3
```

Useful for generating IDs or sequence numbers.

---

## `cycle()`

`cycle()` repeats an iterable indefinitely.

```python
from itertools import cycle

colors = cycle(["red", "green", "blue"])

print(next(colors))
print(next(colors))
print(next(colors))
print(next(colors))
```

```text
red
green
blue
red
```

---

## `repeat()`

`repeat()` produces the same value repeatedly.

```python
from itertools import repeat

for value in repeat("Hello", 3):
    print(value)
```

```text
Hello
Hello
Hello
```

---

## `chain()`

`chain()` joins multiple iterables into a single iterator.

```python
from itertools import chain

numbers = chain([1, 2], [3, 4], [5])

print(list(numbers))
```

```text
[1, 2, 3, 4, 5]
```

Unlike concatenating lists, `chain()` does not create a new list.

---

## `islice()`

`islice()` slices an iterator.

```python
from itertools import islice

numbers = count()

print(list(islice(numbers, 5)))
```

```text
[0, 1, 2, 3, 4]
```

This is especially useful because iterators cannot be sliced using normal indexing.

---

## `takewhile()`

`takewhile()` returns items while a condition remains true.

```python
from itertools import takewhile

numbers = [1, 2, 3, 4, 1, 2]

result = takewhile(lambda x: x < 4, numbers)

print(list(result))
```

```text
[1, 2, 3]
```

Iteration stops as soon as the condition becomes false.

---

## `dropwhile()`

`dropwhile()` skips items while a condition is true, then returns the remaining items.

```python
from itertools import dropwhile

numbers = [1, 2, 3, 4, 1, 2]

result = dropwhile(lambda x: x < 4, numbers)

print(list(result))
```

```text
[4, 1, 2]
```

---

## `groupby()`

`groupby()` groups consecutive items with the same key.

```python
from itertools import groupby

words = ["apple", "ant", "bat", "ball"]

for letter, group in groupby(words, key=lambda word: word[0]):
    print(letter, list(group))
```

```text
a ['apple', 'ant']
b ['bat', 'ball']
```

> **Note**
>
> `groupby()` groups consecutive items. If the data is not already sorted by the grouping key, sort it first.

---

## `product()`

`product()` computes the Cartesian product of multiple iterables.

```python
from itertools import product

for pair in product([1, 2], ["A", "B"]):
    print(pair)
```

```text
(1, 'A')
(1, 'B')
(2, 'A')
(2, 'B')
```

---

## `permutations()`

`permutations()` returns every possible ordering.

```python
from itertools import permutations

print(list(permutations("ABC", 2)))
```

```text
[('A', 'B'),
 ('A', 'C'),
 ('B', 'A'),
 ('B', 'C'),
 ('C', 'A'),
 ('C', 'B')]
```

---

## `combinations()`

`combinations()` returns unique combinations where order does not matter.

```python
from itertools import combinations

print(list(combinations("ABC", 2)))
```

```text
[('A', 'B'),
 ('A', 'C'),
 ('B', 'C')]
```

---

## Choosing the Right Tool

| Task | Tool |
|------|------|
| Infinite sequence | `count()` |
| Repeat values | `repeat()` |
| Cycle forever | `cycle()` |
| Join iterables | `chain()` |
| Slice an iterator | `islice()` |
| Group consecutive items | `groupby()` |
| Cartesian product | `product()` |
| All permutations | `permutations()` |
| Unique combinations | `combinations()` |

---

## Best Practices

- Prefer `itertools` when processing large datasets lazily.
- Use `chain()` instead of repeatedly concatenating iterables.
- Remember that most `itertools` functions return iterators and can only be consumed once.
- Sort data before using `groupby()` if you expect all equal values to appear together.

---

# Context Managers

A context manager manages resources by performing setup before a block of code executes and cleanup afterwards.

The most common example is working with files.

Instead of writing:

```python
file = open("data.txt")

try:
    content = file.read()
finally:
    file.close()
```

use:

```python
with open("data.txt") as file:
    content = file.read()
```

When execution leaves the `with` block, the file is closed automatically—even if an exception occurs.

---

## How It Works

A context manager implements two special methods:

- `__enter__()`
- `__exit__()`

*Best for complex objects, managing persistent state, or if you need to suppress specific errors.*

```python
class Database:

    def __enter__(self):
        print("Connected")
        return self

    def __exit__(self, exc_type, exc, traceback):
        print("Disconnected")
```

```python
with Database():
    print("Working...")
```

Output

```text
Connected
Working...
Disconnected
```

`__enter__()` runs when entering the block.

`__exit__()` always runs when leaving the block.

---

## Creating Context Managers with `contextlib`

For simple cases, use the `contextmanager` decorator.

```python
from contextlib import contextmanager

@contextmanager
def transaction():
    print("Begin")

    try:
        yield
    finally:
        print("Commit")
```

```python
with transaction():
    print("Saving...")
```

Output

```text
Begin
Saving...
Commit
```

This is often simpler than writing a class.

---

## `ExitStack`

Manages a variable number of context managers this is handy when you don't know how many resources you'll need ahead of time.

```python
from contextlib import ExitStack

with ExitStack() as stack:
    files = [stack.enter_context(open(name)) for name in filenames]
    # all files stay open here, all get closed automatically on exit

```
---

## Common Use Cases

Context managers are commonly used for:

- files
- database connections
- network connections
- locks
- transactions
- temporary files
- resource cleanup

---

## Best Practices

- Prefer `with` over manually calling `close()`.
- Keep `with` blocks focused on a single task.
- Use `contextmanager` for simple custom context managers.
- Use `ExitStack` when managing multiple resources dynamically.

> **Developer Note**
>
> Context managers make resource management predictable. They ensure cleanup happens even if an exception interrupts execution, making them the preferred way to work with files, connections, locks, and other resources.

---

# Best Practices

As your codebase grows, good function design becomes more important than clever implementations.

## Keep Functions Small

A function should perform one well-defined task.

Instead of:

```python
def process_user():
    # validate
    # save
    # send email
    # log
```

Prefer:

```python
validate_user()
save_user()
send_welcome_email()
log_activity()
```

Small functions are easier to read, test, and reuse.

---

## Prefer Returning Values

Functions should return results instead of printing them.

```python
def total(values):
    return sum(values)
```

instead of

```python
def total(values):
    print(sum(values))
```

Printing is useful for debugging or user interfaces, not for reusable functions.

---

## Avoid Mutable Default Arguments

Don't do this:

```python
def add(item, items=[]):
    items.append(item)
    return items
```

Instead:

```python
def add(item, items=None):
    if items is None:
        items = []

    items.append(item)
    return items
```

---

## Use Generators for Large Data

If values are processed once, generate them lazily.

```python
sum(x * x for x in range(1_000_000))
```

instead of

```python
sum([x * x for x in range(1_000_000)])
```

The generator avoids creating an unnecessary list.

---

## Prefer Comprehensions

For simple transformations:

```python
squares = [x * x for x in numbers]
```

instead of

```python
squares = list(map(lambda x: x * x, numbers))
```

Comprehensions are generally considered more Pythonic and easier to read.

---

## Use Built-ins

Python's built-in functions are usually faster and clearer than manual implementations.

Prefer:

```python
sum(numbers)
max(numbers)
min(numbers)
any(values)
all(values)
```

over manually looping.

---

## Keep Decorators Focused

A decorator should have one responsibility.

Good examples include:

- logging
- caching
- timing
- authorization
- retries

Avoid decorators that perform unrelated tasks.

---

## Use `with` for Resources

Always prefer context managers when working with resources.

```python
with open("data.txt") as file:
    ...
```

instead of manually opening and closing resources.

---

## Add Type Hints

Type hints improve readability and tooling.

```python
def average(numbers: list[int]) -> float:
    ...
```

While optional, they are recommended for public APIs and shared codebases.

---

## Document Public Functions

Use docstrings for reusable functions.

```python
def square(number):
    """Return the square of a number."""
```

Document:

- purpose
- parameters
- return value
- exceptions (when appropriate)

---

## Write Pythonic Code

Prefer readability over cleverness.

Code is read far more often than it is written.

---

# Other features

---
# Descriptors

Descriptors are objects that control what happens when an attribute is accessed, assigned, or deleted.

The protocol is:

```
__get__()
__set__()
__delete__()
```

A simple descriptor:

```python
class PositiveNumber:
    def __get__(self, instance, owner):
        return instance._value

    def __set__(self, instance, value):
        if value < 0:
            raise ValueError("must be positive")

        instance._value = value
```

Used like:

```python
class Product:
    price = PositiveNumber()
```

Now:

```python
product.price = 10
```

actually invokes the descriptor's `__set__()`.

### Why care?

You probably won't write descriptors every day.

The important thing is understanding that Python's attribute system is programmable.

`property` is itself implemented using the descriptor protocol.

So when you write:

```python
@property
def price(self):
    return self._price
```

you're using a descriptor without having to implement one yourself.

Descriptors become particularly important when working with frameworks, ORMs, and other systems that make attributes behave in non-trivial ways.

---

# Typing

Type hints describe what kinds of values functions and objects are expected to work with.

```python
def total(prices: list[float]) -> float:
    return sum(prices)
```

Type hints don't normally enforce types at runtime.

Tools such as `mypy` and `pyright` can use them to detect problems before the program runs.

### Modern syntax

Prefer modern Python syntax:

```python
str | None
list[str]
dict[str, int]
tuple[str, int]
```

rather than unnecessarily verbose older forms.

### Callable

Use `Callable` when a function accepts another function.

```python
from collections.abc import Callable


def apply(
    fn: Callable[[int], int],
    value: int,
) -> int:
    return fn(value)
```

This says:

- `fn` accepts an `int`
- `fn` returns an `int`

### TypeVar

`TypeVar` lets you describe relationships between input and output types.

```python
from typing import TypeVar

T = TypeVar("T")


def first(items: list[T]) -> T:
    return items[0]
```

If you pass:

```python
first([1, 2, 3])
```

the result is understood as an `int`.

If you pass:

```python
first(["a", "b"])
```

the result is understood as a `str`.

The important idea is that the function preserves the relationship between the input type and output type.

### TypedDict

Useful when working with dictionaries that have a known structure:

```python
from typing import TypedDict


class User(TypedDict):
    id: int
    name: str
    active: bool
```

Now:

```python
user: User = {
    "id": 1,
    "name": "Alice",
    "active": True,
}
```

This gives static type checkers information about dictionary structure without turning the dictionary into a class.

#### TypedDict vs dataclass

Different tools for different jobs — one's a type annotation, the other's a real object.

## TypedDict

Purely a **type-checking construct**. At runtime, it's just a plain `dict`.

```python
from typing import TypedDict

class User(TypedDict):
    id: int
    name: str
    active: bool

user: User = {"id": 1, "name": "Alice", "active": True}
```

- No `__init__`, no validation, no methods — `type(user)` is `dict`, not `User`
- Zero runtime cost — it disappears after static analysis
- Access via `user["name"]`, not `user.name`
- Use when the data genuinely *is* a dict (JSON payloads, API responses, kwargs you're passing through) and you just want the type checker to know its shape

## dataclass

Generates an **actual class** with real behavior.

```python
from dataclasses import dataclass

@dataclass
class User:
    id: int
    name: str
    active: bool = True

user = User(id=1, name="Alice")
```

- Auto-generates `__init__`, `__repr__`, `__eq__` (and `__hash__` if `frozen=True`)
- Real object with identity — `type(user)` is `User`
- Access via `user.name`
- Can hold methods, defaults, validation (`__post_init__`), and be made immutable

## The actual decision

| | TypedDict | dataclass |
|---|---|---|
| Underlying type at runtime | `dict` | your class |
| Access | `d["key"]` | `obj.attr` |
| Methods/behavior | no | yes |
| Validation | no (static only) | yes, via `__post_init__` |
| Best for | describing dict-shaped data you don't own (JSON, external APIs) | data your code owns and operates on |

Rule of thumb: if you're **parsing/deserializing** something (like a webhook payload or an n8n/Zapier response body) and just want mypy to catch typos in key access, TypedDict. If you're **constructing and passing around your own domain objects**, dataclass.

## Worth knowing: Pydantic's BaseModel

Sits between them — it looks like a dataclass but does runtime validation and coercion (e.g., turning `"1"` into `1` for an `int` field). For anything touching untrusted input (webhooks, API request bodies), that's usually the better call than either TypedDict or a plain dataclass, since neither actually validates at runtime.

---

# Pattern Matching

Python's `match` statement can match values and structures.

```python
match command:
    case "start":
        start()

    case "stop":
        stop()

    case _:
        raise ValueError("Unknown command")
```

Its real strength is structural matching.

```python
match request:
    case {"action": "create", "name": name}:
        create_user(name)

    case {"action": "delete", "id": user_id}:
        delete_user(user_id)

    case _:
        raise ValueError("Invalid request")
```

You can also use guards:

```python
match value:
    case int(n) if n > 0:
        print("positive integer")

    case int():
        print("non-positive integer")

    case _:
        print("something else")
```

Use `match` when it makes structural branching clearer.

Don't replace every `if` statement with it.

---

# Exercises

---

## Exercise 1 — Build a Memoization Decorator

Implement a decorator that caches function results.

Requirements:

- Cache results by arguments.
- Return cached values when possible.
- Do not use `functools.lru_cache`.

**Bonus**

Support functions with both positional and keyword arguments.

```python
def my_cache(func):
    cache_store = {}

    @wraps(func)
    def wrapper(*args, **kwargs):
        key = (args, frozenset(kwargs.items()))

        if key in cache_store:
            return cache_store[key]

        result = func(*args, **kwargs)
        cache_store[key] = result
        return result

    return wrapper

@my_cache
def add(a, b):
    print("Computing")
    return a + b

print(add(1, 2))
print(add(1, 2))
```
--- 

## Exercise 2 — CSV Processing Pipeline

Given a CSV containing millions of rows:

- Read the file lazily.
- Filter invalid rows.
- Convert valid rows into dictionaries.
- Compute summary statistics.

Avoid loading the entire file into memory.

```python
import csv
from collections import Counter


def valid(row):
    return (
        row.get("timestamp")
        and row.get("level") in {"INFO", "WARNING", "ERROR"}
        and row.get("service")
        and row.get("message")
    )


def valid_rows(rows):
    for row in rows:
        if valid(row):
            yield row


levels = Counter()
services = Counter()

with open("logs.csv", newline="") as file:
    rows = csv.DictReader(file)

    for row in valid_rows(rows):
        levels[row["level"]] += 1
        services[row["service"]] += 1

print(levels)
print(services)
```
---

## Exercise 3 — Logging Decorator

Create a decorator that logs:

- function name
- arguments
- execution time
- returned value

The decorator should work with any function signature.

```python
from functools import wraps
from time import perf_counter


def log_call(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        start = perf_counter()

        result = func(*args, **kwargs)

        elapsed = perf_counter() - start

        print(f"Function: {func.__name__}")
        print(f"Arguments: {args}, {kwargs}")
        print(f"Execution time: {elapsed:.6f}s")
        print(f"Returned: {result}")

        return result

    return wrapper
```

---

## Exercise 4 — Rate Limiter

Implement a decorator that allows a function to execute at most five times per minute.

Subsequent calls should raise an exception.

```python
from collections import deque
from functools import wraps
from time import monotonic

def rate_limit(func):
    calls = deque()

    @wraps(func)
    def wrapper(*args, **kwargs):
        now = monotonic()

        # Remove calls older than one minute
        while calls and now - calls[0] >= 60:
            calls.popleft()

        if len(calls) >= 5:
            raise RuntimeError("Rate limit exceeded")

        calls.append(now)

        return func(*args, **kwargs)

    return wrapper
```

---


## Exercise 5 — Iterator

Implement a custom iterator representing a countdown.

```python
Countdown(5)
```

should produce

```text
5
4
3
2
1
```

Then implement the same functionality using a generator.

Compare both implementations.

```python
class Countdown:

    def __init__(self, start):
        self.current = start

    def __iter__(self):
        return self

    def __next__(self):
        if self.current <= 0:
            raise StopIteration

        value = self.current
        self.current -= 1

        return value

# Generator version
def countdown(start):
    while start > 0:
        yield start
        start -= 1
```

---

## Exercise 7 — Streaming Log Analyzer

Given a log file containing several gigabytes of data:

- Count requests by endpoint.
- Count requests by HTTP status.
- Find the ten most requested endpoints.

The solution should:

- process the file one line at a time
- avoid storing unnecessary data
- use generators where appropriate

---

## Exercise 8 — Command Dispatcher

Create a command dispatcher using `functools.singledispatch`.

Commands should behave differently for:

- strings
- integers
- lists
- dictionaries

Avoid using `if isinstance(...)`.

```python
from functools import singledispatch


@singledispatch
def handle(command):
    raise TypeError(f"Unsupported command: {type(command)}")


@handle.register
def _(command: str):
    return f"String command: {command}"


@handle.register
def _(command: int):
    return f"Integer command: {command}"


@handle.register
def _(command: list):
    return f"List command: {command}"


@handle.register
def _(command: dict):
    return f"Dictionary command: {command}"
```

---

## Exercise 9 — Database Transaction Context Manager

Create a context manager that:

- opens a transaction
- commits on success
- rolls back on exception

Implement it:

1. as a class
```python
class Transaction:

    def __init__(self, connection):
        self.connection = connection

    def __enter__(self):
        self.connection.begin()
        return self.connection

    def __exit__(self, exc_type, exc_value, traceback):
        if exc_type is None:
            self.connection.commit()
        else:
            self.connection.rollback()

        return False
```
2. using `@contextmanager`
```python
from contextlib import contextmanager


@contextmanager
def transaction(connection):
    connection.begin()

    try:
        yield connection
    except Exception:
        connection.rollback()
        raise
    else:
        connection.commit()
```

---

## Exercise 10 — Data Processing Framework

Design a reusable data-processing pipeline.

Requirements:

- Input may come from a file, database, or API.
- Process records lazily.
- Allow filters to be chained.
- Allow transformations to be chained.
- Produce summary statistics.
- Support datasets that do not fit into memory.

Explain:

- which components should be generators
- where decorators could be useful
- where caching is appropriate
- which `itertools` functions simplify the implementation

---

## Exercise 11

Review the following code.

```python
results = []

for user in users:
    if user["active"]:
        results.append(
            user["name"].upper()
        )

print(sorted(results))
```

Refactor it to make it:

- more Pythonic
- more memory efficient
- easier to extend
- easier to test

Consider:

- generators
- comprehensions
- higher-order functions
- separation of concerns

Explain every design decision.

---

# Module Summary

In this module, you learned how Python treats functions as first-class objects and how that design influences much of the language.

You explored:

- function parameters
- variable scope
- lambda functions
- closures
- decorators
- iterators
- generators
- lazy evaluation
- functional programming utilities
- `itertools`
- context managers

These concepts form the foundation of modern Python frameworks such as FastAPI, Django, Flask, Click, and many data science libraries.

The next module builds on these foundations by exploring Python's object-oriented programming model, including classes, inheritance, protocols, descriptors, and Python's data model.

---

# Module 4 — Exceptions & File Handling (Combined Notes)

## 1. Errors vs Exceptions — what's the difference?

Think of it like cooking:

- An **error** is when you don't even have a valid recipe. The kitchen won't let you start cooking at all. Python's parser catches these *before* your program ever runs — things like `SyntaxError`, `IndentationError`, `TabError`.
- An **exception** is when you're already cooking and something goes wrong midway — you drop the eggs, the oven timer goes off wrong. The recipe was valid, but something at *runtime* broke. The good news: you can plan for this and recover, e.g. `10 / 0` raises `ZeroDivisionError`.

```
Syntax Error  →  Program can't even start
Exception     →  Program CAN recover, if you handle it
```

## 2. The exception family tree

Almost every exception is a descendant of the base `Exception` class — like a big family, where `Exception` is the great-grandparent everyone traces back to.

| Exception | When it happens (in plain terms) |
|---|---|
| `ValueError` | Right type of thing, but a value that doesn't make sense (e.g. `int("banana")`) |
| `TypeError` | You used the wrong *kind* of thing entirely (e.g. adding a string to a number) |
| `KeyError` | You asked a dictionary for a key it doesn't have |
| `IndexError` | You asked a list for a slot that doesn't exist |
| `NameError` | You used a variable that was never created |
| `AttributeError` | You asked an object to do something it doesn't know how to do |
| `FileNotFoundError` | You tried to open a file that isn't there |
| `PermissionError` | The file exists, but you're not allowed to touch it |
| `ZeroDivisionError` | You divided by zero |
| `TimeoutError` | Something took too long and gave up |

**Golden rule:** always catch the most *specific* exception you can. Catching a specific error is like calling a locksmith for a lock problem instead of calling "anyone who might fix anything."

## 3. `try` / `except` — the safety net

```python
try:
    age = int(input("Age: "))
except ValueError:
    print("Please enter a number.")
```

Picture `try` as "attempt this risky move" and `except` as "here's the safety net if you fall." The code inside `try` is the code that might fail.

### Catching more than one thing

```python
try:
    process(data)
except FileNotFoundError:
    ...
except PermissionError:
    ...
except Exception:
    ...
```

Python checks these top to bottom, like a bouncer checking a list of IDs from most specific to "let anyone else in." That's why the broadest one (`Exception`) always goes **last** — if you put it first, it would catch everything and the more specific nets below it would never get a turn.

You can also catch several types in one net using a tuple:

```python
except (TypeError, ValueError):
    ...
```

### Skip the bare `except:`

```python
try:
   print(x)
except:
   print("An exception has occurred!")
```

This works, but it's like putting a net so wide it also scoops up things you *didn't* want to catch — including the signal for "user pressed Ctrl+C" (`KeyboardInterrupt`) or "please quit now" (`SystemExit`). That can make your program impossible to stop cleanly and hides real bugs. Almost always name the exception you expect instead.

## 4. `else` — "only if nothing went wrong"

```python
try:
    user = load_user()
except FileNotFoundError:
    ...
else:
    print(user)
```

Think of `else` as the victory lap — it only runs if the `try` block finished with zero drama. This keeps your "happy path" code visually separate from your error-handling code, which makes both easier to read.

## 5. `finally` — "no matter what"

```python
file = open("users.csv")
try:
    process(file)
finally:
    file.close()
```

`finally` is the cleanup crew that shows up *regardless* of whether the party went well or the kitchen caught fire — exception or not, it runs. This is exactly why it's the natural home for closing files, releasing locks, or disconnecting from a database.

In modern Python you'll almost always prefer:

```python
with open(...) as file:
    ...
```

`with` is `finally`'s cleanup behavior baked in automatically — you get the safety net for free.

## 6. Nesting try/except (use sparingly)

You *can* nest a `try`/`except` inside another block — e.g. inside the `else` of an outer `try`, to check a second thing only if the first thing succeeded:

```python
def divide(x, y):
    try:
        value = 50
        x.append(value)
    except AttributeError as atr_err:
        print(atr_err)
    else:
        try:
            result = [i / y for i in x]
            print(result)
        except ZeroDivisionError:
            print("Please change 'y' to a non-zero value")
    finally:
        print("done")
```

This is like a security checkpoint with a second checkpoint behind it — technically fine, but the more layers you add, the harder the whole thing is to follow. Most real code prefers several simple, sequential `try`/`except` blocks over deep nesting.

## 7. Raising your own exceptions

Don't quietly hand back a wrong answer — raise an alarm instead:

```python
def withdraw(balance, amount):
    if amount > balance:
        raise ValueError("Insufficient funds")
    return balance - amount
```

Think of `raise` as pulling the fire alarm on purpose, the moment you notice something is wrong — much better than waiting for the fire to spread and cause a confusing crash somewhere else later.

## 8. Re-raising — "log it, then let it keep going"

```python
try:
    process_order()
except Exception:
    logger.error("Order failed")
    raise
```

A bare `raise` (no arguments) is like a relay racer handing off the baton without dropping it — it re-throws the *exact same* exception, traceback and all, after you've had a chance to log or clean something up.

## 9. Exception chaining — keeping the paper trail

```python
try:
    load_config()
except FileNotFoundError as exc:
    raise RuntimeError("Application configuration missing.") from exc
```

`from exc` is like a doctor's note that says "this new problem was *caused by* that earlier problem." Python keeps both in the traceback, so whoever reads the crash log can trace the whole chain of events instead of only seeing the last domino that fell. This shows up constantly in real production code — one layer catches a low-level failure and re-raises it as something more meaningful to the layer above.

## 10. Custom exceptions — giving errors their own names

Instead of lumping everything under a generic `ValueError`, give your app's own failures their own identity:

```python
class InsufficientStock(Exception):
    pass
```

Now your code can tell the difference between `ValueError`, `InsufficientStock`, `PaymentFailed`, `AuthenticationError` — instead of treating every failure as the same vague "something broke."

### Building a small family of errors

```python
class InventoryError(Exception):
    pass

class ProductNotFound(InventoryError):
    pass

class OutOfStock(InventoryError):
    pass
```

This is like organizing a filing cabinet: callers can grab one specific folder (`ProductNotFound`) or the whole drawer (`InventoryError`) depending on how broadly they want to react.

## 11. What's new in recent Python versions

Python has kept adding tools that make exception handling nicer to read and debug:

- **Structural pattern matching (3.10):** `match`/`case` lets you branch on the *type* of exception a bit like a switchboard operator routing calls, instead of a long chain of `except` blocks:

  ```python
  try:
      result = 1 / 0
  except Exception as e:
      match e:
          case ZeroDivisionError():
              print("You can't divide by zero.")
          case NameError():
              print("That variable doesn't exist.")
          case _:
              print("Something unexpected happened.")
  ```

- **Precise error locations (3.11):** tracebacks now point at the *exact* sub-expression that failed, not just the whole line — like a mechanic pointing at the one loose bolt instead of "somewhere under the hood."

- **Exception groups & `except*` (3.11):** lets you raise and catch a *bundle* of exceptions at once — handy for async code where several things can fail in parallel:

  ```python
  try:
      raise ExceptionGroup("multiple errors", [ValueError("bad value"), TypeError("bad type")])
  except* ValueError as ve:
      print(f"Handling ValueError: {ve}")
  except* TypeError as te:
      print(f"Handling TypeError: {te}")
  ```

- **`add_note()` (3.11):** stick extra context onto an exception before it propagates, like adding sticky notes to a file before passing it up the chain:

  ```python
  try:
      raise ValueError("Invalid input")
  except ValueError as e:
      e.add_note("This happened while processing user input.")
      raise
  ```

- **Smarter error messages (3.12):** typos in attribute or variable names now get a "did you mean...?" suggestion, e.g. calling `.appendd()` on a list tells you it probably meant `.append()`.

- **Colorized tracebacks (3.13):** the interactive interpreter now highlights error types, line numbers, and messages in color by default, so long tracebacks are faster to scan.

## 12. File handling basics

```python
with open("users.txt") as file:
    content = file.read()
```

Reading line by line (much friendlier to memory on big files):

```python
with open("users.txt") as file:
    for line in file:
        ...
```

For very large files, always **iterate** instead of `readlines()` — `readlines()` is like trying to carry an entire warehouse of boxes in one trip, while iterating is carrying one box at a time.

### File modes cheat sheet

| Mode | Meaning |
|---|---|
| `r` | Read |
| `w` | Write (overwrites existing content!) |
| `a` | Append (adds to the end) |
| `x` | Create new file (fails if it already exists) |
| `rb` | Read binary |
| `wb` | Write binary |

### `pathlib` instead of string paths

```python
from pathlib import Path

path = Path("data") / "users.csv"
```

`pathlib` treats a file path like a proper object instead of a plain string you have to glue together by hand — much less error-prone than `os.path`.

Handy methods: `path.exists()`, `path.read_text()`, `path.write_text(...)`, `path.mkdir()`, `path.glob("*.csv")`.

```python
for file in Path("logs").glob("*.log"):
    print(file)
```

### `contextlib.suppress` — "it's fine if this fails"

```python
from contextlib import suppress

with suppress(FileNotFoundError):
    os.remove("temp.txt")
```

This is a shorter, more expressive way of saying "try this, and if it fails with exactly this one expected reason, just shrug and move on" — instead of writing a whole `try/except/pass`.

## 13. Real-World Exercise — Log Processing Pipeline

You're building an internal monitoring tool. Every application server writes logs like this:

```
2026-08-01T09:10:15 INFO User logged in
2026-08-01T09:10:18 ERROR Payment failed
2026-08-01T09:10:25 WARNING Slow database query
```

There are hundreds of log files, one per server, and each file may be several gigabytes.

**Requirements** — build a pipeline that:

- discovers every `.log` file in a directory
- reads files lazily (never loads an entire file into memory)
- skips corrupted lines
- counts log levels (`INFO`, `WARNING`, `ERROR`)
- writes a daily summary to `summary.json`
- never crashes because of one bad file

**Design questions to think through:**

1. Which exceptions should you catch?
2. Which exceptions should you let propagate?
3. Where should `finally` or context managers be used?
4. Would you use `pathlib` or `os.path`?
5. How would you process 50GB of logs without running out of memory?

*Hint, tying it back to the sections above: `Path("logs").glob("*.log")` handles discovery (§12), iterating line-by-line handles the memory constraint (§12), a `try`/`except` per line with a specific exception (not bare `except:`) handles corrupted lines without crashing the whole pipeline (§3), and `with open(...)` handles cleanup automatically (§5).*

---

# Module 5 — Async Python

## Why Async Exists

Most backend code spends more time *waiting* than *computing*. A typical request looks like this:

```
Receive request
      │
      ▼
Call database ──► WAIT
      │
      ▼
Call another API ──► WAIT
      │
      ▼
Return response
```

The CPU is idle for most of that timeline. Async programming exists to make the waiting productive: instead of sitting still until one operation finishes, the program works on something else in the meantime.

### Concurrency vs. parallelism — not the same thing

| | Concurrency | Parallelism |
|---|---|---|
| What it means | Multiple tasks make progress over the same period | Multiple tasks execute at the literal same instant |
| Typical setup | One thread, one CPU core | Multiple threads/processes, multiple cores |
| Best for | I/O-bound work (waiting on network, disk, DB) | CPU-bound work (heavy computation) |

**Chef analogy:**
- **Concurrency** — one chef puts pasta on to boil, then chops vegetables while it waits. One chef, two things in progress.
- **Parallelism** — two chefs cook two dishes at the same time. Two chefs, two things happening simultaneously.

`asyncio` gives you concurrency, not parallelism. It's one thread hopping between tasks whenever the current one is idle waiting on I/O.

---

## Coroutines: the core syntax

```python
async def fetch_data():
    ...
```

An `async def` function is a **coroutine function**. Calling it does **not** run the body — it produces a coroutine object:

```python
result = fetch_data()
print(type(result))   # <class 'coroutine'>
```

To actually execute it, you need `await` (from inside another async function) or `asyncio.run()` (from synchronous, top-level code):

```python
result = await fetch_data()          # inside another async function
result = asyncio.run(fetch_data())   # entry point from sync code
```

Think of `asyncio.run()` as "start the event loop and run this."

### The classic mistake: forgetting `await`

```python
async def get_message():
    await asyncio.sleep(1)
    return "Hello!"

async def main():
    message = get_message()   # missing await
    print(message)
```

```
<coroutine object get_message at 0x...>
RuntimeWarning: coroutine 'get_message' was never awaited
```

Rule of thumb: **every async function call needs an `await`** — whether it's a built-in like `asyncio.sleep()` or one you wrote yourself. If you see `coroutine 'x' was never awaited`, that's the bug.

---

## The event loop

The event loop is the engine that decides which coroutine runs when.

```
Task A waiting
      │
      ▼
Run Task B
      │
Task B waiting
      │
      ▼
Run Task C
      │
Task A's wait finishes
      │
      ▼
Resume Task A
```

Everything typically runs on **one thread**. Tasks explicitly hand back control at `await` points — this is *cooperative* concurrency, not preemptive. Nothing forces a switch mid-computation, which is exactly why CPU-bound code doesn't benefit from async (see below).

A rough step-by-step for a single coroutine with one `await asyncio.sleep(2)`:

1. `asyncio.run()` creates the event loop and starts the coroutine.
2. Code runs until it hits `await` — the coroutine pauses there.
3. The loop checks: anything else to run? (With one coroutine, no.)
4. When the awaited operation completes, the loop resumes the coroutine exactly where it paused.
5. Coroutine finishes → loop exits.

With only one coroutine, async isn't any faster than sync — the benefit shows up once there's more than one thing to juggle.

---

## Sequential `await` is still sequential

A common misconception: calling several async functions with `await`, one after another, does **not** make them concurrent. Each `await` blocks the rest of the function until that call resolves.

```python
await greet("Alice")     # ~2s
await greet("Bob")       # ~2s
await greet("Charlie")   # ~2s
# total: ~6s
```

`async def` doesn't make code concurrent by itself — it just makes it *pausable*. You still need something that runs multiple coroutines at once.

## `asyncio.gather()` — actual concurrency

```python
results = await asyncio.gather(
    fetch_user(),
    fetch_products(),
    fetch_orders(),
)
# total: ~time of the slowest one, not the sum
```

All three coroutines start immediately and progress during the same window. Results come back in a list, **in the same order the coroutines were passed in** — regardless of which one finishes first.

Use `gather()` whenever the operations don't depend on each other.

## Background tasks with `create_task`

Sometimes you don't want to `await` something immediately — you want it running while you do other work, then collect the result later:

```python
task = asyncio.create_task(fetch_user())

print("Doing something else...")

user = await task   # collect the result when you're ready
```

`create_task()` schedules the coroutine on the event loop right away; the `await` later on just waits for (and returns) its result.

## Async context managers

Just as `with open(...)` manages a resource synchronously, `async with` manages one asynchronously — very common with HTTP clients and databases:

```python
async with session.get(url) as response:
    data = await response.json()
```

---

## Real I/O: `aiohttp` instead of `requests`

`requests` is synchronous — using it inside async code blocks the entire event loop and defeats the point. Use `aiohttp`, and **reuse one `ClientSession`** across requests instead of creating a new one per call.

```python
import aiohttp
import asyncio

async def fetch(session, url):
    async with session.get(url) as response:
        return await response.text()

async def main():
    urls = ["https://example.com"] * 10
    async with aiohttp.ClientSession() as session:
        pages = await asyncio.gather(*[fetch(session, u) for u in urls])

asyncio.run(main())
```

### Why session reuse matters

| Without reuse | With reuse |
|---|---|
| 100 requests → 100 new connections | 1 session → connections pooled and reused |
| Every request pays a fresh TCP + SSL handshake | Only the *first* request to a host pays the handshake cost |

A TCP handshake costs one round trip; an SSL handshake costs two more on top — roughly 100–300ms thrown away per request if you skip pooling. One `ClientSession` keeps a connection pool alive, so the next request to the same host reuses an open connection instead of renegotiating from scratch.

---

## Rate limiting with a semaphore

Firing everything at once can get you rate-limited, throttled, or blocked — or just overwhelm your own machine. `asyncio.Semaphore(n)` caps how many requests are "in flight" simultaneously.

Think of it as a fixed number of permits: a task acquires one to proceed, and waits — without blocking anything else — until one frees up if none are available.

```python
async def fetch_limited(session, url, semaphore):
    async with semaphore:          # acquire a permit, released automatically on exit
        async with session.get(url) as response:
            return await response.text()

semaphore = asyncio.Semaphore(5)   # max 5 concurrent requests
results = await asyncio.gather(*[fetch_limited(session, u, semaphore) for u in urls])
```

Walking through it with 3 permits and 4+ tasks: Task A, B, C each grab a permit and start; Task D wants one but none are free, so it suspends (not spins) until A finishes and returns its permit — then D proceeds.

**Important:** a semaphore limits *concurrency*, not a rate per second. `Semaphore(10)` means "at most 10 in flight," not "10 per second." For strict time-based limits, pair it with delays between batches or a dedicated rate-limiting library (e.g. `aiolimiter`).

### Semaphore vs. rate limiter

| | Semaphore | Rate limiter |
|---|---|---|
| Limits | Concurrent tasks in flight | Requests over a time window |
| Says | "Only 5 at once" | "Only 100 per minute" |

In production you often use both together.

---

## Timeouts with `asyncio.wait_for`

A request can hang indefinitely if a server accepts the connection but never responds. Wrap it with a deadline:

```python
try:
    result = await asyncio.wait_for(fetch(session, url), timeout=5.0)
except asyncio.TimeoutError:
    result = {"error": "timed out"}
```

When a coroutine is cancelled — by timeout or otherwise — Python raises `asyncio.CancelledError` inside it. Use `try`/`finally` if the coroutine is holding a resource (a file handle, a connection) that needs cleanup regardless of how it exits:

```python
async def fetch_with_cleanup(session, url):
    try:
        async with session.get(url) as response:
            return await response.text()
    finally:
        cleanup()   # runs even on cancellation
```

---

## Handling failures inside `gather`

By default, `gather()` is **fail-fast**: one exception cancels the whole batch and propagates up, discarding every result — even the ones that already succeeded.

```python
results = await asyncio.gather(*tasks, return_exceptions=True)

successes = [r for r in results if not isinstance(r, Exception)]
failures  = [r for r in results if isinstance(r, Exception)]
```

`return_exceptions=True` changes that: failures land in the results list as exception objects instead of raising, so you can separate what worked from what didn't and decide what to do with each — retry, log, or drop.

---

## Retry with exponential backoff

Retrying immediately after a failure can make a struggling server worse. Back off longer after each attempt using `2 ** attempt` → 1s, 2s, 4s...

```python
async def fetch_with_retry(session, url, max_retries=3):
    for attempt in range(max_retries):
        try:
            return await fetch(session, url)
        except aiohttp.ClientError:
            if attempt == max_retries - 1:
                return None
            await asyncio.sleep(2 ** attempt)
```

A few rules that matter in practice:

- Catch **specific**, likely-transient exceptions (`aiohttp.ClientError`) — not a bare `except`. A bug in your own code shouldn't trigger a retry loop.
- Add small random **jitter** to the backoff in production, so a batch of failed requests doesn't all retry at exactly the same moment.
- Only retry errors that are actually transient (e.g. `503`) — not permanent ones (`404`, `401`).

---

## Async database access with `aiosqlite`

A synchronous DB driver blocks the event loop during every query — same problem as using `requests`. `aiosqlite` gives SQLite an async interface (it runs operations in a thread pool under the hood) so queries don't stall other coroutines.

```python
import aiosqlite

async def save(db, row):
    await db.execute(
        "INSERT OR REPLACE INTO items (id, name) VALUES (?, ?)",
        (row["id"], row["name"]),
    )

async with aiosqlite.connect("data.db") as db:
    await save(db, {"id": 1, "name": "example"})
    await db.commit()
```

Always use `?` placeholders — never string-format values into SQL.

---

## Putting it together: one pipeline function

Every pattern above composes into a single shape: reused session → semaphore → timeout → retry → `gather(return_exceptions=True)` → storage. Written as one pipeline:

```python
import aiohttp
import aiosqlite
import asyncio

async def fetch_one(session, url, semaphore, timeout=5.0, max_retries=3):
    async with semaphore:
        for attempt in range(max_retries):
            try:
                coro = session.get(url)
                async with await asyncio.wait_for(coro, timeout=timeout) as response:
                    return await response.text()
            except (aiohttp.ClientError, asyncio.TimeoutError):
                if attempt == max_retries - 1:
                    return None
                await asyncio.sleep(2 ** attempt)

async def run_pipeline(urls, db_path, concurrency=5):
    semaphore = asyncio.Semaphore(concurrency)

    async with aiohttp.ClientSession() as session, aiosqlite.connect(db_path) as db:
        await db.execute(
            "CREATE TABLE IF NOT EXISTS pages (url TEXT PRIMARY KEY, body TEXT)"
        )

        results = await asyncio.gather(
            *[fetch_one(session, u, semaphore) for u in urls],
            return_exceptions=True,
        )

        for url, body in zip(urls, results):
            if body and not isinstance(body, Exception):
                await db.execute(
                    "INSERT OR REPLACE INTO pages (url, body) VALUES (?, ?)",
                    (url, body),
                )
        await db.commit()

asyncio.run(run_pipeline(["https://example.com"] * 20, "pages.db"))
```

Every request goes through the semaphore (bounded concurrency), a per-request timeout, and a retry loop with backoff — and one bad URL never sinks the batch, because failures come back as objects in the results list instead of raising.

---

## Async vs. threads

```
threading                          asyncio
  multiple OS threads                typically one thread
  scheduler switches between them    event loop switches between tasks
                                      tasks explicitly yield at await points
```

Async can handle very large numbers of concurrent I/O operations without a thread per operation — thread creation and context-switching have real overhead that async avoids.

## Async vs. parallelism, and when *not* to reach for async

`asyncio` gives concurrency, not CPU parallelism. For CPU-bound work — heavy computation, not waiting — `await` never gets a chance to hand off control, so async gives zero benefit. Use `multiprocessing` or `concurrent.futures` instead.

| Use async for | Don't use async for |
|---|---|
| HTTP requests | Image / video processing |
| Database queries | ML training |
| Web scraping | Heavy numerical computation |
| Reading many files | CPU-intensive algorithms |
| Message queues, websocket servers | — |

---

## The gotcha to never forget

```python
async def fetch():
    time.sleep(5)   # BAD — blocks the whole event loop
```

```python
async def fetch():
    await asyncio.sleep(5)   # correct — yields control
```

`async def` does **not** automatically make blocking code non-blocking. The same applies to any blocking database driver, HTTP client, or file operation — if it lacks an async-native version (`aiohttp` not `requests`, `aiosqlite` not `sqlite3`), it will stall the event loop the moment it's called from async code.

### Common mistakes, at a glance

| ❌ Mistake | ✅ Fix |
|---|---|
| `time.sleep(5)` inside async code | `await asyncio.sleep(5)` |
| New `ClientSession` per request | One shared session, reused |
| Firing thousands of requests with no cap | `asyncio.Semaphore(n)` |
| Running independent tasks with sequential `await` | `asyncio.gather(...)` |
| Bare `except:` around retries | Catch specific, transient exceptions only |

---

## Real-World Exercise — Multi-Source News Scraper

You're building an internal news aggregator. Every morning, it needs to collect the latest articles from **eight different news sites** — some respond in 200ms, others take 5–10 seconds, and a few occasionally fail.

**Build an asynchronous scraper that:**
- fetches articles from multiple sites concurrently
- reuses a single HTTP client session
- limits concurrency to **5 requests at a time**
- applies a **10-second timeout** per request
- retries failed requests up to **3 times**
- skips permanently failed sites without stopping the whole pipeline
- returns all successful articles sorted by publication time

```
BBC · CNN · Reuters · Bloomberg · TechCrunch · The Verge · Hacker News · Al Jazeera
        │
        ▼
 asyncio.gather()
        │
        ▼
 Semaphore (5 concurrent requests)
        │
        ▼
 Retry + Timeout
        │
        ▼
 Merge + Sort Articles
```

**Design questions to work through before coding:**
1. Why is async better than threads for this problem?
2. Where should the semaphore be applied?
3. Why should the HTTP client session be reused?
4. Which exceptions should trigger a retry?
5. What should happen if one site is completely unavailable?
6. How would you avoid overwhelming a news site with hundreds of concurrent requests?

## Challenge — Concurrent Data Ingestion Pipeline

You're ingesting product data from **20 supplier APIs** every hour. Each supplier has different response times, occasional failures, and strict rate limits.

**Design (don't fully implement) a pipeline that supports:**
- concurrent API requests
- retries with exponential backoff
- per-request timeouts
- concurrency limits using semaphores
- graceful handling of partial failures
- structured logging
- returning a combined dataset without failing the entire job because one supplier is down

Think about architecture first, then code.

---

# Module 6 - Object Oriented Programming

---

## 1. What is a class?

A class is a blueprint. It doesn't build anything by itself — it just describes what something *will* look like once you build it.

```python
class User:
    def __init__(self, name):
        self.name = name

    def greet(self):
        return f"Hello, {self.name}"
```

Building an actual thing from the blueprint (an **object**, also called an **instance**):

```python
user = User("Alice")

print(user.name)
print(user.greet())
```

`User` is the blueprint (the class). `user` is the actual house built from it (an instance). A class earns its keep when data and the behavior that acts on that data naturally belong together — like a dog having both a name (data) and a bark (behavior).

## 2. Instance attributes and methods

Instance attributes are the details that belong to one specific object, not the whole blueprint.

```python
class User:
    def __init__(self, name, age):
        self.name = name
        self.age = age
```

Each object keeps its own private notebook of state:

```python
alice = User("Alice", 30)
bob = User("Bob", 25)

alice.name  # "Alice"
bob.name    # "Bob"
```

Instance methods automatically receive the object they were called on as their first argument, conventionally named `self` — think of `self` as the method silently being handed "here's exactly which object you're working on right now."

```python
class User:
    def greet(self):
        return f"Hello, {self.name}"
```

## 3. Class attributes

A class attribute belongs to the blueprint itself, not to any one object — like a company-wide policy that every employee shares unless they personally override it.

```python
class User:
    species = "human"

    def __init__(self, name):
        self.name = name
```

Every instance can read it:

```python
alice = User("Alice")
bob = User("Bob")

alice.species
bob.species
```

If an instance ever sets its own attribute with the same name, that instance's version wins for that instance only — like an employee ignoring the company handbook and writing their own personal note instead.

Be careful with **mutable** class attributes:

```python
class Team:
    members = []
```

Every instance would secretly be sharing the *exact same list* — like everyone on the team accidentally writing in one shared notebook instead of their own. Usually you want:

```python
class Team:
    def __init__(self):
        self.members = []
```

so each instance gets its own fresh list.

## 4. Class methods and alternative constructors

A class method receives the *class itself* as its first argument, conventionally named `cls` — the class-level equivalent of `self`.

```python
class User:
    count = 0

    def __init__(self, name):
        self.name = name
        User.count += 1

    @classmethod
    def total_users(cls):
        return cls.count
```

Class methods make sense when the operation is really about the *class* as a whole (like "how many users have we ever created?"), not about one particular instance.

They're also commonly used as **alternative constructors** — extra "front doors" into building an object from different kinds of input:

```python
class User:
    def __init__(self, name):
        self.name = name

    @classmethod
    def from_email(cls, email):
        name = email.split("@")[0]
        return cls(name)

    @classmethod
    def from_dict(cls, data):
        return cls(data.get("name"))

user1 = User.from_email("alice@example.com")
user2 = User.from_dict({"name": "Bob"})
```

Unlike languages that only give you one constructor, Python lets you offer several clearly-named front doors — one for building from an email string, one from a dictionary, one from a raw API response — while `__init__` stays simple and consistent underneath.

## 5. `staticmethod`

A static method doesn't receive `self` or `cls` at all — it's just a regular function that happens to live inside the class because it's thematically related.

```python
class Math:
    @staticmethod
    def add(a, b):
        return a + b
```

Use it when the function logically belongs to the class but doesn't need instance or class state to do its job. Don't reach for `staticmethod` just because you can — if a function has no real relationship to the class, it may simply belong outside it as a normal function.

## 6. `__init__` and object construction

`__init__` initializes an object that already exists — it's not technically the thing that *creates* the object.

```python
class User:
    def __init__(self, name):
        self.name = name

user = User("Alice")
```

When you write `User("Alice")`, Python first creates a bare instance behind the scenes, then calls `__init__` on it to set it up — like a house being built first, then the moving crew coming in afterward to furnish it.

## 7. The four pillars of OOP — a quick map

Before going deeper, it helps to know the four big ideas everything below eventually connects back to:

| Pillar | In plain terms |
|---|---|
| **Encapsulation** | Keep an object's internal data behind controlled access, not wide open |
| **Inheritance** | Let one class reuse and extend another class's behavior |
| **Polymorphism** | Let different classes respond to the same call/operator in their own way |
| **Abstraction** | Expose a clean "what it does" interface, hide the messy "how it does it" |

The rest of this note walks through each pillar in turn.

## 8. Pillar 1 — Encapsulation

Python doesn't lock attributes away, it relies on naming *conventions* everyone agrees to respect.

- `self.name` → **public**: fair game to access from anywhere
- `self._name` → **protected** (by convention only): "this is an internal detail, don't poke at it unless you have a good reason"
- `self.__name` → **private-ish**, triggers **name mangling**

```python
class BankAccount:
    def __init__(self, owner, balance):
        self.owner = owner          # public
        self._status = "active"     # protected, by convention
        self.__balance = balance    # name-mangled
```

Internally, `self.__balance` becomes something like `self._BankAccount__balance`. This mangling mainly exists to stop *accidental* name clashes when subclassing, not to create a truly unbreakable vault — a determined caller can still reach `acc._BankAccount__balance` if they really want to.

The everyday analogy: you can't reach into the bank's vault yourself — you go through a teller or an ATM (the class's methods), which is exactly what controlled methods like `deposit()` and `withdraw()` give you instead of touching `__balance` directly.

### Properties — an attribute-shaped door with logic behind it

A property lets you keep the simple `object.attribute` syntax while quietly running real code behind it.

```python
class Product:
    def __init__(self, price):
        self._price = price

    @property
    def price(self):
        return self._price
```

Now `product.price` *looks* like plain attribute access but is secretly calling a method — effectively a getter. If the attribute should be read-only, you can stop right there; you only need a matching setter if you want to control what values are allowed on assignment:

```python
class Product:
    def __init__(self, price):
        self.price = price   # goes through the setter below

    @property
    def price(self):
        return self._price

    @price.setter
    def price(self, value):
        if value < 0:
            raise ValueError("Price cannot be negative")
        self._price = value
```

The core idea: a property preserves the friendly attribute-style API while quietly adding validation, calculation, or gatekeeping behind the scenes.

## 9. Pillar 2 — Inheritance

Inheritance lets one class derive behavior from another — a subclass is a specialized version of its parent.

```python
class Animal:
    def speak(self):
        return "sound"

class Dog(Animal):
    pass
```

`class Dog(Animal):` establishes the relationship: "Dog is-a Animal." It doesn't run any code by itself — it just wires up the family tree.

### Overriding

A subclass can swap in its own version of an inherited method:

```python
class Dog(Animal):
    def speak(self):
        return "woof"
```

`Dog().speak()` now returns `"woof"` instead of the generic `"sound"` — the subclass's version wins.

### `super()` — asking "what would the parent do?"

```python
class Animal:
    def __init__(self, name):
        self.name = name

class Dog(Animal):
    def __init__(self, name, breed):
        super().__init__(name)
        self.breed = breed
```

`super().__init__(name)` says "continue up the family tree and run the next implementation of `__init__`" — rather than naming `Animal` directly, which becomes fragile once multiple inheritance is involved. `class Dog(Animal):` *creates* the relationship; `super()` is what you use *afterward* to actually reach into it.

### The shapes inheritance can take

Inheritance isn't always a simple one-parent, one-child line. A few common shapes:

- **Single** — one parent, one child (the `Dog`/`Animal` example above)
- **Multilevel** — a chain: grandparent → parent → child, each adding more
- **Multiple** — one child pulls from two or more parents at once
- **Hierarchical** — several children all share the same one parent
- **Hybrid** — a mix of the above

When a class inherits from more than one parent, Python needs a rulebook for which parent's method wins if both define the same name. That rulebook is the **Method Resolution Order (MRO)**, computed with an algorithm called C3 linearization — you can always inspect it yourself:

```python
print(SomeClass.mro())
```

Deep or tangled inheritance trees get hard to reason about fast, so treat these tools as available when genuinely needed, not something to reach for by default.

### Composition — the alternative to inheritance

Inheritance models an **is-a** relationship (`Dog` is-a `Animal`). Composition models a **has-a** relationship (`Car` has-a `Engine`):

```python
class Engine:
    def start(self):
        print("Engine started")

class Car:
    def __init__(self):
        self.engine = Engine()

    def start(self):
        self.engine.start()
```

`Car` doesn't inherit from `Engine` — it simply *contains* one. In many designs composition beats deep inheritance chains, because a component like `Engine` can be swapped, tested, or reused entirely on its own. The question worth asking whenever you reach for inheritance: does this object genuinely need to *be* a specialized version of another object, or does it just *use* one?

## 10. Pillar 3 — Polymorphism

Polymorphism means different classes can respond to the *same* call in their own way — one interface, many behaviors.

### Overriding

Method overriding is polymorphism's most common everyday form: `Account().kind()` and `SavingsAccount().kind()` can both exist, and which one actually runs is decided at runtime by the real type of the object, not by however the variable happened to be declared.

### Faking overloading

Python doesn't support true method overloading (defining the "same" method multiple times with different parameter lists). Instead, the same effect is faked with default arguments or `*args`/`**kwargs`:

```python
def withdraw(amount, limit=None):
    if limit:
        print(f"Withdrawing {amount}, limit {limit}")
    else:
        print(f"Withdrawing {amount}")

withdraw(500)
withdraw(500, 1000)
```

One function, but its behavior flexes depending on how it's called — a lighter-weight cousin of true overloading.

### Dunder methods and operator overloading

Dunder ("double underscore") methods are Python's plug points into its own built-in syntax — they're how your objects get to participate in `+`, `==`, `len()`, `print()`, and friends, rather than being stuck as second-class citizens next to Python's native types.

```python
class Money:
    def __init__(self, amount):
        self.amount = amount

    def __add__(self, other):
        return Money(self.amount + other.amount)

    def __eq__(self, other):
        return self.amount == other.amount

    def __str__(self):
        return f"${self.amount}"

    def __repr__(self):
        return f"Money({self.amount})"
```

Now `m1 + m2` quietly calls `__add__`, `m1 == m2` calls `__eq__`, and `print(m1)` calls `__str__`. `__str__` is the friendly, user-facing display; `__repr__` is aimed at developers debugging in a console and should ideally be unambiguous enough to help you understand the object at a glance.

Other protocols worth knowing:

```
__len__      → len(obj)
__iter__     → iter(obj)
__getitem__  → obj[key]
__contains__ → value in obj
```

This is polymorphism in action: the *same* `+` symbol means something different depending on whether you're adding two numbers, two strings, or two `Money` objects — the object itself decides how to respond.

## 11. Pillar 4 — Abstraction

Abstraction means exposing a clean "what this does" surface while hiding the messy "how it actually works" underneath — like using an ATM without needing to know how the bank verifies your card or physically counts the cash.

Python does this formally with **abstract base classes**:

```python
from abc import ABC, abstractmethod

class PaymentProcessor(ABC):
    @abstractmethod
    def process(self, amount):
        ...
```

A subclass *must* implement `process()` before Python will let you instantiate it:

```python
class StripeProcessor(PaymentProcessor):
    def process(self, amount):
        print(f"Processing {amount}")
```

Trying to instantiate `PaymentProcessor` directly raises an error, since it's still missing a real implementation. ABCs are worth reaching for when you genuinely need an enforced contract between multiple implementations — not simply to make a design *look* more "object-oriented."

## 12. Dataclasses

Dataclasses exist for the common case where a class is mostly just a bundle of data.

Instead of the boilerplate:

```python
class User:
    def __init__(self, name, age):
        self.name = name
        self.age = age
```

you can write:

```python
from dataclasses import dataclass

@dataclass
class User:
    name: str
    age: int
```

Python auto-generates `__init__`, `__repr__`, and (by default) equality behavior for you.

Defaults work as you'd expect, but mutable defaults need `default_factory` to avoid the shared-list trap:

```python
from dataclasses import dataclass, field

@dataclass
class Team:
    members: list[str] = field(default_factory=list)
```

A **frozen** dataclass blocks normal attribute reassignment after creation — handy when an object's state genuinely shouldn't change once it exists:

```python
@dataclass(frozen=True)
class Point:
    x: int
    y: int
```

## 13. Pydantic models

Pydantic models look similar to dataclasses on the surface, but they're built around **validating and parsing** data, not just storing it.

```python
from pydantic import BaseModel

class User(BaseModel):
    name: str
    age: int

user = User(name="Alice", age=30)
```

Pydantic checks the supplied data against the declared field types — feeding it clearly invalid data raises a validation error instead of silently producing a broken object.

### Dataclass vs Pydantic — which one?

They solve different problems, so the real question isn't "internal vs external" as a hard rule — it's:

> Do I mainly need a convenient Python container for data I already trust, or do I need to validate and parse data crossing a boundary I *don't* fully trust (HTTP requests, config files, external API responses, raw user input)?

Reach for a plain **dataclass** when the former is the concern. Reach for **Pydantic** when validation and untrusted boundaries are the actual problem you're solving.

## 14. Real-world OOP design

The point of OOP was never "turn every piece of data into a class." It's to model state, behavior, and relationships in a way that keeps the system easy to understand and change later.

### Exercise — Banking system

Design a small banking system starting with `Customer`, `Account`, `Transaction`. It should support: creating accounts, depositing, withdrawing, checking balances, recording transactions, preventing invalid withdrawals, and identifying account owners.

Questions to sit with:

- Which data belongs to `Customer` vs `Account`?
- Should a `Transaction` be its own object?
- Which relationships here are composition rather than inheritance?
- Where should validation actually live?
- Which values deserve to be an `Enum`?
- Would a dataclass fit any of these models?

Resist the urge to build one giant `Bank` class that does everything — the exercise is really about deciding *where responsibility belongs*.

```python
from dataclasses import dataclass, field
from datetime import datetime, timezone
from decimal import Decimal
from enum import Enum
from uuid import uuid4

class AccountType(Enum):
    CHECKING = "checking"
    SAVINGS = "savings"


class TransactionType(Enum):
    DEPOSIT = "deposit"
    WITHDRAWAL = "withdrawal"

class AccountError(Exception):
    """Base exception for account-related errors."""


class InvalidAmountError(AccountError):
    """Raised when a deposit or withdrawal amount is invalid."""


class InsufficientFundsError(AccountError):
    """Raised when an account cannot cover a withdrawal."""

@dataclass
class Transaction:
    transaction_type: TransactionType
    amount: Decimal
    timestamp: datetime = field(
        default_factory=lambda: datetime.now(timezone.utc)
    )
    transaction_id: str = field(default_factory=lambda: str(uuid4()))


@dataclass
class Customer:
    customer_id: str
    name: str
    email: str
    accounts: list["Account"] = field(default_factory=list)

    def add_account(self, account: "Account") -> None:
        self.accounts.append(account)


@dataclass
class Account:
    account_number: str
    owner: Customer
    account_type: AccountType
    _balance: Decimal = Decimal("0")
    transactions: list[Transaction] = field(default_factory=list)

    @property
    def balance(self) -> Decimal:
        return self._balance

    def deposit(self, amount: Decimal) -> None:
        self._validate_positive_amount(amount)

        self._balance += amount

        self.transactions.append(
            Transaction(
                transaction_type=TransactionType.DEPOSIT,
                amount=amount,
            )
        )

    def withdraw(self, amount: Decimal) -> None:
        self._validate_positive_amount(amount)

        if amount > self._balance:
            raise InsufficientFundsError(
                f"Cannot withdraw {amount}; "
                f"balance is {self._balance}"
            )

        self._balance -= amount

        self.transactions.append(
            Transaction(
                transaction_type=TransactionType.WITHDRAWAL,
                amount=amount,
            )
        )

    @staticmethod
    def _validate_positive_amount(amount: Decimal) -> None:
        if amount <= 0:
            raise InvalidAmountError(
                "Amount must be greater than zero"
            )


class Bank:
    def __init__(self):
        self.customers: dict[str, Customer] = {}
        self.accounts: dict[str, Account] = {}

    def create_customer(
        self,
        name: str,
        email: str,
    ) -> Customer:
        customer = Customer(
            customer_id=str(uuid4()),
            name=name,
            email=email,
        )

        self.customers[customer.customer_id] = customer

        return customer

    def create_account(
        self,
        customer: Customer,
        account_type: AccountType,
    ) -> Account:
        account = Account(
            account_number=str(uuid4()),
            owner=customer,
            account_type=account_type,
        )

        self.accounts[account.account_number] = account
        customer.add_account(account)

        return account

    def get_account(self, account_number: str) -> Account:
        try:
            return self.accounts[account_number]
        except KeyError:
            raise ValueError("Account not found")

def main():
    bank = Bank()

    alice = bank.create_customer(
        name="Alice",
        email="alice@example.com",
    )

    account = bank.create_account(
        customer=alice,
        account_type=AccountType.CHECKING,
    )

    account.deposit(Decimal("100.00"))
    account.withdraw(Decimal("30.00"))

    for transaction in account.transactions:
        print(
            transaction.transaction_type.value,
            transaction.amount,
        )

    try:
        account.withdraw(Decimal("500"))
    except InsufficientFundsError:
        print("Not enough money")
    except InvalidAmountError:
        print("Invalid amount")


if __name__ == "__main__":
    main()
```

### Exercise — Inventory system

Design an inventory system starting with `Product`, `Inventory`, `Category`. It should support: creating products, tracking and adjusting stock, preventing negative inventory, looking up products, changing prices, and identifying categories.

```
Product
    name, price, category, stock

Inventory
    collection of products
    add/remove stock, find products
```

Consider where dataclasses, enums, properties, composition, validation, plain dictionaries for lookup, and Pydantic models (at any real external boundary) each earn their place.

### Putting several pillars together

A single worked example can show how the pillars cooperate rather than compete — an abstract `Account` base, two concrete subclasses overriding `account_type()`, a `Bank` that *has* a list of accounts (composition), and a `__repr__` for friendly debugging:

```python
from abc import ABC, abstractmethod

class Account(ABC):
    bank_name = "Example Bank"

    def __init__(self, owner, balance=0):
        self.owner = owner
        self._balance = balance

    @abstractmethod
    def account_type(self):
        ...

    def deposit(self, amount):
        if amount > 0:
            self._balance += amount

    def withdraw(self, amount):
        if 0 < amount <= self._balance:
            self._balance -= amount

    def __repr__(self):
        return f"{self.__class__.__name__}(owner={self.owner!r}, balance={self._balance!r})"


class SavingsAccount(Account):
    def __init__(self, owner, balance, interest_rate):
        super().__init__(owner, balance)
        self.interest_rate = interest_rate

    def account_type(self):
        return "Savings"


class CurrentAccount(Account):
    def account_type(self):
        return "Current"


class Bank:
    def __init__(self):
        self.accounts = []          # composition: Bank HAS-A list of Accounts

    def add_account(self, account):
        self.accounts.append(account)

    def total_balance(self):
        return sum(a._balance for a in self.accounts)
```

One small model, and all four pillars are already at work: **abstraction** (`Account` as an ABC), **inheritance** (`SavingsAccount`/`CurrentAccount`), **polymorphism** (`account_type()` behaves differently per subclass), **encapsulation** (`_balance` accessed through methods), and **composition** (`Bank` holding `Account` objects) — without any deep or fragile inheritance chain.

## 15. OOP design principles

- **Keep responsibilities focused.** A class should have one clear purpose. Avoid a "god class" that mixes database access, validation, HTTP handling, business logic, and formatting all in one place.
- **Prefer composition when appropriate.** Don't build an inheritance relationship just to reuse a couple of methods — if an object simply *has* another object, model it that way.
- **Avoid unnecessary inheritance.** Inheritance creates a real relationship between types; use it because the relationship is meaningful, not just because two classes happen to share some code.
- **Put behavior near the data it operates on.** `account.withdraw(100)` is usually clearer than `bank.process_withdrawal(account, 100)` when withdrawing is fundamentally the account's own responsibility.
- **Don't create classes just because you can.** A dictionary, a function, or a plain dataclass may be the better fit. Good OOP isn't measured by how many classes you wrote.

## 16. Module summary

```
classes
    ↓
instances
    ↓
instance / class state
    ↓
methods (instance, class, static)
    ↓
encapsulation & properties
    ↓
inheritance, super(), MRO
    ↓
composition
    ↓
polymorphism (overriding, dunder methods, operator overloading)
    ↓
abstraction (ABCs)
    ↓
dataclasses
    ↓
Pydantic models
    ↓
object-oriented design
```

---
# Module 7: Modules, Packages & Python Projects

### 1. What Is a Module?

A module is simply a Python file.

```python
# math_utils.py

def add(a, b):
    return a + b

def subtract(a, b):
    return a - b
```

Another file can import it:

```python
from math_utils import add

print(add(2, 3))
```

A module lets you split a large program into logical pieces. Instead of:

```text
app.py   (5000 lines)
```

you can have:

```text
app/
    users.py
    products.py
    payments.py
```

### 2. `import`

```python
import math

math.sqrt(16)
```

Or import specific names:

```python
from math import sqrt

sqrt(16)
```

You can also use an alias:

```python
import pandas as pd
```

Prefer explicit imports. Avoid wildcard imports:

```python
from module import *
```

They make it unclear where a name came from, and they can silently overwrite names already in your namespace.

### 3. Modules Are Executed

When Python imports a module, its top-level code runs once per process, then the result is cached in `sys.modules`. Importing the same module again just reuses the cached version instead of re-running it.

```python
# config.py
print("Loading configuration...")
DEBUG = True
```

```python
import config
# prints: Loading configuration...
```

This is why you should generally avoid putting arbitrary executable code (network calls, file writes, heavy computation) at module level. Side effects at import time make a module hard to reason about and hard to test.

### 4. `__name__ == "__main__"`

```python
# app.py

def main():
    print("Running application")

if __name__ == "__main__":
    main()
```

If you run:

```bash
python app.py
```

then `__name__ == "__main__"` is true.

If another module imports `app`:

```python
import app
```

then `__name__ == "app"`, and `main()` does not automatically execute.

This lets the same file be both importable as a module and runnable as a script.

### 5. Packages

A package is a directory containing related modules.

```text
shop/
    users.py
    products.py
    orders.py
```

You can import:

```python
from shop.products import Product
```

Modern Python can use **namespace packages** (no `__init__.py` required, PEP 420) but regular packages still commonly include `__init__.py` because it makes the package boundary explicit and gives you a place for initialization code.

### 6. `__init__.py`

```text
shop/
    __init__.py
    products.py
    users.py
```

`__init__.py` marks the directory as a regular package and can expose selected package-level names:

```python
# shop/__init__.py
from .products import Product
```

Then:

```python
from shop import Product
```

Don't turn `__init__.py` into a dumping ground for imports. Keep package boundaries understandable.

### 7. Absolute vs Relative Imports

Absolute:

```python
from shop.products import Product
```

Relative:

```python
from .products import Product
```

Relative imports are useful inside a package when referring to sibling modules:

```text
shop/
    __init__.py
    products.py
    orders.py
```

Inside `orders.py`:

```python
from .products import Product
```

Note: relative imports only work when the file is imported as part of a package. Running a file directly with `python shop/orders.py` will fail on a relative import, because the script is then executed as `__main__`, not as `shop.orders`. Run it as a module instead: `python -m shop.orders`.

### 8. Circular Imports

A common problem:

```text
users.py  -> imports orders.py
orders.py -> imports users.py
```

This creates a circular dependency. Usually it's a sign that responsibilities or module boundaries need restructuring. Don't immediately "fix" circular imports by moving imports inside functions. That can sometimes be a valid workaround, but ask first: *why do these modules need to depend on each other?* Often the real fix is extracting the shared piece (a type, a constant, an interface) into a third module that both can import from.

### 9. Virtual Environments

A virtual environment isolates a project's Python packages.

Without isolation:

```text
Project A -> requests 2.x
Project B -> requests 3.x
```

can conflict.

With virtual environments:

```text
project-a/.venv
project-b/.venv
```

each project has its own dependencies, installed independently of your system Python.

### 10. uv

`uv` is a modern, Rust-based Python project and package manager from Astral (the team behind Ruff). It replaces the pip + pip-tools + pipx + poetry + pyenv + virtualenv combo with one tool.

Create a project:

```bash
uv init my-project
cd my-project
```

Pin a Python version for the project (writes `.python-version`):

```bash
uv python pin 3.12
```

Create/sync the environment from the lockfile:

```bash
uv sync
```

Add a dependency:

```bash
uv add requests
```

Add a dev-only dependency (test/lint tools that shouldn't ship to production):

```bash
uv add --dev pytest ruff mypy
```

Run a command inside the project's environment (no manual activate/deactivate needed):

```bash
uv run python main.py
uv run pytest
```

Other useful commands:

```bash
uv remove requests          # drop a dependency
uv lock --upgrade           # refresh the lockfile to latest compatible versions
uv tree                     # show the dependency tree
uv build                    # build sdist/wheel
uv publish                  # publish to PyPI
uvx ruff check .            # run a tool without installing it into the project
```

The important idea:

```text
pyproject.toml  -> project configuration + dependencies
uv.lock         -> exact resolved dependency versions (for reproducible installs)
.venv/          -> the actual project environment
```

Commit `pyproject.toml` and `uv.lock`. Do not commit `.venv/`.

### 11. `pyproject.toml`

Modern Python projects use `pyproject.toml` as the central project configuration file.

```toml
[project]
name = "inventory-app"
version = "0.1.0"
description = "Inventory application"
requires-python = ">=3.12"

dependencies = [
    "pydantic",
]

[dependency-groups]
dev = [
    "pytest",
    "ruff",
    "mypy",
]
```

Note: dev dependencies added with `uv add --dev` land in the `[dependency-groups]` table (this follows PEP 735). Older tutorials show `[tool.uv.dev-dependencies]`, an earlier uv-specific convention; both still work but `[dependency-groups]` is the current standard.

Tools such as Ruff, pytest, and mypy can also be configured through `pyproject.toml` under their own `[tool.*]` tables, e.g. `[tool.ruff]`, `[tool.pytest.ini_options]`, `[tool.mypy]`.

### 12. Project Layout

A small project:

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
│       └── main.py
└── tests/
    ├── test_models.py
    └── test_services.py
```

The `src` layout helps prevent accidentally importing your source code from the project root instead of the actually-installed package (a common source of "it works on my machine but not after packaging" bugs).

For a larger application, organize around responsibilities (domain areas, layers) rather than one enormous module.

### 13. Environment Variables

Configuration that changes between environments should not usually be hard-coded.

Bad:

```python
DATABASE_URL = "postgres://production..."
```

Instead:

```python
import os

DATABASE_URL = os.environ["DATABASE_URL"]
```

Use `os.environ["KEY"]` when the variable is required (it raises `KeyError` if missing, which fails fast), and `os.environ.get("KEY", default)` when it's optional.

Environment variables are particularly useful for:

- database URLs
- API keys
- secrets
- per-environment configuration (dev/staging/prod)

Never commit secrets into source control. For local development, a `.env` file loaded with a library like `python-dotenv` (or Pydantic's `BaseSettings`) is a common pattern, paired with `.env` in `.gitignore`.

### 14. `__all__`

A module can define:

```python
__all__ = ["User", "create_user"]
```

This communicates which names are the intended public API of the module, and it controls what `from module import *` pulls in. It's useful for library design, but don't add it everywhere just because it exists; it's most valuable on modules that are actually imported by other code.

### 15. Real-World Exercise: Build a Python Project

Take the inventory domain from the Classes module. Create:

```text
inventory-app/
├── pyproject.toml
├── src/
│   └── inventory/
│       ├── __init__.py
│       ├── models.py
│       ├── inventory.py
│       ├── services.py
│       └── main.py
└── tests/
```

Requirements:

- initialize it with `uv`
- create a virtual environment
- add Pydantic
- separate models from business logic
- create a clean package
- expose only the intended public API
- make the project runnable with `uv run`
- keep tests separate from application code

The goal isn't to build a huge application. It's to understand how the Python files you've been writing become a real, structured project.

---

# Module 8: Standard Library (The Python Toolbox)

Python's standard library solves a surprising number of problems without needing third-party packages. The skill isn't memorizing every module, it's recognizing: *"Python probably already has something for this."*

### 1. `collections`

**`Counter`**, for counting occurrences:

```python
from collections import Counter

counts = Counter(["api", "web", "api", "api", "web"])
print(counts)
counts.most_common(2)
```

Useful for frequencies, log analysis, rankings, top-N values. `Counter` is a `dict` subclass, so it supports normal dict operations, plus arithmetic between counters (`+`, `-`, `&`, `|`).

Remember: `most_common(n)` returns the top `n` entries (including ties that happen to fall within that cutoff), but it does not expand the result to include everyone tied with the nth item. If you need "everyone tied for second," define that explicitly.

**`defaultdict`**, for a container that creates a missing value automatically:

```python
from collections import defaultdict

by_department = defaultdict(list)

for employee in employees:
    by_department[employee["department"]].append(employee)
```

Instead of:

```python
if department not in groups:
    groups[department] = []
groups[department].append(employee)
```

Use `defaultdict` when the missing-key behavior is part of the design.

**`deque`**, for efficient operations from both ends:

```python
from collections import deque

queue = deque()
queue.append("A")
queue.append("B")
queue.popleft()
```

A `list` is O(n) for inserting/removing at the front; a `deque` is O(1) at both ends. Use it for queues, sliding windows, recent-item buffers, and breadth-first search. Pass `maxlen=n` to get a fixed-size rolling buffer that automatically drops old items.

### 2. `itertools`

Useful tools: `chain()`, `islice()`, `groupby()`, `product()`, `permutations()`, `combinations()`. The important skill is recognizing when you're manually re-implementing something `itertools` already provides.

### 3. `functools`

Covered elsewhere: `cache`, `lru_cache`, `partial`, `wraps`, `reduce`, `singledispatch`.

The important production question isn't "can I use `cache`?" It's "is caching correct for this data, and how will stale values be handled?" (e.g. does the underlying data change over time, and does the cache need an eviction policy or a manual `.cache_clear()`?)

### 4. `pathlib`

Use `Path` for filesystem paths; it's the modern, preferred alternative to string-based `os.path`.

```python
from pathlib import Path

data_dir = Path("data")

for file in data_dir.glob("*.csv"):
    print(file)
```

Useful operations: `path.exists()`, `path.is_file()`, `path.is_dir()`, `path.read_text()`, `path.write_text()`, `path.mkdir()`, `Path.cwd()`, `path.resolve()`. Paths join with `/`: `data_dir / "raw" / "file.csv"`.

### 5. `datetime`

```python
from datetime import datetime, timezone

now = datetime.now(timezone.utc)
```

Prefer timezone-aware datetimes for real-world timestamps. Avoid casually mixing naive datetimes with timezone-aware ones (comparing or subtracting a naive and an aware datetime raises `TypeError`).

Correction to a common older habit: `datetime.utcnow()` and `datetime.utcfromtimestamp()` were deprecated in Python 3.12 (they return naive datetimes, which is the source of many subtle bugs) and are scheduled for removal in a future version. Use `datetime.now(timezone.utc)` instead (or `datetime.now(datetime.UTC)` on 3.11+, since `datetime.UTC` is an alias for `timezone.utc`).

Parsing:

```python
timestamp = datetime.fromisoformat("2026-08-26T12:30:00")
```

Formatting:

```python
timestamp.strftime("%Y-%m-%d")
```

`timedelta` is the companion type for durations and date arithmetic: `now + timedelta(days=7)`.

### 6. `json`

```python
import json

data = {"name": "Alice", "age": 30}
text = json.dumps(data)          # serialize
data = json.loads(text)          # deserialize

with open("data.json") as file:
    data = json.load(file)
```

`json` only natively handles the basic JSON types (str, int, float, bool, None, list, dict). It can't serialize things like `datetime` or `set` out of the box; for those, pass a `default=` function to `json.dumps` that converts the object to something serializable, or use a library like Pydantic for structured (de)serialization. Use `json.dumps(data, indent=2)` for readable output.

### 7. `csv`

```python
import csv

with open("users.csv", newline="") as file:
    reader = csv.DictReader(file)
    for row in reader:
        print(row)
```

`DictReader` is useful when CSV columns represent named fields, and it's naturally lazy: rows are produced as you iterate, not loaded all at once into memory. Note that every value comes back as a string; you're responsible for converting types (`int(row["age"])`, etc.). For writing, `csv.writer` and `csv.DictWriter` are the counterparts.

### 8. `sqlite3`

SQLite gives you a lightweight relational database directly in the standard library.

```python
import sqlite3

connection = sqlite3.connect("app.db")

connection.execute("""
    CREATE TABLE IF NOT EXISTS users (
        id INTEGER PRIMARY KEY,
        name TEXT NOT NULL
    )
""")

connection.execute(
    "INSERT INTO users (name) VALUES (?)",
    ("Alice",),
)
connection.commit()
```

The `?` placeholder matters: never build SQL with string interpolation.

```python
# Don't do this:
f"SELECT * FROM users WHERE name = '{name}'"
```

Parameterized queries protect against SQL injection.

Gotcha worth knowing: `with connection:` (see below) commits or rolls back the transaction on exit, but it does **not** close the connection. You still need `connection.close()` when you're done. Setting `connection.row_factory = sqlite3.Row` gets you dict-like access to result rows instead of plain tuples.

### 9. Transactions

Transactions group database operations into an atomic unit.

```python
with connection:
    connection.execute(...)
    connection.execute(...)
```

If the block succeeds, it commits. If an exception occurs, it rolls back. This connects directly to the transaction context manager idea from earlier modules.

### 10. `logging`

Don't use `print()` as your application's logging system.

```python
import logging

logger = logging.getLogger(__name__)

logger.info("Application started")
logger.warning("Slow request")
logger.error("Database connection failed")
```

Typical levels: `DEBUG`, `INFO`, `WARNING`, `ERROR`, `CRITICAL`.

Logging gives you timestamps, severity, structured output, configurable destinations, and production observability. In a library (as opposed to a top-level application), avoid calling `logging.basicConfig()` or otherwise configuring the root logger; let the application that uses your library decide how logs are handled. `logger.addHandler(logging.NullHandler())` is the polite default for library code.

### 11. `heapq`

`heapq` implements a priority queue on top of a plain list.

```python
import heapq

tasks = []
heapq.heappush(tasks, (2, "normal"))
heapq.heappush(tasks, (1, "urgent"))
heapq.heappop(tasks)
```

The smallest item comes out first (`tasks[0]` is always the smallest). Useful for priority queues, scheduling, top-N algorithms, and graph algorithms (like Dijkstra's). If you only need the largest or smallest few values, `heapq.nlargest(n, items)` / `heapq.nsmallest(n, items)` can be more efficient than sorting the whole collection.

### 12. `bisect`

`bisect` works with already-sorted lists using binary search.

```python
from bisect import bisect_left, insort

numbers = [1, 3, 5, 7]
index = bisect_left(numbers, 5)   # finds an insertion position

insort(numbers, 4)
# numbers is now [1, 3, 4, 5, 7]
```

`bisect_left` and `bisect_right` differ in where they place a value that already exists in the list (before vs. after existing equal entries). The list must already be sorted for these functions to behave correctly.

### 13. `statistics`

Basic statistical operations without pulling in a heavier dependency:

```python
from statistics import mean, median, stdev, variance

values = [10, 20, 30]
mean(values)
median(values)
stdev(values)
variance(values)
```

Use the standard library for simple summary statistics. Reach for NumPy or pandas once you need vectorized operations over large datasets, or more advanced statistics.

### 14. `typing` and `collections.abc`

Common types: `list[str]`, `dict[str, int]`, `str | None` (these built-in generics work directly since Python 3.9, without importing `List`/`Dict` from `typing`).

For behavior-based typing: `from collections.abc import Iterable, Iterator, Callable`.

For structural interfaces (duck typing with type checking): `from typing import Protocol`.

### 15. A Few More Worth Knowing

- **`os`**: lower-level system interface (`os.environ`, `os.getcwd()`, `os.rename()`). `pathlib` covers most filesystem needs more pleasantly, but `os.environ` is still the standard way to read environment variables.
- **`re`**: regular expressions, for pattern matching in text such as log lines. `re.match`, `re.search`, `re.findall`, and compiled patterns via `re.compile()` for reuse in a hot loop.
- **`argparse`**: build command-line interfaces with named arguments, flags, and help text, useful for turning a script into a proper CLI tool.
- **`enum`**: define a fixed set of named values (`class Status(Enum): ACTIVE = "active"`), better than magic strings for things like categories or states.
- **`contextlib`**: helpers for building your own context managers, notably the `@contextmanager` decorator for writing a `with`-compatible object as a single generator function.

### Real-World Exercise: Log Analytics Tool

Build a command-line log analyzer.

```text
logs/
    server-01.log
    server-02.log
    server-03.log
```

The program should:

- discover files with `pathlib`
- read them lazily
- parse each line (consider `re` for structured log lines)
- count status codes with `Counter`
- group requests with `defaultdict`
- track recent errors with `deque`
- parse timestamps with `datetime`
- save a JSON report
- store historical summaries in SQLite
- log processing failures with `logging`

Then add: find the 10 most frequently requested endpoints without sorting the entire dataset. Think about whether `heapq.nlargest` is appropriate here.

**Standard Library Rule:** before installing a package, ask "can the standard library already solve this?" Not every problem should be solved with the standard library, but knowing what's available prevents unnecessary dependencies.

---

## Module 9: Testing & Developer Tooling

### 1. Why Test?

Testing isn't primarily about proving your code is correct. It's about making changes safer. A useful test suite lets you ask "did I break anything?" after changing the code.

### 2. `pytest`

Install:

```bash
uv add --dev pytest
```

A basic test:

```python
def add(a, b):
    return a + b

def test_add():
    assert add(2, 3) == 5
```

Run:

```bash
uv run pytest
```

### 3. Arrange, Act, Assert

```python
def test_withdraw():
    # Arrange
    account = Account(balance=100)

    # Act
    account.withdraw(40)

    # Assert
    assert account.balance == 60
```

Keep tests easy to read.

### 4. Test Behavior, Not Implementation

Bad: `assert internal_dictionary_exists`

Better: assert that withdrawing 40 reduces the balance by 40.

Tests should describe what the system promises to do. This makes refactoring safer, because the test doesn't care how the promise is fulfilled internally.

### 5. Fixtures

Fixtures provide reusable test setup.

```python
import pytest

@pytest.fixture
def account():
    return Account(balance=100)

def test_withdraw(account):
    account.withdraw(40)
    assert account.balance == 60
```

Fixtures are especially useful for database setup, temporary files, test clients, and other reusable objects. A few things worth knowing:

- Put fixtures shared across multiple test files in a `conftest.py`; pytest auto-discovers it, no import needed.
- Fixtures have a `scope` (`function` by default, also `class`, `module`, `session`) that controls how often they're recreated.
- `tmp_path` and `monkeypatch` are built-in fixtures worth knowing: `tmp_path` gives you a throwaway directory per test, and `monkeypatch` safely patches attributes, dict items, or environment variables for the duration of a test and undoes it afterward.

### 6. Parametrization

Avoid duplicating almost-identical tests.

```python
@pytest.mark.parametrize(
    "value,expected",
    [
        (1, 2),
        (2, 4),
        (3, 6),
    ],
)
def test_double(value, expected):
    assert double(value) == expected
```

One test definition can cover many cases.

### 7. Testing Exceptions

```python
import pytest

def test_negative_price():
    with pytest.raises(ValueError):
        Product(name="Keyboard", price=-10)
```

You're testing the contract: invalid input should raise this exception.

### 8. Mocking

Sometimes a test shouldn't call a real external service.

```python
def get_weather():
    return requests.get(...).json()
```

A unit test shouldn't need the internet. Mock the dependency instead:

```python
from unittest.mock import Mock

response = Mock()
response.json.return_value = {"temperature": 25}
```

The principle: replace expensive, slow, or unpredictable external dependencies with controlled test doubles. `unittest.mock.patch` (as a decorator or context manager) is the usual way to swap out a real function/object for a mock during a test. The `pytest-mock` plugin wraps this in a convenient `mocker` fixture if you'd rather not manage `patch` context managers directly.

### 9. Mock Only at the Boundary

Don't mock everything. If you're testing business logic, mock the database, the HTTP API, the filesystem, or the message queue, when appropriate. Don't mock your own business logic until the test becomes meaningless.

### 10. Async Testing

Async functions need to be tested asynchronously, and plain pytest can't do this on its own: an `async def test_...` without help gets collected and "passes" instantly without actually running, because the coroutine is never awaited. You need the separate **`pytest-asyncio`** plugin.

```bash
uv add --dev pytest pytest-asyncio
```

```toml
# pyproject.toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
```

With `asyncio_mode = "auto"`, async tests and fixtures are picked up automatically. In `strict` mode (the plugin's default if unset), you need to mark each one explicitly:

```python
import pytest

@pytest.mark.asyncio
async def test_fetch():
    result = await fetch_data()
    assert result == expected
```

Async applications often require mocking async dependencies too (`unittest.mock.AsyncMock`, or an async-aware HTTP mocking library).

### 11. Unit vs Integration Tests

**Unit test**: tests one component in isolation (`function -> test`). Fast and numerous.

**Integration test**: tests multiple real components together (`application -> database -> response`). Slower, but catches integration problems a unit test would miss.

A healthy project normally has both.

### 12. Test the Important Boundaries

Good candidates for tests: validation, business rules, database interactions, API boundaries, error handling, authentication/authorization, and data transformations. Don't chase 100% coverage just because the number looks good; coverage is a signal, not the goal. (`pytest-cov` gives you a coverage report if you want to track this.)

### 13. Ruff

Ruff is a very fast Python linter (and formatter) from Astral, the same team behind `uv`.

```bash
uv run ruff check .
```

It catches unused imports, suspicious code, style problems, and common mistakes, and it has effectively replaced older tools like flake8 and isort for a lot of projects because it reimplements their checks much faster.

Format code:

```bash
uv run ruff format .
```

Ruff's formatter targets Black-compatible output, so if you're already using Ruff for linting, you often don't need a separate formatter.

### 14. Black

Black is the original opinionated Python formatter.

```bash
black .
```

It formats Python code consistently with very few configuration knobs by design. Plenty of projects still use it directly; others have moved to `ruff format` for the speed and one-less-tool benefit. The important thing either way: pick a formatter and use it consistently, don't manually debate formatting in every code review.

### 15. Mypy

Mypy performs static type checking.

```python
def add(a: int, b: int) -> int:
    return a + b

add("hello", 10)  # mypy flags this before runtime
```

Run:

```bash
uv run mypy src/
```

Type hints become much more useful once tooling actually checks them. `mypy --strict` turns on a stricter rule set (no implicit `Any`, requires annotations, etc.); it's a good target for new projects, less practical to bolt onto a large untyped codebase all at once. `pyright` (from Microsoft, also what powers Pylance in VS Code) is a common alternative type checker.

### 16. Debugging

Start with the traceback. Don't immediately jump to the final line. Read:

```text
Traceback -> where the call started -> which function called what -> where the failure actually occurred
```

Ask: what exception happened, where was it raised, what values existed at that point, and why did those values become invalid?

For interactive debugging, Python's built-in `breakpoint()` (Python 3.7+) drops you into `pdb` at that line without an explicit import; `ipdb` is a popular drop-in replacement with a nicer interface if you have it installed.

### 17. Logging During Debugging

Instead of `print(user)`, use `logger.debug("Processing user %s", user.id)`. This lets you enable or disable diagnostic output without modifying the application code, and it keeps a permanent, leveled record instead of one-off print statements you have to remember to remove.

### 18. A Useful Development Workflow

A practical loop:

```text
write code -> run tests -> lint -> format -> type check -> commit
```

For example:

```bash
uv run pytest
uv run ruff check .
uv run ruff format .
uv run mypy src/
```

Eventually automate this in CI, and consider `pre-commit` to run the fast checks (lint, format) automatically before each commit so issues get caught before they even reach CI.

### Real-World Exercise: Test the Inventory Application

Take the inventory domain from the Classes module. Write tests for:

- creating a product
- invalid prices
- adding stock
- removing stock
- preventing negative stock
- looking up products
- unknown product IDs
- category validation
- concurrent stock updates
- external persistence

Mock the database or external service rather than requiring it for every unit test. Then configure `pytest`, `ruff`, and `mypy` so the entire project can be checked consistently.

---

## Module 10: Pythonic Best Practices

This module is less about new syntax and more about how experienced Python developers make decisions.

### 1. Prefer Simple Code

Python rewards readable code.

Bad:

```python
result = [x for x in [y for y in data if y.active]]
```

Good:

```python
active_users = [user for user in users if user.active]
```

Don't optimize for cleverness. Optimize for code another developer can understand quickly.

### 2. Use the Right Data Structure

Ask what operation matters most:

- ordered mutable collection -> `list`
- key to value lookup -> `dict`
- uniqueness -> `set`
- counting -> `Counter`
- grouping -> `defaultdict`
- queue -> `deque`
- fixed immutable record -> `tuple` / `namedtuple`
- structured mutable data -> `dataclass`

Your choice should follow the problem, not personal preference.

### 3. Don't Search a List When You Need a Lookup

Bad:

```python
for user in users:
    if user["id"] == user_id:
        return user
```

If this happens repeatedly, build a lookup once:

```python
users_by_id = {user["id"]: user for user in users}
```

Then: `users_by_id[user_id]`. The difference is small on a tiny list, and enormous as the dataset grows (O(n) scan vs O(1) lookup).

### 4. Use Sets for Membership

Bad, when membership is the primary operation:

```python
if user_id in user_ids_list:
    ...
```

Prefer:

```python
user_ids = set(user_ids)
if user_id in user_ids:
    ...
```

### 5. EAFP vs LBYL

Python often favors EAFP, "Easier to Ask Forgiveness than Permission," over LBYL, "Look Before You Leap."

Instead of checking everything first:

```python
if key in data:
    process(data[key])
```

you may use:

```python
try:
    process(data[key])
except KeyError:
    ...
```

This isn't a rule that every lookup must be wrapped in `try`. Use whichever expresses the intent more clearly for the situation.

### 6. Don't Mutate Unexpectedly

Always know whether an operation mutates the object or returns a new one. For example, `numbers.sort()` mutates the list in place (and returns `None`), while `sorted(numbers)` returns a new sorted list and leaves the original untouched. This distinction is fundamental to writing predictable Python.

### 7. Avoid Mutable Default Arguments

Bad:

```python
def add(item, items=[]):
    items.append(item)
    return items
```

The same list object is reused between calls, so appends accumulate across unrelated calls. Use:

```python
def add(item, items=None):
    if items is None:
        items = []
    items.append(item)
    return items
```

For dataclasses, use `field(default_factory=list)` instead of `items: list = []`.

### 8. Don't Catch Exceptions You Can't Handle

Bad:

```python
try:
    process()
except Exception:
    pass
```

This hides bugs. Instead:

```python
try:
    process()
except FileNotFoundError:
    recover()
```

Catch an exception when you can actually do something useful about it; otherwise let it propagate so it's visible.

### 9. Don't Overuse Classes

Not everything needs to become a class.

- **function**, when behavior is simple
- **dataclass**, when structured data is the main concern
- **class**, when an object has meaningful state and behavior together
- **Pydantic model**, when validation, parsing, and serialization are central

### 10. Don't Overuse Inheritance

Deep hierarchies are often hard to understand. Prefer composition when possible:

```python
class Car:
    def __init__(self, engine):
        self.engine = engine
```

This lets you swap implementations without changing `Car` itself.

### 11. Don't Optimize Before Measuring

Readable code is often better than clever code that's 5% faster, unless performance actually matters. When it does:

```text
measure -> identify bottleneck -> change code -> measure again
```

Use profiling (e.g. `cProfile`, or `timeit` for small snippets) rather than guessing.

### 12. Complexity Matters

Know roughly how your data structures scale:

- dict lookup: O(1) average
- set membership: O(1) average
- list membership: O(n)
- list insertion: O(n), depending on position
- sorting: O(n log n)

You don't need to calculate Big O for every line, but you should recognize when an algorithm accidentally turns 1 million operations into 1 trillion operations.

### 13. Lazy Processing

If data is large:

```python
for row in rows:
    ...
```

is often better than:

```python
rows = list(rows)
```

Generators let you process streams without keeping everything in memory. This matters most for files, database results, APIs, logs, and data pipelines.

### 14. Standard Library First

Before adding a dependency, ask "can the standard library already solve this?"

- JSON -> `json`
- CSV -> `csv`
- paths -> `pathlib`
- queues -> `collections.deque`
- counting -> `Counter`
- dates -> `datetime`
- SQLite -> `sqlite3`

External packages are still appropriate when they provide substantial functionality. The goal isn't "never use dependencies," it's "don't add dependencies unnecessarily."

### 15. Separate Concerns

Avoid functions that do everything.

Bad:

```python
def process():
    read_file()
    parse_data()
    validate()
    call_api()
    save_database()
    print_report()
```

Better, as a pipeline of independently testable steps:

```text
read -> parse -> validate -> transform -> persist
```

### 16. Keep Side Effects at the Boundaries

A pure transformation is easy to test:

```python
def calculate_total(items):
    return sum(item.price for item in items)
```

A function that simultaneously reads a database, modifies global state, calls an API, and prints output is much harder to test. Keep side effects near the edges of your application, and keep the core logic pure where you can.

### 17. Type Hints Improve Design

Type hints don't just help static analysis, they communicate intent.

```python
def find_product(
    products: dict[int, Product],
    product_id: int,
) -> Product | None:
    ...
```

A reader immediately knows what goes in, what comes out, and whether the result can be missing.

### 18. Use `with` for Resources

Whenever an object represents a resource that needs cleanup, look for a context manager:

```python
with open(...) as file:
    ...

with sqlite3.connect(...) as connection:
    ...

async with session.get(...) as response:
    ...
```

The pattern is always: acquire, use, cleanup, even if an exception occurs.

### 19. Use `async` Only When It Helps

Async is excellent for I/O-heavy concurrency. It is not automatically faster. Don't write:

```python
async def calculate():
    return expensive_cpu_computation()
```

just because async is modern; an `async def` around pure CPU work still blocks the event loop while it runs.

- CPU-bound -> processes / parallelism (`multiprocessing`, `concurrent.futures.ProcessPoolExecutor`)
- I/O-bound -> async or threads
- Simple synchronous code -> keep it synchronous

### 20. A Few More Idioms Worth Knowing

- **f-strings** for readable string formatting: `f"{name} has {count} items"`.
- **`enumerate()`** instead of manually tracking an index: `for i, item in enumerate(items):`.
- **`zip()`** to iterate multiple sequences together: `for name, price in zip(names, prices):`.
- **the walrus operator `:=`** to assign and use a value in one expression, useful in a `while` loop or a comprehension condition: `while (chunk := file.read(1024)):`.
- **`match` statements** (Python 3.10+) as a more readable alternative to long `if`/`elif` chains when branching on a value's shape or type.
- Use `functools.reduce` sparingly; an explicit loop or a comprehension is usually more readable for the same result, and `reduce` is best reserved for cases where there's no clearer built-in equivalent (`sum`, `any`, `all`, etc. already cover the common ones).
- Follow PEP 8 for naming and layout conventions (`snake_case` for functions/variables, `PascalCase` for classes, `UPPER_CASE` for constants); Ruff and Black enforce most of this automatically so you don't have to police it by hand.

### 21. Common Interview Patterns

Recognize the underlying problem, not the trick:

```python
# Frequency counting
Counter(items)

# Grouping
groups = defaultdict(list)
for item in items:
    groups[item.key].append(item)

# Lookup table
by_id = {item.id: item for item in items}

# Deduplication
seen = set()

# Queue
queue = deque()

# Top N
heapq.nlargest(n, items)

# Lazy processing
(value for value in values if condition(value))

# Sorting by a field
sorted(users, key=lambda user: user.name)
```

### 22. Thinking in Python

When facing a new problem, don't immediately start writing loops. Ask:

1. What is the data? (list, dict, set, object, stream)
2. What operation matters? (lookup, membership, ordering, counting, grouping, queue)
3. How large can it become? (10 items, 10,000, 10 million)
4. Does it need to be lazy? Can I process one item at a time?
5. Is there already a standard-library solution? (`Counter`, `defaultdict`, `deque`, `heapq`, `itertools`)
6. Does this belong in a class? Don't assume the answer is yes.
7. Is this I/O-bound or CPU-bound? This determines whether async, threads, or processes make sense.
8. What can fail? Design exception handling deliberately.

### 23. A Practical Decision Process

```text
Understand the data
        v
Choose data structures
        v
Define responsibilities
        v
Choose synchronous / async
        v
Consider memory usage
        v
Handle expected failures
        v
Implement
        v
Test
        v
Measure if performance matters
        v
Refactor
```

This is much more valuable than memorizing isolated Python features.

### Final Real-World Exercise: Inventory Backend Design

You now have the major Python building blocks needed to design the domain of the inventory application. Design the system before implementing it.

Requirements:

- products have IDs, names, prices, categories, and stock
- products can be created and updated
- stock can be added or removed
- negative stock is invalid
- products can be looked up quickly by ID
- inventory can be persisted
- external supplier APIs provide stock updates
- supplier APIs may be slow or unavailable
- multiple suppliers can be queried concurrently
- failed supplier requests should be retried
- excessive concurrency should be prevented
- invalid data should be rejected
- important operations should be tested
- application errors should be logged

Decide:

**Data modeling.** Which should be a `dataclass`, a Pydantic model, an `Enum`, a regular class, or a plain dictionary, and why?

**Storage.** Would you use a `dict`, SQLite, or an external database, and why?

**Concurrency.** Where would you use `asyncio`, a `Semaphore`, timeouts, or retries?

**Error handling.** Which errors deserve custom exceptions?

**Testing.** What should be unit tested? What should be integration tested?

**Project structure.** How would you organize models, services, database access, external clients, exceptions, and tests?

Don't write the whole application yet, just the design.



