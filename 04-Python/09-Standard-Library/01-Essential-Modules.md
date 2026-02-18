# Essential Standard Library Modules

## Regular Expressions (re)

### Basic Pattern Matching

```python
import re

# Basic search
text = "Hello, my email is alice@example.com"
match = re.search(r'\w+@\w+\.\w+', text)
if match:
    print(match.group())  # alice@example.com

# Match vs Search
re.match(r'\d+', '123abc')    # Match - only at beginning
re.search(r'\d+', 'abc123')   # Search - anywhere in string

# Find all matches
text = "Call 123-456-7890 or 987-654-3210"
phones = re.findall(r'\d{3}-\d{3}-\d{4}', text)
# ['123-456-7890', '987-654-3210']

# Find with groups
text = "Name: Alice, Age: 30; Name: Bob, Age: 25"
matches = re.findall(r'Name: (\w+), Age: (\d+)', text)
# [('Alice', '30'), ('Bob', '25')]

# Finditer - returns match objects
for match in re.finditer(r'\d+', 'a1b2c3'):
    print(match.group(), match.start(), match.end())
# 1 1 2
# 2 3 4
# 3 5 6
```

### Regex Patterns Reference

```python
"""
Character Classes:
.       Any character except newline
\d      Digit [0-9]
\D      Non-digit
\w      Word character [a-zA-Z0-9_]
\W      Non-word character
\s      Whitespace [ \t\n\r\f\v]
\S      Non-whitespace

Anchors:
^       Start of string (or line with MULTILINE)
$       End of string (or line with MULTILINE)
\b      Word boundary
\B      Non-word boundary

Quantifiers:
*       0 or more (greedy)
+       1 or more (greedy)
?       0 or 1 (greedy)
{n}     Exactly n
{n,}    n or more
{n,m}   Between n and m
*?      0 or more (non-greedy)
+?      1 or more (non-greedy)

Groups:
(...)   Capturing group
(?:...) Non-capturing group
(?P<name>...) Named group
\1, \2  Backreference

Lookahead/Lookbehind:
(?=...) Positive lookahead
(?!...) Negative lookahead
(?<=...) Positive lookbehind
(?<!...) Negative lookbehind

Flags:
re.IGNORECASE or re.I   Case-insensitive
re.MULTILINE or re.M    ^ and $ match line boundaries
re.DOTALL or re.S       . matches newline
re.VERBOSE or re.X      Allow comments in pattern
"""

# Examples
# Email validation (simplified)
email_pattern = r'^[\w.-]+@[\w.-]+\.\w+$'

# Phone number (various formats)
phone_pattern = r'(\d{3}[-.\s]?\d{3}[-.\s]?\d{4})'

# URL extraction
url_pattern = r'https?://[\w.-]+(?:/[\w.-]*)*'

# Password validation (8+ chars, upper, lower, digit)
password_pattern = r'^(?=.*[a-z])(?=.*[A-Z])(?=.*\d).{8,}$'

# Extract domain from email
match = re.search(r'@([\w.-]+)', 'user@example.com')
domain = match.group(1)  # example.com
```

### Substitution and Splitting

```python
# Basic substitution
text = "Hello World"
re.sub(r'World', 'Python', text)  # "Hello Python"

# Replace all digits
re.sub(r'\d', 'X', 'a1b2c3')  # "aXbXcX"

# Substitution with function
def upper_repl(match):
    return match.group().upper()

re.sub(r'\b\w', upper_repl, 'hello world')  # "Hello World"

# Substitution with backreference
re.sub(r'(\w+) (\w+)', r'\2 \1', 'Hello World')  # "World Hello"

# Named group substitution
re.sub(r'(?P<first>\w+) (?P<last>\w+)',
       r'\g<last>, \g<first>', 'John Doe')  # "Doe, John"

# Limit replacements
re.sub(r'\d', 'X', 'a1b2c3', count=2)  # "aXbXc3"

# Split
re.split(r'\s+', 'Hello   World')  # ['Hello', 'World']
re.split(r'[,;]', 'a,b;c,d')  # ['a', 'b', 'c', 'd']

# Split with capture group (keeps separator)
re.split(r'(\d+)', 'a1b2c')  # ['a', '1', 'b', '2', 'c']

# Max splits
re.split(r'\s+', 'a b c d', maxsplit=2)  # ['a', 'b', 'c d']
```

### Compiled Patterns

