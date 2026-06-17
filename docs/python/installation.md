---
icon: lucide/download
---

# Installation

## Requirements

- **Python ≥ 3.10** (3.12+ recommended)
- Runtime dependencies: `pandas` (≥2.0), `numpy` (≥1.24), `pyyaml` (≥6.0), `pydantic` (≥2.0), `plotly` (≥5.0), `jinja2` (≥3.0)

The package is named `resxr` and installs a `resxr` command-line entry point.

## Option 1 — Conda (recommended)

Clone the repository and create the environment from the provided spec file.

=== "Users (minimal install)"

    ```bash
    git clone https://github.com/ResXR/resxr-python-pipeline.git
    cd resxr-python-pipeline
    conda env create -f environment.yml
    conda activate resxr_env
    ```

    Creates the **`resxr_env`** environment and installs the package (non-editable).

=== "Developers (with dev tools)"

    ```bash
    git clone https://github.com/ResXR/resxr-python-pipeline.git
    cd resxr-python-pipeline
    conda env create -f environment-dev.yml
    conda activate resxr_dev
    ```

    Creates the **`resxr_dev`** environment, installs the package in **editable** mode with the `[dev]` extra, and adds `pytest` and `ruff`.

## Option 2 — uv

```bash
git clone https://github.com/ResXR/resxr-python-pipeline.git
cd resxr-python-pipeline
uv sync --all-extras
```

This is the workflow used in CI. It creates a virtual environment from `uv.lock` with all dependencies, including dev extras.

## Verify the install

```bash
resxr --version
```

This prints `ResXR <version>`. You can also run the package as a module:

```bash
python -m resxr --version
```

## Running the tests

The project ships an extensive `pytest` suite (BIDS writers, readers, splitter, validation checks, config, and a full end-to-end integration test).

=== "uv"

    ```bash
    uv run pytest
    ```

=== "conda (dev env)"

    ```bash
    pytest
    ```

!!! tip "Linting"
    The codebase uses [Ruff](https://docs.astral.sh/ruff/) for linting and formatting (line length 100). CI runs `ruff check .` and `ruff format --check` on every pull request, alongside the test suite, on Python 3.10.
