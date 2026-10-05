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


Error handling
```python
# try else
def process_number(text):
    try:
        number = int(text)
    except ValueError:
        print("Conversion failed.")
    else:
        # This only runs if int(text) succeeded
        print(f"Conversion successful: {number}")
        print(f"Square: {number ** 2}")

# try finally
def database_operation(fail_simul):
    print("Opening database connection...")
    try:
        if fail_simul:
            result = 1 / 0  # Simulate a crash
        else:
            print("Query executed.")
    except ZeroDivisionError:
        print("Error: Operation failed.")
    finally:
        print("Closing database connection.")

# Raising exceptions manually
def update_profile(age):
    if age < 0:
        raise ValueError("Age cannot be negative.")
    
    print(f"Profile updated. Age: {age}")

# We inherit from Exception to create a distinct error type
class DatabaseConnectionError(Exception):
    pass

def connect_to_database(url):
    # Simulating a connection failure
    if url == "":
        raise DatabaseConnectionError("URL cannot be empty")
    print(f"Connected to {url}")

try:
    connect_to_database("")
except DatabaseConnectionError as e:
    # We catch our specific error, leaving other errors (like TypeError) alone
    print(f"Connection failed: {e}")

# rewrite parent function
class InsufficientFundsError(Exception):
    def __init__(self, required, available):
        self.required = required
        self.available = available
        # We format a human-readable message for the parent class
        message = f"Attempted to withdraw {required}, but only {available} is available."
        super().__init__(message)
```

Assertion
```python
assert condition, optional_message

def register_user(username, age):
    # Validation: Check input types and values explicitly
    if not isinstance(username, str) or len(username) == 0:
        raise ValueError("Username must be a non-empty string")
    
    if not isinstance(age, int) or age < 13:
        raise ValueError("User must be at least 13 years old")
        
    return f"User {username} registered."

# This raises a ValueError, NOT an AssertionError
# We can catch this with try/except blocks
try:
    register_user("Alice", 10)
except ValueError as e:
    print(f"Registration failed: {e}")
```

 Severity levels
 
1. **`DEBUG`****:** Detailed information, useful only when diagnosing problems (e.g., `"Variable x is 5"`).
    
2. **`INFO:`** Confirmation that things are working as expected (e.g., `"User logged in"`).
    
3. **`WARNING`****:** An indication that something unexpected happened, but the software is still working (e.g., `"Disk space low"`).
    
4. **`ERROR`****:** Due to a more serious problem, the software has not been able to perform some function.
    
5. **`CRITICAL`****:** A serious error, indicating that the program itself may be unable to continue running.

By default, Python sets the logging level to `WARNING`. This acts as a filter: `DEBUG` and `INFO` messages are discarded silently unless we lower the threshold.
```python
import logging

# We simulate a simple data processing step
def process_data(data):
    # This print goes to standard output (User sees this)
    print(f"Processing: {data}")

    # These are logging calls (Developer sees these)
    # 1. Detailed diagnostic info (Ignored by default)
    logging.debug(f"DEBUG: Data type is {type(data)}")
    
    # 2. General runtime info (Ignored by default)
    logging.info("INFO: Transformation started...")
    
    if data is None:
        # 3. Serious issue (Shown by default)
        logging.error("ERROR: Data is empty! processing failed.")

# Run the function with valid data
process_data("User_Input")

# Run the function with invalid data
process_data(None)

```

```python
import logging
# Configure the system to capture EVERYTHING (DEBUG and up) 
# and write it to a file instead of the screen.
logging.basicConfig(
    filename='app_debug.log', 
    level=logging.DEBUG, 
    filemode='w'
)
```

