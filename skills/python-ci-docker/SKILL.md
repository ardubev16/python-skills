---
name: python-ci-docker
description: Use when adding continuous integration or containerization to a Python project managed with uv. Creates GitHub Actions workflows that call the shared reusable workflows in ardubev16/common to run prek (ruff, ty, etc.) and pytest on push/PR, and a multi-stage Dockerfile based on the official uv Docker example.
---

# CI and Docker for uv-managed Python projects

Assumes the project already follows `uv-python-package` (installable
package, src layout, hatch-vcs versioning) and `python-quality-tooling`
(ruff + ty configured).

CI is wired up as thin callers of the reusable workflows in
[`ardubev16/common`](https://github.com/ardubev16/common/tree/main/.github/workflows),
so each project just selects which shared jobs it needs instead of
duplicating their steps.

## 1. Test workflow — `.github/workflows/test.yaml`

```yaml
name: Test

on:
  push:
    branches:
      - main
  pull_request:

jobs:
  lint:
    uses: ardubev16/common/.github/workflows/prek.yaml@main

  test:
    uses: ardubev16/common/.github/workflows/python-test.yaml@main
```

`prek.yaml` runs the full `.pre-commit-config.yaml` pipeline (ruff, ty,
codespell, actionlint, gitleaks, etc. — see `python-quality-tooling`).
`python-test.yaml` installs uv, sets up the Python version pinned in
`.python-version` (override via the `python-version` input), runs
`uv sync --dev`, then `uv run pytest`.

## 2. Release workflow — `.github/workflows/release.yaml` (optional)

Only add this if the project ships a container image. Trigger on
version tags, build and push to GHCR via `docker-build.yaml`, then cut a
GitHub release with auto-generated notes:

```yaml
name: Release

on:
  push:
    tags:
      - "v*"

permissions:
  contents: write
  packages: write
  id-token: write

jobs:
  build-and-push:
    uses: ardubev16/common/.github/workflows/docker-build.yaml@main
    with:
      tags: type=ref,event=tag

  release:
    runs-on: ubuntu-latest
    needs: build-and-push
    steps:
      - name: Create GitHub release
        uses: softprops/action-gh-release@v3
        with:
          tag_name: ${{ github.ref_name }}
          generate_release_notes: true
```

`docker-build.yaml` defaults `image-name` to the repository name and
`dockerfile`/`context` to `Dockerfile`/`.` — only pass `with:` overrides
when a project's layout differs.

### Optional: build + clean up PR images

To also publish a `pr-<number>` tagged image on pull requests and remove
it once the PR closes, add:

```yaml
name: PR Image

on:
  pull_request:
    types: [opened, synchronize, closed]

jobs:
  build:
    if: github.event.action != 'closed'
    uses: ardubev16/common/.github/workflows/docker-build.yaml@main
    with:
      tags: type=ref,event=pr

  clean:
    if: github.event.action == 'closed'
    uses: ardubev16/common/.github/workflows/docker-clean-pr.yaml@main
    with:
      pr-number: ${{ github.event.pull_request.number }}
```

## 3. Dockerfile

Based on the official
[uv Docker example](https://github.com/astral-sh/uv-docker-example/blob/main/multistage.Dockerfile).
Two stages: build the venv with `uv sync`, then copy only the venv (plus
whatever runtime files the app needs) into a clean base image.

```dockerfile
# From example at: https://github.com/astral-sh/uv-docker-example/blob/main/multistage.Dockerfile

# Build app dependencies
FROM python:3.12-slim-bookworm AS builder
WORKDIR /app

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
    # Needed by hatch-vcs to read git history for the version
    git \
    && rm -rf /var/lib/apt/lists/*

ENV UV_COMPILE_BYTECODE=1 UV_LINK_MODE=copy
COPY --from=ghcr.io/astral-sh/uv:latest /uv /bin/

RUN --mount=type=cache,target=/root/.cache/uv \
    --mount=type=bind,source=uv.lock,target=uv.lock \
    --mount=type=bind,source=pyproject.toml,target=pyproject.toml \
    --mount=type=bind,source=.python-version,target=.python-version \
    SETUPTOOLS_SCM_PRETEND_VERSION_FOR_<PROJECT_NAME_UPPER>=0 \
    uv sync --locked --no-dev --no-install-project --no-editable

# Build app
COPY . .
RUN --mount=type=cache,target=/root/.cache/uv \
    --mount=type=bind,source=.git,target=.git \
    # NOTE: reinstall-package is needed to calculate the correct version from git tags
    uv sync --locked --no-dev --no-editable --reinstall-package=<project-name>


# Copy app to runtime stage
FROM python:3.12-slim-bookworm
WORKDIR /app

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
    dumb-init=1.2.5-2 \
    && rm -rf /var/lib/apt/lists/*

COPY --from=builder /app/.venv /app/.venv

# Place executables in the environment at the front of the path
ENV PATH="/app/.venv/bin:$PATH"

ENTRYPOINT ["/usr/bin/dumb-init", "--"]
CMD ["<project-name>"]
```

Adjust per project:
- Add any extra `apt-get install` packages both stages need (e.g.
  `libpq-dev` for `psycopg2`) — keep build-only deps (`build-essential`,
  `git`) in the builder stage only.
- Copy any extra runtime files the app needs (e.g. `alembic.ini` and a
  `migrations/` directory) from the builder stage alongside the venv.
- The `SETUPTOOLS_SCM_PRETEND_VERSION_FOR_*` env var on the
  dependency-only sync step prevents `hatch-vcs` from failing before
  `.git` is available in that layer; it's safe because that step doesn't
  build the project itself (`--no-install-project`).
- Use plain tags for the base images (`python:3.12-slim-bookworm`,
  `ghcr.io/astral-sh/uv:latest` or a specific version) — don't hand-resolve
  digests. Renovate pins and updates image versions on its own once it's
  configured for the project.
- Add a `.dockerignore` with at least `**/__pycache__`.

## Verification checklist

- [ ] `prek run --all-files` and `uv run pytest` both pass locally before
      relying on the workflow
- [ ] `docker build .` succeeds and the resulting image runs the app
