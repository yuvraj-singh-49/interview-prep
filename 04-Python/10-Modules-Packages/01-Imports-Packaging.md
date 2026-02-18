# Modules, Packages, and Imports

## How Python Imports Work

### Import Mechanism

```python
"""
When you write: import mymodule

Python does:
1. Check sys.modules (already imported?)
2. Find the module (search sys.path)
3. Create module object
4. Execute module code
5. Bind name in current namespace
"""

import sys

# Check what's already imported
'os' in sys.modules  # True if already imported

# sys.path - where Python looks for modules
print(sys.path)
# ['', '/usr/lib/python3.10', '/usr/lib/python3.10/lib-dynload', ...]
# '' = current directory (first!)

# Add to path at runtime (generally not recommended)
sys.path.append('/my/custom/path')
sys.path.insert(0, '/my/priority/path')  # Add at beginning

# PYTHONPATH environment variable adds to sys.path
# export PYTHONPATH=/my/path:$PYTHONPATH
```

### Import Statements

```python
# Import module
import os
os.getcwd()

# Import with alias
import numpy as np
np.array([1, 2, 3])

# Import specific names
from os import path, getcwd
getcwd()

# Import with alias
from os import path as ospath

# Import all (avoid in production code!)
from os import *
# Pollutes namespace, hard to track where names come from

# Relative imports (within packages)
from . import sibling_module
from .. import parent_package_module
from ..sibling_package import module

# Import from string (dynamic import)
import importlib
module = importlib.import_module('os.path')
```

### Module Search Order

```
1. Built-in modules (sys, os, etc.)
2. sys.modules cache (already imported)
3. sys.path locations:
   a. Current directory
   b. PYTHONPATH directories
   c. Installation-dependent defaults
   d. Site-packages (pip installed)

Check which file is used:
>>> import os
>>> os.__file__
'/usr/lib/python3.10/os.py'
```

---

## Creating Modules

### Module Basics

```python
# mymodule.py
"""Module docstring - describes the module."""

# Module-level variables
VERSION = "1.0.0"
_private_var = "internal use"  # Convention: underscore = private

# Module-level functions
def greet(name):
    """Greet someone."""
    return f"Hello, {name}!"

def _helper():
    """Private helper function."""
    pass

# Module-level class
class MyClass:
    pass

# Code that runs on import
print("Module loaded!")  # Runs when imported

# Code that runs only when executed directly
if __name__ == "__main__":
    # This only runs when: python mymodule.py
    # NOT when: import mymodule
    print("Running as main script")
    greet("World")
```

### `__name__` and `__main__`

```python
# mymodule.py
print(f"__name__ is: {__name__}")

def main():
    print("Main function")

if __name__ == "__main__":
    main()

# When run directly: python mymodule.py
# __name__ is: __main__
# Main function

# When imported: import mymodule
# __name__ is: mymodule
# (main() not called)

# Why this matters:
# - Allows module to be both imported AND run as script
# - Tests can import without running main code
# - Prevents side effects on import
```

### Module Attributes

```python
# Every module has these attributes
import mymodule

mymodule.__name__      # 'mymodule'
mymodule.__file__      # '/path/to/mymodule.py'
mymodule.__doc__       # Module docstring
mymodule.__dict__      # Module namespace
mymodule.__package__   # Package name (or '' for standalone)
mymodule.__spec__      # Module spec (import machinery info)

# Control what's exported with from module import *
__all__ = ['greet', 'MyClass']  # Only these are exported

# Without __all__, all public names (no leading _) are exported
```

---

## Creating Packages

### Package Structure

```
mypackage/
├── __init__.py          # Makes it a package
├── module1.py
├── module2.py
├── subpackage/
│   ├── __init__.py
│   └── module3.py
└── data/
    └── config.json
```

### `__init__.py`

```python
# mypackage/__init__.py

"""Package docstring."""

# Package version
__version__ = "1.0.0"

# Import submodules to expose at package level
from .module1 import func1, Class1
from .module2 import func2

# Now users can do:
# from mypackage import func1
# Instead of:
# from mypackage.module1 import func1

# Control what's exported
__all__ = ['func1', 'func2', 'Class1']

# Package-level initialization code
print("Package initialized!")

# Lazy loading (for heavy modules)
def __getattr__(name):
    if name == "heavy_module":
        from . import heavy_module
        return heavy_module
    raise AttributeError(f"module {__name__!r} has no attribute {name!r}")
```

