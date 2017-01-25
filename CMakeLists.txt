################################################################################
# Project:  Lib Qt
# Purpose:  CMake build scripts
# Author:   Dmitry Baryshnikov, dmitry.baryshnikov@nexgis.com
################################################################################
# Copyright (C) 2017, NextGIS <info@nextgis.com>
#
# Permission is hereby granted, free of charge, to any person obtaining a
# copy of this software and associated documentation files (the "Software"),
# to deal in the Software without restriction, including without limitation
# the rights to use, copy, modify, merge, publish, distribute, sublicense,
# and/or sell copies of the Software, and to permit persons to whom the
# Software is furnished to do so, subject to the following conditions:
#
# The above copyright notice and this permission notice shall be included
# in all copies or substantial portions of the Software.
#
# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS
# OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
# FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL
# THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
# LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING
# FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER
# DEALINGS IN THE SOFTWARE.
################################################################################
cmake_minimum_required(VERSION 2.8.12 FATAL_ERROR)

project(qt4)

# some init settings
set(CMAKE_COLOR_MAKEFILE ON)
# set path to additional CMake modules
set(CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake ${CMAKE_MODULE_PATH})

if(OSX_FRAMEWORK AND (BUILD_SHARED_LIBS OR BUILD_STATIC_LIBS))
  message(FATAL_ERROR "Only OSX_FRAMEWORK key or any or both BUILD_SHARED_LIBS
                       and BUILD_STATIC_LIBS keys are permitted")
endif()

include(util)
check_version(QT_MAJOR_VERSION QT_MINOR_VERSION QT_REV_VERSION)
set(VERSION "${QT_MAJOR_VERSION}.${QT_MINOR_VERSION}.${QT_REV_VERSION}")
report_version(${PROJECT_NAME} ${VERSION})

if(QT_MAJOR_VERSION LESS 5)
    set(SUBFOLDER "")
    set(CONFIGURE_ARGS -prefix-install -prefix ${CMAKE_INSTALL_PREFIX} -opensource -confirm-license -shared -no-qt3support -no-multimedia -no-gif -no-phonon -system-sqlite -system-zlib -system-libtiff -system-libpng -system-libjpeg -nomake examples -nomake demos -nomake docs -no-rpath)
else()
    set(SUBFOLDER "single/")
    set(CONFIGURE_ARGS "-prefix ${CMAKE_INSTALL_PREFIX} -opensource -confirm-license -shared -no-gif -system-sqlite -system-zlib -system-libpng -system-libjpeg -no-compile-examples")
endif()

if(CMAKE_BUILD_TYPE STREQUAL "Debug")
    set(CONFIGURE_ARGS ${CONFIGURE_ARGS} -debug)
elseif(CMAKE_BUILD_TYPE STREQUAL "RelWithDebInfo")
    set(CONFIGURE_ARGS ${CONFIGURE_ARGS} -debug-and-release)
else()
    set(CONFIGURE_ARGS ${CONFIGURE_ARGS} -release)
endif()

#http://download.qt.io/official_releases/qt/5.7/5.7.1/single/qt-everywhere-opensource-src-5.7.1.tar.gz
set(QT_INPUT_NAME "qt-everywhere-opensource-src-${VERSION}")
set(DOWNLOAD_URL "http://download.qt.io/official_releases/qt/${QT_MAJOR_VERSION}.${QT_MINOR_VERSION}/${VERSION}/${SUBFOLDER}${QT_INPUT_NAME}.tar.gz")
set(OUT_NAME "qt.tar.gz")

if(NOT EXISTS "${CMAKE_CURRENT_BINARY_DIR}/${OUT_NAME}")
    status_message("Download ${DOWNLOAD_URL}")
    file(DOWNLOAD "${DOWNLOAD_URL}" "${CMAKE_CURRENT_BINARY_DIR}/${OUT_NAME}"
         SHOW_PROGRESS
     )
endif()

include(ProcessorCount)
ProcessorCount(CPU_COUNT)
if(NOT CPU_COUNT OR CPU_COUNT EQUAL 0)
    set(CPU_COUNT 1)
endif()

find_package(Git)
if(NOT GIT_FOUND)
  message(FATAL_ERROR "git is required")
endif()

include(FindAnyProject)

macro(create_symlink PATH_TO_LIB LIB_NAME)
    if(OSX_FRAMEWORK)
        warning_message("Create symlink ${PATH_TO_LIB}/lib${LIB_NAME}.so")
        execute_process(COMMAND ${CMAKE_COMMAND} -E create_symlink ${PATH_TO_LIB}/${LIB_NAME} ${PATH_TO_LIB}/lib${LIB_NAME}.so)
    endif()
endmacro()

find_anyproject(ZLIB DEFAULT ON)
if(ZLIB_FOUND)
    set(CONFIGURE_ARGS ${CONFIGURE_ARGS} -I${ZLIB_INCLUDE_DIR})
    if(TARGET ${ZLIB_LIBRARY})
        get_target_property(LINK_SEARCH_PATH ${ZLIB_LIBRARY} LOCATION)
        get_filename_component(PARENT_DIR ${LINK_SEARCH_PATH} DIRECTORY)
        set(CONFIGURE_ARGS ${CONFIGURE_ARGS} -L${PARENT_DIR})

        create_symlink(${PARENT_DIR} "z")
    endif()
endif()

find_anyproject(JPEG DEFAULT ON)
if(JPEG_FOUND)
    set(CONFIGURE_ARGS ${CONFIGURE_ARGS} -I${JPEG_INCLUDE_DIR})
    if(TARGET ${JPEG_LIBRARY})
        get_target_property(LINK_SEARCH_PATH ${JPEG_LIBRARY} LOCATION)
        get_filename_component(PARENT_DIR ${LINK_SEARCH_PATH} DIRECTORY)
        set(CONFIGURE_ARGS ${CONFIGURE_ARGS} -L${PARENT_DIR})

        create_symlink(${PARENT_DIR} "jpeg")
    endif()
endif()

find_anyproject(TIFF DEFAULT ON)
if(TIFF_FOUND)
    set(CONFIGURE_ARGS ${CONFIGURE_ARGS} -I${TIFF_INCLUDE_DIR})
    if(TARGET ${TIFF_LIBRARY})
        get_target_property(LINK_SEARCH_PATH ${TIFF_LIBRARY} LOCATION)
        get_filename_component(PARENT_DIR ${LINK_SEARCH_PATH} DIRECTORY)
        set(CONFIGURE_ARGS ${CONFIGURE_ARGS} -L${PARENT_DIR})

        create_symlink(${PARENT_DIR} "tiff")
    endif()
endif()

find_anyproject(PNG DEFAULT ON)
if(PNG_FOUND)
    set(CONFIGURE_ARGS ${CONFIGURE_ARGS} -I${PNG_INCLUDE_DIR})
    if(TARGET ${PNG_LIBRARY})
        get_target_property(LINK_SEARCH_PATH ${PNG_LIBRARY} LOCATION)
        get_filename_component(PARENT_DIR ${LINK_SEARCH_PATH} DIRECTORY)
        set(CONFIGURE_ARGS ${CONFIGURE_ARGS} -L${PARENT_DIR})

        create_symlink(${PARENT_DIR} "png")
    endif()
endif()

find_anyproject(SQLite3 DEFAULT ON)
if(SQLITE3_FOUND)
    set(CONFIGURE_ARGS ${CONFIGURE_ARGS} -I${SQLITE3_INCLUDE_DIR})
    if(TARGET ${SQLITE3_LIBRARY})
        get_target_property(LINK_SEARCH_PATH ${SQLITE3_LIBRARY} LOCATION)
        get_filename_component(PARENT_DIR ${LINK_SEARCH_PATH} DIRECTORY)
        set(CONFIGURE_ARGS ${CONFIGURE_ARGS} -L${PARENT_DIR})

        create_symlink(${PARENT_DIR} "sqlite3")
    endif()
endif()

find_anyproject(OpenSSL DEFAULT ON)
if(OPENSSL_FOUND)
    set(CONFIGURE_ARGS ${CONFIGURE_ARGS} -I${OPENSSL_INCLUDE_DIR})
    if(TARGET ${OPENSSL_LIBRARY})
        get_target_property(LINK_SEARCH_PATH ${OPENSSL_LIBRARY} LOCATION)
        get_filename_component(PARENT_DIR ${LINK_SEARCH_PATH} DIRECTORY)
        set(CONFIGURE_ARGS ${CONFIGURE_ARGS} -L${PARENT_DIR})

        create_symlink(${PARENT_DIR} "openssl")

        set(CONFIGURE_ARGS ${CONFIGURE_ARGS} -openssl-linked)
        set(ENV{OPENSSL_LIBS} "-L${PARENT_DIR} -lopenssl")
    endif()
endif()

if(WIN32)
    set(CONFIGURE_CMD_NAME configure.exe)
else()
    set(CONFIGURE_CMD_NAME configure)
endif()

message(STATUS "Configure args: ${CONFIGURE_ARGS}")

if(NOT EXISTS ${CMAKE_CURRENT_BINARY_DIR}/${QT_INPUT_NAME})
    status_message("Extract ${CMAKE_CURRENT_BINARY_DIR}/${OUT_NAME}")
    execute_process(COMMAND ${CMAKE_COMMAND} -E tar xzf ${CMAKE_CURRENT_BINARY_DIR}/${OUT_NAME})
endif()

if(APPLE)
    status_message("Apply patch ${CMAKE_CURRENT_SOURCE_DIR}/patch/mac.patch")
    execute_process(COMMAND ${GIT_EXECUTABLE} apply -v ${CMAKE_CURRENT_SOURCE_DIR}/patch/mac.patch
        WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/${QT_INPUT_NAME}
    )
endif()

status_message("Apply patch ${CMAKE_CURRENT_SOURCE_DIR}/patch/openssl.patch")
execute_process(COMMAND ${GIT_EXECUTABLE} apply -v ${CMAKE_CURRENT_SOURCE_DIR}/patch/openssl.patch
    WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/${QT_INPUT_NAME}
)

status_message("Configure")
execute_process(COMMAND ${CMAKE_CURRENT_BINARY_DIR}/${QT_INPUT_NAME}/${CONFIGURE_CMD_NAME} ${CONFIGURE_ARGS}
    WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/${QT_INPUT_NAME}
)

#TODO: Fix on WINDOWS
add_custom_target(build_qt ALL make -j ${CPU_COUNT}
    WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/${QT_INPUT_NAME}
    VERBATIM
)

install(CODE "execute_process(
              COMMAND make install
              WORKING_DIRECTORY  ${CMAKE_CURRENT_BINARY_DIR}/${QT_INPUT_NAME}
             )"
)

