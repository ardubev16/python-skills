---
name: python-ci-docker
description: Use when adding continuous integration or containerization to a Python project managed with uv. Creates a GitHub Actions workflow that runs prek (ruff, ty, etc.) and pytest on push/PR, and a multi-stage Dockerfile based on the official uv Docker example.
---

# CI and Docker for uv-managed Python projects

Assumes the project already follows `uv-python-package` (installable
package, src layout, hatch-vcs versioning) and `python-quality-tooling`
(ruff + ty configured).

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
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v7

      - name: Run prek
        uses: j178/prek-action@v2

  test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v7

      - name: Install uv
        uses: astral-sh/setup-uv@v8

      - name: Set up Python
        run: uv python install

      - name: Install dependencies
        run: uv sync --dev

      - name: Run tests
        run: uv run pytest
```

Use plain version tags for `uses:` (e.g. `actions/checkout@v7`) — no need
to hand-resolve commit SHAs. Renovate (configured by the
`python-quality-tooling` skill) tracks and bumps these versions on its own.

The `lint` job runs the full `.pre-commit-config.yaml` pipeline (ruff, ty,
codespell, actionlint, gitleaks, etc. — see `python-quality-tooling`) via
the official `j178/prek-action`, so ty checking happens there instead of as
a separate step. `uv python install` reads `.python-version`, so CI always
matches the locally pinned interpreter. `uv sync --dev` installs the dev
dependency group (pytest, ruff, ty) on top of the locked `uv.lock`.

## 2. Release workflow — `.github/workflows/release.yaml` (optional)

Only add this if the project ships a container image. Trigger on
version tags, build and push to GHCR, then cut a GitHub release with
auto-generated notes:

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
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v7

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v4

      - name: Login to GHCR
        uses: docker/login-action@v4
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Docker image metadata
        id: meta
        uses: docker/metadata-action@v6
        with:
          images: ghcr.io/${{ github.repository }}
          tags: type=ref,event=tag

      - name: Build and push Docker image
        uses: docker/build-push-action@v7
        with:
          context: .
          push: true
          labels: ${{ steps.meta.outputs.labels }}
          tags: ${{ steps.meta.outputs.tags }}

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

As above, use plain version tags for `uses:` — Renovate keeps them current.

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