```python
# Compile for repeated use (faster)
pattern = re.compile(r'\d{3}-\d{3}-\d{4}')

pattern.search('Call 123-456-7890')
pattern.findall('123-456-7890 and 987-654-3210')
pattern.sub('XXX-XXX-XXXX', '123-456-7890')

# With flags
pattern = re.compile(r'hello', re.IGNORECASE)
pattern.search('HELLO')  # Matches

# Verbose pattern (readable)
phone_pattern = re.compile(r'''
    (\d{3})     # Area code
    [-.\s]?     # Optional separator
    (\d{3})     # First 3 digits
    [-.\s]?     # Optional separator
    (\d{4})     # Last 4 digits
''', re.VERBOSE)
```

---

## os and sys Modules

### os Module - Operating System Interface

```python
import os

# Environment variables
os.environ['HOME']                  # Get env var
os.environ.get('API_KEY', 'default')  # With default
os.getenv('API_KEY', 'default')     # Same thing

os.environ['MY_VAR'] = 'value'      # Set env var
del os.environ['MY_VAR']            # Delete env var

# Current directory
os.getcwd()                         # Current working directory
os.chdir('/path/to/dir')           # Change directory

# Directory operations
os.listdir('.')                     # List directory contents
os.mkdir('new_dir')                # Create directory
os.makedirs('a/b/c', exist_ok=True)  # Create nested dirs
os.rmdir('empty_dir')              # Remove empty directory
os.removedirs('a/b/c')             # Remove nested empty dirs

# File operations
os.remove('file.txt')              # Delete file
os.rename('old.txt', 'new.txt')    # Rename file
os.stat('file.txt')                # File info (size, times, etc.)
os.path.exists('file.txt')         # Check if exists
os.path.isfile('path')             # Is it a file?
os.path.isdir('path')              # Is it a directory?

# Path operations
os.path.join('dir', 'subdir', 'file.txt')  # 'dir/subdir/file.txt'
os.path.dirname('/path/to/file.txt')       # '/path/to'
os.path.basename('/path/to/file.txt')      # 'file.txt'
os.path.splitext('file.txt')               # ('file', '.txt')
os.path.abspath('relative/path')           # Full absolute path
os.path.expanduser('~/file.txt')           # Expand ~

# Walk directory tree
for root, dirs, files in os.walk('/path'):
    for file in files:
        filepath = os.path.join(root, file)
        print(filepath)

# Process info
os.getpid()                        # Current process ID
os.getppid()                       # Parent process ID
os.getlogin()                      # Current username
os.cpu_count()                     # Number of CPUs

# Execute commands (prefer subprocess)
os.system('ls -la')                # Run shell command
```

### sys Module - System-Specific

```python
import sys

# Python version
sys.version                        # '3.10.0 (default, ...)'
sys.version_info                   # (3, 10, 0, 'final', 0)
sys.version_info.major            # 3

# Command line arguments
sys.argv                           # ['script.py', 'arg1', 'arg2']
sys.argv[0]                       # Script name
sys.argv[1:]                      # Arguments

# Module search path
sys.path                           # List of paths Python searches
sys.path.append('/my/module/path') # Add to path

# Loaded modules
sys.modules                        # Dict of loaded modules
'os' in sys.modules               # True if os is imported

# Standard streams
sys.stdin                          # Standard input
sys.stdout                         # Standard output
sys.stderr                         # Standard error

# Redirect output
old_stdout = sys.stdout
sys.stdout = open('output.txt', 'w')
print("This goes to file")
sys.stdout = old_stdout

# Exit program
sys.exit(0)                        # Exit with code 0 (success)
sys.exit(1)                        # Exit with code 1 (error)
sys.exit("Error message")          # Exit with message to stderr

# Memory and recursion
sys.getsizeof(obj)                 # Size of object in bytes
sys.getrecursionlimit()           # Current recursion limit
sys.setrecursionlimit(2000)       # Set recursion limit

# Platform info
sys.platform                       # 'linux', 'darwin', 'win32'
sys.executable                     # Path to Python interpreter
sys.prefix                         # Python installation prefix
```

---

## pathlib - Modern Path Handling

