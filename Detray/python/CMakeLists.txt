# This file is part of the ACTS project.
#
# Copyright (C) 2016 CERN for the benefit of the ACTS project
#
# This Source Code Form is subject to the terms of the Mozilla Public
# License, v. 2.0. If a copy of the MPL was not distributed with this
# file, You can obtain one at https://mozilla.org/MPL/2.0/.

# Set up the python stall directories
set(_python_dir "${CMAKE_BINARY_DIR}/python")
file(MAKE_DIRECTORY ${_python_dir})
file(MAKE_DIRECTORY ${_python_dir}/detray)

# Set up the source script in the python directory
configure_file(setup.sh.in ${_python_dir}/setup.sh @ONLY)
install(FILES ${_python_dir}/setup.sh DESTINATION python)

# Set up the init file for the detray python module
set(_target ${_python_dir}/detray/__init__.py)
file(
    CREATE_LINK ${CMAKE_CURRENT_SOURCE_DIR}/detray/__init__.py ${_target}
    SYMBOLIC
)

# Allow access to the python detectors package from the main __init__.py
file(
    CREATE_LINK
        ${CMAKE_CURRENT_SOURCE_DIR}/detray/detectors
        ${_python_dir}/detray/detectors
    SYMBOLIC
)

install(
    FILES ${CMAKE_CURRENT_SOURCE_DIR}/detray/__init__.py
    DESTINATION ${DETRAY_PYTHON_INSTALL_DIR}
)

# Generate stubs for a user-facing Python module using pybind11-stubgen.
#
# _module: public dotted import path of the wrapper (e.g. detray.core)
# _target: the pybind11 module target; its .so is importable as detray.<_target>
#          and carries the real signatures, while <_module> just re-exports it.
# [ARGN]:  extra targets that must be built before stub generation.
#
# Unlike acts (where `acts` is a package and stubgen recurses into the compiled
# submodule), detray.<_module> is a plain module, so stubgen is run twice: once on
# the raw compiled module `detray.<_target>` (produces <_target>.pyi with the real
# signatures) and once on the wrapper `<_module>` (the re-export stub).
function(detray_pybind_generate_stubs _module _target)
    if(NOT DETRAY_GENERATE_PYTHON_STUBS)
        return()
    endif()

    find_program(uv_exe uv REQUIRED)

    set(_extra_deps ${ARGN})
    set(_python_dir "${CMAKE_BINARY_DIR}/python")
    set(_stub_file "${_python_dir}/detray/${_target}.pyi")

    add_custom_command(
        OUTPUT ${_stub_file}
        COMMAND
            ${CMAKE_COMMAND} -E env "PYTHONPATH=${_python_dir}" ${uv_exe} tool
            run --python ${Python_EXECUTABLE} pybind11-stubgen detray.${_target}
            --exit-code --output-dir ${_python_dir} --ignore-invalid-expressions
            ".*" --ignore-unresolved-names ".*"
        COMMAND
            ${CMAKE_COMMAND} -E env "PYTHONPATH=${_python_dir}" ${uv_exe} tool
            run --python ${Python_EXECUTABLE} pybind11-stubgen ${_module}
            --exit-code --output-dir ${_python_dir} --ignore-invalid-expressions
            ".*" --ignore-unresolved-names ".*"
        DEPENDS ${_target} ${_extra_deps}
        COMMENT "Generating python stubs for ${_module}"
        VERBATIM
    )

    add_custom_target(${_target}Stubs ALL DEPENDS ${_stub_file})
endfunction()

if(DETRAY_BUILD_PYTHON_BINDINGS)
    add_subdirectory(Core)
    add_subdirectory(Examples)
endif()

# Install the generated stub files alongside the package.
if(DETRAY_GENERATE_PYTHON_STUBS)
    install(
        DIRECTORY ${_python_dir}/detray/
        DESTINATION ${DETRAY_PYTHON_INSTALL_DIR}
        FILES_MATCHING
        PATTERN "*.pyi"
    )
endif()
