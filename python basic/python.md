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

by_id = {r['id']: r for r in rows}
regions = {r['region'] for r in rows}
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

Mock in unittest
```python
from unittest.mock import Mock

api_client = Mock(return_value=200)

response_code = api_client("http://example.com")

```

File operation, using 'with' auto close the file after the indent finished
```python
filename = "message.txt"
try:
	with open(filename, 'r') as file_handle:
		print(f"File opened: {file_handle}")
		content = file_handle.read()
		
excpect FileNotFoundError:
	print(f"Error: The file '{file_name}' not found.")
	
# Read with lines to handle large files
with open('file_name', 'r') as log_file:
	for line in log_file:
		if "ERROR" in line:
			print("...")

with open('data.txt', 'r', encoding='utf-8') as f:
	for line in f:
		clean_line = line.strip()
```
Reading csv
```python
with open('inventory.csv', mode = 'r', encoding='utf-8') as csv_file:
	reader = csv.DictReader(csv_file)
	for row in reader:
		name = row["product_name"]
		...
```
Write file, if file not exist, will create a new file
```python
file_name = "daily_report.txt"

with open(filename, "w", encoding="utf-8") as file:
	file.write("Daily status Report\n")
	file.write("Date: 2023-10-27\n")
# "a" apend mode, cursor at the end of the file	
with open("log_file", 'a', "utf-8") as f:
	f.write("")

```
For binary:
- `'rb'` for reading binary data
- `'wb'` for writing binary data

path Handling
```python
from pathlib import Path

base_dir = Path.cwd()
log_dir = base_dir/"app_data"/"logs"
error_log = log_dir/"error.txt"

file_path = Path("data/reports/...")
```
```python
from pathlib import Path
p = Path("settings.ini")
p.touch()

if p.exists():
	if p.is_file():
	
	elif p.is_dir:

p.unlink()

# iterdir() return every file and subdirectories in the location
# glob() return wildecard partterns similar to those used in the cmd lie

data_dir = Path("dataset")
for item in data_dir.iterdir():
	print (item.name)

for image in data_dir.glob("*.png"):
    print(f"Found image: {image.name}")
```
Serialization
dict
- `dict` →→ JSON Object `{"key": value}`
- `list` / `tuple` →→ JSON Array `[value, ...]`
- `str` →→ String
- `int` / `float` →→ Number
- `True` / `False` →→ `true` / `false`
- `None` →→ `null`

- `json.dumps(obj)`**:** Returns a JSON string from a Python object.
- `json.loads(json_str)`**:** Returns a Python object from a JSON string.
- `json.dump(obj, file)`**:** Writes a Python object directly to a file.
- `json.load(file)`**:** Reads a JSON object directly from a file.

```python
import json

user_config = {
	"user_name": "developer",
	"theme": "dark"
}
with open("user.txt", 'w', encoding='utf-8') as file:
	json.dump(use_config, file, indent = 4)

with open("user.txt", 'r', encoding='utf-8') as file:
	loaded_config = json.load(file)
```

Iterator and iterable
```python
cursor = iter(container_name)
first_item = next(cursor)
second_item = next(cursor)
```

enumerate
`enumerate(iterable, start=0)` 返回索引和值组成的元组
```python
fruits = ["apple", "banana", "orange"]

for index, fruit in enumerate(fruits):
    print(index, fruit)
```

Python 的 `zip()` 用来把多个可迭代对象中相同位置的元素配对，返回一个迭代器，每次产出一个 tuple。
```python
names = ["Alice", "Bob", "Charlie"]
scores = [90, 85, 95]

result = zip(names, scores)

print(list(result))
# [('Alice', 90), ('Bob', 85), ('Charlie', 95)]
```
generator
Avoid storing all values in memory. 适合处理**无限数据、超大文件、streaming 数据**：它不需要一次性把所有结果放进内存。
We create a generator by using the `yield` keyword instead of `return`. When Python executes a function that contains `yield`, the following sequence occurs:
- **Pause and return:** The function yields a value to the caller and then pauses execution.
- **Save state:** Python preserves the function’s entire state, including local variables and the current execution position.
- **Resume:** When the next value is requested (using `next()` or a `for` loop), execution resumes exactly where it left off, immediately after the previous `yield`.
```python
# 2 methods for generator
# Generator Expression (Lazy)
g = (x * 2 for x in range(10))
# Could use next and for loop to iterate over it 
for num in g:
    print(num)
    
# Generator function
def error_log_reader(file_path):
	with open(file_path, "r") as file:
		for line in file:
			if "ERROR" in line:
				yield line.strip()
log_gen = error_log_reader("system.log")

for error in log_gen:
    print(error)
```

