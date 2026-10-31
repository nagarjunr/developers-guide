# Modern Python Tooling: Ruff, Mypy, and Pre-commit

A comprehensive guide to setting up modern Python development tools for fast, consistent, and type-safe code.

## Table of Contents

1. [Why Modern Tooling?](#why-modern-tooling)
2. [Ruff: The Fast All-in-One Linter](#ruff-the-fast-all-in-one-linter)
3. [Mypy: Static Type Checking](#mypy-static-type-checking)
4. [Pre-commit: Automated Git Hooks](#pre-commit-automated-git-hooks)
5. [Makefile: Developer Commands](#makefile-developer-commands)
6. [Complete Setup](#complete-setup)
7. [Best Practices](#best-practices)

---

## Why Modern Tooling?

**The old way** required multiple tools:
```bash
black .           # Formatting
isort .           # Import sorting
flake8 .          # Linting
pylint .          # More linting
mypy .            # Type checking
```

**The modern way** consolidates with Ruff:
```bash
ruff format .     # Formatting (replaces Black)
ruff check .      # Linting (replaces flake8, isort, pylint)
mypy .            # Type checking (still needed)
```

**Benefits:**
- ⚡ **10-100x faster** than traditional tools
- 🎯 **Single tool** for formatting and linting
- 🔧 **Auto-fix** for most issues
- 📦 **Zero config** to get started
- 🔄 **Drop-in replacement** for Black, isort, flake8

---

## Ruff: The Fast All-in-One Linter

### Installation

```bash
pip install ruff
```

### Basic Usage

```bash
# Check for issues
ruff check .

# Check and auto-fix
ruff check . --fix

# Format code (like Black)
ruff format .

# Check specific files
ruff check src/main.py src/utils.py

# Show statistics
ruff check . --statistics
```

### Configuration

**File: `pyproject.toml`**

```toml
[tool.ruff]
# Target Python 3.11+
target-version = "py311"

# Line length (default: 88, Black-compatible)
line-length = 100

# Exclude directories
exclude = [
    ".git",
    ".venv",
    "venv",
    "__pycache__",
    "build",
    "dist",
    "*.egg-info",
]

[tool.ruff.lint]
# Enable rule sets
select = [
    "E",   # pycodestyle errors
    "W",   # pycodestyle warnings
    "F",   # pyflakes
    "I",   # isort (import sorting)
    "N",   # pep8-naming
    "UP",  # pyupgrade (modernize Python syntax)
    "B",   # flake8-bugbear (find likely bugs)
    "C4",  # flake8-comprehensions
    "SIM", # flake8-simplify
]

# Disable specific rules
ignore = [
    "E501",  # Line too long (handled by formatter)
    "B008",  # Do not perform function calls in argument defaults
]

# Allow autofix for all enabled rules
fixable = ["ALL"]
unfixable = []

[tool.ruff.lint.per-file-ignores]
# Allow print statements in example files
"examples/**/*.py" = ["T201"]
# Allow unused imports in __init__.py
"__init__.py" = ["F401"]

[tool.ruff.format]
# Use double quotes (default)
quote-style = "double"

# Indent with spaces
indent-style = "space"

# Respect magic trailing comma
skip-magic-trailing-comma = false
```

### Common Ruff Rules

| Rule | Description | Example |
|------|-------------|---------|
| `E501` | Line too long | Lines exceeding line-length |
| `F401` | Unused import | `import os` but never used |
| `F841` | Unused variable | `x = 5` but never used |
| `I001` | Import order | Imports not sorted correctly |
| `N806` | Variable naming | `MyVar` should be `my_var` |
| `UP` | Python upgrades | `List[str]` → `list[str]` (3.9+) |
| `B` | Bugbear | Common bugs and design problems |

### Formatting Examples

**Before:**
```python
from typing import List,Dict
import os,sys
from mypackage import utils,helpers

def my_function(x:int,y:int)->int:
    result=x+y
    return result
```

**After `ruff format`:**
```python
from typing import Dict, List
import os
import sys

from mypackage import helpers, utils


def my_function(x: int, y: int) -> int:
    result = x + y
    return result
```

---

## Mypy: Static Type Checking

### Installation

```bash
pip install mypy
```

### Basic Usage

```bash
# Check entire project
mypy .

# Check specific files
mypy src/main.py

# Ignore missing imports
mypy . --ignore-missing-imports

# Show error codes
mypy . --show-error-codes
```

### Configuration

**File: `pyproject.toml`**

```toml
[tool.mypy]
# Python version
python_version = "3.11"

# Strictness
warn_return_any = true
warn_unused_configs = true
disallow_untyped_defs = false  # Set true for strict typing

# Import handling
ignore_missing_imports = true
follow_imports = "silent"

# Output
show_error_codes = true
pretty = true

# Exclude patterns
exclude = [
    "venv/",
    ".venv/",
    "build/",
    "dist/",
]

# Per-module options
[[tool.mypy.overrides]]
module = "tests.*"
ignore_errors = true
```

### Type Hints Examples

```python
from typing import List, Dict, Optional, Union, Callable

# Function signatures
def greet(name: str) -> str:
    return f"Hello, {name}!"

def process_items(items: List[str]) -> Dict[str, int]:
    return {item: len(item) for item in items}

# Optional parameters
def find_user(user_id: int) -> Optional[dict]:
    # Returns dict or None
    pass

# Union types
def parse_id(value: Union[str, int]) -> int:
    return int(value)

# Callable (function types)
def apply_operation(x: int, operation: Callable[[int], int]) -> int:
    return operation(x)

# Generic types (Python 3.9+)
def first_item(items: list[str]) -> str | None:
    return items[0] if items else None
```

### Common Mypy Errors

```python
# Error: Incompatible return value type
def get_age() -> int:
    return "25"  # ❌ Expected int, got str

# Fix:
def get_age() -> int:
    return 25  # ✅

# Error: Missing type annotation
def calculate(x, y):  # ❌ Missing types
    return x + y

# Fix:
def calculate(x: int, y: int) -> int:  # ✅
    return x + y

# Error: Argument has incompatible type
def greet(name: str) -> str:
    return f"Hello, {name}"

greet(123)  # ❌ Expected str, got int

# Fix:
greet("Alice")  # ✅
```

---

## Pre-commit: Automated Git Hooks

### Installation

```bash
pip install pre-commit
```

### Setup

**1. Create configuration file:**

**File: `.pre-commit-config.yaml`**

```yaml
# Pre-commit hooks configuration
repos:
  # Ruff for linting and formatting
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.1.15
    hooks:
      # Run linter
      - id: ruff
        args: [--fix, --exit-non-zero-on-fix]
      # Run formatter
      - id: ruff-format

  # Mypy for type checking
  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.8.0
    hooks:
      - id: mypy
        args: [--ignore-missing-imports, --show-error-codes]
        additional_dependencies: [types-requests]

  # General file checks
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-json
      - id: check-added-large-files
        args: [--maxkb=1000]
      - id: check-merge-conflict
      - id: detect-private-key
```

**2. Install hooks:**

```bash
# Install pre-commit hooks
pre-commit install

# Verify installation
pre-commit --version
```

**3. Run manually:**

```bash
# Run on all files
pre-commit run --all-files

# Run on staged files only
pre-commit run

# Run specific hook
pre-commit run ruff --all-files
```

### Pre-commit Workflow

```bash
# 1. Make changes to your code
vim src/main.py

# 2. Stage changes
git add src/main.py

# 3. Commit (hooks run automatically)
git commit -m "Add new feature"

# If hooks fail:
# - Code is automatically fixed (if possible)
# - Review the changes
# - Stage the fixes
git add src/main.py
git commit -m "Add new feature"

# If hooks still fail:
# - Fix issues manually
# - Stage and commit again
```

### Skip Hooks (Use Sparingly)

```bash
# Skip all hooks
git commit --no-verify -m "Emergency fix"

# Skip specific hook
SKIP=mypy git commit -m "WIP: incomplete types"
```

---

## Makefile: Developer Commands

Create a `Makefile` for common development tasks:

**File: `Makefile`**

```makefile
.PHONY: help setup format lint lint-fix typecheck check clean pre-commit-run pre-commit-update

# Default target
help:
	@echo "Available commands:"
	@echo "  make setup            - Install development dependencies"
	@echo "  make format           - Format code with Ruff"
	@echo "  make lint             - Check code with Ruff (no fixes)"
	@echo "  make lint-fix         - Check and auto-fix issues with Ruff"
	@echo "  make typecheck        - Run type checking with Mypy"
	@echo "  make check            - Run all checks (format, lint, typecheck)"
	@echo "  make clean            - Remove cache files"
	@echo "  make pre-commit-run   - Run pre-commit on all files"
	@echo "  make pre-commit-update - Update pre-commit hooks"

# Install dependencies
setup:
	pip install -r requirements-dev.txt
	pre-commit install
	@echo "✅ Development environment ready!"

# Format code
format:
	ruff format .
	@echo "✅ Code formatted"

# Lint without fixing
lint:
	ruff check .
	@echo "✅ Linting complete"

# Lint and auto-fix
lint-fix:
	ruff check . --fix
	@echo "✅ Linting complete with fixes"

# Type check
typecheck:
	mypy .
	@echo "✅ Type checking complete"

# Run all checks
check: format lint-fix typecheck
	@echo "✅ All checks passed!"

# Clean cache files
clean:
	find . -type d -name "__pycache__" -exec rm -rf {} + 2>/dev/null || true
	find . -type d -name ".mypy_cache" -exec rm -rf {} + 2>/dev/null || true
	find . -type d -name ".ruff_cache" -exec rm -rf {} + 2>/dev/null || true
	find . -type d -name "*.egg-info" -exec rm -rf {} + 2>/dev/null || true
	find . -type f -name "*.pyc" -delete
	@echo "✅ Cache cleaned"

# Run pre-commit hooks
pre-commit-run:
	pre-commit run --all-files
	@echo "✅ Pre-commit hooks executed"

# Update pre-commit hooks
pre-commit-update:
	pre-commit autoupdate
	@echo "✅ Pre-commit hooks updated"
```

### Usage

```bash
# Show available commands
make help

# Complete setup
make setup

# Format code
make format

# Lint and fix
make lint-fix

# Run all checks
make check

# Clean cache
make clean
```

---

## Complete Setup

### Project Structure

```
my-project/
├── src/
│   ├── __init__.py
│   └── main.py
├── tests/
│   └── test_main.py
├── .pre-commit-config.yaml
├── pyproject.toml
├── Makefile
├── requirements.txt
├── requirements-dev.txt
└── README.md
```

### Step-by-Step Setup

**1. Create `requirements-dev.txt`:**

```txt
# Development dependencies
ruff>=0.1.15
mypy>=1.8.0
pre-commit>=3.6.0
pytest>=7.4.0
```

**2. Create `pyproject.toml`:**

```toml
[build-system]
requires = ["setuptools>=61.0", "wheel"]
build-backend = "setuptools.build_meta"

[project]
name = "my-project"
version = "1.0.0"
requires-python = ">=3.11"

[tool.ruff]
target-version = "py311"
line-length = 100

[tool.ruff.lint]
select = ["E", "W", "F", "I", "N", "UP", "B", "C4", "SIM"]
ignore = ["E501"]

[tool.mypy]
python_version = "3.11"
ignore_missing_imports = true
show_error_codes = true
```

**3. Create `.pre-commit-config.yaml`:**

```yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.1.15
    hooks:
      - id: ruff
        args: [--fix, --exit-non-zero-on-fix]
      - id: ruff-format

  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.8.0
    hooks:
      - id: mypy
        args: [--ignore-missing-imports]

  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-json
```

**4. Install everything:**

```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements-dev.txt

# Install pre-commit hooks
pre-commit install

# Run initial check
make check
```

---

## Best Practices

### 1. Commit Workflow

```bash
# 1. Write code
vim src/main.py

# 2. Format and lint
make check

# 3. Stage and commit (hooks run automatically)
git add .
git commit -m "Add new feature"
```

### 2. Project Standards

**Naming conventions (PEP 8):**
```python
# Modules/packages: lowercase_with_underscores
my_module.py

# Classes: PascalCase
class MyClass:
    pass

# Functions/methods: lowercase_with_underscores
def my_function():
    pass

# Constants: UPPERCASE_WITH_UNDERSCORES
MAX_SIZE = 100
```

**Type hints:**
```python
# Always add type hints for public APIs
def public_function(name: str, age: int) -> dict[str, str | int]:
    return {"name": name, "age": age}

# Private functions can skip types if obvious
def _internal_helper(x):
    return x * 2
```

### 3. CI/CD Integration

Add to your CI pipeline:

```yaml
# GitHub Actions example
name: Code Quality

on: [push, pull_request]

jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-python@v4
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: pip install ruff mypy

      - name: Lint with Ruff
        run: ruff check .

      - name: Type check with Mypy
        run: mypy .
```

### 4. Editor Integration

**VS Code (`settings.json`):**
```json
{
  "[python]": {
    "editor.defaultFormatter": "charliermarsh.ruff",
    "editor.formatOnSave": true,
    "editor.codeActionsOnSave": {
      "source.fixAll": true,
      "source.organizeImports": true
    }
  },
  "ruff.lint.args": ["--fix"],
  "python.linting.enabled": true,
  "python.linting.mypyEnabled": true
}
```

**PyCharm:**
- Install Ruff plugin from marketplace
- Enable Mypy in Settings → Tools → External Tools

### 5. Performance Tips

```bash
# Ruff is fast - use it frequently
ruff check .  # < 1 second for most projects

# Mypy can be slow - cache results
mypy .  # Creates .mypy_cache/ for faster subsequent runs

# Pre-commit caches results per file
# Only changed files are re-checked
```

---

## Migration Guide

### From Black + isort + flake8

**Before:**
```bash
# Old approach (slow)
black .
isort .
flake8 .
```

**After:**
```bash
# New approach (fast)
ruff format .
ruff check . --fix
```

**Update configuration:**

Remove old tools:
```bash
pip uninstall black isort flake8 pylint
```

Install Ruff:
```bash
pip install ruff
```

Update `pyproject.toml`:
```toml
# Remove old tool configs
# [tool.black]
# [tool.isort]

# Add Ruff config
[tool.ruff]
line-length = 100
```

---

## Troubleshooting

### Issue 1: Ruff and Mypy Conflict

**Problem:** Ruff auto-fixes code that Mypy then complains about.

**Solution:**
```toml
# pyproject.toml - Disable conflicting Ruff rules
[tool.ruff.lint]
ignore = ["UP006", "UP007"]  # Keep old-style type hints if needed
```

### Issue 2: Pre-commit Hooks Fail

**Problem:** Hooks fail on every commit.

**Solution:**
```bash
# Update hooks to latest versions
pre-commit autoupdate

# Clear cache
pre-commit clean

# Re-run
pre-commit run --all-files
```

### Issue 3: Ruff Too Strict

**Problem:** Too many linting errors.

**Solution:**
```toml
# Start with minimal rules
[tool.ruff.lint]
select = ["E", "F"]  # Only errors and pyflakes

# Gradually add more
# select = ["E", "F", "I", "N", "UP"]
```

---

## Summary

**Modern Python tooling stack:**

1. **Ruff** - Fast linting and formatting (replaces Black, isort, flake8, pylint)
2. **Mypy** - Static type checking
3. **Pre-commit** - Automated git hooks
4. **Makefile** - Convenience commands

**Benefits:**
- ⚡ 10-100x faster than traditional tools
- 🎯 Single tool for multiple tasks
- 🤖 Automated checks on every commit
- 📦 Minimal configuration needed
- 🔧 Auto-fix for most issues

**Workflow:**
```bash
make setup      # One-time setup
make check      # Run all checks
git commit      # Hooks run automatically
```

## Further Reading

- [Ruff Documentation](https://docs.astral.sh/ruff/)
- [Mypy Documentation](https://mypy.readthedocs.io/)
- [Pre-commit Documentation](https://pre-commit.com/)
- [PEP 8 Style Guide](https://pep8.org/)

---

**Created:** 2026-02-06
**Tags:** #python #ruff #mypy #pre-commit #tooling #linting #formatting #dev-tools
