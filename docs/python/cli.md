---
icon: lucide/terminal
---

# CLI Reference

Installing the package provides the `resxr` command. The same entry point is available as `python -m resxr`.

## Synopsis

```bash
resxr [-c CONFIG] [--dry-run] [-v | -q] [--version]
```

Running `resxr` with no arguments uses the default config path `config/pipeline_config.yaml`.

## Options

| Option | Short | Default | Description |
| ------ | ----- | ------- | ----------- |
| `--config` | `-c` | `config/pipeline_config.yaml` | Path to the pipeline YAML config. |
| `--dry-run` | | off | Validate the config and print the plan; write nothing. |
| `--verbose` | `-v` | off | DEBUG-level logging; also prints a traceback on failure. |
| `--quiet` | `-q` | off | Show only ERROR-level logging. |
| `--version` | | | Print `ResXR <version>` and exit. |

`--quiet` takes precedence over `--verbose` if both are given.

## Examples

```bash
# Run with the default config
resxr

# Run with an explicit config
resxr -c config/my_study.yaml

# Validate the config without processing anything
resxr -c config/my_study.yaml --dry-run

# Verbose run (full debug logging + tracebacks)
resxr -c config/my_study.yaml -v

# Run as a module
python -m resxr -c config/my_study.yaml
```

## Dry run

`--dry-run` loads and validates the configuration, then logs the execution plan — the config path, input directory, output directory, the number of configured session mappings (shown as `0` when you rely on auto-discovery), and the list of enabled tracking systems. No data is loaded and no output is written.

```bash
resxr -c config/pipeline_config.yaml --dry-run
```

## Exit codes

| Code | Meaning |
| ---- | ------- |
| `0` | Success (pipeline completed, or dry run validated OK). |
| `1` | Config file not found, invalid config, or a pipeline error. |
| `130` | Interrupted (`Ctrl-C`). |

!!! tip "One bad session won't fail the run"
    If an individual session errors during processing, it is logged and skipped while the rest continue. The process still exits `0` unless a top-level error occurs. Run with `-v` to see per-session tracebacks.
