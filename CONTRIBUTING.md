# Contributing

Contributions are welcome.

This project is intentionally small and focused: an async client for S3-compatible object storage. Keep changes simple, explicit, and scoped.

## Issues

Open an issue for:
- bugs
- unclear behavior
- missing documentation
- small feature requests
- compatibility issues with S3-compatible providers

Include:
- Python version
- package version or commit hash
- storage provider (if relevant)
- minimal reproduction
- expected vs actual behavior

## Pull requests

Before opening a PR, ensure all checks pass:

uv sync --dev
uv run ruff check .
uv run ruff format --check .
uv run pyright
uv run pytest
uv run pre-commit run --all-files

Guidelines:
- keep PRs focused
- avoid mixing refactor + behavior changes
- prefer minimal diffs
- preserve runtime behavior unless explicitly intended

## Development

Setup:

uv sync --dev

Run tests:

uv run pytest

Run all checks:

uv run ruff check .
uv run ruff format --check .
uv run pyright
uv run pytest

Install hooks:

uv run pre-commit install

## Code style

- modern Python (3.12+)
- explicit > clever
- small, stable public API
- absolute imports only
- typed interfaces
- minimal dependencies

## Compatibility

- Python 3.12+
- S3-compatible APIs (not AWS-only)

Provider-specific quirks should be documented.
