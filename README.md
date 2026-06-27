# python-skills

A collection of Claude Code skills that scaffold Python projects using a
personal, opinionated, [Astral](https://astral.sh)-centric toolchain.

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
  multi-stage Docker image.

## Skills

| Skill | Use when |
|---|---|
| [`uv-python-package`](./skills/uv-python-package/SKILL.md) | Starting a brand-new Python project/package. |
| [`python-quality-tooling`](./skills/python-quality-tooling/SKILL.md) | Wiring up ruff, ty, pre-commit and Renovate. |
| [`python-ci-docker`](./skills/python-ci-docker/SKILL.md) | Adding GitHub Actions CI and a Dockerfile. |
| [`python-library-choices`](./skills/python-library-choices/SKILL.md) | Picking a library for a given use case (web, HTTP, CLI, DB, logging, etc.). |

These skills are designed to be applied in order, but each is self-contained
and can be invoked on its own against an existing project.

## Installing as a plugin

This repo doubles as a Claude Code plugin marketplace containing a single
plugin. Add the marketplace, then install the plugin:

```
/plugin marketplace add ardubev16/python-skills
/plugin install python-skills@python-skills
```

or, for local development, run Claude Code with `--plugin-dir <path-to-this-repo>`.
Skills are then invoked as `/python-skills:<skill-name>`.
