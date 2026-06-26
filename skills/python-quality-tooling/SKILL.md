---
name: python-quality-tooling
description: Use when setting up or auditing linting, formatting, type checking, pre-commit hooks, or dependency-update automation for a Python project. Configures ruff (lint + format, select=ALL), the ty type checker, a prek-run pre-commit pipeline, and Renovate.
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
of rules. The list above is a starting baseline, not a fixed set: extend it
freely on a per-project basis as real friction shows up, commenting *why*
each addition exists.

Run `uv run ruff check --fix .` and `uv run ruff format .` locally; both
also run via pre-commit and CI.

## 2. ty — type checking

Add `ty` to the dev dependency group (`uv add --dev ty`). No project
config is required by default — `ty check` runs against the package as-is.
ty runs through the `astral-sh/ty-pre-commit` hook configured below, not as
a separate CI step.

## 3. pre-commit (via `prek`)

Run the hooks with [`prek`](https://github.com/j178/prek) — a Rust
reimplementation of `pre-commit` that reads the same
`.pre-commit-config.yaml`, runs noticeably faster, and adds the
`builtin`/`meta` pseudo-repos used below for hooks that would otherwise
need their own venv. Install it with `uvx prek` for one-off runs, or a
standalone install (e.g. your package manager or `cargo install --locked
prek`) if you want it on `PATH` permanently — it isn't a Python dependency.

Write `.pre-commit-config.yaml`:

```yaml
repos:
  - repo: builtin
    hooks:
      - id: check-added-large-files
      - id: check-case-conflict
      - id: check-executables-have-shebangs
      - id: check-json
      - id: check-json5
      - id: check-merge-conflict
      - id: check-shebang-scripts-are-executable
        exclude: ^\.envrc$
      - id: check-symlinks
      - id: check-toml
      - id: check-xml
      - id: check-yaml
        args: [--allow-multiple-documents]
      - id: detect-private-key
      - id: end-of-file-fixer
      - id: fix-byte-order-marker
      - id: mixed-line-ending
        args: [--fix=lf]
      - id: trailing-whitespace

  - repo: meta
    hooks:
      - id: check-useless-excludes

  - repo: https://github.com/codespell-project/codespell
    rev: 2ccb47ff45ad361a21071a7eedda4c37e6ae8c5a # frozen: v2.4.2
    hooks:
      - id: codespell

  - repo: https://github.com/rhysd/actionlint
    rev: 914e7df21a07ef503a81201c76d2b11c789d3fca # frozen: v1.7.12
    hooks:
      - id: actionlint

  - repo: https://github.com/renovatebot/pre-commit-hooks
    rev: 574e0c355d91ac32f4f5377d5d6523942c9a4d76 # frozen: 43.242.2
    hooks:
      - id: renovate-config-validator
        args: [--strict]

  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.15.20
    hooks:
      - id: ruff
        types_or: [python, pyi]
        args: [--fix]
      - id: ruff-format
        types_or: [python, pyi]

  - repo: https://github.com/astral-sh/ty-pre-commit
    rev: v0.0.54
    hooks:
      - id: ty

  - repo: https://github.com/astral-sh/uv-pre-commit
    rev: 0.11.24
    hooks:
      - id: uv-lock

  - repo: https://github.com/gitleaks/gitleaks
    rev: 83d9cd684c87d95d656c1458ef04895a7f1cbd8e # frozen: v8.30.1
    hooks:
      - id: gitleaks

  - repo: https://github.com/shellcheck-py/shellcheck-py
    rev: 745eface02aef23e168a8afb6b5737818efbea95 # frozen: v0.11.0.1
    hooks:
      - id: shellcheck

  - repo: https://github.com/DavidAnson/markdownlint-cli2
    rev: v0.22.1
    hooks:
      - id: markdownlint-cli2

  - repo: https://github.com/hadolint/hadolint
    rev: 57e1618d78fd469a92c1e584e8c9313024656623 # frozen: v2.14.0
    hooks:
      - id: hadolint-docker
```

Notes:
- `repo: builtin` covers the generic hygiene hooks with prek's native
  implementations (no extra venv); `repo: meta` adds prek's own
  self-checks (e.g. flagging unused `exclude` patterns).
- `check-shebang-scripts-are-executable` excludes `.envrc`, since direnv
  sources it rather than executing it.
- `shellcheck` and `hadolint-docker` only bite if the project actually has
  shell scripts or a Dockerfile; `markdownlint-cli2` lints the repo's
  Markdown (including `SKILL.md` files). Drop whichever don't apply.
- Only add the `sqlfluff-fix` hook (`sqlfluff/sqlfluff`) if the project
  has raw SQL files (e.g. Alembic migrations) to lint.
- Pin each hook to a tag (`rev: vX.Y.Z`) or, where Renovate pins digests,
  a frozen commit hash with the tag in a trailing comment
  (`rev: <sha> # frozen: vX.Y.Z`) — never `main`/`master`. Renovate (below)
  keeps both forms current.
- After writing the file, run `prek install` and `prek run --all-files` to
  verify everything passes before committing.

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
- [ ] `uv run ty check` passes
- [ ] `prek run --all-files` passes
- [ ] `renovate.json` validates against the Renovate schema
