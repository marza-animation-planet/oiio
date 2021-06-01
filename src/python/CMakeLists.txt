# Copyright 2008-present Contributors to the OpenImageIO project.
# SPDX-License-Identifier: BSD-3-Clause
# https://github.com/OpenImageIO/oiio/blob/master/LICENSE.md

# from pythonutils.cmake
checked_find_package (Pybind11 REQUIRED
                      VERSION_MIN 2.4.2)


if (CMAKE_SYSTEM_NAME STREQUAL "Darwin")
    set (PYTHON_VISIBILITY_MAP_FILE "${PROJECT_SOURCE_DIR}/src/build-scripts/exportsymbols_python_osx.map")
else ()
    set (PYTHON_VISIBILITY_MAP_FILE "${PROJECT_SOURCE_DIR}/src/build-scripts/hidesymbols_python.map")
endif ()
file (GLOB python_srcs *.cpp)
setup_python_module (TARGET    PyOpenImageIO
                     MODULE    OpenImageIO
                     VISMAP    ${PYTHON_VISIBILITY_MAP_FILE}
                     SOURCES   ${python_srcs}
                     LIBS      OpenImageIO
                     )
