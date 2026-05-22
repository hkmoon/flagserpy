cmake_minimum_required(VERSION 3.15)
project(flagser_pybind LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 14)

find_package(pybind11 CONFIG REQUIRED)

set(BINDINGS_DIR "src")

# When scikit-build-core is invoked in editable.mode = "inplace" the source and
# binary directories coincide. In that case the compiled modules must land
# inside the Python package (flagserpy/modules) so that `from .modules.* import ...`
# works without any wheel-install step. For regular (out-of-tree) builds the
# normal install(TARGETS ... DESTINATION flagserpy/modules) rules below apply.
if("${CMAKE_CURRENT_SOURCE_DIR}" STREQUAL "${CMAKE_CURRENT_BINARY_DIR}"
   AND DEFINED SKBUILD)
  set(FLAGSERPY_INPLACE_OUTPUT_DIR
      "${CMAKE_CURRENT_SOURCE_DIR}/flagserpy/modules")
  file(MAKE_DIRECTORY "${FLAGSERPY_INPLACE_OUTPUT_DIR}")
endif()

# flagser
pybind11_add_module(flagser_pybind "${BINDINGS_DIR}/flagser_bindings.cpp")
target_compile_definitions(flagser_pybind PRIVATE RETRIEVE_PERSISTENCE=1 MANY_VERTICES=1)
target_include_directories(flagser_pybind PRIVATE .)
if(DEFINED FLAGSERPY_INPLACE_OUTPUT_DIR)
    set_target_properties(flagser_pybind PROPERTIES
        LIBRARY_OUTPUT_DIRECTORY "${FLAGSERPY_INPLACE_OUTPUT_DIR}$<0:>")
endif()
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
if(DEFINED FLAGSERPY_INPLACE_OUTPUT_DIR)
    set_target_properties(flagser_coeff_pybind PROPERTIES
        LIBRARY_OUTPUT_DIRECTORY "${FLAGSERPY_INPLACE_OUTPUT_DIR}$<0:>")
endif()
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
if(DEFINED FLAGSERPY_INPLACE_OUTPUT_DIR)
    set_target_properties(flagser_count_pybind PROPERTIES
        LIBRARY_OUTPUT_DIRECTORY "${FLAGSERPY_INPLACE_OUTPUT_DIR}$<0:>")
endif()
if(MSVC)
    target_compile_options(flagser_count_pybind PUBLIC $<$<CONFIG:RELEASE>: /Wall /O2>)
    target_compile_options(flagser_count_pybind PUBLIC $<$<CONFIG:DEBUG>:/O1 /DEBUG:FULL /Zi /Zo>)
else()
    target_compile_options(flagser_count_pybind PUBLIC $<$<CONFIG:RELEASE>: -Ofast>)
    target_compile_options(flagser_count_pybind PUBLIC $<$<CONFIG:DEBUG>: -O2 -ggdb -D_GLIBCXX_DEBUG>)
endif()
install(TARGETS flagser_count_pybind DESTINATION flagserpy/modules)
