# CCI Repository — Python 3.12 + Ubuntu 24.04 Compatibility Audit

**Date**: 2026-05-08
**Scope**: `cci` repo (`nsappinfo` service + 14 NS upstream packages)
**Target runtime**: Python 3.12.3 (Ubuntu 24.04 system Python)
**Target OS**: Ubuntu 24.04 LTS (Noble Numbat) — glibc 2.39, OpenSSL 3.x, GCC 13
**Current runtime**: Python 3.10.16 (conda env), CI images: Ubuntu 16.04/20.04

---

## Executive Summary & Go/No-Go Recommendation

**Recommendation: GO with conditions**

Python 3.12 is the **default system Python** for Ubuntu 24.04. No deadsnakes PPA or custom builds required. The migration is viable but requires a **focused remediation sprint** addressing:

1. **This repo (`nsappinfo`) has 5 direct code-level issues** — Python 2 compat shims (`basestring`, `from builtins`, `ConfigParser` try/except with `six`), deprecated `Logger.warn()` calls, and `SQLAlchemy 1.3.20` which does not support Python 3.12.
2. **3 critical dependency blockers** — `SQLAlchemy==1.3.20` (no 3.12 support), `pymongo==3.11.1` (EOL, no 3.12 wheels), `jaeger-client==4.8.0` (abandoned).
3. **NS upstream packages** have pre-existing Python 2 legacy issues (not 3.12 regressions) that must be fixed before the packages can install on 3.12.

**Key runtime facts at Python 3.12:**
- `gevent 24.2.1` ships `cp312` wheels — not a blocker.
- `Logger.warn()` is deprecated but functional in 3.12.
- Dead batteries (`cgi`, `pipes`, `telnetlib`) are deprecated but still present in 3.12.
- `distutils` is **removed** in 3.12 — this affects `setup.py` and some transitive deps.

---

## Part 1: This Repository (`nsappinfo`) — Direct Code & Dependency Audit

### 1.1 Repository Configuration

| Item | Current | Required for 3.12 |
|------|---------|-------------------|
| Conda env Python | `3.10.16` (`app-env-py310.yaml`) | `3.12.x` |
| `tox.ini` envlist | `py310` | `py312` |
| `Makefile` PYTHON_VERSION | `py310` | `py312` |
| `setup.py` `python_requires` | Not declared | Add `>=3.10,<3.13` |
| `setup.cfg` | pycodestyle config only | No change needed |
| Drone CI python_image | `python:3.8` | `python:3.12` |
| Drone CI builder images | `builder:v31` (Ubuntu 16.04), `builder-2004:v11` | Need Ubuntu 24.04 builder |
| GitHub Actions | Miniforge + conda, `py310` env | Needs `py312` env yaml |

### 1.2 Code-Level Issues Found in `nsappinfo/src/`

| File | Line | Issue | Severity | Fix |
|------|------|-------|----------|-----|
| `svc/routes/mongo_update.py` | 1 | `from past.builtins import basestring` | **HIGH** | Replace with `str` (Python 3 equivalent) |
| `svc/routes/mongo_update.py` | 2 | `from builtins import object` | LOW | Remove (no-op in Python 3) |
| `svc/routes/mongo_update.py` | 54 | `isinstance(apps_string, basestring)` | **HIGH** | Replace with `isinstance(apps_string, str)` |
| `run_config.py` | 5 | `from __future__ import print_function` | LOW | Remove (no-op in Python 3) |
| `run_config.py` | 6 | `from builtins import str` | LOW | Remove (no-op in Python 3) |
| `run_config.py` | 7 | `from builtins import object` | LOW | Remove (no-op in Python 3) |
| `config/meta_manager.py` | 13-15 | `try: import ConfigParser ... except: from six.moves import configparser` | **MEDIUM** | Replace with `import configparser` (stdlib in Python 3) |
| `svc/routes/install.py` | 12-14 | `try: import ConfigParser ... except: from six.moves import configparser` | **MEDIUM** | Replace with `import configparser` (stdlib in Python 3) |
| `cci_calc.py` | 51 | `self._logger.warn(string)` | Technical Debt | Replace with `.warning()` |
| `app_index/mongodb_loader.py` | 133 | `_logger.warn(...)` | Technical Debt | Replace with `.warning()` |
| `database/org_info.py` | 5, 12 | `from sqlalchemy.ext.declarative import declarative_base` | **HIGH** | Move to `from sqlalchemy.orm import DeclarativeBase` (requires SQLAlchemy ≥1.4) |
| `config/meta_manager.py` | 20, 25 | `from sqlalchemy.ext.declarative import declarative_base` | **HIGH** | Same as above |
| `svc/routes/install.py` | 17, 22 | `from sqlalchemy.ext.declarative import declarative_base` | **HIGH** | Same as above |
| `setup.py` | 14 | `del os.link` (comment: "hack for distutils") | **MEDIUM** | Verify still needed; `distutils` removed in 3.12, but `setuptools` provides the shim |

