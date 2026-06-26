---
name: uv-python-package
description: Use when starting a new Python project, or when an existing project needs to be converted into a proper installable package. Scaffolds a uv-managed package (src layout, hatch-vcs versioning) with .python-version, .editorconfig, .gitignore and direnv support, matching the conventions used in ardubev16/carpoolerbot.
---

# uv Python package scaffolding

Bootstrap a Python project as a real, installable package managed entirely
by `uv` — never `pip`, `poetry`, or a bare `requirements.txt`.

## Steps

1. **Initialize the package.**

   ```bash
   uv init --package <project-name>
   ```

   This creates a `src/<project_name>/` layout with `__init__.py`,
   `pyproject.toml`, and `.python-version`. Do not use `uv init` without
   `--package` — flat-layout, non-installable projects are not the
   convention here.

2. **Pin the Python version.** Check `.python-version` contains a single
   line with the target minor version (e.g. `3.12`), matching
   `requires-python` in `pyproject.toml`.

3. **Switch versioning to git tags via hatch-vcs.** Edit `pyproject.toml`:

   ```toml
   [project]
   name = "<project-name>"
   dynamic = ["version"]
   description = "..."
   readme = "README.md"
   authors = [{ name = "...", email = "..." }]
   requires-python = ">=3.12"
   dependencies = []

   [dependency-groups]
   dev = ["pytest>=8.0.0", "ruff>=0.14.2", "ty>=0.0.5"]

   [build-system]
   requires = ["hatchling", "hatch-vcs"]
   build-backend = "hatchling.build"

   [tool.hatch.version]
   source = "vcs"

   [tool.hatch.version.raw-options]
   version_scheme = "no-guess-dev"
   ```

   The project must have at least one git tag (or commits since init) for
   `hatch-vcs` to resolve a version; an untagged repo still builds with a
   `0.0.0`-style dev version.

4. **Add an entry point if the project is runnable**, not just a library:

   ```toml
   [project.scripts]
   <project-name> = "<project_name>:main"
   ```

   and define `main()` in `src/<project_name>/__init__.py` or a `__main__`
   module.

5. **Add `.editorconfig`:**

   ```ini
   root = true

   [*]
   end_of_line = lf
   insert_final_newline = true
   indent_style = space
   indent_size = 4

   [*.{md,yaml,yml,json}]
   indent_size = 2
   ```

6. **Add direnv support (`.envrc`)** so contributors get an activated venv
   automatically on `cd`:

   ```bash
   #!/usr/bin/env bash

   set -euo pipefail

   if ! command -v uv &>/dev/null; then
       echo "uv could not be found, please install it first"
       exit
   fi

   uv sync
   source ./.venv/bin/activate
   ```

   Make it executable (`chmod +x .envrc`) and tell the user to run
   `direnv allow` once. This file is intentionally excluded from the
   pre-commit `check-shebang-scripts-are-executable` hook's shebang check
   in the quality-tooling skill, since direnv invokes it by sourcing.

7. **Use `uv`'s generated `.gitignore`** (it already covers `.venv/`,
   `__pycache__/`, build artifacts, etc.) — don't hand-roll one.

8. **Verify the package installs and runs:**

   ```bash
   uv sync
   uv run python -c "import <project_name>"
   ```

## Notes

- Keep `src/<project_name>/` as the only top-level package; tests live in a
  sibling `tests/` directory outside `src/`, not inside the package.
- Don't add a version string by hand anywhere — `hatch-vcs` derives it from
  git tags at build time. Tag releases as `vX.Y.Z`.
- This skill only covers scaffolding. For linting/formatting/type-checking
  and pre-commit/Renovate wiring, follow up with the
  `python-quality-tooling` skill. For CI and Docker, follow up with
  `python-ci-docker`.