if(OSX_FRAMEWORK)
    find_package(PythonInterp REQUIRED)
    install(CODE "execute_process(COMMAND ${PYTHON_EXECUTABLE} ${CMAKE_SOURCE_DIR}/cmake/mac_rpath_fix.py
            WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
        )"
    )
endif()

if (REGISTER_PACKAGE)
    set(INSTALL_INC_DIR ${CMAKE_INSTALL_PREFIX}/include)
    set(INSTALL_LIB_DIR ${CMAKE_INSTALL_PREFIX}/lib)
    set(INSTALL_BIN_DIR ${CMAKE_INSTALL_PREFIX}/bin)

# TODO:   export(TARGETS ${LIB_NAME} FILE ${LIB_NAME}-exports.cmake)

    set(EXPORT_PACKAGE_NAME Qt4)
    set(EXPORT_PACKAGE_NAME_UPPER QT4)
    set(EXPORT_PACKAGE_LIB_NAME qt4)

    export(PACKAGE ${EXPORT_PACKAGE_NAME})

    include(CMakePackageConfigHelpers)
    configure_package_config_file(
        ${CMAKE_SOURCE_DIR}/cmake/PackageConfig.cmake.in
        ${CMAKE_CURRENT_BINARY_DIR}/${EXPORT_PACKAGE_NAME}Config.cmake
        INSTALL_DESTINATION ${CMAKE_CURRENT_BINARY_DIR}
        PATH_VARS INSTALL_INC_DIR )
    write_basic_package_version_file(
        ${CMAKE_CURRENT_BINARY_DIR}/${EXPORT_PACKAGE_NAME}ConfigVersion.cmake
        VERSION ${VERSION}
        COMPATIBILITY AnyNewerVersion )
#else()
#    export(TARGETS ${LIB_NAME} FILE ${LIB_NAME}-exports.cmake EXPORT_LINK_INTERFACE_LIBRARIES)
endif()