################################################################################
# Project:  Lib Qt
# Purpose:  CMake build scripts
# Author:   Dmitry Baryshnikov, dmitry.baryshnikov@nexgis.com
################################################################################
# Copyright (C) 2017-2018, NextGIS <info@nextgis.com>
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

set(INSTALL_CMAKECONF_DIR CMake CACHE INTERNAL "Installation directory for cmake config files" FORCE)

include(util)
check_version(QT_MAJOR_VERSION QT_MINOR_VERSION QT_REV_VERSION)
set(VERSION "${QT_MAJOR_VERSION}.${QT_MINOR_VERSION}.${QT_REV_VERSION}")
report_version(${PROJECT_NAME} ${VERSION})


set(SUBFOLDER "")
set(CONFIGURE_ARGS -opensource -confirm-license -no-qt3support -no-phonon -nomake examples -nomake demos -nomake docs ${QT_CONFIGURE_ARGS})

if(OSX_FRAMEWORK)
    set(CONFIGURE_ARGS ${CONFIGURE_ARGS} -framework -no-rpath)
elseif(BUILD_SHARED_LIBS)
    set(CONFIGURE_ARGS ${CONFIGURE_ARGS} -shared)
else()
    set(CONFIGURE_ARGS ${CONFIGURE_ARGS} -static)
endif()

if(WIN32)
    if(MSVC)
        # Visual Studio 2005	Whidbey	8.0	             1400
        # Visual Studio 2008	Orcas	9.0 	         1500
        # Visual Studio 2010	Dev10/Rosario	10.0     1600
        # Visual Studio 2012	Dev11	11.0             1700
        # Visual Studio 2013	Dev12	12.0             1800
        # Visual Studio 2015	Dev14	14.0             1900
        # Visual Studio 2017	Dev15	15.0             1910-1919
        if(MSVC_VERSION EQUAL 1400)
            set(CONFIGURE_ARGS ${CONFIGURE_ARGS} -platform win32-msvc2005)
        elseif(MSVC_VERSION EQUAL 1500)
            set(CONFIGURE_ARGS ${CONFIGURE_ARGS} -platform win32-msvc2008)
        elseif(MSVC_VERSION EQUAL 1600)
            set(CONFIGURE_ARGS ${CONFIGURE_ARGS} -platform win32-msvc2010)
        elseif(MSVC_VERSION EQUAL 1700)
            set(CONFIGURE_ARGS ${CONFIGURE_ARGS} -platform win32-msvc2012)
        elseif(MSVC_VERSION EQUAL 1800)
            set(CONFIGURE_ARGS ${CONFIGURE_ARGS} -platform win32-msvc2013)
        elseif(MSVC_VERSION EQUAL 1900)
            set(CONFIGURE_ARGS ${CONFIGURE_ARGS} -platform win32-msvc2015)
        elseif(MSVC_VERSION GREATER 1900)
            set(CONFIGURE_ARGS ${CONFIGURE_ARGS} -platform win32-msvc2015) # 2017
        endif()
    endif()
endif()

if(CMAKE_BUILD_TYPE STREQUAL "Debug")
    set(CONFIGURE_ARGS ${CONFIGURE_ARGS} -debug)
elseif(CMAKE_BUILD_TYPE STREQUAL "RelWithDebInfo")
    set(CONFIGURE_ARGS ${CONFIGURE_ARGS} -debug-and-release)
else()
    set(CONFIGURE_ARGS ${CONFIGURE_ARGS} -release)
endif()

include(ProcessorCount)
ProcessorCount(CPU_COUNT)
if(NOT CPU_COUNT OR CPU_COUNT EQUAL 0)
    set(CPU_COUNT 1)
endif()

if(CPU_COUNT GREATER 1 AND WIN32)
    set(CONFIGURE_ARGS ${CONFIGURE_ARGS} -mp )
endif()

if(WIN32)
    file(REMOVE_RECURSE "$ENV{USERPROFILE}\\source")
endif()

include(FindAnyProject)

find_anyproject(ICONV DEFAULT ON)
if(ICONV_FOUND)
	set(CONFIGURE_ARGS ${CONFIGURE_ARGS} -iconv)
	set(CONFIGURE_ARGS_INCLUDE_DIRS ${CONFIGURE_ARGS_INCLUDE_DIRS} ${ICONV_INCLUDE_DIRS})
	if(TARGET ${ICONV_LIBRARIES} AND WIN32)
		get_target_property(LINK_SEARCH_PATH ${ICONV_LIBRARIES} IMPORTED_IMPLIB_RELEASE)
        get_filename_component(LINK_NAME ${LINK_SEARCH_PATH} NAME_WE)
		list(APPEND CONFIGURE_ARGS -l ${LINK_NAME})
		set(ENV{ICONV_LIBS} "-L${LINK_SEARCH_PATH}")
		get_filename_component(PARENT_DIR ${LINK_SEARCH_PATH} PATH)
        set(CONFIGURE_ARGS_LINK_LIBS ${CONFIGURE_ARGS_LINK_LIBS} ${PARENT_DIR})
		set(ENV{LIB} "$ENV{LIB};${PARENT_DIR}")
		set(ENV{INCLUDE} "$ENV{INCLUDE};${ICONV_INCLUDE_DIRS}")

        # Check if build needed
        build_if_needed(${LINK_SEARCH_PATH} "ICONV" CPU_COUNT)
	endif()
