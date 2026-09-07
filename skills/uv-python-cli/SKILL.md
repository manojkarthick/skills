---
name: uv-python-cli
description: Build, package, and release installable Python CLIs with uv. Use when scaffolding a new CLI, migrating an existing pure-Python CLI to uv_build, configuring project.scripts or a src layout, standardizing on Ruff, ty, and pytest, verifying uv tool installation, or publishing the CLI to a package index.
---

# Build a Python CLI with uv

Create a packaged application that runs in its project with `uv run` and installs as an isolated command with `uv tool install`.

Match the workflow to the request:

- New CLI: establish its names, then initialize it.
- Existing project: inspect it, skip initialization, and migrate supported projects to uv_build, Ruff, ty, and pytest.
- Release: skip scaffolding and validate metadata, tests, artifacts, and installation before publishing.

Use `uv_build` for pure Python and replace any other existing backend. Preserve or select another backend only when uv_build is unsupported, such as for extension modules requiring Maturin or scikit-build-core. When migrating, replace `[build-system]` with a bounded, installed-version-compatible `uv_build` requirement and remove obsolete backend configuration.

## Establish the names

For a new project, determine these separately, using conventional defaults when safe:

- Distribution name: published to a package index; default to the project directory name.
- Import package: valid Python identifier under `src/`; default to the normalized distribution name with dots and hyphens replaced by underscores.
- Command name: executable users type; default to the distribution name.
- Minimum supported Python version.
- Runtime dependencies; assume none if unmentioned.
- Development tools: Ruff, ty, and pytest.

A minimum Python version is required. If it is missing, ask for it and wait before initializing the project; do not substitute the agent's preferred version or the machine's current version. Do not impose naming suffixes such as `-tool`.

## Initialize the application

Use uv's native build backend explicitly so the command remains clear across uv versions:

```bash
uv init --app --build-backend uv --python <minimum-python> --name <distribution-name> <project-directory>
cd <project-directory>
```

Use `.` when initializing an empty current directory. The explicit backend makes the app distributable and creates a `src/` layout. Do not add `__main__.py` unless the user separately requests `python -m` support.

If the requested import package differs from uv_build's normalized distribution name, rename the generated module directory and declare it explicitly:

```toml
[tool.uv.build-backend]
module-name = "<import_package>"
```

Confirm this shape:

```text
.
├── .gitignore
├── .python-version
├── README.md
├── pyproject.toml
├── src/<import_package>/__init__.py
└── uv.lock  # created by the first lock, sync, or run
```

## Pin Python deliberately

Use the minimum version supplied by the user in both places while preserving their different meanings:

- `.python-version` selects the development interpreter. Set it with `uv python pin <minimum-python>` and commit it so contributors share the pin.
- `project.requires-python` declares the versions supported by installed users. Confirm that `uv init` produced `requires-python = ">=<minimum-python>"`.

Prefer a portable major/minor pin over `uv python pin --resolved`, whose path is machine-specific.

## Define the command

Put command behavior in `src/<import_package>/cli.py`:

```python
import argparse
from typing import Optional, Sequence


def main(argv: Optional[Sequence[str]] = None) -> int:
    """Run the command-line application."""
    parser = argparse.ArgumentParser()
    parser.parse_args(argv)
    print("Hello")
    return 0
```

Expose the callable in `pyproject.toml`:

```toml
[project.scripts]
<command-name> = "<import_package>.cli:main"
```

The value is `importable.module:callable`, not a filesystem path. A build system is required for the entry point to be installed. Remove the generated `main` from `__init__.py` if it is no longer used. As the CLI grows, separate argument parsing from domain logic so both remain testable.

Add at least one behavior test so a fresh scaffold has a meaningful passing test suite:

```python
from <import_package>.cli import main


def test_main(capsys) -> None:
    assert main([]) == 0
    assert capsys.readouterr().out == "Hello\n"
```

## Manage dependencies

Let uv update `pyproject.toml` and `uv.lock` together, using Ruff, ty, and pytest as the quality stack:

```bash
uv add <runtime-package>
uv add --dev pytest ruff ty
```

For an existing project, remove dependencies and configuration for quality tools that these replace.

Runtime requirements belong in `project.dependencies`. Development-only tools belong in `[dependency-groups].dev` and are not published as package requirements. Use `uv add --group <group> <package>` only when separate groups such as `test` and `lint` add value.

Configure Ruff with the following baseline, but treat the rule selection as a starting point that can change with project needs:

```toml
[tool.ruff]
src = ["src"]

[tool.ruff.lint]
select = ["E", "F", "I", "UP"]
```

Do not set `tool.ruff.target-version`; keep Python compatibility centralized in `project.requires-python`.

```bash
uv sync
uv lock --check
```

Commit `pyproject.toml` and `uv.lock`. In CI, use `uv sync --locked` so a stale lockfile fails instead of changing silently.

## Run the quality gate

Apply formatting and safe lint fixes while developing:

```bash
uv run ruff format .
uv run ruff check --fix .
```

Then run the complete non-mutating gate:

```bash
uv run ruff format --check .
uv run ruff check .
uv run ty check src
uv run pytest
```

Address errors rather than broadly suppressing them. If a formatter or lint fix changes code, rerun the complete gate. Require it to pass before installation verification.

## Verify both usage paths

Exercise the project environment and then the installed-tool boundary:

```bash
uv run <command-name> --help
uv tool install --editable .
<command-name> --help
```

Editable installation reflects source changes immediately. Before release, test an ordinary local installation:

```bash
uv tool install . --force
<command-name> --help
```

After publishing to a package index, users install the distribution and invoke the command:

```bash
uv tool install <distribution-name>
<command-name> --help
```

If uv reports that its executable directory is missing from `PATH`, suggest `uv tool update-shell` and restarting the shell.

## Build and publish when requested

Ensure the metadata has a useful description, README, version, and license information. Then:

```bash
uv version <version>
uv build --no-sources
```

Inspect the wheel and source distribution in `dist/`. Verify the built wheel with `uv tool install <wheel-path> --force`, followed by `<command-name> --help`.

Publish to PyPI or another configured index:

```bash
uv publish
```

Prefer trusted publishing from GitHub Actions for public projects so a long-lived PyPI token is not stored as a secret. A GitHub repository alone does not make `uv tool install <distribution-name>` resolve from PyPI; publish the distribution to the selected package index too.
