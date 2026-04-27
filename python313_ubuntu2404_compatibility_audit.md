# NS Packages — Python 3.13 + Ubuntu 24.04 Compatibility Audit

**Date**: 2026-04-24
**Packages audited**: 14 (from CCI requirements.in + nsconst as transitive dep)
**Target**: Python 3.13 + Ubuntu 24.04 (Noble Numbat)

---

## Summary Table

| Package | Repo | Owner(s) | Verdict |
|---|---|---|---|
| `nssysutil` | [ns-python-sysutil](https://github.com/netSkope/ns-python-sysutil) | CMouliNS, ns-aliang, ns-dgate | **BROKEN** |
| `nsconst` | [ns-python-const](https://github.com/netSkope/ns-python-const) | CMouliNS, ns-csingh, ns-jiew | **Compatible** |
| `nslog` | [ns-python-log](https://github.com/netSkope/ns-python-log) | CMouliNS, ns-svemu | **Needs fixes** |
| `nsdbfusion` | [ns-python-dbfusion](https://github.com/netSkope/ns-python-dbfusion) | ns-rradhakrishnan | **BROKEN** |
| `nsjson` | [ns-python-json](https://github.com/netSkope/ns-python-json) | CMouliNS | **Needs minor fix** |
| `nsconfwatcher` | [ns-python-confwatcher](https://github.com/netSkope/ns-python-confwatcher) | CMouliNS | **Needs 1 fix** |
| `nsmongo` | [ns-python-mongo](https://github.com/netSkope/ns-python-mongo) | ns-cshyi, ns-pschandel | **Needs fixes** |
| `nsnightingale` | [ns-python-nightingale](https://github.com/netSkope/ns-python-nightingale) | CMouliNS, wooseung-sim | **Needs fixes** |
| `nsdocker` | [ns-python-docker](https://github.com/netSkope/ns-python-docker) | CMouliNS, wooseung-sim | **Needs fixes (infra only)** |
| `nsfilewatch` | [ns-python-filewatch](https://github.com/netSkope/ns-python-filewatch) | CMouliNS, ns-csingh | **BROKEN** |
| `nsclustermapping` | [ns-python-clustermapping](https://github.com/netSkope/ns-python-clustermapping) | CMouliNS | **Needs fixes (minor)** |
| `nsconfig` | [ns-python-stagedconfig](https://github.com/netSkope/ns-python-stagedconfig) | CMouliNS, ns-cyang, ns-cyen | **Needs fixes** |
| `nsutil` | **repo not found** | — | **Not assessed** |
| `nswatcher` | **repo not found** | — | **Not assessed** |

**Primary contact**: Chandra Mouli Maddala (CMouliNS) — chandramouli@netskope.com
_(de facto platform owner; appears as primary human committer in 9 of 12 identified repos)_

---

## Verdict Definitions

| Verdict | Meaning |
|---|---|
| **BROKEN** | Hard runtime or install failure on Python 3.13 / Ubuntu 24.04. Cannot be used as-is. |
| **Needs fixes** | Specific code or dependency changes required before Python 3.13 / Ubuntu 24.04 works. |
| **Needs minor fix** | Single small change (e.g. lift a version ceiling). Code itself is already compatible. |
| **Compatible** | Works on Python 3.13 and Ubuntu 24.04 with no changes. |
| **Not assessed** | Repo not found under netSkope org. |

---

## Per-Package Details

---

### 1. `nssysutil` — **BROKEN**

**Repo**: netSkope/ns-python-sysutil
**Owners**: CMouliNS, ns-aliang, ns-dgate
**Bazel python_version**: 3.8
**CI matrix**: 3.8, 3.10 only
**python_requires in wheel**: Not declared

**Blocking issues**:
- `netskope/utility/cryptokey.py`: uses `raw_input()` and `print` as a statement — hard `SyntaxError` on any Python 3.x at parse time
- `netskope/utility/hexdump.py`: uses `xrange()` — `NameError` on Python 3
- `netskope/util/func_util.py`: `from builtins import zip` — relies on the `future` package; breaks if not installed
- Bazel toolchain targets Python 3.8 (EOL Oct 2024, not available on Ubuntu 24.04 via apt)

**Required fixes**:
1. Rewrite `cryptokey.py` with Python 3 syntax (`input()`, `print()`)
2. Replace `xrange` with `range` in `hexdump.py`
3. Remove `from builtins import zip` (not needed in Python 3)
4. Bump `MODULE.bazel` `python_version` to `3.12`
5. Add `python_requires = ">=3.10,<=3.13"` to wheel BUILD rule
6. Update CI matrix to include 3.12 and 3.13

---

### 2. `nsconst` — **Compatible**

**Repo**: netSkope/ns-python-const
**Owners**: CMouliNS, ns-csingh, ns-jiew
**Bazel python_version**: 3.12
**CI matrix**: 3.10, 3.12
**python_requires in wheel**: `>=3.10,<=3.13`

**Notes**:
- Clean, pure-Python code — no Python 2 constructs, no removed stdlib usage
- Uses `six.iteritems()` (unnecessary on Python 3 but not broken)
- CI matrix does not include 3.13, but `python_requires` ceiling allows it

**Recommended (non-blocking)**:
- Remove `six` dependency; replace `six.iteritems(...)` with `.items()`
- Add `3.13` to CI test matrix

---

### 3. `nslog` — **Needs fixes**

**Repo**: netSkope/ns-python-log
**Owners**: CMouliNS, ns-svemu
**Bazel python_version**: 3.12
**CI matrix**: 3.10, 3.12
**python_requires in wheel**: `>=3.10,<3.13` (explicitly excludes 3.13)

**Blocking issues**:
- `logapi.py` line 374: `logging.getLogger(modname).warn(...)` — `Logger.warn()` was removed in Python 3.13; raises `AttributeError`

**Safe (non-blocking) findings**:
- `try: from cStringIO import StringIO` — has correct `except ImportError` fallback to `io.StringIO`
- `try: import SocketServer` — has correct fallback to `socketserver`
- Python 2 dead code branches (`IS_PY3 = False`) — never reached, harmless

**Required fixes**:
1. Replace `Logger.warn(...)` with `Logger.warning(...)` in `logapi.py` line 374 (and any other call sites)
2. Update `python_requires` from `<3.13` to `<3.14` in `BUILD.bazel`
3. Add `3.13` to CI matrix

---

### 4. `nsdbfusion` — **BROKEN**

**Repo**: netSkope/ns-python-dbfusion
**Owners**: ns-rradhakrishnan
**Bazel python_version**: 3.10
**CI matrix**: 3.10 only
**python_requires in wheel**: Not declared

**Blocking issues**:
- `netskope/storage/analytics.py` line 10: `from collections import Iterable` — removed in Python 3.10; raises `ImportError` (already broken on the currently declared target!)
- `gevent==24.2.1`: no Python 3.13 wheels; gevent only added 3.13 support in 24.10.x
- `gevent-inotifyx==0.2`: abandoned C extension (2013); no Python 3.13 wheel; likely fails to build on Ubuntu 24.04 with GCC 13
- `_logger.warn(...)` calls in `analytics.py` and `mongodb.py` (multiple sites) — removed in Python 3.12

**Additional concerns**:
- Widespread use of `future`, `six`, `past` shims (never cleaned up from Python 2 era)
- `numpy` imported in source but absent from Bazel lockfile — runtime dependency gap
- No `python_requires` declared in wheel metadata

**Required fixes**:
1. Replace `from collections import Iterable` with `from collections.abc import Iterable`
2. Upgrade `gevent` to ≥24.10.x for Python 3.13 support
3. Resolve `gevent-inotifyx 0.2` — find a maintained alternative or build for Python 3.13
4. Replace all `_logger.warn(...)` with `_logger.warning(...)`
5. Add `numpy` to Bazel dependency declarations
6. Remove `future`, `six`, `past` shims (Python 3 cleanup)

---

### 5. `nsjson` — **Needs minor fix**

**Repo**: netSkope/ns-python-json
**Owners**: CMouliNS
**Bazel python_version**: 3.12
**CI matrix**: 3.10, 3.12
**python_requires in wheel**: `>=3.10,<3.13` (explicitly excludes 3.13)

**Notes**:
- Source code (`nsjson.py`) is simple, clean Python 3 — no compatibility issues
- The only blocker is the `<3.13` ceiling in `BUILD.bazel`

**Required fixes**:
1. Update `python_requires` from `<3.13` to `<3.14` in `BUILD.bazel`
2. Add `3.13` to CI matrix and verify tests pass

---

### 6. `nsconfwatcher` — **Needs 1 fix**

**Repo**: netSkope/ns-python-confwatcher
**Owners**: CMouliNS
**Bazel python_version**: 3.8
**CI matrix**: 3.8, 3.10
**python_requires in wheel**: Not declared

**Blocking issues**:
- `conf_watcher.py` line 227: `_logger.warn(...)` — removed in Python 3.12; raises `AttributeError`

**Notes**:
- Source code is clean idiomatic Python 3 — `threading`, `os`, `watchdog`
- `watchdog 4.0.2` has full Python 3.13 support

**Required fixes**:
1. Replace `_logger.warn(...)` with `_logger.warning(...)` in `conf_watcher.py`
2. Bump `MODULE.bazel` `python_version` from `3.8` to `3.12`
3. Update CI matrix from `["3.8", "3.10"]` to `["3.10", "3.12"]` or `["3.12", "3.13"]`
4. Add `python_requires` to wheel BUILD rule

---

### 7. `nsmongo` — **Needs fixes**

**Repo**: netSkope/ns-python-mongo
**Owners**: ns-cshyi, ns-pschandel
**Bazel python_version**: 3.10
**CI matrix**: 3.10 only
**python_requires in wheel**: Not declared
**Builder base OS**: ubuntu2004

**Blocking issues**:
- `jaeger-client==4.8.0`: abandoned since 2022; last tested on Python 3.9; no cp313 wheel; ships as sdist-only — functional compatibility with 3.13 unverified

**Additional concerns**:
- `opentracing==2.4.0`: deprecated standard; classifiers only through 3.7 (pure Python — likely works but untested)
- `docker==7.1.0`: classifiers stop at 3.12; no 3.13 wheel confirmed
- `six` usage in `mongo_client.py` and `pymongo_opentracing.py` — works but unnecessary
- Builder images are Ubuntu 20.04-based

**Required fixes**:
1. Replace `jaeger-client` + `opentracing` with OpenTelemetry SDK
2. Verify `docker==7.1.0` on Python 3.13 or upgrade
3. Remove `six` usage
4. Update builder images to Ubuntu 24.04
5. Expand CI matrix to include 3.12 and 3.13

---

### 8. `nsnightingale` — **Needs fixes**

**Repo**: netSkope/ns-python-nightingale
**Owners**: CMouliNS, wooseung-sim
**Bazel python_version**: 3.8
**CI matrix**: 3.8, 3.10
**python_requires in wheel**: Not declared
**Builder base OS**: ubuntu2004

**Blocking issues**:
- `datadog==0.52.1`: classifiers only through Python 3.9; not tested on 3.12/3.13 — functional compatibility unverified
- Bazel toolchain targets Python 3.8 (EOL, unavailable on Ubuntu 24.04 via apt)

**Additional concerns**:
- `from __future__ import absolute_import`, `with_statement` in source — harmless no-ops but signals old code heritage
- `six.iteritems` in test files — unnecessary Python 3 compat code
- Builder images are Ubuntu 20.04-based

**Required fixes**:
1. Verify or upgrade `datadog` for Python 3.13 compatibility
2. Bump `MODULE.bazel` `python_version` from `3.8` to `3.12`
3. Update CI matrix from `["3.8", "3.10"]` to `["3.10", "3.12"]`
4. Update builder images to Ubuntu 24.04
5. Clean up `from __future__` imports and `six.iteritems` in tests

---

### 9. `nsdocker` — **Needs fixes (infra only)**

**Repo**: netSkope/ns-python-docker
**Owners**: CMouliNS, wooseung-sim
**Bazel python_version**: 3.8
**CI matrix**: 3.8, 3.10
**python_requires in wheel**: Not declared
**Builder base OS**: ubuntu2004

**Notes**:
- Source code is clean Python 3 — no runtime compatibility issues
- `from builtins import object/str`, `from __future__ import print_function` — harmless no-ops in Python 3
- No C extensions

**Required fixes (infrastructure only)**:
1. Bump `MODULE.bazel` `python_version` from `3.8` to `3.12`
2. Update CI matrix from `["3.8", "3.10"]` to `["3.10", "3.12"]`
3. Update builder images to Ubuntu 24.04
4. Verify internal deps `nssysutil` and `nslog` are Python 3.13-compatible

---

### 10. `nsfilewatch` — **BROKEN**

**Repo**: netSkope/ns-python-filewatch
**Owners**: CMouliNS, ns-csingh
**Bazel python_version**: 3.12
**CI matrix**: 3.10, 3.12
**python_requires in wheel**: `>=3.10,<3.13` (hard cap)
**Builder base OS**: ubuntu2004

**Blocking issues**:
- `gevent==24.2.1` (exact pin): no Python 3.13 wheels; gevent only added 3.13 support in 24.10.x
- `gevent-inotifyx==0.2` (exact pin): abandoned C extension; no Python 3.13 wheel; almost certainly fails to build on Ubuntu 24.04 with GCC 13
- `python_requires = ">=3.10,<3.13"` hard ceiling in wheel metadata

**Additional concerns**:
- `from imp import reload` dead branch in `watch.py` line 16 — `imp` was removed in Python 3.12; the primary `from importlib import reload` path is correct, but the dead fallback is a code quality issue
- Builder images are Ubuntu 20.04-based

**Required fixes**:
1. Upgrade `gevent` from `24.2.1` to `>=24.10` for Python 3.13 support
2. Resolve `gevent-inotifyx==0.2` — find maintained alternative or build for 3.13 (consider replacing with `watchdog`, already a transitive dep)
3. Update `python_requires` ceiling from `<3.13` to `<3.14` after above
4. Remove dead `from imp import reload` fallback in `watch.py`
5. Update CI matrix and builder images for 3.13 / Ubuntu 24.04

---

### 11. `nsclustermapping` — **Needs fixes (minor)**

**Repo**: netSkope/ns-python-clustermapping
**Owners**: CMouliNS
**Bazel python_version**: 3.10
**CI matrix**: 3.10 only
**python_requires in wheel**: Not declared
**Builder base OS**: ubuntu2004

**Notes**:
- Source code is clean modern Python 3 across all 5 source files — f-strings, idiomatic patterns, no Python 2 constructs
- No C extensions; `requests` and all direct deps are pure Python
- Internal dep `nswatcher` compatibility with 3.13 needs separate verification

**Required fixes**:
1. Add `python_requires = ">=3.10"` to wheel BUILD rule
2. Bump `MODULE.bazel` `python_version` to `3.12`
3. Expand CI matrix from `["3.10"]` to `["3.10", "3.12"]` or `["3.12", "3.13"]`
4. Update builder images to Ubuntu 24.04
5. Verify `nswatcher` (internal dep) is Python 3.13-compatible

---

### 12. `nsconfig` — **Needs fixes**

**Repo**: netSkope/ns-python-stagedconfig
**Owners**: CMouliNS, ns-cyang, ns-cyen
**Bazel python_version**: 3.8
**CI matrix**: 3.8, 3.10
**python_requires in wheel**: Not declared
**Builder base OS**: ubuntu2004

**Blocking issues**:
- `pyinotify==0.9.6` declared in `requirements.in` and `requirements.bzl` but **not imported anywhere in source code** — phantom dependency. `pyinotify` is abandoned since 2015, has no wheels for Python 3.10+, and fails to build from source on Python 3.12+ due to `distutils` removal. Will cause `pip install` / `bazel fetch` to fail.

**Notes**:
- Source code (`staged_config.py`, `file_watch_watchdog.py`) is notably clean modern Python 3 — f-strings, `pathlib.Path`, `from __future__ import annotations`, full type annotations, `watchdog`-based file watching
- `watchdog 4.0.2` has full Python 3.13 support

**Required fixes**:
1. Remove `pyinotify==0.9.6` from `requirements.in` and `requirements.bzl` (unused in code)
2. Bump `MODULE.bazel` `python_version` from `3.8` to `3.12`
3. Update CI matrix from `["3.8", "3.10"]` to `["3.10", "3.12"]`
4. Update builder images to Ubuntu 24.04
5. Verify `nsnightingale` (transitive dep) is Python 3.13-compatible

---

### 13. `nsutil` — **Not assessed**

**Repo**: Not found under netSkope org (no `ns-python-util` repo exists)
**Note**: `nsutil` is the root hub package — it pulls in almost all other ns packages. Its compatibility is the highest-priority gap in this audit. It may be embedded inside a service monorepo rather than having its own `ns-python-*` repo.

---

### 14. `nswatcher` — **Not assessed**

**Repo**: Not found under netSkope org (no `ns-python-watcher` repo exists)
**Note**: Used as a transitive dep by `nsclustermapping`. May be embedded in another repo.

---

## Cross-Cutting Issues (All Repos)

| Issue | Affected repos | Severity |
|---|---|---|
| Builder images based on Ubuntu 20.04 | All 12 identified repos | HIGH — must update to Ubuntu 24.04 images |
| Bazel `python_version` at 3.8 (EOL) | sysutil, confwatcher, nightingale, docker, stagedconfig | HIGH — py3.8 not available on Ubuntu 24.04 via apt |
| No `python_requires` in wheel metadata | sysutil, dbfusion, confwatcher, mongo, nightingale, docker, clustermapping, stagedconfig | MEDIUM — silent breakage risk on wrong Python |
| `six` dependency (py2/3 shim) | All via transitive deps | LOW — works on 3.13 but is unnecessary dead weight |
| CI matrix does not include Python 3.13 | All 12 repos | MEDIUM — no automated 3.13 regression coverage |
| `_logger.warn()` (removed in 3.13) | nslog, nsconfwatcher, nsdbfusion | HIGH — `AttributeError` at runtime on 3.13 |
| `gevent-inotifyx==0.2` (abandoned C ext) | nsfilewatch, nsdbfusion (transitive) | CRITICAL — no 3.13 wheel, likely fails on Ubuntu 24.04 |
| `gevent==24.2.1` (no 3.13 support) | nsfilewatch, nsdbfusion (transitive) | HIGH — upgrade to ≥24.10 required |

---

## Recommended Action Order

### P0 — Blockers (fix before any migration attempt)

1. **nssysutil** — fix Python 2 syntax in `cryptokey.py` and `hexdump.py`
2. **nsdbfusion** — fix `collections.Iterable` import (already broken on 3.10+)
3. **nsfilewatch / nsdbfusion** — resolve `gevent-inotifyx==0.2` (no 3.13 wheel; consider replacing with `watchdog`)
4. **nsfilewatch / nsdbfusion** — upgrade `gevent` to ≥24.10 for Python 3.13 support

### P1 — Fix during migration

5. **nslog / nsconfwatcher / nsdbfusion** — replace `_logger.warn()` with `_logger.warning()`
6. **nsconfig** — remove phantom `pyinotify==0.9.6` dependency
7. **nsmongo** — replace `jaeger-client` + `opentracing` with OpenTelemetry SDK
8. **All repos** — update builder images from Ubuntu 20.04 to Ubuntu 24.04
9. **All repos** — bump Bazel `python_version` to 3.12 where still on 3.8/3.10
10. **nsjson / nslog** — lift `python_requires <3.13` ceiling after code fixes

### P2 — Post-migration cleanup

11. **All repos** — remove `six`, `future`, `past` shims (Python 3-only cleanup)
12. **All repos** — expand CI matrix to include Python 3.13
13. **All repos** — add `python_requires` to all wheel BUILD rules
14. **nsutil / nswatcher** — locate source repos and audit separately