endif()

find_anyproject(ZLIB DEFAULT ON)
if(ZLIB_FOUND)
    set(CONFIGURE_ARGS_INCLUDE_DIRS ${CONFIGURE_ARGS_INCLUDE_DIRS} ${ZLIB_INCLUDE_DIRS})
    if(TARGET ${ZLIB_LIBRARIES})
        set(CONFIGURE_ARGS ${CONFIGURE_ARGS} -system-zlib)
        get_target_property(LINK_SEARCH_PATH ${ZLIB_LIBRARIES} IMPORTED_LOCATION_RELEASE)
        if(OSX_FRAMEWORK)
            get_filename_component(PARENT_DIR ${LINK_SEARCH_PATH} PATH)

            set(CONFIGURE_ARGS_LINK_LIBS ${CONFIGURE_ARGS_LINK_LIBS} ${PARENT_DIR})

            create_symlink(${PARENT_DIR} "zlib")
            # set(CONFIGURE_ARGS ${CONFIGURE_ARGS} "-lzlib")
            # set(ENV_ARGS "${ENV_ARGS} LIBZ_LIBS=\"-lzlib\"")
            set(ENV{ZLIB_LIBS} "-L${PARENT_DIR} -lzlib")
        elseif(WIN32)
            get_target_property(LINK_SEARCH_PATH_I ${ZLIB_LIBRARIES} IMPORTED_IMPLIB_RELEASE)
            get_filename_component(LINK_NAME ${LINK_SEARCH_PATH_I} NAME_WE)
    		list(APPEND CONFIGURE_ARGS -l ${LINK_NAME})
    		get_filename_component(PARENT_DIR ${LINK_SEARCH_PATH_I} PATH)
    		set(ENV{LIBZ_LIBS} "-L${PARENT_DIR} -l${LINK_NAME}")
            set(CONFIGURE_ARGS_LINK_LIBS ${CONFIGURE_ARGS_LINK_LIBS} ${PARENT_DIR})
    		set(ENV{LIB} "$ENV{LIB};${PARENT_DIR}")
    		set(ENV{INCLUDE} "$ENV{INCLUDE};${ZLIB_INCLUDE_DIRS}")

            get_filename_component(PATH_DIR ${LINK_SEARCH_PATH} PATH)
            set(ENV{PATH} "$ENV{PATH};${PATH_DIR}")
            # Stupid Qt4 build fix
            warning_message("Copy ${LINK_SEARCH_PATH_I} to ${PARENT_DIR}/zdll.lib")
            execute_process(COMMAND ${CMAKE_COMMAND} -E copy ${LINK_SEARCH_PATH_I} ${PARENT_DIR}/zdll.lib)
            warning_message("Copy ${LINK_SEARCH_PATH} to $ENV{USERPROFILE}\\source\\bin\\")
            execute_process(
                COMMAND ${CMAKE_COMMAND} -E make_directory "$ENV{USERPROFILE}\\source" "$ENV{USERPROFILE}\\source\\bin"
                COMMAND ${CMAKE_COMMAND} -E copy_if_different ${LINK_SEARCH_PATH} "$ENV{USERPROFILE}\\source\\bin\\zlib1.dll"
            )
            if(NOT EXISTS $ENV{USERPROFILE}/source/bin/zlib1.dll)
                message(FATAL_ERROR "$ENV{USERPROFILE}\\source\\bin\\zlib1.dll not exists!")
            endif()
        endif()
        # Check if build needed
        build_if_needed(${LINK_SEARCH_PATH} "ZLIB" CPU_COUNT)
    endif()
endif()