```python
from pathlib import Path

# Creating paths
p = Path('/usr/local/bin')
p = Path.home()                    # User's home directory
p = Path.cwd()                     # Current working directory
p = Path('relative/path')

# Path components
p = Path('/home/user/file.txt')
p.name                             # 'file.txt'
p.stem                             # 'file'
p.suffix                           # '.txt'
p.suffixes                         # ['.txt'] (handles .tar.gz)
p.parent                           # Path('/home/user')
p.parents                          # [Path('/home/user'), Path('/home'), Path('/')]
p.parts                            # ('/', 'home', 'user', 'file.txt')
p.anchor                           # '/'

# Path operations
p = Path('/home/user')
p / 'subdir' / 'file.txt'          # Path('/home/user/subdir/file.txt')
p.joinpath('subdir', 'file.txt')   # Same thing

# Checking path
p.exists()                         # Does path exist?
p.is_file()                        # Is it a file?
p.is_dir()                         # Is it a directory?
p.is_absolute()                    # Is path absolute?
p.is_symlink()                     # Is it a symbolic link?

# File operations
p = Path('file.txt')
p.read_text()                      # Read entire file as string
p.read_bytes()                     # Read as bytes
p.write_text('content')            # Write string to file
p.write_bytes(b'content')          # Write bytes to file

# Directory operations
p = Path('mydir')
p.mkdir(parents=True, exist_ok=True)  # Create directory
p.rmdir()                          # Remove empty directory

# Listing files
p = Path('.')
list(p.iterdir())                  # All items in directory
list(p.glob('*.txt'))              # All .txt files
list(p.glob('**/*.py'))            # All .py files recursively
list(p.rglob('*.py'))              # Same as above

# File info
p.stat()                           # os.stat() result
p.stat().st_size                   # File size
p.stat().st_mtime                  # Modification time

# Path manipulation
p.resolve()                        # Absolute path (resolving symlinks)
p.absolute()                       # Absolute path
p.relative_to('/home')             # Relative path
p.with_name('other.txt')           # Change filename
p.with_suffix('.md')               # Change extension
p.with_stem('newname')             # Change stem (Python 3.9+)

# pathlib vs os.path
# Prefer pathlib for new code - more readable and Pythonic
# os.path still useful for string paths in some libraries
```

---

## datetime Module

### Basic Date and Time

```python
from datetime import datetime, date, time, timedelta

# Current date/time
now = datetime.now()               # Local time
today = date.today()               # Current date
utc_now = datetime.utcnow()        # UTC time (naive - see below)

# Creating datetime objects
dt = datetime(2024, 1, 15, 10, 30, 45)  # year, month, day, hour, min, sec
d = date(2024, 1, 15)
t = time(10, 30, 45)

# From string (parsing)
datetime.strptime('2024-01-15', '%Y-%m-%d')
datetime.strptime('Jan 15, 2024 10:30', '%b %d, %Y %H:%M')

# To string (formatting)
dt.strftime('%Y-%m-%d %H:%M:%S')   # '2024-01-15 10:30:45'
dt.strftime('%B %d, %Y')           # 'January 15, 2024'

# ISO format
dt.isoformat()                     # '2024-01-15T10:30:45'
datetime.fromisoformat('2024-01-15T10:30:45')

# Components
dt.year, dt.month, dt.day
dt.hour, dt.minute, dt.second
dt.weekday()                       # 0=Monday, 6=Sunday
dt.isoweekday()                    # 1=Monday, 7=Sunday

# Date arithmetic
tomorrow = today + timedelta(days=1)
next_week = today + timedelta(weeks=1)
delta = datetime(2024, 12, 31) - datetime(2024, 1, 1)
delta.days                         # 365
```

### Format Codes

```python
"""
%Y  4-digit year (2024)
%y  2-digit year (24)
%m  Month as zero-padded decimal (01-12)
%B  Full month name (January)
%b  Abbreviated month name (Jan)
%d  Day of month, zero-padded (01-31)
%A  Full weekday name (Monday)
%a  Abbreviated weekday (Mon)
%H  Hour (24-hour), zero-padded (00-23)
%I  Hour (12-hour), zero-padded (01-12)
%p  AM/PM
%M  Minute, zero-padded (00-59)
%S  Second, zero-padded (00-59)
%f  Microsecond (000000-999999)
%z  UTC offset (+0000)
%Z  Timezone name (UTC)
%j  Day of year (001-366)
%U  Week number (Sunday as first day)
%W  Week number (Monday as first day)
"""
```

### Timezone Handling (Critical for Production)