Chaining generators
```python
raw_data = "  10,  20,  bad_data, 30  "

# Step 1: Split the string (creates a list, unavoidable here for split)
items = raw_data.split(',')

# Step 2: Generator to strip whitespace
clean_items = (item.strip() for item in items)

# Step 3: Generator to filter only digits
valid_items = (item for item in clean_items if item.isdigit())

# Step 4: Generator to convert to integer
integers = (int(item) for item in valid_items)

# Consume the pipeline
print(f"Sum of valid integers: {sum(integers)}")
```

sum([r['amount'] for r in rows])  # []：列表推导式
sum(r['amount'] for r in rows)  # ()：生成器表达式(save memory, )

Iterator class
To make an object compatible with a `for` loop, it must follow the iterator protocol. This protocol requires two special dunder methods:
1. **`__iter__()`****:** This is the entry point. When we start a loop, Python calls this method to get the iterator object.
    
2. **`__next__()`****:** This is the engine. Python calls this repeatedly to fetch the next item in the sequence
an iterator must raise the `StopIteration` exception for ending
```python
from datetime import timedelta, date
class DateRange:
	def __init__(self, start:date, end:date):
		if start > end:
			raise ValueErro("")
		self.current_date = start
		self.end_date = end
		
	def __iter__:
		return self
	
	def __next__ (self):
		if self.current_date >= self.end_date:
			raise StopIteratoin
		today = self.current_date
		self.current_date += timedelta(days=1)
		return today
```
We now have two ways to **create custom iteration logic**. Here is how to choose between them:

1. **Use generators (Functions with yield)** when you want concise code for a simple stream of data. They handle state and `StopIteration` automatically. This is the default choice for 90% of cases.
2. **Use class-based iterators** when you need:
    **Complex state:** Multiple attributes tracking progress.
    **Public attributes:** You need to access the current state or configuration (e.g., `.current_date`) from outside the loop.
    **Inheritance:** You want to create a base iterator and subclass it for specific behaviors.

Requests
Use response .json to transfer response body into python obj (dict or list)
```python
response.json()

query_params = {"postId": 1}

# Headers: Metadata about the request
custom_headers = {
    "User-Agent": "MyPythonApp/1.0",
    "Accept": "application/json"
}

response = requests.get(url, params=query_params, headers=custom_headers)
# Send a POST request. The 'json' parameter handles formatting.
response = requests.post(url, json=new_post)

try:
    # We set a timeout of 2 seconds. Since the server takes 5, this will fail.
    response = requests.get(url, timeout=2)
    print("Success:", response.status_code)
except requests.exceptions.Timeout:
    print("The request timed out. The server took too long to respond.")
```
`raise_for_status()` is essential because `requests` considers error pages (like 404) "successful" downloads. This method converts those error codes into actual exceptions so our `try/except` block can handle them.
```python
try:
    response = requests.get(url, timeout=3)
     # raise_for_status() triggers an HTTPError for 4xx/5xx responses
    response.raise_for_status()
    print(f"Connected to {url}")
        
except requests.exceptions.ConnectionError:
    print(f"Failed to connect to {url}. Is the internet down?")
except requests.exceptions.HTTPError as e:
    print(f"Server returned an error: {e}")
except requests.exceptions.RequestException as e:
    # Catch-all for any other request-related error
    print(f"An ambiguous error occurred: {e}")
```

Package
package is a directory containing modules and a special file named `__init__.py`.
Calls the function using the full namespace path `package.module.function`.The `__init__.py` file is the package’s entry point. It is executed automatically the moment the package (or any module within it) is imported.

In init file, if import related functions, others can directly call function from package namespace.
```python
# File: pixel_art/__init__.py

# We import specific items from submodules into the package namespace
from pixel_art.shapes import draw_circle
from pixel_art.colors import get_red  # Assuming colors.py exists

VERSION = "1.0.0"

# File: pixel_art/__init__.py
from pixel_art.shapes import draw_circle, draw_square

# Only these names will be imported during 'from pixel_art import *'
__all__ = ['draw_circle']

```

we use **relative imports** with dot notation:
- `.` refers to the current package.
- `..` refers to the parent package.

- **`os.environ['KEY']`****:** Raises a `KeyError` if the variable is not defined. This approach is appropriate when the configuration is mandatory, and the application should fail immediately if it is missing.
    
- **`os.getenv('KEY', default)`****:** Returns `None`, or a specified default value, if the variable does not exist. This method is preferable for optional configuration settings.
venv
```shell
python -m venv .venv

source .venv/bin/activate

pip freeze > requirements.txt
```