find_anyproject(OpenSSL DEFAULT ON)
if(OPENSSL_FOUND)
    set(CONFIGURE_ARGS_INCLUDE_DIRS ${CONFIGURE_ARGS_INCLUDE_DIRS} ${OPENSSL_INCLUDE_DIRS})
    if(TARGET ${OPENSSL_LIBRARIES})
        set(CONFIGURE_ARGS ${CONFIGURE_ARGS} -openssl-linked)
        if(OSX_FRAMEWORK)
            get_target_property(LINK_SEARCH_PATH ${OPENSSL_LIBRARIES} IMPORTED_LOCATION_RELEASE)
            get_filename_component(PARENT_DIR ${LINK_SEARCH_PATH} DIRECTORY)

            set(CONFIGURE_ARGS_LINK_LIBS ${CONFIGURE_ARGS_LINK_LIBS} ${PARENT_DIR})

            create_symlink(${PARENT_DIR} "openssl")
            # set(CONFIGURE_ARGS ${CONFIGURE_ARGS} "-lopenssl")
            # set(ENV_ARGS "${ENV_ARGS} OPENSSL_LIBS=\"-lopenssl\"")
            set(ENV{OPENSSL_LIBS} "-L${PARENT_DIR} -lopenssl")

        elseif(WIN32)
            get_target_property(LINK_SEARCH_PATH_I ${OPENSSL_LIBRARIES} IMPORTED_IMPLIB_RELEASE)
            get_filename_component(LINK_NAME ${LINK_SEARCH_PATH_I} NAME_WE)
    		list(APPEND CONFIGURE_ARGS -l ${LINK_NAME} OPENSSL_LIBS="-l${LINK_NAME}")
            set(ENV{OPENSSL_LIBS} "-L${LINK_SEARCH_PATH_I} -l${LINK_NAME}")
            get_filename_component(PARENT_DIR ${LINK_SEARCH_PATH_I} PATH)
            set(CONFIGURE_ARGS_LINK_LIBS ${CONFIGURE_ARGS_LINK_LIBS} ${PARENT_DIR})
            set(ENV{LIB} "$ENV{LIB};${PARENT_DIR}")
            set(ENV{INCLUDE} "$ENV{INCLUDE};${OPENSSL_INCLUDE_DIRS}")
        endif()
        # Check if build needed
        build_if_needed(${LINK_SEARCH_PATH} "OpenSSL" ${CPU_COUNT})
    endif()
endif()

if(NOT WIN32)
find_anyproject(Freetype DEFAULT ON)
endif()
if(FREETYPE_FOUND)
    set(CONFIGURE_ARGS_INCLUDE_DIRS ${CONFIGURE_ARGS_INCLUDE_DIRS} ${FREETYPE_INCLUDE_DIRS})
    if(TARGET ${FREETYPE_LIBRARIES})
        set(CONFIGURE_ARGS ${CONFIGURE_ARGS} -system-freetype)
        if(OSX_FRAMEWORK)
            get_target_property(LINK_SEARCH_PATH ${FREETYPE_LIBRARIES} IMPORTED_LOCATION_RELEASE)
            get_filename_component(PARENT_DIR ${LINK_SEARCH_PATH} DIRECTORY)

            set(CONFIGURE_ARGS_LINK_LIBS ${CONFIGURE_ARGS_LINK_LIBS} ${PARENT_DIR})

            create_symlink(${PARENT_DIR} "freetype")

            # set(CONFIGURE_ARGS ${CONFIGURE_ARGS} "-lfreetype")
            # set(ENV_ARGS "${ENV_ARGS} FREETYPE_LIBS=\"-lfreetype\"")
            set(ENV{FREETYPE_LIBS} "-L${PARENT_DIR} -lfreetype")

        elseif(WIN32)
            get_target_property(LINK_SEARCH_PATH ${FREETYPE_LIBRARIES} IMPORTED_IMPLIB_RELEASE)
            get_filename_component(LINK_NAME ${LINK_SEARCH_PATH} NAME_WE)
    		list(APPEND CONFIGURE_ARGS -l ${LINK_NAME})
            set(ENV{FREETYPE_LIBS} "-L${LINK_SEARCH_PATH}")
            get_filename_component(PARENT_DIR ${LINK_SEARCH_PATH} PATH)
            set(CONFIGURE_ARGS_LINK_LIBS ${CONFIGURE_ARGS_LINK_LIBS} ${PARENT_DIR})
            set(ENV{LIB} "$ENV{LIB};${PARENT_DIR}")
            set(ENV{INCLUDE} "$ENV{INCLUDE};${FREETYPE_INCLUDE_DIRS}")
            file(WRITE ${FREETYPE_INCLUDE_DIRS}/freetype2/unistd.h "/* dummy */")
        endif()
        # Check if build needed
        build_if_needed(${LINK_SEARCH_PATH} "Freetype" ${CPU_COUNT})
    endif()
endif()

# TODO: -harfbuzz ............ Select used HarfBuzz-NG [system/qt/no]

