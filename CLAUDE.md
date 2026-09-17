# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Package Management

This project uses `uv` for dependency management (Python 3.12).

```bash
uv add <package>          # add a dependency
uv run python main.py     # run the entry point
uv run <command>          # run any command in the project environment
uv sync                   # sync dependencies from pyproject.toml
```

## Project Status

Early-stage project. `main.py` is the entry point stub. No tests or linting are configured yet.
