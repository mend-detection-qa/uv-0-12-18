# Probe: uv-pip-json-output

Mend SCA detection probe for uv 0.12.18 `--output-format json` on
`uv pip install` / `uv pip sync`.

## Probe metadata

| Field                 | Value                                    |
|-----------------------|------------------------------------------|
| Pattern               | `uv-pip-json-output`                     |
| PM                    | uv                                       |
| PM version under test | 0.12.18                                  |
| Category              | tree_command                             |
| Status                | untested                                 |
| Generated at          | 2026-09-22T23:31:32Z                     |
| Schema version        | 1.2                                      |

## Feature under test

uv 0.12.18 added `--output-format json` to `uv pip install` and
`uv pip sync`. When this flag is present, the command emits a JSON
array of installed-package records to stdout rather than human-readable
installation progress text.

This is Mend-SCA-relevant because the Unified Agent uses shell-outs
to package-manager CLIs (`tree_command`) to gather installed-package
information. If the UA calls `uv pip install` (now or in a future
UA release), the output format it must parse changes from text to
JSON with this uv version. Probes that verify Mend can still produce
a correct dependency tree after this CLI change belong here.

## Project structure

This probe is in **pip-mode** (not uv-workspace mode):

- `requirements.txt` — the manifest Mend's Pip resolver reads.
- `requirements-pinned.txt` — full pinned set (`pip freeze`
  equivalent) for reproducibility audit.
- `pyproject.toml` — PEP 621 minimal metadata; no
  `[tool.uv.workspace]`, no `[project.dependencies]`. Presence
  of this file alongside `requirements.txt` should not confuse
  Mend into treating the project as a uv-workspace scan.
- `.python-version` — single-line literal `3.11` (see
  Python version detection below).
- `src/` — minimal source stub (not scanned by Mend).
- `expected-tree.json` — ground truth for downstream comparison.

## Dependencies (pip-mode)

Direct (from `requirements.txt`):

- `requests==2.32.3`
- `click==8.1.7`

Transitive (from `requests`):

- `certifi==2024.8.30`
- `charset-normalizer==3.4.0`
- `idna==3.10`
- `urllib3==2.2.3`

`click` has no non-stdlib transitive dependencies.

The tree is intentionally two levels deep so the JSON output from
`uv pip install --output-format json` carries real parent-child
structure and is not trivially a flat list.

## Python version detection

Mend reads Python version from files in PIP-chain precedence order.
`.python-version` has **higher precedence** than `pyproject.toml`'s
`requires-python`. This probe ships `.python-version` containing
`3.11` — Mend will use `3.11` as the Python version for this scan.
`pyproject.toml`'s `requires-python = ">=3.11"` is consistent with
this.

## Mend config

**Bucket B — no `.whitesource`** (dynamic Python detection covers it;
uv tool itself is not in the `install-tool` list and cannot be
pinned via `versioning`).

The `python` key is in the `install-tool` list and could be pinned
if a version-specific Python regression were being tested, but this
pattern is about the uv CLI output format change, not a Python
version regression. Dynamic detection from `.python-version` is
sufficient.

No `whitesource.config` is emitted. `configMode` defaults to `AUTO`.

## What the downstream comparator must check

1. All six packages present in the Mend scan output:
   `requests`, `click`, `certifi`, `charset-normalizer`,
   `idna`, `urllib3`.
2. `requests` reports `certifi`, `charset-normalizer`, `idna`,
   and `urllib3` as its dependencies (second-level).
3. `click` reports no non-stdlib dependencies.
4. All packages have `source = "registry"` (PyPI).
5. No packages from `pyproject.toml` appear as extra dependencies
   (the `pyproject.toml` has no `[project.dependencies]` — Mend
   must not invent deps from it).

## Resolver notes (Mend UA behavior)

Mend's UA scans this project via the **Pip resolver** (not a uv
resolver — there is no separate uv resolver in the UA). The UA
auto-detects the `requirements.txt` manifest and:

1. Runs `pip download -r requirements.txt --no-deps` to get direct
   deps.
2. Runs `pip download -r requirements.txt` to get transitives.
3. Parses downloaded filenames for name + version.
4. If `python.resolveHierarchyTree=true`: creates a venv, runs
   `pip install -r requirements.txt`, then `pipdeptree --json` for
   the hierarchy.

The `uv pip install --output-format json` change affects the UA
only if the UA has been updated to shell out to `uv pip install`
instead of (or in addition to) bare `pip`. This probe captures the
expected tree so the downstream comparator can detect if the UA's
new code path produces different results.

`MEND_SCA_UV_PROJECTS` env var is not set for this probe (it is
only needed when uv-workspace projects are present that Mend should
ignore).

## Source

https://github.com/astral-sh/uv/releases/tag/0.12.18