find_anyproject(JPEG DEFAULT ON)
if(JPEG_FOUND)
    set(CONFIGURE_ARGS_INCLUDE_DIRS ${CONFIGURE_ARGS_INCLUDE_DIRS} ${JPEG_INCLUDE_DIRS})
    if(TARGET ${JPEG_LIBRARIES})
        set(CONFIGURE_ARGS ${CONFIGURE_ARGS} -system-libjpeg)
        if(OSX_FRAMEWORK)
            get_target_property(LINK_SEARCH_PATH ${JPEG_LIBRARIES} IMPORTED_LOCATION_RELEASE)
            get_filename_component(PARENT_DIR ${LINK_SEARCH_PATH} DIRECTORY)

            set(CONFIGURE_ARGS_LINK_LIBS ${CONFIGURE_ARGS_LINK_LIBS} ${PARENT_DIR})

            create_symlink(${PARENT_DIR} "jpeg")

            # set(CONFIGURE_ARGS ${CONFIGURE_ARGS} "-ljpeg")
            # set(ENV_ARGS "${ENV_ARGS} LIBJPEG_LIBS=\"-ljpeg\"")
            set(ENV{LIBJPEG_LIBS} "-L${PARENT_DIR} -ljpeg")
        elseif(WIN32)
            get_target_property(LINK_SEARCH_PATH_I ${JPEG_LIBRARIES} IMPORTED_IMPLIB_RELEASE)
            get_filename_component(LINK_NAME ${LINK_SEARCH_PATH_I} NAME_WE)
    		list(APPEND CONFIGURE_ARGS -l ${LINK_NAME})
            get_filename_component(PARENT_DIR ${LINK_SEARCH_PATH_I} PATH)
    		set(ENV{LIBJPEG_LIBS} "-L${PARENT_DIR} -l${LINK_NAME}")
            set(CONFIGURE_ARGS_LINK_LIBS ${CONFIGURE_ARGS_LINK_LIBS} ${PARENT_DIR})
            set(ENV{LIB} "$ENV{LIB};${PARENT_DIR}")
            set(ENV{INCLUDE} "$ENV{INCLUDE};${JPEG_INCLUDE_DIRS}")

            # Stupid Qt4 build fix
            warning_message("Copy ${LINK_SEARCH_PATH_I} to ${PARENT_DIR}/libjpeg.lib")
            execute_process(COMMAND ${CMAKE_COMMAND} -E copy ${LINK_SEARCH_PATH_I} ${PARENT_DIR}/libjpeg.lib)
        endif()
        # Check if build needed
        build_if_needed(${LINK_SEARCH_PATH} "JPEG" ${CPU_COUNT})
    endif()
endif()

find_anyproject(PNG DEFAULT ON)
if(PNG_FOUND)
    set(CONFIGURE_ARGS_INCLUDE_DIRS ${CONFIGURE_ARGS_INCLUDE_DIRS} ${PNG_INCLUDE_DIRS})
    if(TARGET ${PNG_LIBRARIES})
        set(CONFIGURE_ARGS ${CONFIGURE_ARGS} -system-libpng)
        if(OSX_FRAMEWORK)
            get_target_property(LINK_SEARCH_PATH ${PNG_LIBRARIES} IMPORTED_LOCATION_RELEASE)
            get_filename_component(PARENT_DIR ${LINK_SEARCH_PATH} DIRECTORY)

            set(CONFIGURE_ARGS_LINK_LIBS ${CONFIGURE_ARGS_LINK_LIBS} ${PARENT_DIR})

            create_symlink(${PARENT_DIR} "png")

            # set(CONFIGURE_ARGS ${CONFIGURE_ARGS} "-lpng")
            # set(ENV_ARGS "${ENV_ARGS} LIBPNG_LIBS=\"-lpng\"")
            set(ENV{LIBPNG_LIBS} "-L${PARENT_DIR} -lpng")
        elseif(WIN32)
            get_target_property(LINK_SEARCH_PATH_I ${PNG_LIBRARIES} IMPORTED_IMPLIB_RELEASE)
            get_filename_component(LINK_NAME ${LINK_SEARCH_PATH_I} NAME_WE)
    		list(APPEND CONFIGURE_ARGS -l ${LINK_NAME})
            get_filename_component(PARENT_DIR ${LINK_SEARCH_PATH_I} PATH)
            set(ENV{LIBPNG_LIBS} "-L${PARENT_DIR} -l${LINK_NAME}")
            set(CONFIGURE_ARGS_LINK_LIBS ${CONFIGURE_ARGS_LINK_LIBS} ${PARENT_DIR})
            set(ENV{LIB} "$ENV{LIB};${PARENT_DIR}")
            set(ENV{INCLUDE} "$ENV{INCLUDE};${PNG_INCLUDE_DIRS}")

            # Stupid Qt4 build fix
            warning_message("Copy ${LINK_SEARCH_PATH_I} to ${PARENT_DIR}/libpng.lib")
            execute_process(COMMAND ${CMAKE_COMMAND} -E copy ${LINK_SEARCH_PATH_I} ${PARENT_DIR}/libpng.lib)
        endif()
        # Check if build needed
        build_if_needed(${LINK_SEARCH_PATH} "PNG" ${CPU_COUNT})
    endif()