**Note**: The `try: import ConfigParser` blocks work at runtime (the `except` branch fires on Python 3), but they rely on `six` which is unnecessary baggage. On Python 3.12 these paths execute the `six.moves.configparser` fallback, which ultimately resolves to `configparser` — functionally safe but should be cleaned up.

### 1.3 Direct Dependency Compatibility (from `requirements.in`)

#### CRITICAL — Will not install or run on Python 3.12

| Package | Pinned Version | Issue | Resolution |
|---------|---------------|-------|------------|
| `sqlalchemy` | `1.3.20` | **EOL since 2023**. Does not support Python 3.12. `declarative_base` import path changed in 1.4+; removed in 2.0. | Upgrade to `sqlalchemy>=1.4.50` (minimum for 3.12 compat) or `>=2.0` for future-proofing |
| `pymongo` | `3.11.1` | No `cp312` wheels. PyMongo 3.x is EOL (last release 2021). Python 3.12 support requires PyMongo ≥4.6. | Upgrade to `pymongo>=4.6` (breaking API changes — audit all MongoClient usage) |
| `jaeger-client` | `4.8.0` | Abandoned (2022). No `cp312` wheel. Pure Python but depends on `threadloop` + `thrift` with C components. | Replace with OpenTelemetry SDK |
| `gevent-inotifyx` | `0.2` | Abandoned (2013). C extension with no `cp312` wheel. Source build will fail against Python 3.12 headers. | Remove or replace with `watchdog` (already in deps) |

#### HIGH — Likely to break or emit warnings on Python 3.12

| Package | Pinned Version | Issue | Resolution |
|---------|---------------|-------|------------|
| `pydantic` | `1.10.22` | Pydantic v1 emits deprecation warnings on 3.12; no further v1 releases planned. `cp312` wheels exist but library is EOL. | Evaluate upgrade to Pydantic v2 (major refactor) or pin at 1.10.x with acceptance of tech debt |
| `docker` | `4.4.4` | Classifiers only through 3.9. Pure Python — likely works but untested at 3.12. docker-py 4.x is EOL; 7.x is current. | Upgrade to `docker>=7.0` |
| `celery` | `5.2.2` | Celery 5.2 not officially tested on 3.12. Celery 5.3+ has 3.12 in CI matrix. | Upgrade to `celery>=5.3.6` |
| `pyzmq` | `22.3.0` | Older C extension build; `cp312` wheels available only from pyzmq ≥25.x. | Upgrade to `pyzmq>=25.1` |
| `redis-py-cluster` | `1.0.0` | Abandoned. Modern `redis>=4.1` has native cluster support. | Replace with `redis[cluster]>=5.0` |
| `opentracing` | `2.4.0` | Deprecated standard (archived 2022). Only needed by `jaeger-client`. | Remove when jaeger-client is replaced |
| `datadog` | `0.38.0` | Older version; 3.12 compat confirmed from `0.47.0`+. | Upgrade to `datadog>=0.47.0` |
| `numpy` | `1.26.4` | Works on 3.12 (`cp312` wheels available). OK as-is. | No change needed |

#### MEDIUM — Functional but with deprecation warnings or tech debt

