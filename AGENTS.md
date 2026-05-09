# AGENTS.md

## Build System

- **Build firmware**: `scons` (SCons, not make/cmake)
- **Full test pipeline**: `./test.sh` (runs: `source ./setup.sh && scons && ruff check . && pytest`)
- **Individual steps**:
  - `source ./setup.sh` - creates `.venv` via `uv`, installs deps including `opendbc`
  - `ruff check .` - lint (configured in `pyproject.toml`, line-length 160)
  - `pytest` - runs tests (ignores `tests/hitl` and `tests/som` by default)
  - `scons` - build all firmware targets (panda_h7, panda_jungle_h7, body_h7)

## Firmware Variants

Three distinct firmware targets built from `board/`:
- `panda_h7` - main panda board (CAN FD)
- `panda_jungle_h7` - panda jungle (debug harness)
- `body_h7` - panda body

Each has separate entrypoints and flash procedures.

## Key Constraints

- **Compiler strictness**: Firmware built with `-Wall -Wextra -Wstrict-prototypes -Werror`
- **Python version**: 3.11–3.12 only (macOS breaks on 3.13 due to pycapnp from opendbc)
- **Safety logic**: External, lives in `opendbc` repo - don't modify there expecting CI to catch it

## Hardware Testing

- `tests/hitl/` - hardware-in-the-loop tests requiring actual panda hardware
- `tests/hitl/conftest.py` auto-discovers connected pandas and sets up fixtures
- Set `NO_JUNGLE=1` to skip jungle-dependent tests
- These tests are excluded from default `pytest` run via `pyproject.toml`

## Flashing

```bash
cd board/
./flash.py        # flash application
./recover.py      # flash bootstub (for bricked devices)
```

Troubleshooting: green LED on + won't flash → `recover.py`. Fast blinking green → `flash.py`. LED off + invisible → DFU mode (need panda paw).

## Python Package Structure

Package is `panda` but source is at root (setuptools package-dir mapping). Import path:
```python
from panda import Panda  # works
```
NOT `from python import ...`

## Dependencies

- **opendbc**: External repo, loaded via `git+https://...@master`. Provides safety logic and CAN structs.
- **udev rules**: Required on Linux for USB access to panda. See README for rules file.

## CI

- GitHub Actions: `test.yaml` - runs `./test.sh` on macOS + Ubuntu, builds on Ubuntu
- Jenkins: hardware testing pipeline (runs hitl tests on real hardware via SSH)

## Code Rigor

- MISRA C:2012 checking via cppcheck addon (see `tests/misra/`)
- Mutation testing on MISRA coverage: `tests/misra/test_mutation.py`
- Safety logic verified by opendbc unit tests (external)