endif()

find_anyproject(TIFF DEFAULT ON)
if(TIFF_FOUND)
    set(CONFIGURE_ARGS_INCLUDE_DIRS ${CONFIGURE_ARGS_INCLUDE_DIRS} ${TIFF_INCLUDE_DIRS})
    if(TARGET ${TIFF_LIBRARIES})
        set(CONFIGURE_ARGS ${CONFIGURE_ARGS} -system-libtiff)
        if(OSX_FRAMEWORK)
            get_target_property(LINK_SEARCH_PATH ${TIFF_LIBRARIES} IMPORTED_LOCATION_RELEASE)
            get_filename_component(PARENT_DIR ${LINK_SEARCH_PATH} DIRECTORY)

            set(CONFIGURE_ARGS_LINK_LIBS ${CONFIGURE_ARGS_LINK_LIBS} ${PARENT_DIR})

            create_symlink(${PARENT_DIR} "tiff")

            # set(CONFIGURE_ARGS ${CONFIGURE_ARGS} "-ltiff")
            # set(ENV_ARGS "${ENV_ARGS} TIFF_LIBS=\"-ltiff\"")
            set(ENV{TIFF_LIBS} "-L${PARENT_DIR} -ltiff")
        elseif(WIN32)
            get_target_property(LINK_SEARCH_PATH_I ${TIFF_LIBRARIES} IMPORTED_IMPLIB_RELEASE)
            get_filename_component(LINK_NAME ${LINK_SEARCH_PATH_I} NAME_WE)
    		list(APPEND CONFIGURE_ARGS -l ${LINK_NAME})
            get_filename_component(PARENT_DIR ${LINK_SEARCH_PATH_I} PATH)
            set(ENV{TIFF_LIBS} "-L${PARENT_DIR} -l${LINK_NAME}")
            set(CONFIGURE_ARGS_LINK_LIBS ${CONFIGURE_ARGS_LINK_LIBS} ${PARENT_DIR})
            set(ENV{LIB} "$ENV{LIB};${PARENT_DIR}")
            set(ENV{INCLUDE} "$ENV{INCLUDE};${TIFF_INCLUDE_DIRS}")

            # Stupid Qt4 build fix
            warning_message("Copy ${LINK_SEARCH_PATH_I} to ${PARENT_DIR}/libtiff.lib")
            execute_process(COMMAND ${CMAKE_COMMAND} -E copy ${LINK_SEARCH_PATH_I} ${PARENT_DIR}/libtiff.lib)
        endif()
        # Check if build needed
        build_if_needed(${LINK_SEARCH_PATH} "TIFF" ${CPU_COUNT})
    endif()
endif()

# find_anyproject(GIF DEFAULT ON)
# if(GIF_FOUND)
#     set(CONFIGURE_ARGS_INCLUDE_DIRS ${CONFIGURE_ARGS_INCLUDE_DIRS} ${GIF_INCLUDE_DIRS})
#     if(TARGET ${GIF_LIBRARIES})
#         set(CONFIGURE_ARGS ${CONFIGURE_ARGS} -system-gif)
#         if(WIN32)
#             get_target_property(LINK_SEARCH_PATH ${GIF_LIBRARIES} IMPORTED_IMPLIB_RELEASE)
#         else()
#             get_target_property(LINK_SEARCH_PATH ${GIF_LIBRARIES} LOCATION)
#         endif()
#         get_filename_component(PARENT_DIR ${LINK_SEARCH_PATH} DIRECTORY)
#         set(CONFIGURE_ARGS ${CONFIGURE_ARGS} -L ${PARENT_DIR})
#
#         if(WIN32)
#             warning_message("Copy ${LINK_SEARCH_PATH}")
#             execute_process(COMMAND ${CMAKE_COMMAND} -E copy ${LINK_SEARCH_PATH} ${PARENT_DIR}/libgiff.lib)
#         endif()
#
#         create_symlink(${PARENT_DIR} "giff")
#     endif()
# endif()

if(NOT WIN32) # On windows use internal qt sqlite
    find_anyproject(SQLite3 DEFAULT ON)