```python
from datetime import datetime, timezone, timedelta
from zoneinfo import ZoneInfo  # Python 3.9+

# IMPORTANT: Always use timezone-aware datetimes in production!

# UTC timezone
utc = timezone.utc
now_utc = datetime.now(utc)        # Timezone-aware UTC

# Using zoneinfo (Python 3.9+)
eastern = ZoneInfo('America/New_York')
pacific = ZoneInfo('America/Los_Angeles')

now_eastern = datetime.now(eastern)
now_pacific = datetime.now(pacific)

# Convert between timezones
utc_time = datetime.now(timezone.utc)
eastern_time = utc_time.astimezone(eastern)

# Naive vs Aware datetimes
naive = datetime.now()             # No timezone info - AVOID!
aware = datetime.now(timezone.utc) # Has timezone info - PREFER!

# Check if aware
naive.tzinfo is None               # True - naive
aware.tzinfo is not None           # True - aware

# Make naive datetime aware (assuming it's UTC)
naive_dt = datetime(2024, 1, 15, 10, 30)
aware_dt = naive_dt.replace(tzinfo=timezone.utc)

# Common gotcha: comparing naive and aware
# naive == aware  # TypeError in Python 3!

# Best practice: store everything as UTC, convert for display
def to_utc(dt):
    """Convert any datetime to UTC."""
    if dt.tzinfo is None:
        # Assume local, convert to UTC
        return dt.astimezone(timezone.utc)
    return dt.astimezone(timezone.utc)

def to_local(dt, tz=None):
    """Convert UTC datetime to local timezone."""
    if tz is None:
        tz = ZoneInfo('America/New_York')
    return dt.astimezone(tz)
```

### timedelta

```python
from datetime import timedelta

# Creating timedeltas
delta = timedelta(days=5)
delta = timedelta(hours=2, minutes=30)
delta = timedelta(weeks=1, days=2, hours=3)

# Arithmetic
tomorrow = datetime.now() + timedelta(days=1)
yesterday = datetime.now() - timedelta(days=1)
two_weeks_ago = datetime.now() - timedelta(weeks=2)

# Difference between datetimes
dt1 = datetime(2024, 1, 1)
dt2 = datetime(2024, 12, 31)
diff = dt2 - dt1
diff.days                          # 365
diff.total_seconds()               # Total seconds as float

# Components
delta = timedelta(days=5, hours=3, minutes=30)
delta.days                         # 5
delta.seconds                      # 12600 (3*3600 + 30*60)
delta.total_seconds()              # 444600.0
```

---

## JSON Module

### Reading and Writing JSON

```python
import json

# Python to JSON string
data = {'name': 'Alice', 'age': 30, 'active': True}
json_string = json.dumps(data)
# '{"name": "Alice", "age": 30, "active": true}'

# Pretty print
json.dumps(data, indent=2)
json.dumps(data, indent=2, sort_keys=True)

# JSON string to Python
json.loads('{"name": "Alice", "age": 30}')
# {'name': 'Alice', 'age': 30}

# Read from file
with open('data.json') as f:
    data = json.load(f)

# Write to file
with open('data.json', 'w') as f:
    json.dump(data, f, indent=2)
```

### Type Mapping

```python
"""
Python          JSON
dict            object
list, tuple     array
str             string
int, float      number
True            true
False           false
None            null
"""

# JSON doesn't support:
# - datetime (convert to ISO string)
# - set (convert to list)
# - bytes (convert to base64)
# - custom objects (need custom encoder)
```

### Custom Encoding/Decoding

```python
import json
from datetime import datetime

# Custom encoder
class CustomEncoder(json.JSONEncoder):
    def default(self, obj):
        if isinstance(obj, datetime):
            return obj.isoformat()
        if isinstance(obj, set):
            return list(obj)
        return super().default(obj)

data = {'timestamp': datetime.now(), 'tags': {'a', 'b'}}
json.dumps(data, cls=CustomEncoder)

# Using default function (simpler)
def json_serializer(obj):
    if isinstance(obj, datetime):
        return obj.isoformat()
    raise TypeError(f"Object of type {type(obj)} is not JSON serializable")

json.dumps(data, default=json_serializer)

# Custom decoder
def json_deserializer(dct):
    for key, value in dct.items():
        if key == 'timestamp':
            dct[key] = datetime.fromisoformat(value)
    return dct

json.loads(json_string, object_hook=json_deserializer)
```

### JSON Best Practices

```python
# Handle missing keys safely
data = json.loads(json_string)
name = data.get('name', 'Unknown')

# Validate JSON structure
import json

def safe_json_loads(s):
    try:
        return json.loads(s)
    except json.JSONDecodeError as e:
        print(f"Invalid JSON: {e}")
        return None

# Pretty print for debugging
def pprint_json(obj):
    print(json.dumps(obj, indent=2, default=str))

# Streaming large JSON files
import ijson  # pip install ijson

with open('large.json', 'rb') as f:
    for item in ijson.items(f, 'items.item'):
        process(item)
```

