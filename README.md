![](banner.jpg)

# Python Development Standards

A comprehensive skill for Python development that enforces zero-fabrication, test-driven development with strict quality gates and real integrations only.

## Purpose

This skill provides development standards and practices for Python projects that require:

- Rigorous testing with real integrations (no mocks, fakes, or stubs)
- Strict code quality enforcement
- Clear project structure and organization
- Consistent coding patterns and naming conventions

## Installation

No installation is required. This skill is automatically available when working on Python projects within the Claude Code environment.

## Usage

Invoke the skill when starting work on a Python project:

```
/python
```

The skill will guide development practices including:

- Project structure and layout
- Coding standards and naming conventions
- Testing patterns (integration-first, pytest-based)
- Error handling approaches
- Linting and quality gates

## Examples

### Starting a New Python Project

```
/python

I need to create a new Python project for processing customer data.
```

### Working on an Existing Project

```
/python

Help me add a new feature to fetch and transform API responses.
```

### Running Quality Checks

After making changes, ensure quality gates pass:

```bash
# Run linter
ruff check --line-length 120

# Run tests for a specific file
pytest src/path/file_test.py

# Run full quality check before committing
./run check
```

## Key Principles

1. **No Fabrication** - Never use mocks, fakes, stubs, or simulated data
2. **Real Tests Only** - All tests are real integration tests with actual dependencies
3. **Test Failure = Task Failure** - Any failing test means the task is incomplete
4. **Quality Gates Must Pass** - No commits without passing `./run check`
5. **DRY > KISS > YAGNI** - Remove duplication first, then simplify, then avoid unneeded features

## License

This project is licensed under [CC BY-NC 4.0](https://darren-static.waft.dev/license) - free to use and modify, but no commercial use without permission.
