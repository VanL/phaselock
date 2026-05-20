# Vendoring Phaselock

`phaselock.py` is the vendorable artifact. It is intentionally a single
stdlib-only file.

## Copying

Copy from a tagged release:

```bash
curl -fsSLo your_project/_vendor/phaselock.py \
  https://raw.githubusercontent.com/VanL/phaselock/v1.0/phaselock.py
```

Use a local import wrapper if you want to hide the vendored path from your
application code:

```python
from your_project._vendor.phaselock import Phase, PhaseLockService
```

Keep the copyright and SPDX header at the top of the file when vendoring it.

## Stable API

The stable public API is:

- `AdvisoryFileLock`
- `PHASELOCK_ENABLE_XATTRS`
- `Phase`
- `PhaseLockService`
- `PhaseLockTimeout`
- `PhaseLockUnavailable`
- `PhaseRunResult`
- `__version__`

Names starting with `_` are private implementation details. They can change in
minor releases.

## Updating

Replace the vendored file, then run your lock and setup tests with xattrs both
enabled and disabled:

```bash
pytest
PHASELOCK_ENABLE_XATTRS=0 pytest
```

If you rely on SQLite setup markers as a barrier, keep
`strict_marker_locking=True`.

## Local Changes

Prefer configuring `PhaseLockService` over editing the vendored file. The most
common project-specific settings are:

- `namespace`
- `lock_suffix`
- `status_suffix`
- `timeout`
- `retry_delay`
- `use_xattrs`
- `strict_marker_locking`

If you must patch the vendored file, keep a short note next to it that explains
the local delta. That makes future updates much cheaper.
