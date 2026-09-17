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

## Linting, Formatting & Types

```bash
uv run ruff check .          # lint
uv run ruff check . --fix    # lint and auto-fix
uv run ruff format .         # format
uv run mypy .                # type check
uv run pre-commit run --all-files   # run every hook over the whole tree
```

Ruff uses Google-style docstrings at line length 120. Rules: pycodestyle (E), Pyflakes (F), isort (I), pyupgrade (UP), pydocstyle (D).

mypy runs with `disallow_untyped_defs`, so **every function needs annotations** — including `-> None`. Unannotated defs fail the commit.

`pre-commit install` wires ruff and mypy into the commit hook; hook revisions are pinned in `.pre-commit-config.yaml` and bumped with `uv run pre-commit autoupdate`.

## Layout

Flat script layout — there is no installable package and no `[build-system]`. `statlearn` is not importable; notebooks and scripts resolve modules from the working directory. Adding a package later means creating `statlearn/` and restoring a `[build-system]` block.

## Project Status

Early-stage project. `main.py` is the entry point stub. No tests are configured yet.