---

## CSV Module

```python
import csv

# Reading CSV
with open('data.csv', newline='') as f:
    reader = csv.reader(f)
    header = next(reader)  # Skip header
    for row in reader:
        print(row)  # row is a list

# Reading as dict (recommended)
with open('data.csv', newline='') as f:
    reader = csv.DictReader(f)
    for row in reader:
        print(row['name'], row['age'])

# Writing CSV
with open('output.csv', 'w', newline='') as f:
    writer = csv.writer(f)
    writer.writerow(['name', 'age'])
    writer.writerow(['Alice', 30])
    writer.writerows([['Bob', 25], ['Charlie', 35]])

# Writing from dicts
with open('output.csv', 'w', newline='') as f:
    fieldnames = ['name', 'age']
    writer = csv.DictWriter(f, fieldnames=fieldnames)
    writer.writeheader()
    writer.writerow({'name': 'Alice', 'age': 30})

# Different delimiters
reader = csv.reader(f, delimiter='\t')  # TSV
reader = csv.reader(f, delimiter=';')   # Semicolon

# Handling quotes
reader = csv.reader(f, quotechar='"', quoting=csv.QUOTE_MINIMAL)

# CSV dialects
csv.list_dialects()  # ['excel', 'excel-tab', 'unix']
reader = csv.reader(f, dialect='excel')
```

---

## subprocess Module

### Running External Commands

```python
import subprocess

# Simple command (Python 3.5+)
result = subprocess.run(['ls', '-la'], capture_output=True, text=True)
result.stdout              # Output as string
result.stderr              # Error output
result.returncode          # Exit code (0 = success)

# Check for errors
result = subprocess.run(['ls', 'nonexistent'], capture_output=True, text=True)
if result.returncode != 0:
    print(f"Error: {result.stderr}")

# Raise exception on error
try:
    subprocess.run(['ls', 'nonexistent'], check=True, capture_output=True, text=True)
except subprocess.CalledProcessError as e:
    print(f"Command failed: {e}")

# Timeout
try:
    subprocess.run(['sleep', '10'], timeout=5)
except subprocess.TimeoutExpired:
    print("Command timed out")

# Shell commands (use with caution!)
# NEVER pass user input to shell=True - command injection risk!
result = subprocess.run('echo $HOME', shell=True, capture_output=True, text=True)

# Input to command
result = subprocess.run(
    ['grep', 'pattern'],
    input='line1\npattern here\nline3',
    capture_output=True,
    text=True
)

# Working directory
subprocess.run(['ls'], cwd='/tmp')

# Environment variables
subprocess.run(['printenv'], env={'MY_VAR': 'value'})

# Piping commands
# ls | grep .py
ls = subprocess.Popen(['ls'], stdout=subprocess.PIPE)
grep = subprocess.Popen(['grep', '.py'], stdin=ls.stdout, stdout=subprocess.PIPE)
output = grep.communicate()[0]
```

---

## Interview Patterns

```python
# Q1: Parse log file and extract IPs
import re
from collections import Counter

def extract_ips(log_file):
    ip_pattern = re.compile(r'\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}')
    ips = Counter()
    with open(log_file) as f:
        for line in f:
            for ip in ip_pattern.findall(line):
                ips[ip] += 1
    return ips.most_common(10)

# Q2: Find all Python files modified in last 24 hours
from pathlib import Path
from datetime import datetime, timedelta

def recent_python_files(directory):
    cutoff = datetime.now().timestamp() - 86400  # 24 hours
    for path in Path(directory).rglob('*.py'):
        if path.stat().st_mtime > cutoff:
            yield path

# Q3: Parse dates from various formats
def parse_date(date_string):
    formats = [
        '%Y-%m-%d',
        '%d/%m/%Y',
        '%B %d, %Y',
        '%Y-%m-%dT%H:%M:%S',
    ]
    for fmt in formats:
        try:
            return datetime.strptime(date_string, fmt)
        except ValueError:
            continue
    raise ValueError(f"Cannot parse date: {date_string}")

# Q4: Safe JSON loading with defaults
def load_config(path, defaults=None):
    defaults = defaults or {}
    try:
        with open(path) as f:
            config = json.load(f)
            return {**defaults, **config}
    except (FileNotFoundError, json.JSONDecodeError):
        return defaults
```
