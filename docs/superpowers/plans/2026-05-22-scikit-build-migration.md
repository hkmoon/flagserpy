# scikit-build-core Migration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the hand-rolled `setup.py` + `CMakeExtension` build system in flagserpy with `scikit-build-core`, modeled after the `bigrandomgraphs` project. Wheel layout and Python API are unchanged.

**Architecture:** scikit-build-core becomes the PEP 517 build backend. `pyproject.toml` carries all project metadata. `CMakeLists.txt` uses `find_package(pybind11)` (PyPI-installed at build time) and `install(TARGETS … DESTINATION flagserpy/modules)`. The `flagser` C++ source remains a git submodule. All Python code is unchanged except dropping the `_version.py` indirection.

**Tech Stack:** scikit-build-core ≥ 0.10, pybind11 ≥ 2.12, CMake ≥ 3.15, cibuildwheel for wheel matrix.

**Reference spec:** `docs/superpowers/specs/2026-05-22-scikit-build-migration-design.md`

---

## Setup

### Task 0: Create the branch

**Files:** none

- [ ] **Step 1: Confirm we're on master with a clean tree**

Run: `git status --short && git rev-parse --abbrev-ref HEAD`
Expected: empty status output; current branch is `master`.

- [ ] **Step 2: Make sure the flagser submodule is checked out**

Run: `git submodule update --init`
Expected: either no output (already initialized) or "Submodule path 'flagser': checked out '<sha>'".

- [ ] **Step 3: Create and switch to the implementation branch**

Run: `git checkout -b modernize_python_build`
Expected: "Switched to a new branch 'modernize_python_build'".

---

## Phase 1: Replace the build configuration

### Task 1: Rewrite pyproject.toml

**Files:**
- Modify: `pyproject.toml` (full rewrite — current file is 3 lines)

- [ ] **Step 1: Overwrite pyproject.toml**

Replace the entire contents of `pyproject.toml` with:

```toml
[build-system]
requires = ["scikit-build-core>=0.10", "pybind11>=2.12"]
build-backend = "scikit_build_core.build"

[project]
name = "flagserpy"
version = "0.4.7"
description = "Python bindings for the flagser C++ library."
readme = "README.rst"
license = { file = "LICENSE" }
authors = [
    { name = "Jason P. Smith", email = "jason.smith@ntu.ac.uk" },
    { name = "Guillaume Tauzin" },
]
keywords = [
    "topological data analysis",
    "persistent homology",
    "directed flags complex",
    "persistence diagrams",
]
classifiers = [
    "Intended Audience :: Science/Research",
    "Intended Audience :: Developers",
    "License :: OSI Approved",
    "Programming Language :: C++",
    "Programming Language :: Python",
    "Topic :: Software Development",
    "Topic :: Scientific/Engineering",
    "Operating System :: Microsoft :: Windows",
    "Operating System :: POSIX",
    "Operating System :: Unix",
    "Operating System :: MacOS",
    "Programming Language :: Python :: 3.7",
    "Programming Language :: Python :: 3.8",
    "Programming Language :: Python :: 3.9",
    "Programming Language :: Python :: 3.10",
    "Programming Language :: Python :: 3.11",
    "Programming Language :: Python :: 3.12",
    "Programming Language :: Python :: 3.13",
    "Programming Language :: Python :: 3.14",
]
requires-python = ">=3.7"
dependencies = [
    "numpy >= 1.17.0",
    "scipy >= 0.17.0",
]

[project.urls]
Homepage = "https://github.com/TopNetBio/flagserpy"
Download = "https://github.com/TopNetBio/flagserpy/tarball/v0.4.7"

[project.optional-dependencies]
tests = [
    "pytest",
    "pytest-timeout",
    "pytest-cov",
    "pytest-azurepipelines",
    "pytest-benchmark",
    "flake8",
]
doc = [
    "sphinx",
    "sphinx-gallery",
    "sphinx-issues",
    "sphinx_rtd_theme",
    "numpydoc",
]

[tool.scikit-build]
minimum-version = "0.10"
cmake.version = ">=3.15"
wheel.packages = ["flagserpy"]

[tool.cibuildwheel]
build = ["cp37-*", "cp38-*", "cp39-*", "cp310-*", "cp311-*", "cp312-*", "cp313-*", "cp314-*"]
skip = ["*-win32", "*-musllinux_*", "*_i686"]

[tool.cibuildwheel.macos]
archs = ["x86_64", "arm64"]

[tool.pytest.ini_options]
junit_family = "xunit1"
addopts = [
    "--ignore=flagser",
    "--ignore=flagserpy/tests/__main__.py",
    "--ignore=flagserpy/tests/conftest.py",
    "-ra",
]

[tool.flake8]
exclude = ["flagser"]
```