### Namespace Packages (PEP 420)

```python
# Python 3.3+ allows packages without __init__.py
# Called "namespace packages"

# Directory structure:
# company_package/
#     subpackage1/
#         module.py
#     subpackage2/
#         module.py

# No __init__.py needed!
# Useful for splitting a package across multiple directories/distributions

# Can have parts in different locations:
# /path1/company_package/subpackage1/
# /path2/company_package/subpackage2/

# Both contribute to same package namespace
```

---

## Import Best Practices

### Import Order (PEP 8)

```python
# 1. Standard library imports
import os
import sys
from datetime import datetime

# 2. Blank line

# 3. Third-party imports
import numpy as np
import requests
from flask import Flask

# 4. Blank line

# 5. Local application imports
from mypackage import module1
from mypackage.utils import helper

# Use isort to auto-format:
# pip install isort
# isort myfile.py
```

### Avoiding Circular Imports

```python
# Problem: Circular dependency
# module_a.py
from module_b import func_b  # Imports module_b
def func_a():
    return func_b()

# module_b.py
from module_a import func_a  # Imports module_a - CIRCULAR!
def func_b():
    return func_a()

# Solutions:

# 1. Import inside function (lazy import)
# module_a.py
def func_a():
    from module_b import func_b  # Import when needed
    return func_b()

# 2. Import at bottom of file
# module_a.py
def func_a():
    return func_b()

from module_b import func_b  # After all definitions

# 3. Restructure code
# - Move shared code to separate module
# - Use dependency injection
# - Merge modules if tightly coupled

# 4. Import module, not name
# module_a.py
import module_b

def func_a():
    return module_b.func_b()  # Resolved at runtime
```

### Absolute vs Relative Imports

```python
# Package structure:
# mypackage/
#     __init__.py
#     module1.py
#     subpackage/
#         __init__.py
#         module2.py

# In module2.py:

# Absolute imports (preferred - explicit)
from mypackage.module1 import func1
from mypackage import module1

# Relative imports (useful within packages)
from .. import module1      # Parent package
from ..module1 import func1 # Specific name from parent
from . import sibling       # Same package
from .sibling import func   # Name from sibling

# When to use which:
# - Absolute: External code, top-level scripts
# - Relative: Within a package (makes package relocatable)

# PEP 8 recommends absolute imports for clarity
# But relative imports are acceptable within packages
```

---

## Virtual Environments

### venv (Built-in)

```bash
# Create virtual environment
python -m venv myenv

# Activate (Linux/Mac)
source myenv/bin/activate

# Activate (Windows)
myenv\Scripts\activate

# Deactivate
deactivate

# Check which Python
which python  # Should point to myenv/bin/python

# Install packages (isolated)
pip install requests

# Freeze requirements
pip freeze > requirements.txt

# Install from requirements
pip install -r requirements.txt

# Delete environment
rm -rf myenv
```

### requirements.txt

```txt
# Exact versions (for reproducibility)
requests==2.28.0
numpy==1.23.0
pandas==1.5.0

# Minimum version
requests>=2.20.0

# Version range
requests>=2.20.0,<3.0.0

# From git
git+https://github.com/user/repo.git@v1.0.0

# Local package
-e ./mypackage

# From specific index
--index-url https://pypi.org/simple/
```

### Poetry (Modern Alternative)

```bash
# Install poetry
pip install poetry

# Create new project
poetry new myproject

# Or initialize existing project
poetry init

# Add dependencies
poetry add requests
poetry add pytest --group dev

# Install all dependencies
poetry install

# Run in virtual environment
poetry run python script.py

# Activate shell
poetry shell

# Update dependencies
poetry update

# Build package
poetry build

# Publish to PyPI
poetry publish
```

### pyproject.toml

