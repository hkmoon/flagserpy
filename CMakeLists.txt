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