- [ ] **Step 2: Sanity-check the TOML parses**

Run: `python -c "import tomllib; tomllib.load(open('pyproject.toml','rb'))"` (Python 3.11+) or `python -c "import tomli; tomli.load(open('pyproject.toml','rb'))"` (3.7–3.10).
Expected: no output, exit code 0.

- [ ] **Step 3: Commit**

```bash
git add pyproject.toml
git commit -m "build: switch pyproject.toml to scikit-build-core backend"
```

---

### Task 2: Rewrite CMakeLists.txt

**Files:**
- Modify: `CMakeLists.txt`

- [ ] **Step 1: Overwrite CMakeLists.txt**

Replace the entire contents of `CMakeLists.txt` with:

```cmake
cmake_minimum_required(VERSION 3.15)
project(flagser_pybind LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 14)

find_package(pybind11 CONFIG REQUIRED)

set(BINDINGS_DIR "src")

# flagser
pybind11_add_module(flagser_pybind "${BINDINGS_DIR}/flagser_bindings.cpp")
target_compile_definitions(flagser_pybind PRIVATE RETRIEVE_PERSISTENCE=1 MANY_VERTICES=1)
target_include_directories(flagser_pybind PRIVATE .)
if(MSVC)
    target_compile_options(flagser_pybind PUBLIC $<$<CONFIG:RELEASE>: /Wall /O2>)
    target_compile_options(flagser_pybind PUBLIC $<$<CONFIG:DEBUG>:/O1 /DEBUG:FULL /Zi /Zo>)
else()
    target_compile_options(flagser_pybind PUBLIC $<$<CONFIG:RELEASE>: -Ofast>)
    target_compile_options(flagser_pybind PUBLIC $<$<CONFIG:DEBUG>: -O2 -ggdb -D_GLIBCXX_DEBUG>)
endif()
install(TARGETS flagser_pybind DESTINATION flagserpy/modules)

# flagser with USE_COEFFICIENTS
pybind11_add_module(flagser_coeff_pybind "${BINDINGS_DIR}/flagser_bindings.cpp")
target_compile_definitions(flagser_coeff_pybind PRIVATE RETRIEVE_PERSISTENCE=1 USE_COEFFICIENTS=1 MANY_VERTICES=1)
target_include_directories(flagser_coeff_pybind PRIVATE .)
if(MSVC)
    target_compile_options(flagser_coeff_pybind PUBLIC $<$<CONFIG:RELEASE>: /Wall /O2>)
    target_compile_options(flagser_coeff_pybind PUBLIC $<$<CONFIG:DEBUG>:/O1 /DEBUG:FULL /Zi /Zo>)
else()
    target_compile_options(flagser_coeff_pybind PUBLIC $<$<CONFIG:RELEASE>: -Ofast>)
    target_compile_options(flagser_coeff_pybind PUBLIC $<$<CONFIG:DEBUG>: -O2 -ggdb -D_GLIBCXX_DEBUG>)
endif()
install(TARGETS flagser_coeff_pybind DESTINATION flagserpy/modules)

# flagser-count
pybind11_add_module(flagser_count_pybind "${BINDINGS_DIR}/flagser_count_bindings.cpp")
target_compile_definitions(flagser_count_pybind PRIVATE MANY_VERTICES=1)
target_include_directories(flagser_count_pybind PRIVATE .)
if(MSVC)
    target_compile_options(flagser_count_pybind PUBLIC $<$<CONFIG:RELEASE>: /Wall /O2>)
    target_compile_options(flagser_count_pybind PUBLIC $<$<CONFIG:DEBUG>:/O1 /DEBUG:FULL /Zi /Zo>)
else()
    target_compile_options(flagser_count_pybind PUBLIC $<$<CONFIG:RELEASE>: -Ofast>)
    target_compile_options(flagser_count_pybind PUBLIC $<$<CONFIG:DEBUG>: -O2 -ggdb -D_GLIBCXX_DEBUG>)
endif()
install(TARGETS flagser_count_pybind DESTINATION flagserpy/modules)
```

- [ ] **Step 2: Commit**

```bash
git add CMakeLists.txt
git commit -m "build: use find_package(pybind11) and install rules in CMakeLists"
```

---

### Task 3: Update __init__.py and remove _version.py

**Files:**
- Modify: `flagserpy/__init__.py`
- Delete: `flagserpy/_version.py`

