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
| `--dry-run` | | off | Check the config as a run does and print the plan; read no session data and write nothing. |
| `--verbose` | `-v` | off | DEBUG-level logging; also shows the traceback of an unexpected error. |
| `--quiet` | `-q` | off | Show only ERROR-level logging: warnings and the run summary are hidden (the summary is shown when no session was processed, since it is then an error). |
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

`--dry-run` reads the configuration and runs the same checks as a run (see [Configuration errors](configuration.md#configuration-errors)), including that `input.data_dir` is an existing folder. If they pass, it logs `Configuration valid: <config path>` and the execution plan: the input directory, the output directory, the number of configured session mappings (shown as `0` when you rely on auto-discovery), and the systems set to `enabled: true` under `systems`. No session data is loaded and no output is written. Failures are reported as in a run: a configuration error as `Configuration error in <config path>: ...`, any other failure as `Pipeline failed: ...` (with its traceback under `-v`), both with exit code `1`; `Ctrl-C` gives exit code `130`.

```bash
resxr -c config/pipeline_config.yaml --dry-run
```

## Run summary

Every run ends with a summary: one line with the number of sessions found, processed and skipped, then one line per session, with the reason for a skipped session and the checks that did not run on a processed one ([Checks not run](validation.md#checks-not-run)):

```
Run summary: <n> session(s) found, <p> processed, <s> skipped.
  processed <source_dir> (sub-<id> ses-<label>): <w> warning(s), <e> error(s)
  processed <source_dir> (sub-<id> ses-<label>): <w> warning(s), <e> error(s); checks not run: <check> on <stream>: <reason>
  skipped <source_dir> (sub-<id> ses-<label>): <reason>
```

The warning and error counts are the WARNING and ERROR messages logged while that session was processed. The summary is logged at INFO when every session was processed and every check ran, at WARNING when a session was skipped or a check did not run, and at ERROR when no session was processed. `Pipeline complete. Output at: <bids_root>` follows when at least one session was processed. The same information is returned by [`resxr.run()`](api.md#run).

## Exit codes

| Code | Meaning |
| ---- | ------- |
| `0` | Every session found was processed, or the dry run found the configuration valid. |
| `1` | A session was skipped, no session was found, the config file was not found, the configuration is invalid, or an error outside the sessions stopped the run or the dry run (`Pipeline failed: ...`). |
| `130` | Interrupted (`Ctrl-C`), also during `--dry-run`. In a run, the output the interrupted session had written is removed first. |

!!! tip "One bad session won't stop the run"
    A session that cannot be processed is logged as `Failed to process <source_dir>: <reason>` and skipped, and the run continues with the others; what the run had written for it is removed (see [Skipped sessions](pipeline-stages.md#skipped-sessions)). An unexpected error inside a session (one that is not a `ResXRError`) is handled the same way, with the reason `unexpected error: <type>: <message>`; run with `-v` to see its traceback. The run then exits `1`.
