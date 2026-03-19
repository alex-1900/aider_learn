# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Aider is AI pair programming in your terminal - a CLI tool that lets users pair program with LLMs to edit code in their git repository. It supports multiple LLM providers and various edit formats for code modifications.

## Development Setup

```bash
# Create and activate virtual environment
python3 -m venv ../aider_venv && source ../aider_venv/bin/activate

# Install in editable mode with dependencies
pip install -e .
pip install -r requirements.txt
pip install -r requirements/requirements-dev.txt

# Optional: install pre-commit hooks
pre-commit install
```

## Commands

### Testing
```bash
# Run all tests
pytest

# Run specific test file
pytest tests/basic/test_coder.py

# Run specific test case
pytest tests/basic/test_coder.py::TestCoder::test_allowed_to_edit
```

### Linting and Formatting
```bash
# Run all pre-commit checks
pre-commit run --all-files

# Individual tools (line length 100)
black --line-length 100 .
isort --profile black .
flake8 --show-source .
codespell
```

### Building
```bash
# Build package
pip install build && python -m build
```

## Architecture

### Entry Points
- `aider/main.py:main()` - CLI entry point, argument parsing, setup
- `aider/__main__.py` - Allows running via `python -m aider`

### Core Components

**Coder Classes** (`aider/coders/`):
- `base_coder.py:Coder` - Base class handling the main chat loop, file management, and LLM interaction
- Multiple subclasses for different edit formats:
  - `EditBlockCoder` - Search/replace blocks (default "diff" format)
  - `WholeFileCoder` - Complete file rewrites
  - `UnifiedDiffCoder` - Standard unified diffs
  - `PatchCoder` - Git patch format
  - `ArchitectCoder`, `AskCoder`, `HelpCoder`, etc. - Special modes

**Model Management** (`aider/models.py`):
- `Model` class wraps LLM configuration
- Model aliases (e.g., "sonnet" -> "claude-sonnet-4-5")
- Uses litellm for unified API access (lazy-loaded via `aider/llm.py`)

**I/O System** (`aider/io.py`):
- `InputOutput` class handles all terminal I/O
- Uses prompt_toolkit for rich input with completion
- Uses rich for formatted output

**Git Integration** (`aider/repo.py`):
- `GitRepo` class wraps git operations
- Auto-commits, manages .gitignore, tracks changes

**Code Mapping** (`aider/repomap.py`):
- `RepoMap` builds a map of the codebase using tree-sitter
- Provides context to LLMs about code structure

**Commands** (`aider/commands.py`):
- `Commands` class implements slash commands (/add, /drop, /diff, etc.)

### Edit Formats

The coder subclasses differ primarily in how they:
1. Parse LLM responses for code changes (`get_edits()`)
2. Apply those changes to files (`apply_edits()`)

Each coder has a corresponding prompts module defining the system prompts that instruct the LLM how to format edits.

## Code Style

- **No type hints** - The project explicitly avoids type annotations
- **Line length**: 100 characters max
- **Formatting**: Black with isort (black profile)
- **Python versions**: 3.10 through 3.14

## Key Patterns

- `from .dump import dump` is used for debugging (imported but often noqa'd)
- Tests use `unittest.TestCase` with `MagicMock`/`patch` for mocking
- `GitTemporaryDirectory` context manager creates temp git repos for tests
- LLM communication goes through litellm abstraction layer

## Dependencies

- `litellm` - Unified interface to LLM APIs
- `prompt_toolkit` - Terminal input handling
- `rich` - Terminal output formatting
- `tree-sitter` / `grep-ast` - Code parsing for repo map
- `gitpython` - Git operations
- `configargparse` - CLI argument parsing with config file support

## Project Conventions

- `learn/` - Directory for project learning notes (markdown documents)