- [ ] **Step 1: Edit flagserpy/__init__.py**

Current first line is `from ._version import __version__`. Replace it with `__version__ = "0.4.7"`.

The full file should read:

```python
__version__ = "0.4.7"

from .flagio import load_unweighted_flag, load_weighted_flag, \
    save_unweighted_flag, save_weighted_flag
from .flagser import flagser_unweighted, flagser_weighted
from .flagser_count import flagser_count_unweighted, \
    flagser_count_weighted

__all__ = ['load_unweighted_flag',
           'load_weighted_flag',
           'save_unweighted_flag',
           'save_weighted_flag',
           'flagser_unweighted',
           'flagser_weighted',
           'flagser_count_unweighted',
           'flagser_count_weighted',
           '__version__']
```

- [ ] **Step 2: Delete flagserpy/_version.py**

Run: `git rm flagserpy/_version.py`
Expected: "rm 'flagserpy/_version.py'".

- [ ] **Step 3: Commit**

```bash
git add flagserpy/__init__.py
git commit -m "build: hardcode __version__ in __init__.py, drop _version.py"
```

---

### Task 4: Delete legacy build files

**Files:**
- Delete: `setup.py`
- Delete: `setup.cfg`
- Delete: `MANIFEST.in`
- Delete: `requirements.txt`

- [ ] **Step 1: Remove the four legacy files**

Run: `git rm setup.py setup.cfg MANIFEST.in requirements.txt`
Expected: four "rm '<file>'" lines.

- [ ] **Step 2: Commit**

```bash
git commit -m "build: delete setup.py, setup.cfg, MANIFEST.in, requirements.txt"
```

---

### Task 5: Update .gitignore

**Files:**
- Modify: `.gitignore` (lines 111-112)

- [ ] **Step 1: Remove the pybind11 entry**

Open `.gitignore` and delete these two lines (currently at the bottom of the Pybind11 section):

```
# Pybind11
pybind11
```

- [ ] **Step 2: Commit**

```bash
git add .gitignore
git commit -m "build: drop pybind11 ignore (no longer git-cloned at build)"
```

---

## Phase 2: Local validation

### Task 6: Build the wheel and verify contents

**Files:** none (validation only)

- [ ] **Step 1: Install the build front-end**

Run: `python -m pip install --upgrade build`
Expected: build installed.

- [ ] **Step 2: Clean any prior build artifacts**

Run: `rm -rf build dist *.egg-info wheelhouse`
Expected: no output.

- [ ] **Step 3: Build the wheel**

Run: `python -m build --wheel .`
Expected: terminates with "Successfully built flagserpy-0.4.7-cp<NN>-cp<NN>-<plat>.whl" in `dist/`.
If pybind11 or scikit-build-core is missing in the isolated build env, the backend will install them automatically — that is normal output.

- [ ] **Step 4: Inspect wheel contents**

Run: `python -m zipfile -l dist/flagserpy-0.4.7-*.whl | grep modules/`
Expected: three `flagserpy/modules/*.so` entries (or `.pyd` on Windows): `flagser_pybind*`, `flagser_coeff_pybind*`, `flagser_count_pybind*`.

If any of the three is missing, the corresponding `install(TARGETS …)` line in CMakeLists.txt is wrong — fix Task 2 before proceeding.

- [ ] **Step 5: Install the built wheel and smoke-test import**

Run: `python -m pip install --force-reinstall dist/flagserpy-0.4.7-*.whl`
Then: `python -c "from flagserpy import flagser_unweighted, flagser_weighted, flagser_count_unweighted, flagser_count_weighted; from flagserpy.modules.flagser_pybind import compute_homology; from flagserpy.modules.flagser_coeff_pybind import compute_homology as ch_coeff; from flagserpy.modules.flagser_count_pybind import compute_cell_count; print('ok')"`
Expected: `ok`.

- [ ] **Step 6: Run the existing test suite**

Run: `python -m pip install pytest pytest-cov pytest-timeout && pytest flagserpy/tests -ra`
Expected: tests pass (or, at worst, same failures the repo already had on master — confirm by stashing branch and re-running on master if a failure looks suspicious).

- [ ] **Step 7: Build an sdist and inspect**

Run: `python -m build --sdist . && tar -tzf dist/flagserpy-0.4.7.tar.gz | head -30`
Expected: sdist tarball lists `pyproject.toml`, `CMakeLists.txt`, `flagserpy/__init__.py`, `src/flagser_bindings.cpp`, and crucially files under `flagser/` (the submodule). If `flagser/` content is missing, the sdist cannot build on PyPI consumers — investigate scikit-build-core sdist include rules before continuing.

