datetime in python

```python
from datetime import date
d = date(2026, 9, 29)
print(d.year)
# 2026
print(d.month)
# 9
print(d.day)
# 29

today = date.today()

from datetime import time
t = time(14, 30, 15)
print(t)
# 14:30:15
t.hour
# 14
t.minute
# 30
t.second
# 15

from datetime import datetime
dt = datetime(2026, 9, 29, 14, 30, 15)
print(dt)
# 2026-09-29 14:30:15

datetime.now()
datetime.utcnow()

from datetime import datetime, timezone
now = datetime.now(timezone.utc)

# time delta
delta = timedelta(days=3)
timedelta(
    days=1,
    hours=2,
    minutes=30,
    seconds=10
)

now = datetime.now()
tomorrow = now + timedelta(days=1)

delta.total_seconds()
delta.day()

# 字符串 → datetime：`strptime`
s = "2026-09-29 15:30:00"
dt = datetime.strptime(
    s,
    "%Y-%m-%d %H:%M:%S"
)
# datetime → 字符串：`strftime`
dt.strftime("%Y-%m-%d")

# isotime
s = "2026-09-29T15:30:00"
dt = datetime.fromisoformat(s)

now = datetime.now(timezone.utc)

# # `weekday()` / `isoweekday()`
dt.weekday()
```

|`%Y`|4位年份|
|`%y`|2位年份|
|`%m`|月|
|`%d`|日|
|`%H`|24小时制小时|
|`%I`|12小时制小时|
|`%M`|分钟|
|`%S`|秒|
|`%z`|UTC offset|
|`%Z`|timezone name|

string
string is immutable
should use to change char of certain index
```python
fixed_text = text[0] + "e" + text[2:]
```

- `strip()`: Removes whitespace (spaces, tabs, newlines) from the beginning and end of the string.
    
- `upper()`/`lower()`: Converts text to all upper or lower case.
    
- `replace(old, new)`: Swaps occurrences of a substring with another.
    
- `split(delimiter)`: Breaks a string into a list of substrings based on a separator character (like a comma or space).

list
```python
batch_1 = [1, 2, 3]
batch_2 = [4, 5]

# Pythonic approach: Use extend
batch_1.extend(batch_2)

# sort with reverse order
scores.sort(reverse=True)

# advanced looping
[expression for item in iterable]

numbers = [1, 2, 3, 4, 5]
# Create the list in one line
squares = [number ** 2 for number in numbers]

slow_responses = [t for t in response_times if t > 100]
slow_responses = [s if s > 0 else "Incomplete" for s in scores]

[ item  for group in collection  for item in group ]
matrix = [
    [1, 2, 3],
    [4, 5, 6],
    [7, 8, 9]
]

# Flattening the matrix into a single list
flat_numbers = [num for row in matrix for num in row]
 
colors = ["Red", "Blue"]
sizes = ["S", "M", "L"]

# Generate all color-size combinations
inventory = [(color, size) for color in colors for size in sizes]
```

1. `remove(value)`: Deletes the _first_ occurrence of a specific value. If the value isn't found, Python raises a `ValueError`. While `remove()` handles single items, later we will combine these techniques to learn how to remove duplicates from a list in Python efficiently.
    
2. `pop(index)`: Removes the item at a specific index and _returns_ it so we can use it. If no index is provided, it removes the last item.
    
3. `clear()`: Wipes the entire list, leaving it empty.

- `index(value)`: The `index()` method returns the position of the first occurrence of the `value` in the list and raises a `ValueError` if it is not present.
    
- `count(value)`: The `count()` method, on the other hand, returns the total number of times the `value` appears in the list. If it is not present, the `count()` method returns `0`.

Tuple
A **tuple** is an ordered sequence, similar to a list, but it is immutable. Once created, it acts as a sealed container that can not be altered.

```python
# Tuple Packing: Python groups these integers into one tuple object
coordinates = 10, 20
print(f"Coordinates: {coordinates} (Type: {type(coordinates)})")

# Tuple Unpacking: Python distributes the tuple items into variables x and y
x, y = coordinates
print(f"x is {x}, y is {y}")

# Practical usage: Swapping variables
a = 5
b = 10
a, b = b, a  # Packs (10, 5) then unpacks immediately
print(f"a: {a}, b: {b}")
```

Dict
Python provides the `in` operator for this purpose, which returns `True` if the key is present in the dictionary and `False` otherwise.

- `.keys()`: Returns keys (default behavior).
    
- `.values()`: Returns only the values.
    
- `.items()`: Returns pairs of `(key, value)`, which allows us to access both variables directly without using `[]`.
```python
for student, score in scores.items():
    print(f"{student} scored {score}")
    
# Dict comprehension
fruits = ["apple", "banana", "cherry", "date"]
# Create a dictionary mapping the fruit name to its length
fruit_lengths = {fruit: len(fruit) for fruit in fruits}
# Invert to create: Country Name -> Currency Code
country_lookup = {country: code for code, country in currency_map.items()}
```

set

```python
numbers = set([1, 2, 2, 3, 3, 4])
empty_set = set() 

# set comprehensions
{expression for item in iterable}
raw_tags = ["Python", "python", "Code", "CODE", "java", "JAVA"]
# Create a set of unique, lowercase tags
unique_tags = {tag.lower() for tag in raw_tags}
```

- **Union (**`|`**):** Combines elements from both sets (all unique items).
- **Intersection (**`&`**):** Keeps only items found in _both_ sets.
- **Difference (**`-`**):** Keeps items in the first set but _not_ the second.
- **Symmetric difference (**`^`**):** Keeps items found in either set, but _not_ both (opposites of intersection).

- `.remove(item)`: This method deletes the specified item from the set. However, if the item is not present, Python raises a `KeyError`, which can interrupt program execution.
    
- `.discard(item)`: This method also attempts to delete the specified item, but if the item is not found, it does nothing. Because it does not raise an error, `.discard()` is generally the safer choice when you are unsure whether the item exists.
dedup
```python
raw_data = ["apple", "banana", "apple", "orange", "banana"]

# Convert to set to remove duplicates
unique_set = set(raw_data)

# Convert back to list (order is not guaranteed to be preserved)
clean_data = list(unique_set)
```

Functions
```python
counter = 0  # Global variable

def increment():
    counter  # Declare intent to modify the global variable
    counter += 1
    print(f"Counter is now: {counter}")

increment()

# Using nolocal for inner function to avoid recreating new 
def make_counter():
    count = 0  # This variable will be modified
    
    def increment():
        nonlocal count  # Explicit permission to modify 'count'
        count += 1
        return count
    
    return increment


# 1. Positional call (Order matters strictly)
describe_pet("Luna", "cat", 4)

# 2. Keyword call (Order does not matter)
describe_pet(age=4, pet_name="Luna", animal_type="cat")

# Lambda Function

filter (function, iterable)
multiples_of_six_iter = filter(lambda x: x % 6 == 0, numbers)
```

map filter reduce
```python
map(function, iterable)

price_iterator = map(parse_price, raw_prices)
# Consume the iterator to see the list
clean_prices = list(price_iterator)

squared = list(map(lambda x: x**2, numbers))
# Keep only even numbers
evens = list(filter(lambda x: x % 2 == 0, numbers))


functools.reduce(function, iterable, initializer)
def add_and_trace(accumulated_value, current_item):
    print(f"Accumulator: {accumulated_value}, Current: {current_item}")
    return accumulated_value + current_item

```