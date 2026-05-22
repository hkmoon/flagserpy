# Migrate flagserpy build to scikit-build-core

**Date:** 2026-05-22
**Status:** Approved
**Reference project:** `bigrandomgraphs` (uses scikit-build-core + pybind11 from PyPI)

## Goal

Replace the hand-rolled `setup.py` + `CMakeExtension` build system with `scikit-build-core`. The runtime behavior of the package — the Python API, the three compiled pybind modules, the integration with the `flagser` C++ library — does not change. Only the *how-to-build* changes.

## Non-goals

- Refactoring Python code in `flagserpy/`.
- Changing the C++ bindings in `src/`.
- Removing the `flagser` git submodule.
- Unrelated CI improvements beyond what the build-system change requires.

## Architecture

```
                BEFORE                                AFTER
  pip install .                            pip install .
        │                                        │
        ▼                                        ▼
  setup.py (CMakeBuild)                   scikit_build_core.build backend
        │                                        │
        ├─ git clone pybind11/                   ├─ pip installs pybind11 to build env
        ├─ cmake configure (custom flags)        ├─ cmake configure (scikit-build-core
        ├─ cmake build                           │  injects PYTHON_EXECUTABLE,
        └─ copy .so → flagserpy/modules/         │  pybind11_DIR, etc.)
                                                 ├─ cmake build
                                                 └─ cmake install (DESTINATION
                                                    flagserpy/modules) → wheel
```

Wheel layout is unchanged: `flagserpy/modules/flagser_pybind*.so`, `flagserpy/modules/flagser_coeff_pybind*.so`, `flagserpy/modules/flagser_count_pybind*.so`. Python imports continue to work without code changes.

## Decisions

| Question | Decision |
|---|---|
| Where do compiled modules install? | `flagserpy/modules/` (same as today) |
| Python version floor | Try `>=3.7` first; fall back to `>=3.8` if CI fails |
| Version metadata source | Static `0.4.7` in pyproject.toml; delete `_version.py` |
| CI workflows | Update both `ci.yml` and `wheels.yml` to match new build |
| pybind11 source | PyPI (build-time dep), not git clone |

## File-level changes

### Replace

**`pyproject.toml`** — rewrite with full `[project]` metadata, `[tool.scikit-build]`, `[tool.cibuildwheel]`, and pytest/flake8 config moved in from `setup.cfg`. Static version `0.4.7`.

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
    "pytest", "pytest-timeout", "pytest-cov",
    "pytest-azurepipelines", "pytest-benchmark", "flake8",
]
doc = [
    "sphinx", "sphinx-gallery", "sphinx-issues",
    "sphinx_rtd_theme", "numpydoc",
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

**`CMakeLists.txt`** — bump minimum to 3.15; replace `add_subdirectory(pybind11)` with `find_package(pybind11 CONFIG REQUIRED)`; add `install(TARGETS … DESTINATION flagserpy/modules)` for each of the three modules. Keep the three targets, their compile definitions, compile options, and `target_include_directories(... .)` so `#include <flagser/src/flagser.cpp>` still resolves.

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

### Delete

- `setup.py`
- `setup.cfg`
- `MANIFEST.in`
- `requirements.txt`
- `flagserpy/_version.py`

### Edit

- **`flagserpy/__init__.py`** — drop `from ._version import __version__`; hardcode `__version__ = "0.4.7"` directly in the file. (We do not use `importlib.metadata` because Python 3.7 lacks it in stdlib; we want a single source-of-truth pattern that works across the whole supported matrix without backport deps. Future bumps update both `pyproject.toml` and this line.)
- **`.gitignore`** — drop the `# Pybind11\npybind11` lines (we no longer git-clone it).

### CI updates

**`.github/workflows/ci.yml`:**
- Add `submodules: true` to the `actions/checkout` step.
- Update Python matrix to `[3.7, 3.8, 3.9, '3.10', '3.11', '3.12', '3.13', '3.14']`.
- Upgrade actions to current versions (`actions/checkout@v4`, `actions/setup-python@v5`, `actions/cache@v4`, `actions/upload-artifact@v4`).
- Fix the `pytest --cov pyflagser` typo → `pytest --cov flagserpy`.
- `pip install -e ".[doc,tests]"` keeps working (pip uses the new backend transparently).

**`.github/workflows/wheels.yml`:**
- Add `submodules: true` to the `actions/checkout` step.
- Update `CIBW_BUILD` to `"cp37-* cp38-* cp39-* cp310-* cp311-* cp312-* cp313-* cp314-*"`.
- Update `CIBW_SKIP` to drop the per-version macOS x86_64 skip list (now controlled by `[tool.cibuildwheel.macos] archs`).
- Upgrade `pypa/cibuildwheel` to a current version (≥ v2.21).

## Risks and validations

1. **flagser submodule must be checked out at build time.** Both workflows need `submodules: true`. For sdist installs from PyPI, scikit-build-core includes git-tracked files by default, so submodule contents get bundled — verify by inspecting `python -m build --sdist` output locally.
2. **Python 3.7 floor:** If `cp37-*` builds fail in CI (cibuildwheel's newer manylinux images may have dropped 3.7), bump `requires-python` to `>=3.8`, drop `cp37-*` and the `3.7` classifier, and update `__init__.py` if any 3.7-only branch was added.
3. **Local validation before push:**
   - `git submodule update --init` to ensure flagser is checked out.
   - `pip install build && python -m build --wheel .` — confirms scikit-build-core invocation succeeds and the wheel contains all three `.so` files under `flagserpy/modules/`.
   - `pip install dist/flagserpy-0.4.7-*.whl && python -c "from flagserpy import flagser_unweighted, flagser_count_unweighted; print('ok')"` — confirms imports work end-to-end.
   - `pytest` — runs the existing test suite against the installed wheel.
4. **The Windows `-A x64` hack** in old setup.py is dropped: scikit-build-core picks the right generator/arch.

## Implementation branch

Per user request: create branch `modernize_python_build`, do the migration, validate, push, open PR.