`load_dotenv()` 是 Python `python-dotenv` 库里的函数，作用 把 `.env` 文件里的配置加载到当前 Python 进程的环境变量 `os.environ`  （Standard `load_dotenv()` behavior respects existing environment variables）
```python
from dotenv import load_dotenv
import os

load_dotenv()

print(os.getenv("DATABASE_URL"))
print(os.getenv("API_KEY"))
```

Concurrency(threading in python) and Parallelism(processes in python)
In an I/O-bound scenario (like a web server), concurrency drastically improves throughput.
In CPU-bound workloads, concurrency based on time slicing can reduce performance

Standard Python (CPython) includes a constraint called the Global Interpreter Lock (GIL). The GIL is a mutual exclusion mechanism that allows only one thread to execute Python bytecode at a time.
a multithreaded Python program will typically utilize only one core at a time for executing Python instructions.
```python
import threading
import time
def simulate_download(file_name):
	sleep(2)
thread1 = threading.Thread(target=simulate_download, args=("Photo.jpg"))
thread2 = threading.Thread(target=simulate_download, args=("adsg.pdf"))
thread1.start()
thread2.start()
thread1.join()
thread2.join()
```

Synchronization with locks
Use mutex and with to acquire lock at the start of the block and release at the end of the block

```python
import threading
lock = threading.Lock()

counter = 0
def safe_counter():
	global counter
	for _ in range(100):
		with lock:
			counter += 1
threads = []
for i in range(3):
	t = threading.Thread(target = safe_counter)
	threads.append(t)
	t.start()
for t in threads:
	t.join()

```

Daemon thread
A daemon thread runs in the background and does not prevent the program from shutting down. Once all non-daemon threads (typically including the main thread) have completed, the interpreter terminates any remaining daemon threads automatically.
```python
import threading
import time
def background_logger():
	while True:
		print("Logger: Writing")
		sleep(1)
log_thread =threading.Thread(target= background_logger, daemon = True)
log_thread.start()


```

Unlike threads, which operate within a single process and share both memory and the Global Interpreter Lock (GIL), each process has an independent memory space and its own GIL.
When using `multiprocessing`, all executable code that spawns processes must be protected within an `if __name__ == "__main__":`
```python
import multiprocessing
import time

def heavy_compueation(name):
	for i in range(1000):
		result += i

if __name__ == "__main__":
	p1 = multiprocessing.Process(target=heavy_computation, args=("A",))
    p2 = multiprocessing.Process(target=heavy_computation, args=("B",))
    
    p1.start()
    p2.start()
    
    p1.join()
    p2.join()
```

Process Pool with map function
```python
import multiprocessing
def square_number(n):
	return n*n

if __name__ == '__main__':
	numbers = [1,2,3,4,5,6]
	with multiprocessing.Pool() as pool:
		results = pool.map(square_number, numbers)

```

Interprocess Communication (IPC) 
use multiprocessing.Queue() to transmit data among the processes
```python
import multiprocessing

def producer(queue):
    """Generates data and puts it into the queue."""
    for i in range(3):
        item = f"Data-{i}"
        queue.put(item)
        print(f"Produced: {item}")
    # Signal that production is done
    queue.put(None)

def consumer(queue):
    """Retrieves data from the queue and processes it."""
    while True:
        item = queue.get()
        if item is None:  # Check for the stop signal
            break
        print(f"Consumed: {item}")

if __name__ == "__main__":
    # Create a thread-safe, process-safe queue
    q = multiprocessing.Queue()

    # Create processes sharing the queue
    p1 = multiprocessing.Process(target=producer, args=(q,))
    p2 = multiprocessing.Process(target=consumer, args=(q,))

    p1.start()
    p2.start()

    p1.join()
    p2.join()
```

Using queue to safely pass messages between threads, it internally implements locking
```python
import threading
import queue
import time

def producer(q, n):
	for i in range(n):
		item = str(n)
		q.put(n)
	q.put(None)
	
def comsumer(q):
	while True:
		item = q.get()
		if item is None:
			break
queue1 = queue.Queue()
prod = threading.Thread(target = producer, args=(q,5))
coms = threading.Thread(target = consumer, args=(q))
prod.start()
coms.start()
prod.join()
coms.join()
```

signal with events
- `set()`**:** Sets the flag to `True`.
- `clear()`**:** Sets the flag to `False`.
- `wait()`**:** Blocks execution until the flag becomes `True`.
```python
import threading
import time

# Create an event object
stop_event = threading.Event()
def backend():
	while not stop_event.is_set()
		print("working")
		if stop_event.wait(timeout = 1.0):
			print("stop working")
			break
t = threading.Thread(target=backend)
t.start()
stop_event.set()
t.join()
```