| Package | Pinned Version | Issue | Resolution |
|---------|---------------|-------|------------|
| `future` | `1.0.0` | Python 2/3 compat shim. Functional on 3.12 but unnecessary. | Remove after cleaning `from builtins`/`from past` imports |
| `six` | `1.16.0` | Python 2/3 compat shim. Functional on 3.12 but unnecessary. | Remove after cleaning `six.moves` imports |
| `mysqlclient` | `2.2.7` | C extension; `cp312` wheels available from 2.2.1+. Requires `libmysqlclient-dev` on Ubuntu 24.04. | Verify `default-libmysqlclient-dev` package on Noble works |
| `lmdb` | `1.6.2` | C extension; has `cp312` wheels on `manylinux_2_28`. | No change needed |
| `uwsgi` | `2.0.30` | Has `cp312` support. Conda build may need adjusting for 3.12. | Verify conda-forge has `uwsgi` for py312 |
| `pillow` | `12.1.1` | Full 3.12 support. | No change needed |
| `pyarrow` | `17.0.0` | Has `cp312` wheels. | No change needed |
| `cryptography` | `45.0.5` | Full 3.12 support. Requires OpenSSL 3.x (provided by Ubuntu 24.04). | No change needed |

### 1.4 `distutils` Removal Impact (Python 3.12)

`distutils` was **removed** from the stdlib in Python 3.12 (PEP 632). `setuptools` provides a shim (`setuptools._distutils`), so projects using `setuptools` in `setup.py` are generally unaffected **if `setuptools` is installed**.

**Impact on this repo:**
- `setup.py` line 14: `del os.link` — comment says "hack for distutils + virtualbox filesystem". This is a monkey-patch on `os`, not an import of `distutils`. **No breakage**, but the comment is misleading.
- `setup.py` uses `from setuptools import setup, find_packages` — **not** `from distutils`. Safe.
- **Transitive risk**: Some NS packages that use `distutils` directly in their build systems (e.g., `pyinotify` in `nsconfig`) will fail. This is captured in the NS packages section.

### 1.5 C-Extension Compatibility (Ubuntu 24.04 ABI)

Ubuntu 24.04 ships:
- **glibc 2.39** — all `manylinux_2_28` wheels are compatible
- **GCC 13.2** — C extensions built from source will use this compiler
- **OpenSSL 3.x** — no more `libssl1.1`; packages requiring OpenSSL 1.1 will break

| C-Extension Package | Wheel Status (cp312, manylinux_2_28) | Source Build Risk |
|--------------------|-----------------------------------------|-------------------|
| `gevent==24.2.1` | Available | N/A |
| `cffi==1.17.1` | Available | N/A |
| `cryptography==45.0.5` | Available | N/A |
| `greenlet==3.2.3` | Available | N/A |
| `hiredis==2.4.0` | Available | N/A |
| `lmdb==1.6.2` | Available | N/A |
| `mysqlclient==2.2.7` | Available (needs libmysqlclient) | Low risk |
| `numpy==1.26.4` | Available | N/A |
| `pillow==12.1.1` | Available | N/A |
| `pyarrow==17.0.0` | Available | N/A |
| `ujson==5.10.0` | Available | N/A |
| `uwsgi==2.0.30` | Conda-built (needs py312 recipe) | Medium risk |
| `gevent-inotifyx==0.2` | **NOT available** | **HIGH risk — will fail** |
| `pyzmq==22.3.0` | **NOT available** for cp312 | HIGH risk at this version |

---

## Part 2: NS Upstream Packages Audit

### Summary Table

