# uv

## What is it?

* Python package manager
* Replacement for `poetry`, `pip`, `pipx`, `pyenv`, `virtualenv`, and others
* Developed by Astral (same company that develops `ruff`)
* Written in Rust

## What does it do?

* Installs libraries/tools globally
* Manages python versions
* Manages project dependencies (via `pyproject.toml` file)
  * Maintains a "universal" lock file that keeps dependencies in sync across platforms, operating systems, and python versions
  * Can serve as a drop-in replacement for pip using `uv pip`
* Manages project virtual environments
  * Can work with an existing virtualenv (including Conda) or create a new one
* Runs single-file scripts (dependencies can be specified in `uv`-specific metadata format)
* Introduces the concept of workspaces for projects with multiple inter-dependent packages

## Basic commands

* `uv python list`: list available python versions to install
  * Currently, only `cpython` and `pypy` are supported
  * By default, only versions that are not EoL are shown (i.e., 3.8+)
  * Use the `--only-installed` flag to only show installed versions
* `uv python find`: find a specific installed python version
  * Looks first in `uv`-managed locations, then looks on the system path
* `uv python install`: install a new python version
  * By default, installing python with `uv` only makes it available to `uv run` and in `uv`-managed virtual environments
  * Adding the `--preview` flag also makes the installed python available globally as e.g., `python3.12`
  * Also adding the `--default` flag makes the installed python available as `python` and `python3`
* `uv python pin`: pin the current project to a specific python version

```console
❯ uv python install 3.12 --preview --default
Installed Python 3.12.9 in 1.18s
 + cpython-3.12.9-macos-aarch64-none (python, python3, python3.12)
❯ which python
/Users/jdidion/.local/bin/python
❯ python --version
Python 3.12.9
```

## Dependencies

`uv` is configured via the project's `pyproject.toml` file.

* Python version requirement is set in `project.requires-python`
* Runtime dependencies go in the `project.dependencies` section
* Optional dependencies go in the `project.optional-dependencies` section
  * These are similar to Rust features
* Development dependencies go in the `dependency-groups` section
  * The `dev` group is included by default unless the default groups are changed
  * This is recommended over the legacy `uv`-specific `dev-dependencies` field
* Build-specific dependencies go in `build-system.requires`
* Other [`uv`-specific options](https://docs.astral.sh/uv/reference/settings/#build-constraint-dependencies) are set in the `tool.uv` section
  * `workspace`: specify included sub-packages
    * Dependencies on workspace sub-packages must be define in `tool.uv.sources`
  * `default-groups`: list of default dev dependency groups to instalL

### Commands

* `uv init`: initialize a new project
* `uv add/remove`: add/remove a dependency
* `uv lock`: create/update the lock file
  * If the lockfile exists, the locked dependencies are not changed unless they are no longer compatible with the version spec in `pyproject.toml`
  * If the `--upgrade` flag is provided, then locked dependencies are updated to the latest version that matches the spec
* `uv sync`: update the venv from the lock file
* `uv run`: run a command in the venv
  * Automatically updates syncs the lockfile and venv first
* `uv build`: build a wheel and/or source distribution
* `uv publish`: publish a package to pypi

## Tools

* `uv` can manage python-based tools
* The `uvx` command is similar to `pipx` in that it allows you to install and run tools
  * The difference s that `uvx` installs the tool in a temporary, isolated environment
  * `uvx <tool> install` installs the tool into a permanent folder

```console
$ uvx pycowsay hello from uv
```

## Scripts

* `uv` can run an arbitrary python script
  * If run in a project folder, the project is installed first (unless the `--no-project` flag is specified)
  * If the script has dependencies, they can be specified either on the command line or in a special "metadata" section of the script itself

```console
$ uv run --with numpy example.py
```

```python
# /// script
# dependencies = [
#   "requests<3",
#   "rich",
# ]
# ///

import requests
from rich.pretty import pprint

resp = requests.get("https://peps.python.org/api/peps.json")
data = resp.json()
pprint([(k, v["title"]) for k, v in data.items()][:10])
```

## Interoperation with Conda

* `uv` can install dependencies into an existing virtualenv
  * The active environment, based on the `VIRTUAL_ENV` environment variable
  * Conda, based on the `CONDA_PREFIX` environment variable
  * A `.venv` folder in the current directory (or nearest parent)
* Conda/Mamba/etc do *not* support using `uv` to install python dependencies
* `pixi` [*does* support using `uv`](https://prefix.dev/blog/uv_in_pixi) to install python dependencies
  * It is recommended to use `pyproject.toml` to manage all dependencies
  * Dependencies under `project.dependencies` are python dependencies installed by `uv`
  * Dependencies under `tool.pixi.dependencies` are conda dependencies installed by `pixi`

## Interoperation with Docker

* Base and distroless images are provided
* [https://docs.astral.sh/uv/guides/integration/docker/](details)
* [https://github.com/astral-sh/uv-docker-example](example)

```docker
# Use a Python image with uv pre-installed
FROM ghcr.io/astral-sh/uv:python3.12-bookworm-slim

# Install the project into `/app`
WORKDIR /app

# Enable bytecode compilation
ENV UV_COMPILE_BYTECODE=1

# Copy from the cache instead of linking since it's a mounted volume
ENV UV_LINK_MODE=copy

# Install the project's dependencies using the lockfile and settings
RUN --mount=type=cache,target=/root/.cache/uv \
    --mount=type=bind,source=uv.lock,target=uv.lock \
    --mount=type=bind,source=pyproject.toml,target=pyproject.toml \
    uv sync --frozen --no-install-project --no-dev

# Then, add the rest of the project source code and install it
# Installing separately from its dependencies allows optimal layer caching
ADD . /app
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --frozen --no-dev

# Place executables in the environment at the front of the path
ENV PATH="/app/.venv/bin:$PATH"
```