endif()
if(SQLITE3_FOUND)
    set(CONFIGURE_ARGS_INCLUDE_DIRS ${CONFIGURE_ARGS_INCLUDE_DIRS} ${SQLITE3_INCLUDE_DIRS})
    if(TARGET ${SQLITE3_LIBRARIES})
        set(CONFIGURE_ARGS ${CONFIGURE_ARGS} -system-sqlite)
        get_target_property(LINK_SEARCH_PATH ${SQLITE3_LIBRARIES} IMPORTED_LOCATION_RELEASE)
        get_filename_component(PARENT_DIR ${LINK_SEARCH_PATH} DIRECTORY)

        set(CONFIGURE_ARGS_LINK_LIBS ${CONFIGURE_ARGS_LINK_LIBS} ${PARENT_DIR})

        create_symlink(${PARENT_DIR} "sqlite3")

        # set(CONFIGURE_ARGS ${CONFIGURE_ARGS} "-lsqlite3")
        # set(ENV_ARGS "${ENV_ARGS} SQLITE_LIBS=\"-lsqlite3\"")
        set(ENV{SQLITE_LIBS} "-L${PARENT_DIR} -lsqlite3")

        # Check if build needed
        build_if_needed(${LINK_SEARCH_PATH} "SQLite3" ${CPU_COUNT})
    endif()
endif()

find_anyproject(PostgreSQL DEFAULT ON NAMES PQ pq libpq)
if(POSTGRESQL_FOUND)
    set(CONFIGURE_ARGS_INCLUDE_DIRS ${CONFIGURE_ARGS_INCLUDE_DIRS} ${POSTGRESQL_INCLUDE_DIRS})
    if(TARGET ${POSTGRESQL_LIBRARIES})
        set(CONFIGURE_ARGS ${CONFIGURE_ARGS} -plugin-sql-psql)
        if(OSX_FRAMEWORK)
            # set(CONFIGURE_ARGS ${CONFIGURE_ARGS} -feature-psql)
            get_target_property(LINK_SEARCH_PATH ${POSTGRESQL_LIBRARIES} IMPORTED_LOCATION_RELEASE)
            get_filename_component(PARENT_DIR ${LINK_SEARCH_PATH} DIRECTORY)

            set(CONFIGURE_ARGS_LINK_LIBS ${CONFIGURE_ARGS_LINK_LIBS} ${PARENT_DIR})

            create_symlink(${PARENT_DIR} "pq")

            # set(CONFIGURE_ARGS ${CONFIGURE_ARGS} "-lpq")
            # set(ENV_ARGS "${ENV_ARGS} PSQL_LIBS=\"-lpq\"")
            set(ENV{PSQL_LIBS} "-L${PARENT_DIR} -lpq")
        elseif(WIN32)
            get_target_property(LINK_SEARCH_PATH ${POSTGRESQL_LIBRARIES} IMPORTED_IMPLIB_RELEASE)
            get_filename_component(LINK_NAME ${LINK_SEARCH_PATH} NAME_WE)
    		list(APPEND CONFIGURE_ARGS -l ${LINK_NAME})
            set(ENV{PSQL_LIBS} "-L${LINK_SEARCH_PATH}")
            get_filename_component(PARENT_DIR ${LINK_SEARCH_PATH} PATH)
            set(CONFIGURE_ARGS_LINK_LIBS ${CONFIGURE_ARGS_LINK_LIBS} ${PARENT_DIR})
            set(ENV{LIB} "$ENV{LIB};${PARENT_DIR}")
            set(ENV{INCLUDE} "$ENV{INCLUDE};${POSTGRESQL_INCLUDE_DIRS}")
        endif()
        # Check if build needed
        build_if_needed(${LINK_SEARCH_PATH} "PostgreSQL" ${CPU_COUNT})
    endif()
endif()

# Add OBDC support
if(WIN32)
    set(CONFIGURE_ARGS ${CONFIGURE_ARGS} -plugin-sql-odbc)
endif()

foreach(CONF_INC_DIR ${CONFIGURE_ARGS_INCLUDE_DIRS})
    list(APPEND CONFIGURE_ARGS -I ${CONF_INC_DIR})
endforeach()

foreach(CONF_LINK_LIB ${CONFIGURE_ARGS_LINK_LIBS})
    # if(OSX_FRAMEWORK)
        # list(APPEND CONFIGURE_ARGS -F${CONF_LINK_LIB})
    # else()
        list(APPEND CONFIGURE_ARGS -L ${CONF_LINK_LIB})
    # endif()
endforeach()

#http://download.qt.io/official_releases/qt/5.7/5.7.1/single/qt-everywhere-opensource-src-5.7.1.tar.gz
#http://mirrors.sohu.com/qt-all/archive/qt/4.8/4.8.7/qt-everywhere-opensource-src-4.8.7.tar.gz
set(QT_INPUT_NAME "qt-everywhere-opensource-src-${VERSION}")
#set(DOWNLOAD_URL "http://download.qt.io/archive/qt/${QT_MAJOR_VERSION}.${QT_MINOR_VERSION}/${VERSION}/${SUBFOLDER}${QT_INPUT_NAME}.tar.gz")
# set(DOWNLOAD_URL "http://mirrors.sohu.com/qt-all/archive/qt/${QT_MAJOR_VERSION}.${QT_MINOR_VERSION}/${VERSION}/${SUBFOLDER}${QT_INPUT_NAME}.tar.gz")
# https://ftp.osuosl.org/pub/blfs/conglomeration/qt4/
set(DOWNLOAD_URL "https://ftp.osuosl.org/pub/blfs/conglomeration/qt4/${QT_INPUT_NAME}.tar.gz")
set(OUT_NAME "qt.tar.gz")

