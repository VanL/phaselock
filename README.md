# Phaselock

  [![CI](https://github.com/VanL/phaselock/actions/workflows/test.yml/badge.svg)](https://github.com/VanL/phaselock/actions/workflows/test.yml)
  [![codecov](https://codecov.io/gh/VanL/phaselock/branch/main/graph/badge.svg)](https://codecov.io/gh/VanL/phaselock)
  [![PyPI version](https://badge.fury.io/py/phaselock.svg)](https://badge.fury.io/py/phaselock)
  [![Python versions](https://img.shields.io/pypi/pyversions/phaselock.svg)](https://pypi.org/project/phaselock/)

*A small vendorable helper for advisory file locks and ordered setup phases.
No runtime dependencies.*

```bash
$ uv add phaselock
```

Or vendor the single source file:

```bash
curl -fsSLO https://raw.githubusercontent.com/VanL/phaselock/v1.0/phaselock.py
```

Phaselock exists for code that needs one process to perform file-backed setup
while other processes wait or skip completed work. The main use case is SQLite:
schema creation, PRAGMA setup, migrations, or cache preparation around one
database file. It also works for any local file where cooperating processes can
agree on the same lock path.

## Recommended For

- **SQLite setup coordination.** Run connection, schema, or migration phases
  once around a database path, even when several processes start at the same
  time.
- **Vendored library code.** Copy `phaselock.py` into your project when adding a
  dependency would be worse than carrying a small stdlib-only helper.
- **Local file coordination.** Use `AdvisoryFileLock` when you need a direct
  advisory lock around a cache, index, generated file, or sidecar resource.
- **Cross-platform tools.** Phaselock uses `fcntl.flock` on POSIX and
  `msvcrt.locking` on Windows, with a process-local mutex to serialize threads
  in the same interpreter.

**Good for:** local files, SQLite setup, small tools, vendored libraries  
**Not for:** distributed locks, uncooperative writers, or network filesystem
correctness guarantees

## Features

- **Single file** - `phaselock.py` is the vendorable artifact.
- **No runtime dependencies** - only the Python standard library.
- **Cross-platform advisory locks** - POSIX and Windows lock primitives.
- **Ordered phases** - idempotent actions run once and resume after partial
  failure.
- **Durable completion markers** - extended attributes when available, atomic
  status-file fallback when not.
- **Thread-aware** - a process-local lock closes the gap between OS file locks
  and same-process concurrency.
- **Explicit failure mode** - lock timeouts include useful diagnostics.

## Installation

```bash
# Use as a normal package
uv add phaselock
pip install phaselock

# Or clone for vendoring and tests
git clone git@github.com:VanL/phaselock.git
```

**Requirements:**

- Python 3.10+
- A local filesystem with `fcntl` or `msvcrt` support

## Quick Start

```python
from pathlib import Path

from phaselock import Phase, PhaseLockService

db_path = Path("app.db")
service = PhaseLockService(db_path, strict_marker_locking=True)

result = service.run_phases(
    (
        Phase("connection-v1", lambda: configure_connection_defaults(db_path)),
        Phase("schema-v3", lambda: create_or_migrate_schema(db_path)),
        Phase("optimize-v1", lambda: apply_optimization_pragmas(db_path)),
    )
)

print(result.completed)
print(result.skipped)
```

Each phase action must be idempotent. Phaselock writes the completion marker
only after the action returns successfully. If a later phase fails, the next
run skips the earlier completed phases and resumes at the missing work.

## Plain File Locking

Use `AdvisoryFileLock` when you already have a lock sidecar path and do not
need phase markers.

```python
from phaselock import AdvisoryFileLock

with AdvisoryFileLock("cache.lock", timeout=5.0, retry_delay=0.05):
    rebuild_cache()
```

This is advisory locking. It protects only code paths that take the same lock.
It does not stop another process from opening or modifying the protected file
directly.

## Vendoring

The public vendoring contract is the single `phaselock.py` file from a tagged
release. Copy that file into your project, preserve its license header, and
import from your local module.

```bash
curl -fsSLo your_project/_vendor/phaselock.py \
  https://raw.githubusercontent.com/VanL/phaselock/v1.0/phaselock.py
```

Recommended vendored import:

```python
from your_project._vendor.phaselock import Phase, PhaseLockService
```

See [VENDORING.md](VENDORING.md) for the update policy and the API surface that
is meant to be stable.

## Python API

### `Phase`

```python
Phase(name, action, attr_name=None, attr_value=b"1")
```

A named idempotent setup action. Names must be non-empty and cannot contain NUL
or newlines.

### `PhaseLockService`

```python
PhaseLockService(
    target,
    namespace="user.phaselock",
    lock_suffix=".lock",
    status_suffix=".status",
    timeout=10.0,
    retry_delay=0.05,
    use_xattrs=None,
    strict_marker_locking=None,
)
```

`target` is the resource being protected. The default lock path for `app.db` is
`app.lock`; the default fallback status path is `app.status`.

Use `strict_marker_locking=True` when marker observation must be a
happens-after edge. That is usually the right setting for SQLite setup, because
waiters should trust a completed marker only after the prior owner has released
the setup lock.

### `PhaseRunResult`

`run_phases()` returns:

- `completed`: phase names this process ran
- `skipped`: phase names already marked complete
- `xattrs_available`: whether xattrs were usable
- `lock_path`: the lock sidecar path
- `status_paths`: fallback status files in use

### `AdvisoryFileLock`

```python
AdvisoryFileLock(path, timeout=10.0, retry_delay=0.05)
```

Use it as a context manager or call `acquire()` and `release()` directly.

## How Markers Work

Phaselock prefers extended attributes because they keep completion state on the
target file. When xattrs are missing, disabled, or rejected by the filesystem,
it uses an atomically replaced UTF-8 status sidecar.

The xattr path is an optimization, not a correctness requirement. Phase actions
still need to be idempotent because a process can crash after doing useful work
but before writing the marker.

## Environment Variables

`PHASELOCK_ENABLE_XATTRS` controls xattr probing:

- `auto` or unset: use xattrs when they work
- `1`, `true`, `yes`, `on`, `xattr`, `xattrs`: force xattr probing on
- `0`, `false`, `no`, `off`, `fallback`, `status`, `none`: use status files

## Critical Safety Notes

Advisory locks are cooperation contracts. They do not protect against code that
does not take the lock.

Network filesystems vary. Some support advisory locks well, some only partly,
and some lie under load. Phaselock does not promise distributed lock semantics.

Do not delete lock files as a cleanup strategy. File existence is not lock
ownership. A process can hold an advisory lock on an open file handle while
another process unlinks the path.

## Development

Phaselock uses [`uv`](https://github.com/astral-sh/uv), [`pytest`](https://pytest.org/),
[`ruff`](https://docs.astral.sh/ruff/), and [`mypy`](https://mypy.readthedocs.io/).

```bash
git clone git@github.com:VanL/phaselock.git
cd phaselock
uv sync --all-extras

uv run pytest
PHASELOCK_ENABLE_XATTRS=0 uv run pytest tests/test_phaselock.py
uv run ruff check phaselock.py tests
uv run ruff format --check phaselock.py tests
uv run mypy phaselock.py
uv run python -m build
```

## License

MIT License. See [LICENSE](LICENSE).