- [ ] **Step 8: Build the wheel from the sdist (end-to-end check)**

Run: `python -m pip wheel --no-deps -w dist-from-sdist dist/flagserpy-0.4.7.tar.gz`
Expected: produces `dist-from-sdist/flagserpy-0.4.7-*.whl`. This proves a downstream user can `pip install flagserpy` from PyPI source distribution.

- [ ] **Step 9: Commit validation cleanup if anything changed**

If Task 6 surfaced and fixed any issue (e.g. an sdist include rule for the `flagser/` submodule), the fix lives in `pyproject.toml`. Stage and commit it:

```bash
git status --short
# if pyproject.toml shows up:
git add pyproject.toml
git commit -m "build: fix sdist to include flagser submodule sources"
```

If nothing changed, skip the commit.

---

## Phase 3: CI updates

### Task 7: Update .github/workflows/ci.yml

**Files:**
- Modify: `.github/workflows/ci.yml`

- [ ] **Step 1: Overwrite ci.yml**

Replace the entire contents of `.github/workflows/ci.yml` with:

```yaml
name: Build package on different OSs and Python versions

on : [push, pull_request]

jobs:

  build_package:
    name: Build ${{ github.event.repository.name }} on ${{ matrix.os }} for Python-${{ matrix.python-version }}
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        python-version: ['3.7', '3.8', '3.9', '3.10', '3.11', '3.12', '3.13', '3.14']
        include:
          - os: ubuntu-latest
            path: ~/.cache/pip
          - os: macos-latest
            path: ~/Library/Caches/pip
          - os: windows-latest
            path: ~\AppData\Local\pip\Cache

    steps:
      - uses: actions/checkout@v4
        with:
          submodules: true

      - uses: actions/setup-python@v5
        name: Install Python-${{ matrix.python-version }}
        with:
          python-version: ${{ matrix.python-version }}

      - name: Activating Python cache
        uses: actions/cache@v4
        id: cache_python
        continue-on-error: true
        with:
          path: ${{ matrix.path }}
          key: ${{ runner.os }}-pip-${{ matrix.python-version }}-${{ hashFiles('pyproject.toml') }}

      - name: Build ${{ github.event.repository.name }}
        run: |
          python -m pip install --upgrade pip
          python -m pip install -e ".[doc,tests]"

      - name: Install and run flake8 on Mac
        if: ${{ runner.os == 'macOS' }}
        run: |
          flake8

      - name: Run test on Mac with coverage generated
        if: ${{ runner.os == 'macOS' }}
        run: |
          pytest --cov flagserpy --cov-report xml

      - name: Run test on Linux and Windows with no coverage generated
        if: ${{ runner.os != 'macOS' }}
        run: |
          python -m pytest --no-cov --no-coverage-upload

      - name: Build sphinx doc on Linux
        if: ${{ runner.os == 'Linux' }}
        run: |
          cd doc
          python -m pip install -r requirements.txt
          sphinx-build -b html . build

      - name: Upload built documentation and coverage as artifacts
        uses: actions/upload-artifact@v4
        with:
          name: ArtifactsCI-${{ matrix.os }}-${{ matrix.python-version }}
          if-no-files-found: ignore
          path: |
            doc/build
            coverage.xml
            htmlcov
```

Key diffs vs the old file: actions upgraded to v4/v5, `submodules: true` on checkout, cache key now hashes `pyproject.toml` instead of `requirements.txt`, Python matrix expanded to 3.7–3.14, `--cov pyflagser` typo fixed to `--cov flagserpy`, artifact name made matrix-unique (v4 requirement).

- [ ] **Step 2: Commit**

```bash
git add .github/workflows/ci.yml
git commit -m "ci: update ci.yml for scikit-build-core (submodules, action versions, cov path)"
```

---

### Task 8: Update .github/workflows/wheels.yml

**Files:**
- Modify: `.github/workflows/wheels.yml`

- [ ] **Step 1: Overwrite wheels.yml**

Replace the entire contents of `.github/workflows/wheels.yml` with:

