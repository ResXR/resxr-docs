---
icon: lucide/download
---

# Installation

## Requirements

- **Python ≥ 3.10** (CI tests Python 3.10 and 3.13)
- Runtime dependencies: `pandas` (≥2.0), `numpy` (≥1.24), `pyyaml` (≥6.0), `pydantic` (≥2.0), `plotly` (≥5.0), `jinja2` (≥3.0)
- **[Git LFS](https://git-lfs.com/)** for the demo data: the demo data (`DATA/`) and the reference output (`output/`) are stored with Git LFS, about 860 MB together. Install Git LFS and run `git lfs install` once before you clone. Without it, the clone holds small pointer files in place of these files: the package still installs, but the demo run fails.

The package is named `resxr` and installs a `resxr` command-line entry point.

## Installing a release

The commands below clone the default branch. To install a specific release, such as the one a paper or dataset names, clone its tag:

```bash
git lfs install                    # once per machine
git clone --branch v0.2.2 https://github.com/ResXR/resxr-python-pipeline.git
```

Then continue with Option 1 or Option 2 from `cd resxr-python-pipeline`. Git reports a "detached HEAD" after cloning a tag; that is expected. The releases are listed on the repository's [releases page](https://github.com/ResXR/resxr-python-pipeline/releases). Release v0.2.2 is also archived on Zenodo ([10.5281/zenodo.23562103](https://doi.org/10.5281/zenodo.23562103)) as a zip file that includes the demo data; in the unpacked folder, run the same environment commands. To cite the pipeline, see [How to cite](index.md#how-to-cite).

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

This creates a virtual environment (`.venv`) from `uv.lock` with all dependencies, including the dev extras. CI runs the same command with `--locked`, which fails if `uv.lock` is out of date. Run the commands on these pages through `uv run` (for example `uv run resxr -c config/pipeline_config.yaml`), or activate `.venv` first.

## Verify the install

```bash
resxr --version
```

This prints `ResXR <version>` (`ResXR 0.2.2` for the v0.2.2 release). You can also run the package as a module:

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
    The codebase uses [Ruff](https://docs.astral.sh/ruff/) for linting and formatting (line length 100). CI runs `ruff check .` and `ruff format --check` on every pull request, alongside the test suite, on Python 3.10 and 3.13.