TEST
1 unittest class:
extends unittest.TestCase, functions in the class starting with test_ will automatically run
```python
import unittest

def add_positive(a, b):
    if a < 0 or b < 0:
        return 0
    return a + b

class TestMathOperations(unittest.TestCase):
    def test_addition(self):
        result = add_positive(5, 10)
        self.assertEqual(result, 15)

    def test_negative_input(self):
        result = add_positive(-5, 10)
        self.assertEqual(result, 0)

if __name__ == '__main__':
    unittest.main()
```
- **`assertEqual(a, b)`****:** Verifies that `a == b`.
- **`assertNotEqual(a, b)`****:** Verifies that `a != b`.
- **`assertTrue(x)`****:** Verifies that `bool(x)` is `True`.
- **`assertIn(item, container)`****:** Verifies that `item` exists within `container`

assertRaises
```python
def parse_percentage(value):
    if not (0 <= value <= 100):
        raise ValueError("Percentage must be between 0 and 100")
    return value / 100

class TestValidation(unittest.TestCase):
    def test_value_error(self):
        with self.assertRaises(ValueError):
            parse_percentage(150)
```

When we run " `python -m unittest`" command:

1. Python searches the current directory for files matching `test*.py`.
    
2. It loads those files and finds classes inheriting from `unittest.TestCase`.
    
3. It runs every method starting with `test_`.

2 pytest

There are no required base classes or special assertion methods. The only rule is naming: Test functions must start with `test_`.

A fixture is a helper function marked with `@pytest.fixture`. It creates and returns the data we need.

```python
import pytest
import sys

# 1. Define the source of data (The Fixture)
@pytest.fixture
def sample_user():
    # This runs BEFORE the test to prepare data
    return {"id": 1, "username": "test_user", "roles": ["admin"]}
# 2. Inject the data into the test (The Consumer)
# We request 'sample_user' by name as an argument
def test_user_role(sample_user):
    # pytest has already run the fixture and passed the result here
    assert "admin" in sample_user["roles"]

def test_user_id(sample_user):
    assert sample_user["id"] == 1

if __name__ == "__main__":
    sys.exit(pytest.main(["-v", __file__]))
```

`pytest` handles this elegantly using the `yield` keyword within a fixture.
1. Code before `yield` is the Setup (runs before the test).
2. The `yield` statement pauses the fixture and hands the value to the test.
3. Code after `yield` is the Teardown (runs after the test finishes)

```python
import pytest
import sys

class MockDatabase:
    def __init__(self):
        self.connected = False

    def connect(self):
        self.connected = True
        print("\n[DB Connected]")
    
    def disconnect(self):
        self.connected = False
        print("\n[DB Disconnected]")

@pytest.fixture
def db_connection():
    # --- SETUP PHASE ---
    db = MockDatabase()
    db.connect()
    
    # --- TESTING PHASE ---
    # We provide the connected db object to the test
    yield db
    
    # --- TEARDOWN PHASE ---
    # This runs guaranteed after the test finishes
    db.disconnect()

def test_query(db_connection):
    # The test receives the object yielded above
    print("Executing query logic...")
    assert db_connection.connected is True

if __name__ == "__main__":
    # We use -s to allow 'print' statements to show up in the output
    sys.exit(pytest.main(["-v", "-s", __file__]))
```
parametrize
```python
def calculate_discount(price, expected_price):
    return price * 0.9 == expected_price

# We define the variable names ("input_price, expected")
# And a list of values to test against those names
@pytest.mark.parametrize("input_price, expected", [
    (100, 90.0),    # Case 1: Standard
    (50, 45.0),     # Case 2: Lower value
    (0, 0.0),       # Case 3: Edge case
])
def test_discount_calculation(input_price, expected):
    # This function runs 3 separate times!
    discounted = input_price * 0.9
    assert discounted == expected

if __name__ == "__main__":
    sys.exit(pytest.main(["-v", __file__]))
```

scope = function/ module/ session(for expensive resources like spark)
```python
import pytest
from pyspark.sql import SparkSession


@pytest.fixture(scope="session")
def spark():
    spark = (
        SparkSession.builder
        .master("local[2]")
        .appName("test")
        .getOrCreate()
    )

    yield spark

    spark.stop()
```