| Package | Pinned Version | Repo | Owner(s) | Bazel py_ver | CI Matrix | `python_requires` | Builder OS | Verdict (3.12) | Blocking Issues | Required Fixes |
|---|---|---|---|---|---|---|---|---|---|---|
| `nssysutil` | 25.90.203 | [ns-python-sysutil](https://github.com/netSkope/ns-python-sysutil) | CMouliNS, ns-aliang, ns-dgate | 3.8 | 3.8, 3.10 | Not declared | — | **BROKEN** | `raw_input()`, `xrange`, `print` statement — hard `SyntaxError` on Python 3.x | Rewrite Py2 syntax; bump Bazel to 3.12; add `python_requires`; update CI matrix |
| `nsconst` | 25.90.13 | [ns-python-const](https://github.com/netSkope/ns-python-const) | CMouliNS, ns-csingh, ns-jiew | 3.12 | 3.10, 3.12 | `>=3.10,<=3.13` | — | **Compatible** | None | None (optional: remove `six` dep) |
| `nslog` | 25.90.13 | [ns-python-log](https://github.com/netSkope/ns-python-log) | CMouliNS, ns-svemu | 3.12 | 3.10, 3.12 | `>=3.10,<3.13` | — | **Needs minor fix** | `python_requires` ceiling excludes future 3.12 strict resolvers | Lift ceiling to `<3.14`; replace `Logger.warn()` with `.warning()` |
| `nsdbfusion` | 26.82.212 | [ns-python-dbfusion](https://github.com/netSkope/ns-python-dbfusion) | ns-rradhakrishnan | 3.10 | 3.10 | Not declared | — | **Needs fixes** | `from collections import Iterable` (removed in 3.10); `gevent-inotifyx==0.2` (no cp312 wheel); `_logger.warn()` | Fix `collections.abc`; replace `gevent-inotifyx` with `watchdog`; add numpy to Bazel deps |
| `nsjson` | 25.90.11 | [ns-python-json](https://github.com/netSkope/ns-python-json) | CMouliNS | 3.12 | 3.10, 3.12 | `>=3.10,<3.13` | — | **Needs minor fix** | `python_requires` ceiling | Lift ceiling to `<3.14` |
| `nsconfwatcher` | 25.90.11 | [ns-python-confwatcher](https://github.com/netSkope/ns-python-confwatcher) | CMouliNS | 3.8 | 3.8, 3.10 | Not declared | — | **Needs minor fix** | Bazel targets EOL Python 3.8; `_logger.warn()` deprecated | Bump Bazel to 3.12; update CI matrix; add `python_requires`; fix `.warn()` |
| `nsmongo` | 25.90.219 | [ns-python-mongo](https://github.com/netSkope/ns-python-mongo) | ns-cshyi, ns-pschandel | 3.10 | 3.10 | Not declared | ubuntu2004 | **Needs fixes** | `jaeger-client==4.8.0` (abandoned, no cp312); `opentracing==2.4.0` (deprecated); `docker==7.1.0` unverified on 3.12; `six` usage | Replace jaeger+opentracing with OpenTelemetry; verify docker; update builder to 24.04; expand CI |
| `nsnightingale` | 25.90.20 | [ns-python-nightingale](https://github.com/netSkope/ns-python-nightingale) | CMouliNS, wooseung-sim | 3.8 | 3.8, 3.10 | Not declared | ubuntu2004 | **Needs fixes** | `datadog==0.52.1` unverified on 3.12; Bazel targets 3.8 | Verify/upgrade datadog; bump Bazel to 3.12; update CI matrix; update builder to 24.04 |
| `nsdocker` | 25.90.208 | [ns-python-docker](https://github.com/netSkope/ns-python-docker) | CMouliNS, wooseung-sim | 3.8 | 3.8, 3.10 | Not declared | ubuntu2004 | **Needs fixes (infra only)** | Bazel targets 3.8; builder on 20.04; depends on nssysutil+nslog | Bump Bazel to 3.12; update CI matrix; update builder to 24.04; gated on nssysutil/nslog fixes |
| `nsfilewatch` | 25.90.19 | [ns-python-filewatch](https://github.com/netSkope/ns-python-filewatch) | CMouliNS, ns-csingh | 3.12 | 3.10, 3.12 | `>=3.10,<3.13` | ubuntu2004 | **Needs fixes** | `from imp import reload` dead branch (`imp` removed in 3.12 → `ImportError`); `gevent-inotifyx==0.2` (no cp312); `python_requires` ceiling | Remove `imp` fallback; lift ceiling to `<3.14`; replace `gevent-inotifyx` with `watchdog`; update builder |
| `nsclustermapping` | 25.90.208 | [ns-python-clustermapping](https://github.com/netSkope/ns-python-clustermapping) | CMouliNS | 3.10 | 3.10 | Not declared | ubuntu2004 | **Needs fixes (minor)** | No `python_requires`; CI doesn't test 3.12; depends on `nswatcher` (unlocated) | Add `python_requires`; bump Bazel to 3.12; expand CI; update builder; verify `nswatcher` |
| `nsconfig` | 25.90.16 | [ns-python-stagedconfig](https://github.com/netSkope/ns-python-stagedconfig) | CMouliNS, ns-cyang, ns-cyen | 3.8 | 3.8, 3.10 | Not declared | ubuntu2004 | **Needs fixes** | `pyinotify==0.9.6` phantom dep (unused but blocks install — `distutils` removal in 3.12 breaks build); Bazel targets 3.8 | Remove `pyinotify` from requirements; bump Bazel to 3.12; update CI; update builder; verify `nsnightingale` |
| `nsutil` | 26.82.208 | **Not found** | — | — | — | — | — | **Not assessed** | Repo not located under netSkope org; root hub package pulling in most other ns packages | **Highest-priority gap** — locate source repo and audit separately |
| `nswatcher` | 25.90.12 | **Not found** | — | — | — | — | — | **Not assessed** | Repo not located; transitive dep of `nsclustermapping` | Locate source repo and audit separately |

**Primary contact**: Chandra Mouli Maddala (CMouliNS) — chandramouli@netskope.com
_(de facto platform owner; appears as primary human committer in 9 of 12 identified repos)_

### Per-Package Details

#### `nssysutil` — **BROKEN**

**Repo**: netSkope/ns-python-sysutil | **Owners**: CMouliNS, ns-aliang, ns-dgate

Pre-existing Python 2 syntax (`raw_input`, `xrange`, `print` statement) — hard `SyntaxError` on any Python 3.x. Not a 3.12 regression.

**Required fixes:**
1. Rewrite `cryptokey.py` — `raw_input()` → `input()`, `print` statement → `print()`
2. Replace `xrange` with `range` in `hexdump.py`
3. Remove `from builtins import zip`
4. Bump Bazel `python_version` to `3.12`, CI matrix to `["3.10", "3.12"]`

---

#### `nsconst` — **Compatible**

**Repo**: netSkope/ns-python-const | Bazel `python_version`: 3.12, CI: 3.10/3.12

No action required. Clean Python 3 code with `python_requires = ">=3.10,<=3.13"`.

---

#### `nslog` — **Needs minor fix**

**Repo**: netSkope/ns-python-log | `python_requires`: `>=3.10,<3.13`

Fix: Update `python_requires` ceiling from `<3.13` to `<3.14`.
Tech debt: `Logger.warn()` → `Logger.warning()`.

---

#### `nsdbfusion` — **Needs fixes**

**Repo**: netSkope/ns-python-dbfusion | Bazel: 3.10

Blocking: `from collections import Iterable` (removed in 3.10 — broken on its own declared target).
Blocking: `gevent-inotifyx==0.2` — no cp312 wheel, source build will fail.

Fixes:
1. `from collections import Iterable` → `from collections.abc import Iterable`
2. Resolve `gevent-inotifyx` — replace with `watchdog`
3. Replace `_logger.warn()` with `_logger.warning()`

---

#### `nsjson` — **Needs minor fix**

Fix: Update `python_requires` from `<3.13` to `<3.14`.

---

#### `nsconfwatcher` — **Needs minor fix**

Fixes: Bump Bazel from 3.8 → 3.12, update CI matrix, add `python_requires`.

---

#### `nsmongo` — **Needs fixes**

Blocking: `jaeger-client==4.8.0` — abandoned, no cp312 wheel.
Fix: Replace jaeger-client + opentracing with OpenTelemetry SDK.

---

#### `nsnightingale` — **Needs fixes**

Fix: Verify/upgrade `datadog` for 3.12, bump Bazel from 3.8 → 3.12.

---

#### `nsdocker` — **Needs fixes (infra only)**

Source code is clean. Needs Bazel bump from 3.8 → 3.12 and Ubuntu 24.04 builder images.

---

#### `nsfilewatch` — **Needs fixes**

Blocking: `from imp import reload` — `imp` was **removed in Python 3.12**. If the dead fallback branch is reached, raises `ImportError`.
Blocking: `gevent-inotifyx==0.2` — same as nsdbfusion.

Fixes:
1. Remove dead `from imp import reload` fallback
2. Update `python_requires` ceiling to `<3.14`
3. Replace `gevent-inotifyx` with `watchdog`

---

#### `nsclustermapping` — **Needs fixes (minor)**

Clean code. Needs Bazel bump, CI matrix expansion, `python_requires` declaration.

---

#### `nsconfig` — **Needs fixes**

Blocking: `pyinotify==0.9.6` declared in requirements but not imported. Abandoned (2015), fails to install on 3.12 due to `distutils` removal.

Fix: Remove `pyinotify` from requirements (unused phantom dependency).

---

#### `nsutil` / `nswatcher` — **Not assessed**

Repos not found under netSkope org. These are the **highest-priority gaps** in this audit.

---

## Part 3: CI/CD Pipeline Changes Required

### 3.1 Drone CI (`.drone.yml`)

| Item | Current | Required |
|------|---------|----------|
| `python_image` global var | `python:3.8` | `python:3.12` |
| `builder:v31` image | Ubuntu 16.04 based | Need `builder-2404:vXX` (Ubuntu 24.04) |
| `builder-2004:v11` image | Ubuntu 20.04 based | Need `builder-2404:vXX` |
| Test step coverage runner | `.tox/py310/bin/coverage` | `.tox/py312/bin/coverage` |
| Pylint install | `pip3 install pylint==2.10.2` | Upgrade to `pylint>=3.0` (3.12 compat) |
| Deb distribution | `xenial` (16.04) | `noble` (24.04) |
| Package path | `$DEB_DIR/xenial/` | `$DEB_DIR/noble/` |

### 3.2 GitHub Actions (`ci.yml`)

| Item | Current | Required |
|------|---------|----------|
| Miniforge conda env | `app-env-py310.yaml` | Create `app-env-py312.yaml` |
| `make test` tox env | `py310` | `py312` |
| Coverage reporter path | `.tox/py310/bin/coverage` | `.tox/py312/bin/coverage` |
| `libmysqlclient-dev` apt pkg | Installed via `apt-get` | Verify package name on Noble (`default-libmysqlclient-dev`) |
| Runner image | `arc-default-basic-cheap-m-set` | Verify runner has Ubuntu 24.04 or is compatible |

### 3.3 Build System Files to Update

| File | Change |
|------|--------|
| `nsappinfo/Makefile` | `PYTHON_VERSION ?= py312` |
| `nsappinfo/tox.ini` | `envlist = py312`, `conda_env = app-env-py312.yaml` |
| `nsappinfo/app-env-py310.yaml` | Clone → `app-env-py312.yaml`, change `python=3.12.x`, `python_abi=3.12` |
| `nsappinfo/setup.py` | Add `python_requires=">=3.10,<3.13"` to `setup()` |

---

## Part 4: System-Level Dependencies (Ubuntu 24.04)

### 4.1 Package Availability

| System Package | Ubuntu 20.04 | Ubuntu 24.04 | Impact |
|----------------|-------------|-------------|--------|
| `python3` | 3.8.10 | **3.12.3** | Target runtime |
| `libssl-dev` | OpenSSL 1.1.1 | **OpenSSL 3.x** (`libssl-dev` → `libssl3`) | `cryptography` OK; legacy packages expecting 1.1 will break |
| `libmysqlclient-dev` | 8.0.x | `default-libmysqlclient-dev` (MariaDB 10.11) | Name change; verify `mysqlclient` builds |
| `libffi-dev` | 3.3 | 3.4 | Compatible; `cffi` wheels handle this |
| `gcc` | 9.4 | **13.2** | C extensions built from source use newer ABI |
| `libc6` | glibc 2.31 | **glibc 2.39** | `manylinux_2_28` wheels compatible |
| `pkg-config` | Available | Available | No change |
| `fakeroot` | Available | Available | No change (used in deb build) |

### 4.2 Removed/Changed Packages

| Legacy Package | Status on 24.04 | Impact |
|---------------|-----------------|--------|
| `libssl1.1` | **Removed** | Any binary linked against 1.1 needs rebuild |
| `python3.8` | **Not available** via apt | Bazel toolchains targeting 3.8 break |
| `python3.10` | Available via `deadsnakes` only | Not default; avoid depending on it at runtime |
| `distutils` (Python module) | **Removed from stdlib** in 3.12 | `setuptools` provides shim; explicit `import distutils` fails |

---

## Part 5: Action Items Table (Implementation-Ready)

### P0 — Blockers (fix before any migration attempt)

| # | Package/File | Action | Owner | Estimated Effort (AI-generated) | Team Estimate |
|---|---|---|---|---|---|
| 1 | `requirements.in` | Upgrade `sqlalchemy` from `1.3.20` to `>=1.4.50` | CCI team | 2-3 days (API changes in declarative_base import path) | _TBD_ |
| 2 | `requirements.in` | Upgrade `pymongo` from `3.11.1` to `>=4.6` | CCI team | 3-5 days (breaking API: `Collection.find()` returns cursor changes, auth mechanism changes) | _TBD_ |
| 3 | `nsappinfo/src/` (3 files) | Update `from sqlalchemy.ext.declarative import declarative_base` to `from sqlalchemy.orm import declarative_base` (1.4+) or `DeclarativeBase` class (2.0+) | CCI team | 1 day | _TBD_ |
| 4 | `requirements.in` | Remove `jaeger-client==4.8.0` + `opentracing==2.4.0` + `threadloop==1.0.2` + `thrift==0.13.0`; add `opentelemetry-sdk` + `opentelemetry-exporter-otlp` | CCI team + nsmongo owner | 2-3 days | _TBD_ |
| 5 | `requirements.in` | Remove `gevent-inotifyx==0.2` (already have `watchdog==4.0.1` in deps) | CCI team | 0.5 day | _TBD_ |
| 6 | NS upstream: `nssysutil` | Fix Python 2 syntax (`raw_input`, `xrange`, `print` statement) | CMouliNS | 1 day | _TBD_ |
| 7 | NS upstream: `nsdbfusion` | Fix `from collections import Iterable` → `from collections.abc import Iterable` | ns-rradhakrishnan | 0.5 day | _TBD_ |
| 8 | NS upstream: `nsfilewatch` | Remove dead `from imp import reload` fallback (`imp` removed in 3.12) | CMouliNS | 0.5 day | _TBD_ |
| 9 | NS upstream: `nsconfig` | Remove phantom `pyinotify==0.9.6` from requirements | CMouliNS | 0.5 day | _TBD_ |

### P1 — Fix during migration sprint

| # | Package/File | Action | Owner | Estimated Effort (AI-generated) | Team Estimate |
|---|---|---|---|---|---|
| 10 | `requirements.in` | Upgrade `pyzmq` from `22.3.0` to `>=25.1` | CCI team | 1 day (API stable) | _TBD_ |
| 11 | `requirements.in` | Upgrade `celery` from `5.2.2` to `>=5.3.6` | CCI team | 1 day | _TBD_ |
| 12 | `requirements.in` | Upgrade `docker` from `4.4.4` to `>=7.0` | CCI team | 1 day (API changes in 7.x) | _TBD_ |
| 13 | `requirements.in` | Upgrade `datadog` from `0.38.0` to `>=0.47.0` | CCI team | 0.5 day | _TBD_ |
| 14 | `requirements.in` | Remove `redis-py-cluster==1.0.0`; use `redis[cluster]>=5.0` | CCI team | 1 day | _TBD_ |
| 15 | `svc/routes/mongo_update.py` | Remove `from past.builtins import basestring`; replace `basestring` with `str` | CCI team | 0.5 day | _TBD_ |
| 16 | `config/meta_manager.py`, `svc/routes/install.py` | Replace `try: import ConfigParser` blocks with `import configparser` | CCI team | 0.5 day | _TBD_ |
| 17 | `run_config.py`, `mongo_update.py` | Remove `from builtins import str/object`, `from __future__ import print_function` | CCI team | 0.5 day | _TBD_ |
| 18 | NS upstream (all) | Lift `python_requires` ceilings from `<3.13` to `<3.14` | CMouliNS | 1 day | _TBD_ |
| 19 | NS upstream (all) | Bump Bazel `python_version` from 3.8/3.10 → 3.12 | CMouliNS | 1 day | _TBD_ |
| 20 | `app-env-py310.yaml` | Create `app-env-py312.yaml` with `python=3.12.x`, `python_abi=3.12` | CCI team | 1 day | _TBD_ |
| 21 | `tox.ini` + `Makefile` | Update `PYTHON_VERSION` to `py312`, `envlist` to `py312` | CCI team | 0.5 day | _TBD_ |
| 22 | `.drone.yml` | Update `python_image` to `python:3.12`, builder images to Ubuntu 24.04, distribution to `noble` | CCI team + DevOps | 2 days | _TBD_ |
| 23 | `.github/workflows/ci.yml` | Update conda env reference, coverage path, verify apt packages on Noble | CCI team | 1 day | _TBD_ |

### P2 — Post-migration cleanup (Technical Debt)

| # | Package/File | Action | Owner | Estimated Effort (AI-generated) | Team Estimate |
|---|---|---|---|---|---|
| 24 | `cci_calc.py`, `mongodb_loader.py` | Replace `_logger.warn()` with `_logger.warning()` | CCI team | 0.5 day | _TBD_ |
| 25 | `requirements.in` | Remove `future==1.0.0` and `six==1.16.0` after all `from builtins`/`from past`/`six.moves` usage is cleaned | CCI team | 1 day | _TBD_ |
| 26 | `requirements.in` | Evaluate `pydantic` v1 → v2 migration (major effort, separate initiative) | CCI team | 5-10 days | _TBD_ |
| 27 | NS upstream (all) | Expand CI matrices to include Python 3.12 | CMouliNS | 1 day | _TBD_ |
| 28 | NS upstream (all) | Remove `six`, `future`, `past` shims from all packages | CMouliNS | 2 days | _TBD_ |
| 29 | NS upstream | Locate `nsutil` and `nswatcher` source repos; audit for 3.12 | CCI team | 1-2 days | _TBD_ |
| 30 | `setup.py` | Add `python_requires=">=3.10,<3.13"` to prevent accidental install on incompatible interpreters | CCI team | 0.5 day | _TBD_ |

---

## Part 6: Migration Execution Order

```
Phase 0: Preparation (parallel with normal work)
├── Create app-env-py312.yaml (conda env)
├── Create builder-2404 Docker images (DevOps)
├── Locate nsutil / nswatcher repos
└── Get P0 NS package fixes merged + released

Phase 1: Core Dependency Upgrades (requires feature branch)
├── Upgrade SQLAlchemy 1.3 → 1.4+ (update declarative_base imports)
├── Upgrade pymongo 3.11 → 4.6+ (audit all MongoClient usage)
├── Remove jaeger-client → add OpenTelemetry
├── Remove gevent-inotifyx
├── Upgrade pyzmq, celery, docker, datadog
└── Run full test suite on Python 3.12 conda env

Phase 2: Code Cleanup + CI Update
├── Remove Python 2 compat code (basestring, builtins, six.moves)
├── Update tox.ini, Makefile, setup.py
├── Update .drone.yml (images, paths, distribution)
├── Update .github/workflows/ci.yml
└── Validate full CI pipeline on Ubuntu 24.04

Phase 3: Deployment
├── Build .deb on Ubuntu 24.04 builder
├── Deploy to staging with Python 3.12 runtime
├── Smoke test all routes (CCI, CASB inline, mongo_update, install)
└── Production rollout
```

---

## Appendix: Dependency Version Matrix

Complete mapping of current → target versions for all packages requiring changes:

| Package | Current | Minimum for 3.12 | Recommended | Notes |
|---------|---------|-------------------|-------------|-------|
| `sqlalchemy` | 1.3.20 | 1.4.50 | 2.0.x | Major API changes in 2.0; 1.4 is compat bridge |
| `pymongo` | 3.11.1 | 4.6.0 | 4.9.x | Auth, cursor, codec changes |
| `pyzmq` | 22.3.0 | 25.1.0 | 26.x | Stable API |
| `celery` | 5.2.2 | 5.3.6 | 5.4.x | Minor changes |
| `docker` | 4.4.4 | 7.0.0 | 7.1.x | `APIClient` → `DockerClient` |
| `datadog` | 0.38.0 | 0.47.0 | 0.50.x | Additive changes |
| `jaeger-client` | 4.8.0 | — | Remove | Replace with OTel |
| `opentracing` | 2.4.0 | — | Remove | Deprecated standard |
| `gevent-inotifyx` | 0.2 | — | Remove | Use `watchdog` |
| `redis-py-cluster` | 1.0.0 | — | Remove | Use `redis[cluster]` |
| `future` | 1.0.0 | — | Remove (P2) | After code cleanup |
| `six` | 1.16.0 | — | Remove (P2) | After code cleanup |
| `pydantic` | 1.10.22 | 1.10.22 (works) | 2.x (P2) | Major effort |

---

## Revision History

| Date | Change |
|------|--------|
| 2026-05-08 | v2 — Consolidated audit: added repo-specific code issues, direct dependency analysis, CI/CD pipeline details, distutils/C-extension coverage, and system-level dependency mapping. Replaces earlier NS-packages-only audit. |