```toml
[project]
name = "mypackage"
version = "1.0.0"
description = "My awesome package"
readme = "README.md"
requires-python = ">=3.8"
license = {text = "MIT"}
authors = [
    {name = "Your Name", email = "you@example.com"}
]
dependencies = [
    "requests>=2.20.0",
    "numpy>=1.20.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=7.0.0",
    "black>=22.0.0",
]

[project.scripts]
mycli = "mypackage.cli:main"

[build-system]
requires = ["setuptools>=61.0"]
build-backend = "setuptools.build_meta"

[tool.pytest.ini_options]
testpaths = ["tests"]

[tool.black]
line-length = 88

[tool.isort]
profile = "black"
```

---

## Creating Distributable Packages

### Package Structure for Distribution

```
mypackage/
├── src/
│   └── mypackage/
│       ├── __init__.py
│       ├── core.py
│       └── utils.py
├── tests/
│   ├── __init__.py
│   └── test_core.py
├── docs/
│   └── index.md
├── pyproject.toml
├── README.md
├── LICENSE
└── .gitignore
```

### Building and Publishing

```bash
# Build package
pip install build
python -m build

# Creates:
# dist/mypackage-1.0.0.tar.gz
# dist/mypackage-1.0.0-py3-none-any.whl

# Upload to PyPI
pip install twine
twine upload dist/*

# Upload to TestPyPI first
twine upload --repository testpypi dist/*

# Install from TestPyPI
pip install --index-url https://test.pypi.org/simple/ mypackage
```

### Entry Points (CLI Tools)

```python
# mypackage/cli.py
import argparse

def main():
    parser = argparse.ArgumentParser(description='My CLI tool')
    parser.add_argument('name', help='Name to greet')
    parser.add_argument('-v', '--verbose', action='store_true')
    args = parser.parse_args()

    if args.verbose:
        print(f"Verbose mode enabled")
    print(f"Hello, {args.name}!")

if __name__ == '__main__':
    main()
```

```toml
# pyproject.toml
[project.scripts]
mycommand = "mypackage.cli:main"

# After pip install mypackage:
# $ mycommand Alice
# Hello, Alice!
```

---

## Dynamic Imports

```python
import importlib

# Import module by name
module = importlib.import_module('os.path')
module.join('a', 'b')

# Import class by name
def get_class(module_path, class_name):
    module = importlib.import_module(module_path)
    return getattr(module, class_name)

MyClass = get_class('mypackage.models', 'User')
user = MyClass()

# Reload module (during development)
import mymodule
importlib.reload(mymodule)

# Import from file path
import importlib.util

spec = importlib.util.spec_from_file_location("mymodule", "/path/to/mymodule.py")
module = importlib.util.module_from_spec(spec)
spec.loader.exec_module(module)

# Plugin system example
def load_plugins(plugin_dir):
    plugins = {}
    for path in Path(plugin_dir).glob('*.py'):
        spec = importlib.util.spec_from_file_location(path.stem, path)
        module = importlib.util.module_from_spec(spec)
        spec.loader.exec_module(module)
        if hasattr(module, 'Plugin'):
            plugins[path.stem] = module.Plugin
    return plugins
```

---

## Interview Questions

```python
# Q1: What's the difference between import X and from X import Y?
"""
import X:
- Imports module, binds to name X
- Access via X.name
- Module namespace separate

from X import Y:
- Imports specific name into current namespace
- Access directly as Y
- Can cause name conflicts
"""

# Q2: How do you prevent code from running on import?
"""
Use if __name__ == "__main__":

This code only runs when the file is executed directly,
not when imported as a module.
"""

# Q3: What is __all__ and when would you use it?
"""
__all__ = ['func1', 'Class1']

Controls what's exported when using: from module import *
Without it, all public names (no leading _) are exported.
Use it to explicitly define the public API.
"""

# Q4: How do you resolve circular imports?
"""
1. Import inside function (lazy import)
2. Import module, not name (deferred resolution)
3. Import at bottom of file
4. Restructure code (separate shared code)
5. Use dependency injection
"""

# Q5: Explain the module search path
"""
Python searches in order:
1. Built-in modules
2. sys.modules cache
3. sys.path: current dir, PYTHONPATH, defaults, site-packages
"""
```