```yaml
name: Build Wheels

on : workflow_dispatch

jobs:

  build_wheels:
    name: Build wheels on ${{ matrix.os }} for ${{ github.event.repository.name }}
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]

    steps:
      - uses: actions/checkout@v4
        with:
          submodules: true

      - name: Build wheels
        uses: pypa/cibuildwheel@v2.21.3
        env:
          CIBW_BUILD: "cp37-* cp38-* cp39-* cp310-* cp311-* cp312-* cp313-* cp314-*"
          CIBW_SKIP: "*-win32 *-musllinux_* *_i686"
          CIBW_BEFORE_BUILD_WINDOWS: sed -i $'s/\r$//' README.rst && python -m pip install delvewheel
          CIBW_REPAIR_WHEEL_COMMAND_WINDOWS: "delvewheel repair -vv -w {dest_dir} {wheel}"
          CIBW_MANYLINUX_X86_64_IMAGE: manylinux2014
          CIBW_ARCHS_MACOS: x86_64 arm64

      - name: Set-up python 3.10 for upload
        uses: actions/setup-python@v5
        with:
          python-version: "3.10"
      - name: Publish
        env:
          TWINE_USERNAME: ${{ secrets.PYPI_USERNAME }}
          TWINE_PASSWORD: ${{ secrets.PYPI_PASSWORD }}
        run: |
          pip install twine
          twine check ./wheelhouse/*.whl
          twine upload --skip-existing ./wheelhouse/*.whl
```

Key diffs vs the old file: actions upgraded, `submodules: true` on checkout, cibuildwheel bumped to v2.21.3, Python matrix expanded to cp37–cp314, per-version macOS x86_64 skip list replaced with `CIBW_ARCHS_MACOS: x86_64 arm64` (separate wheels per arch, which `[tool.cibuildwheel.macos]` in pyproject.toml mirrors).

- [ ] **Step 2: Commit**

```bash
git add .github/workflows/wheels.yml
git commit -m "ci: update wheels.yml for new build matrix and cibuildwheel v2.21"
```

---

## Phase 4: Push and open PR

### Task 9: Push branch and open PR

**Files:** none

- [ ] **Step 1: Push the branch**

Run: `git push -u origin modernize_python_build`
Expected: pushes the branch and prints a PR-create URL.

- [ ] **Step 2: Open the PR with gh**

```bash
gh pr create --title "Modernize Python build with scikit-build-core" --body "$(cat <<'EOF'
## Summary
- Replace the hand-rolled `setup.py` + `CMakeExtension` build with `scikit-build-core` (modeled after the `bigrandomgraphs` project).
- `CMakeLists.txt` now uses `find_package(pybind11)` (PyPI-installed at build time) and `install(TARGETS … DESTINATION flagserpy/modules)`; the build no longer git-clones pybind11.
- All project metadata moves into `pyproject.toml`. `setup.py`, `setup.cfg`, `MANIFEST.in`, `requirements.txt`, and `flagserpy/_version.py` are removed.
- CI workflows updated: `submodules: true` on checkout, modern action versions, Python matrix 3.7–3.14, cibuildwheel v2.21.

Design spec: `docs/superpowers/specs/2026-05-22-scikit-build-migration-design.md`

## Test plan
- [ ] CI green on Linux / macOS / Windows across the 3.7–3.14 matrix
- [ ] Manually verified `python -m build --wheel .` produces a wheel containing all three `flagserpy/modules/*.so` files
- [ ] Manually verified `python -m build --sdist .` includes the `flagser/` submodule sources
- [ ] Smoke-tested `from flagserpy import flagser_unweighted, flagser_count_unweighted` against the installed wheel
- [ ] Existing `pytest flagserpy/tests` suite passes
EOF
)"
```

Expected: prints PR URL.

- [ ] **Step 3: Report PR URL back to user**

Print the URL returned by `gh pr create` so the user can review.

---

## Fallback: if Python 3.7 wheel builds fail in CI

The spec calls for trying `>=3.7` first and falling back to `>=3.8` if cibuildwheel cannot produce 3.7 wheels. If Task 9's CI run shows 3.7 failing across the board:

- [ ] **Step 1: Edit pyproject.toml**

Change:
- `requires-python = ">=3.7"` → `requires-python = ">=3.8"`
- Drop the `"Programming Language :: Python :: 3.7"` classifier
- In `[tool.cibuildwheel]`, change `build = ["cp37-*", "cp38-*", …]` → `build = ["cp38-*", …]`

- [ ] **Step 2: Edit `.github/workflows/ci.yml`**

Remove `'3.7'` from the `python-version` matrix.

- [ ] **Step 3: Edit `.github/workflows/wheels.yml`**

Change `CIBW_BUILD: "cp37-* cp38-* …"` → `CIBW_BUILD: "cp38-* …"`.

- [ ] **Step 4: Commit and push**

```bash
git add pyproject.toml .github/workflows/ci.yml .github/workflows/wheels.yml
git commit -m "build: drop Python 3.7 support (cibuildwheel no longer produces 3.7 wheels)"
git push
```