Async IO
Also used for I/O bound tasks

A coroutine is defined using the `async def` syntax.
```python
import asyncio
async def stop():
	print("start")
	await asyncio.sleep(1)
	print("stop")
if __name__ == "__main__":
	asyncio.run(stop())
```
 
asyncio.run() 从同步程序开启一个异步环境

async function run only create a coroutine object, we have to use await or asyncio to trigger it

**`await` = 启动/执行这个 coroutine，并等待它完成。**  
**`asyncio.create_task()` = 把 coroutine 安排到 event loop 中运行，但当前代码不立即等待它完成。**

Use asyncio.gather(),the event loop can manage them concurrently rather than one at a time.
With `asyncio.gather()` could guarantee, no matter which task complete first
```python
import asyncio
import time
async def simulate_fetch(id, duration):
	await asyncio.sleep(duration)
	return
async def main():
	results = await asyncio.gather(
        simulated_fetch(1, 1.0),
        simulated_fetch(2, 1.0),
        simulated_fetch(3, 1.0)
    )
if __name__ == "__main__":
	asyncio.run(main())
```

Decorator function
```python
def my_decorator(func):
	def wrapper():
		print("decorator")
		func()
	return wrapper

@my_decorator
def my_func():
	print("func")
	
# Decorator with random args
# args for positional argument, kwargs for keyword arguments
def log_decorator(func):
	def wrapper(*args, *kwargs):
		print("calling function")
		func(*args, *kwargs)
		print("function_end")
	return wrapper
@log_decorator
def add(a, b):
	return a + b
	
# Use functool.wrap, we can maintain the metadata of the decorated function
def robust_decorator(func):
    @functools.wraps(func)  # Fixes the metadata
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return wrapper
```

**parameterized decorator** is actually a function that accepts arguments and _returns a decorator_. This outer function is often called a **decorator factory** because its sole job is to manufacture the actual decorator based on the settings we provide.
```python
import functools
def repeat(times): # the factory
	def decorator(func): # the decorator
		@functools.wrap(func)
		def wrapper(*args, **kwargs):
			for _ in range(times):
				result = func(*args, **kwargs)
			return result
		return wrapper
	return decorator

@repeat(times = 2)
def greet(name):
	print(name)
```
Lifecycle of the with block

1. **`__enter__(self)`**: Python calls this immediately when the `with` statement begins. If the `with` statement includes an `as target` clause, the return value of `__enter__` is assigned to `target`.
    
2. **`__exit__(self, exc_type, exc_val, exc_tb)`**: Python calls this when the execution leaves the `with` block. This happens whether the block finishes normally or stops due to an exception. exc_type: error type, exc_val: error val, exc_tb: error traceback

If we implement these two methods in a class, instances of that class can be used in `with` statements.
```python
import time
class Timer:
	def __init__(self, label):
		self.label = lable
		self.start = None
	def __enter__(self):
		 self.start = time.time()
        return self
	def __exit__(self, exc_type, exc_val, exc_tb):
		end =  time.time()
		duration = end -self.start
		return False
with Timer("my_label") as t:
	time.sleep(1)

# Use the class to supress certain type of error
class ErrorSuppressor:
    def __init__(self, error_type):
        self.error_type = error_type

    def __enter__(self):
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        # Check if an exception happened and if it matches the type we want to suppress
        if exc_type is not None and issubclass(exc_type, self.error_type):
            print(f"Suppressed error: {exc_val}")
            return True  # Swallow the exception
        
        # If it's a different error, or no error, returning None (default) propagates it.
        return False

print("Start")
```
Some build in functions
object.attribute
getattr(object, name, default)

```python
import argparse
def main():
	parser = argparse.ArgumentParser(description = "a simple file reader")
	# Positional argument
	parser.add_argument("filename", type = string, help = "Name of file to read")
	# Optinal argument
	parser.add_argument("-n", "--lines", type = int, help = "Numbers of lines")
	# restrict input to specific choices
	
	parser.add_argument(
		"-op",
		choice = ["r", "w", "wr"],
		default = "add"
	)
	args = parser.parse_args()
	
	try:
		with open(args.filename, 'r') as f:
			lines = f.readlines()
			limit = args.lines if args.lines else len(lines)
			
			for i in range(limit):
				print(lines[i])

```

```shell
# 2. Create the source code directories
# The -p flag creates 'src' AND 'simple_tool' at the same time
mkdir -p src/simple_tool

# 4. View the structure
tree .
```