if(NOT EXISTS "${CMAKE_CURRENT_BINARY_DIR}/${OUT_NAME}")
    status_message("Download ${DOWNLOAD_URL}")
    file(DOWNLOAD "${DOWNLOAD_URL}" "${CMAKE_CURRENT_BINARY_DIR}/${OUT_NAME}"
         SHOW_PROGRESS
     )
endif()

if(NOT EXISTS ${CMAKE_CURRENT_BINARY_DIR}/${QT_INPUT_NAME})
    status_message("Extract ${CMAKE_CURRENT_BINARY_DIR}/${OUT_NAME}")
    execute_process(COMMAND ${CMAKE_COMMAND} -E tar xzf ${CMAKE_CURRENT_BINARY_DIR}/${OUT_NAME})
endif()

find_package(Git)
if(NOT GIT_FOUND)
  message(FATAL_ERROR "git is required")
endif()

if(APPLE)
    status_message("Apply patch ${CMAKE_CURRENT_SOURCE_DIR}/patch/mac.patch")
    execute_process(COMMAND ${GIT_EXECUTABLE} apply --ignore-whitespace -v  ${CMAKE_CURRENT_SOURCE_DIR}/patch/mac.patch
        WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/${QT_INPUT_NAME}
        OUTPUT_QUIET ERROR_QUIET
    )
    status_message("Apply patch ${CMAKE_CURRENT_SOURCE_DIR}/pointer_comparison_to_0.patch")
    execute_process(COMMAND ${GIT_EXECUTABLE} apply --ignore-whitespace -v  ${CMAKE_CURRENT_SOURCE_DIR}/patch/pointer_comparison_to_0.patch
        WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/${QT_INPUT_NAME}
        OUTPUT_QUIET ERROR_QUIET
    )
    status_message("Apply patch ${CMAKE_CURRENT_SOURCE_DIR}/xcode91.patch")
    execute_process(COMMAND ${GIT_EXECUTABLE} apply --ignore-whitespace -v  ${CMAKE_CURRENT_SOURCE_DIR}/patch/xcode91.patch
        WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/${QT_INPUT_NAME}
        OUTPUT_QUIET ERROR_QUIET
    )
endif()

if(OPENSSL_FOUND AND OPENSSL_VERSION VERSION_GREATER "1.0.2")
    status_message("Apply patch ${CMAKE_CURRENT_SOURCE_DIR}/patch/openssl.patch")
    execute_process(COMMAND ${GIT_EXECUTABLE} apply --ignore-whitespace -v  ${CMAKE_CURRENT_SOURCE_DIR}/patch/openssl.patch
        WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/${QT_INPUT_NAME}
        OUTPUT_QUIET ERROR_QUIET
    )
endif()

status_message("Apply patch ${CMAKE_CURRENT_SOURCE_DIR}/patch/fix_build_with_msvc2015.patch")
execute_process(COMMAND ${GIT_EXECUTABLE} apply --ignore-whitespace -v  ${CMAKE_CURRENT_SOURCE_DIR}/patch/fix_build_with_msvc2015.patch
    WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/${QT_INPUT_NAME}
    OUTPUT_QUIET ERROR_QUIET
)

set(CMAKE_INSTALL_PREFIX ${CMAKE_CURRENT_BINARY_DIR}/inst)

message(STATUS "Configure args: ${CONFIGURE_ARGS}")
status_message("Configure in ${CMAKE_CURRENT_BINARY_DIR}/${QT_INPUT_NAME}")

