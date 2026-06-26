# python-skills

A collection of Claude Code skills that scaffold Python projects using the
opinionated, [Astral](https://astral.sh)-centric toolchain used across
[ardubev16](https://github.com/ardubev16) repositories (see
[carpoolerbot](https://github.com/ardubev16/carpoolerbot) as the reference
implementation).

## Conventions encoded by these skills

- **Packaging**: every project is a real installable package, created with
  `uv init --package` (src layout), versioned from git tags via `hatch-vcs`.
- **Tooling**: [`uv`](https://docs.astral.sh/uv/) for dependency/environment
  management, [`ruff`](https://docs.astral.sh/ruff/) for linting and
  formatting, [`ty`](https://docs.astral.sh/ty/) for type checking. No pip,
  poetry, black, flake8, isort or mypy.
- **Quality gates**: `pre-commit` runs generic hygiene hooks plus codespell,
  actionlint, gitleaks, ruff and `uv lock` checks. Renovate keeps
  dependencies (and pre-commit hook pins) up to date automatically.
- **CI/CD**: a GitHub Actions workflow installs `uv`, then runs `ty check`
  and `pytest` on every push/PR. Releases are tag-triggered and build a
  multi-stage, digest-pinned Docker image.

## Skills

| Skill | Use when |
|---|---|
| [`uv-python-package`](./uv-python-package/SKILL.md) | Starting a brand-new Python project/package. |
| [`python-quality-tooling`](./python-quality-tooling/SKILL.md) | Wiring up ruff, ty, pre-commit and Renovate. |
| [`python-ci-docker`](./python-ci-docker/SKILL.md) | Adding GitHub Actions CI and a Dockerfile. |

These skills are designed to be applied in order, but each is self-contained
and can be invoked on its own against an existing project.
