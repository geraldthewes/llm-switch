# AGENTS.md - Guidelines for AI Coding Agents

## Repository Overview
This repository contains the BMad (Bulletproof Method for AI Development) framework implementation. The primary component is the `bmad-init` skill which handles project configuration bootstrapping and config loading.

## Build, Lint, and Test Commands

### Testing
Run all tests:
```bash
python -m unittest discover -s .opencode/skills/bmad-init/scripts/tests -p "test_*.py"
```

Run a specific test file:
```bash
python -m unittest .opencode/skills/bmad-init/scripts/tests/test_bmad_init.py
```

Run a specific test method:
```bash
python -m unittest .opencode/skills/bmad-init/scripts/tests/test_bmad_init.py.TestFindProjectRoot.test_finds_bmad_folder
```

Run tests with verbose output:
```bash
python -m unittest discover -s .opencode/skills/bmad-init/scripts/tests -p "test_*.py" -v
```

### Linting
Check Python code style with flake8:
```bash
flake8 .opencode/skills/bmad-init/scripts/
```

Check Python code style with pylint:
```bash
pylint .opencode/skills/bmad-init/scripts/
```

### Formatting
Format Python code with black:
```bash
black .opencode/skills/bmad-init/scripts/
```

Sort imports with isort:
```bash
isort .opencode/skills/bmad-init/scripts/
```

### Type Checking
Run mypy for type checking:
```bash
mypy .opencode/skills/bmad-init/scripts/
```

### Security Scanning
Run bandit for security vulnerabilities:
```bash
bandit -r .opencode/skills/bmad-init/scripts/
```

## Code Style Guidelines

### Python Specific Guidelines

#### Imports
- Standard library imports first
- Third-party imports second
- Local application imports third
- Each section separated by a blank line
- Use absolute imports from project root when possible
- Never use wildcard imports (`from module import *`)
- Alphabetize imports within each section

Example:
```python
# Standard library imports
import argparse
import json
import os
import sys
from pathlib import Path

# Third-party imports
import yaml

# Local application imports (none in this file)
```

#### Formatting
- Maximum line length: 88 characters (Black default)
- Use 4 spaces per indentation level (no tabs)
- Blank lines:
  - 2 blank lines between top-level functions and classes
  - 1 blank line between method definitions
  - Use blank lines to separate logical sections within functions
- Trailing commas:
  - Use trailing commas in tuples, lists, and dicts that span multiple lines
  - No trailing commas on single-line constructs

#### Types and Type Hints
- Use Python 3.6+ type hinting syntax
- Import typing module only for complex types (List, Dict, etc.)
- For Python 3.9+, use built-in collection types (list, dict) instead of typing.List, typing.Dict
- Always specify return types for functions
- Specify types for function parameters when not obvious
- Use Optional[T] for values that can be None
- Avoid Any type when possible

Example:
```python
def find_project_root(llm_provided=None):
    """
    Find project root by looking for _bmad folder.

    Args:
        llm_provided: Path explicitly provided via --project-root.

    Returns:
        Path to project root, or None if not found.
    """
    if llm_provided:
        candidate = Path(llm_provided)
        if (candidate / '_bmad').exists():
            return candidate
        # First run — _bmad won't exist yet but LLM path is still valid
        if candidate.is_dir():
            return candidate

    for start_dir in [Path.cwd(), Path(__file__).resolve().parent]:
        current_dir = start_dir
        while current_dir != current_dir.parent:
            if (current_dir / '_bmad').exists():
                return current_dir
            current_dir = current_dir.parent

    return None
```

#### Naming Conventions
- Functions and variables: snake_case
- Classes: PascalCase
- Constants: UPPER_SNAKE_CASE
- Modules: snake_case
- Private functions/variables: _snake_case (single leading underscore)
- Descriptive names preferred over short abbreviations
- Boolean variables: use is_ or has_ prefix when appropriate

#### Error Handling
- Use specific exception types rather than bare except:
- Catch exceptions as close to the source as possible
- Always handle or re-raise caught exceptions
- Don't suppress exceptions with empty except blocks
- Use finally blocks for cleanup operations
- Raise appropriate exceptions with descriptive messages
- In CLI applications, exit with non-zero status code on error

Example:
```python
try:
    with open(path, 'r', encoding='utf-8') as f:
        raw = yaml.safe_load(f)
except Exception:
    return None

if not raw or not isinstance(raw, dict):
    return None
```

#### Documentation
- Use docstrings for all public modules, functions, classes, and methods
- Follow Google-style or NumPy-style docstrings
- Include Args, Returns, and Raises sections when applicable
- Keep docstrings concise but informative
- Use inline comments sparingly, only for non-obvious code
- Update docstrings when modifying functions
- Use triple double quotes for docstrings

Example:
```python
def load_module_yaml(path):
    """
    Load and parse a module.yaml file, separating metadata from variable definitions.

    Returns:
        Dict with 'meta' (code, name, etc.) and 'variables' (var definitions)
        and 'directories' (list of dir templates), or None on failure.
    """
```

#### CLI Applications
- Use argparse for command-line argument parsing
- Provide clear help text for all arguments
- Use subparsers for distinct commands
- Return JSON output for machine-consumption when appropriate
- Exit with status code 0 for success, 1 for error
- Provide meaningful error messages to stderr
- Don't print to stdout unless it's the intended output

#### Testing
- Write unit tests for all public functions
- Test edge cases and error conditions
- Use descriptive test method names
- Follow Arrange-Act-Assert pattern in tests
- Mock external dependencies when necessary
- Keep tests focused on a single unit of behavior
- Use setUp and tearDown for test fixtures
- Don't test private implementation details unless necessary

## Project Structure Guidelines

### BMad Skills
- Each skill should be self-contained in its own directory under `.opencode/skills/`
- Skill directories should follow the pattern: `.opencode/skills/{skill-name}/`
- Each skill should contain:
  - `scripts/` - Main implementation code
  - `resources/` - Static resources (templates, configs, etc.)
  - `tests/` - Unit tests for the skill
- Skills should not have external dependencies beyond what's declared

### Configuration Files
- Use YAML for configuration files (.yaml or .yml extension)
- Configuration files should be flat (no nested structures beyond one level)
- Use `{placeholder}` syntax for template expansion
- Include header comments with generation info
- Separate metadata from variable definitions in module.yaml files

## Development Workflow

### Making Changes
1. Create a new branch for each feature or bug fix
2. Write tests before implementing new functionality (TDD preferred)
3. Implement the minimum code to pass tests
4. Run all tests to ensure nothing is broken
5. Run linting and formatting tools
6. Update documentation as needed
7. Submit pull request with clear description

### Code Review
- Check for adherence to style guidelines
- Verify tests cover new functionality
- Ensure error handling is appropriate
- Confirm documentation is updated
- Look for potential security issues
- Check for performance improvements where relevant

## Dependencies Management
- Pin exact versions in requirements.txt or package.json
- Use virtual environments for development
- Update dependencies regularly but test thoroughly after updates
- Document why specific versions are required if not latest
- Remove unused dependencies

## Security Practices
- Never commit secrets or API keys
- Use environment variables for sensitive configuration
- Validate all inputs
- Use parameterized queries for database operations
- Keep dependencies updated to patch security vulnerabilities
- Follow principle of least privilege
