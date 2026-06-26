---
name: python-quality-tooling
description: Use when setting up or auditing linting, formatting, type checking, pre-commit hooks, or dependency-update automation for a Python project. Configures ruff (lint + format, select=ALL), the ty type checker, a pre-commit pipeline, and Renovate.
---

# Python quality tooling (ruff, ty, pre-commit, Renovate)

This project's stack is **Astral-only**: `ruff` for linting and formatting,
`ty` for type checking. Do not introduce black, isort, flake8, pylint, or
mypy alongside or instead of them.

## 1. ruff — lint + format

Add to `pyproject.toml` (adjust `target-version` to match
`requires-python`):

```toml
[tool.ruff]
target-version = "py312"
line-length = 120

[tool.ruff.lint]
select = ["ALL"]
ignore = [
    "ANN204", # Missing type annotation for special method
    "D1",     # Missing docstring in public module/class/method/function/package/magic method/nested class/init
    "D203",   # 1 blank line required before class docstring
    "D212",   # Multi-line docstring summary should start at the first line
    "DTZ",    # No naive datetime
    "FIX",    # Line contains FIXME, TODO, XXX, HACK
    "S101",   # Use of assert
    "S311",   # Standard pseudo-random generators not suitable for crypto
    "TD002",  # TODO comment must include an author
    "TRY400", # Use logging.exception() instead of logging.error()
]

[tool.ruff.lint.per-file-ignores]
"tests/**/*.py" = [
    "FBT001",  # Boolean-typed positional argument in function definition
    "FBT002",  # Boolean default positional argument in function definition
    "INP001",  # File is part of an implicit namespace package
    "PLR0913", # Too many arguments in function definition
    "PLR2004", # Magic value used in comparison
]
```

Start from `select = ["ALL"]` always — the default is maximal strictness,
with deviations as explicit, commented ignores rather than an opt-in subset
of rules. Only add a new ignore when a rule is genuinely a false positive
for this codebase's idioms, and comment *why*.

Run `uv run ruff check --fix .` and `uv run ruff format .` locally; both
also run via pre-commit and CI.

## 2. ty — type checking

Add `ty` to the dev dependency group (`uv add --dev ty`). No project
config is required by default — `ty check` runs against the package as-is.
ty does not yet have an official pre-commit hook
(astral-sh/ty#269), so it is invoked directly as a CI step, not via
`.pre-commit-config.yaml` — see the `python-ci-docker` skill.

## 3. pre-commit

Install `pre-commit` (`uv add --dev pre-commit` or rely on it being run via
`uvx pre-commit`) and write `.pre-commit-config.yaml`:

```yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v6.0.0
    hooks:
      - id: check-added-large-files
      - id: check-executables-have-shebangs
      - id: check-json
      - id: check-merge-conflict
      - id: check-shebang-scripts-are-executable
        exclude: ^\.envrc$
      - id: check-symlinks
      - id: check-toml
      - id: check-yaml
      - id: end-of-file-fixer
      - id: mixed-line-ending
        args: [--fix=lf]
      - id: trailing-whitespace

  - repo: https://github.com/codespell-project/codespell
    rev: v2.4.2
    hooks:
      - id: codespell

  - repo: https://github.com/rhysd/actionlint
    rev: v1.7.12
    hooks:
      - id: actionlint

  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.15.20
    hooks:
      - id: ruff
        types_or: [python, pyi]
        args: [--fix]
      - id: ruff-format
        types_or: [python, pyi]

  - repo: https://github.com/astral-sh/uv-pre-commit
    rev: 0.11.24
    hooks:
      - id: uv-lock

  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.30.1
    hooks:
      - id: gitleaks
```

Notes:
- `check-shebang-scripts-are-executable` excludes `.envrc`, since direnv
  sources it rather than executing it.
- Only add the `sqlfluff-fix` hook (`sqlfluff/sqlfluff`) if the project
  has raw SQL files (e.g. Alembic migrations) to lint.
- Pin every hook to a tag (`rev:`), never `main`/`master` — Renovate (below)
  keeps these current.
- After writing the file, run `uv run pre-commit install` and
  `uv run pre-commit run --all-files` to verify everything passes before
  committing.

## 4. Renovate

Add `renovate.json` to keep `uv.lock`, pre-commit hook pins, and GitHub
Actions versions current automatically:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["github>ardubev16/common"]
}
```

If the project pins a Python minor version deliberately (e.g. via
`requires-python` and `.python-version`) and shouldn't be auto-bumped to
the next minor by Renovate, add:

```json
{
  "packageRules": [
    {
      "matchPackageNames": ["python"],
      "matchUpdateTypes": ["minor"],
      "enabled": false
    }
  ]
}
```

## Verification checklist

- [ ] `uv run ruff check .` passes with no unexplained ignores
- [ ] `uv run ruff format --check .` passes
- [ ] `uv run ty check` passes (or is wired into CI, see `python-ci-docker`)
- [ ] `uv run pre-commit run --all-files` passes
- [ ] `renovate.json` validates against the Renovate schema