if(WIN32)
    set(CONFIGURE_CMD_NAME configure.exe)
    get_filename_component(VC_COMPILER_DIR ${CMAKE_CXX_COMPILER} DIRECTORY)
    find_file(VCVARSALL_BAT
        NAMES vsdevcmd.bat vcvarsall.bat
        PATHS "${VC_COMPILER_DIR}/.." "${VC_COMPILER_DIR}/../.." "${VC_COMPILER_DIR}/../../../../../../../.." "${VC_COMPILER_DIR}/../../../../../../.."
        PATH_SUFFIXES "Common7/Tools" "Tools"
    )

    if(DEFINED CMAKE_GENERATOR)
        # TODO: Need different arguments for vsdevcmd.bat and vcvarsall.bat. Now support only VC2017
        string(FIND ${CMAKE_GENERATOR} "Win64" POS)
        if(POS GREATER -1)
            set(VCVARSALL_BAT_ARG -arch=amd64)
        else()
            set(VCVARSALL_BAT_ARG -arch=x86)
        endif()
    endif()

    message(STATUS "VCVARSALL_BAT: ${VCVARSALL_BAT} ${VCVARSALL_BAT_ARG}")

    set(CONFIGURE_CMD ${VCVARSALL_BAT} ${VCVARSALL_BAT_ARG} && ${CMAKE_CURRENT_BINARY_DIR}/${QT_INPUT_NAME}/${CONFIGURE_CMD_NAME} ${CONFIGURE_ARGS})

    execute_process(
        COMMAND ${CONFIGURE_CMD} -prefix ${CMAKE_INSTALL_PREFIX}
        WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/${QT_INPUT_NAME}
    )

    add_custom_target(build_qt ALL
        COMMAND ${VCVARSALL_BAT} ${VCVARSALL_BAT_ARG} && nmake
        WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/${QT_INPUT_NAME}
        COMMENT "${VCVARSALL_BAT} ${VCVARSALL_BAT_ARG} && nmake"
        VERBATIM
    )
else()
    set(CONFIGURE_CMD_NAME configure)
    set(CONFIGURE_CMD ${CMAKE_CURRENT_BINARY_DIR}/${QT_INPUT_NAME}/${CONFIGURE_CMD_NAME} ${CONFIGURE_ARGS})

    execute_process(
        COMMAND ${CONFIGURE_CMD} -prefix ${CMAKE_INSTALL_PREFIX}
        WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/${QT_INPUT_NAME}
    )

    add_custom_target(build_qt ALL
        COMMAND make -j ${CPU_COUNT}
        WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/${QT_INPUT_NAME}
        COMMENT "${CMAKE_CURRENT_BINARY_DIR}/${QT_INPUT_NAME}/make -j ${CPU_COUNT}"
        VERBATIM
    )
endif()

# Get cpack archive name
get_cpack_filename(${VERSION} PROJECT_CPACK_FILENAME)

if(WIN32)
    set(MAKE_CMD ${VCVARSALL_BAT} ${VCVARSALL_BAT_ARG} && nmake install)
else()
    set(MAKE_CMD make install)
endif()

add_custom_target(preinstall_qt ALL
    COMMAND ${MAKE_CMD}
    DEPENDS build_qt
    WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/${QT_INPUT_NAME}
)

if(OSX_FRAMEWORK)
    find_package(PythonInterp REQUIRED)
    # FIXME: Uncomment fix zlib paths for WITH_ZLIB_EXTERNAL
    install(CODE "execute_process(COMMAND ${PYTHON_EXECUTABLE} ${CMAKE_SOURCE_DIR}/cmake/mac_rpath_fix.py
            WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
        )"
    )
endif()

set(PACKAGE_NAME ${PROJECT_NAME})
string(TOUPPER ${PACKAGE_NAME} PACKAGE_UPPER_NAME)

if(REGISTER_PACKAGE)
    # Export the package for use from the build-tree
    # (this registers the build-tree with a global CMake-registry)
    export(PACKAGE ${PACKAGE_UPPER_NAME})
endif()

# Create the <Package>Config.cmake file
configure_file(cmake/PackageConfig.cmake.in
    ${PROJECT_BINARY_DIR}/${PACKAGE_UPPER_NAME}Config.cmake @ONLY)

if(NOT SKIP_INSTALL_LIBRARIES AND NOT SKIP_INSTALL_ALL )
    install(DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/inst/ DESTINATION "." USE_SOURCE_PERMISSIONS)
    install(FILES
      ${PROJECT_BINARY_DIR}/${PACKAGE_UPPER_NAME}Config.cmake
      DESTINATION ${INSTALL_CMAKECONF_DIR} COMPONENT dev)
endif()

# Archiving ====================================================================

set(CPACK_PACKAGE_NAME "${PACKAGE_NAME}")
set(CPACK_PACKAGE_VENDOR "${PACKAGE_VENDOR}")
set(CPACK_PACKAGE_VERSION "${VERSION}")
set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "${PACKAGE_NAME} Installation")
set(CPACK_PACKAGE_RELOCATABLE TRUE)
set(CPACK_ARCHIVE_COMPONENT_INSTALL ON)
set(CPACK_GENERATOR "ZIP")
set(CPACK_MONOLITHIC_INSTALL ON)
set(CPACK_STRIP_FILES TRUE)

set(CPACK_PACKAGE_FILE_NAME ${PROJECT_CPACK_FILENAME})

include (CPack)
