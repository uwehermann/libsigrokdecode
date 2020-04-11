##
## This file is part of the libsigrokdecode project.
##
## Copyright (C) 2020 Uwe Hermann <uwe@hermann-uwe.de>
##
## This program is free software: you can redistribute it and/or modify
## it under the terms of the GNU General Public License as published by
## the Free Software Foundation, either version 2 of the License, or
## (at your option) any later version.
##
## This program is distributed in the hope that it will be useful,
## but WITHOUT ANY WARRANTY; without even the implied warranty of
## MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
## GNU General Public License for more details.
##
## You should have received a copy of the GNU General Public License
## along with this program.  If not, see <http://www.gnu.org/licenses/>.
##

cmake_minimum_required(VERSION 3.12)

project(sigrokdecode VERSION 6.0.0 LANGUAGES C)

include(GNUInstallDirs)

find_package(Git QUIET)
if(Git_FOUND)
	execute_process(COMMAND ${GIT_EXECUTABLE} rev-parse --short HEAD
		WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR} ERROR_QUIET
		OUTPUT_VARIABLE GITHASH OUTPUT_STRIP_TRAILING_WHITESPACE)
	execute_process(COMMAND
		${GIT_EXECUTABLE} name-rev --tags --name-only ${GITHASH}
		WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR} ERROR_QUIET
		OUTPUT_VARIABLE GITTAG OUTPUT_STRIP_TRAILING_WHITESPACE)
	if((DEFINED GITTAG) AND ("${GITTAG}" STREQUAL "undefined"))
		string(APPEND PROJECT_VERSION "-git-${GITHASH}")
	endif()
endif()

option(BUILD_SHARED_LIBS "Shared (non-static) build" TRUE)

if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
	set_property(CACHE CMAKE_BUILD_TYPE PROPERTY VALUE RelWithDebInfo)
endif()

set_property(GLOBAL PROPERTY USE_FOLDERS TRUE)

find_package(PkgConfig REQUIRED)
pkg_check_modules(LIBGLIB REQUIRED IMPORTED_TARGET "glib-2.0>=2.34.0")
pkg_search_module(LIBPYTHON REQUIRED IMPORTED_TARGET "python3-embed>=3.2;python3>=3.2")
message(STATUS "  Found Python, version ${LIBPYTHON_VERSION}")
# Ugly workaround, *_MODULE_NAME only exists in cmake >= 3.16.
pkg_check_modules(LIBPYTHONEMBED QUIET "python3-embed>=3.2")
pkg_check_modules(LIBPYTHONNOEMBED QUIET "python3>=3.2")
set(SRD_PKGLIBS "python3")
if(LIBPYTHONEMBED_FOUND)
	set(SRD_PKGLIBS "python3-embed")
endif()
pkg_check_modules(LIBCHECK IMPORTED_TARGET "check>=0.9.4")

include(TestBigEndian)
test_big_endian(WORDS_BIGENDIAN)

add_library(${PROJECT_NAME} "")
add_library(sigrok::${PROJECT_NAME} ALIAS ${PROJECT_NAME})

target_sources(${PROJECT_NAME} PRIVATE decoder.c error.c exception.c
	instance.c log.c module_sigrokdecode.c session.c srd.c type_decoder.c
	util.c version.c lib${PROJECT_NAME}-internal.h
	${CMAKE_CURRENT_SOURCE_DIR}/include/lib${PROJECT_NAME}/lib${PROJECT_NAME}.h
	${CMAKE_CURRENT_BINARY_DIR}/include/lib${PROJECT_NAME}/lib${PROJECT_NAME}_version.h
	${CMAKE_CURRENT_BINARY_DIR}/include/lib${PROJECT_NAME}/lib${PROJECT_NAME}_export.h)

set_target_properties(${PROJECT_NAME} PROPERTIES PUBLIC_HEADER "\
${CMAKE_CURRENT_SOURCE_DIR}/include/lib${PROJECT_NAME}/lib${PROJECT_NAME}.h;\
${CMAKE_CURRENT_BINARY_DIR}/include/lib${PROJECT_NAME}/lib${PROJECT_NAME}_version.h;\
${CMAKE_CURRENT_BINARY_DIR}/include/lib${PROJECT_NAME}/lib${PROJECT_NAME}_export.h")

set_target_properties(${PROJECT_NAME} PROPERTIES
	VERSION ${PROJECT_VERSION} SOVERSION ${PROJECT_VERSION_MAJOR})

target_compile_features(${PROJECT_NAME} PRIVATE c_std_99)

target_compile_options(${PROJECT_NAME} PRIVATE
	$<IF:$<C_COMPILER_ID:MSVC>,/W4,-Wall -Wextra>)

target_compile_definitions(${PROJECT_NAME} PRIVATE
	$<IF:$<BOOL:${MINGW}>,__STDC_FORMAT_MACROS -D__STDC_LIMIT_MACROS -D_POSIX -D__printf__=__gnu_printf__,>)

target_include_directories(${PROJECT_NAME} PUBLIC
	$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
	$<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>
	$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
	$<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/include>
	$<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>)

if(BUILD_SHARED_LIBS)
	target_link_libraries(${PROJECT_NAME} PUBLIC
		PkgConfig::LIBGLIB PkgConfig::LIBPYTHON)
else()
	target_include_directories(${PROJECT_NAME} PUBLIC
		${LIBGLIB_STATIC_INCLUDE_DIRS} ${LIBPYTHON_STATIC_INCLUDE_DIRS})
	target_link_libraries(${PROJECT_NAME} PUBLIC
		${LIBGLIB_STATIC_LDFLAGS} ${LIBPYTHON_STATIC_LDFLAGS})
endif()

set(CMAKE_CXX_VISIBILITY_PRESET hidden)
set(VISIBILITY_INLINES_HIDDEN TRUE)
include(GenerateExportHeader)
generate_export_header(${PROJECT_NAME} BASE_NAME lib${PROJECT_NAME}
	PREFIX_NAME "SRD_" EXPORT_MACRO_NAME "API" NO_EXPORT_MACRO_NAME "PRIV"
	DEPRECATED_MACRO_NAME "DEPR" NO_DEPRECATED_MACRO_NAME "NO_DEPR"
	EXPORT_FILE_NAME ${CMAKE_CURRENT_BINARY_DIR}/include/lib${PROJECT_NAME}/lib${PROJECT_NAME}_export.h)

include(CMakePackageConfigHelpers)
set(INST_INC_DIR ${CMAKE_INSTALL_INCLUDEDIR})
set(INST_LIB_DIR ${CMAKE_INSTALL_LIBDIR})
configure_package_config_file(${PROJECT_NAME}Config.cmake.in
	${PROJECT_NAME}Config.cmake PATH_VARS INST_INC_DIR INST_LIB_DIR
	INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME})
write_basic_package_version_file(
	${PROJECT_NAME}ConfigVersion.cmake COMPATIBILITY SameMajorVersion)

install(TARGETS ${PROJECT_NAME} EXPORT ${PROJECT_NAME}Targets
	RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
	LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
	ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
	PUBLIC_HEADER DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/lib${PROJECT_NAME})

install(EXPORT ${PROJECT_NAME}Targets
	FILE ${PROJECT_NAME}Targets.cmake NAMESPACE sigrok::
	DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME})

install(FILES ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Config.cmake
	${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}ConfigVersion.cmake
	DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME})

install(FILES ${CMAKE_CURRENT_BINARY_DIR}/lib${PROJECT_NAME}.pc
	DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig)

install(DIRECTORY decoders DESTINATION ${CMAKE_INSTALL_DATADIR}/lib${PROJECT_NAME} PATTERN "decoders/*" EXCLUDE) # Dummy, only used to get a log message.
install(DIRECTORY decoders DESTINATION ${CMAKE_INSTALL_DATADIR}/lib${PROJECT_NAME} MESSAGE_NEVER)

message(STATUS "CMake: ${CMAKE_COMMAND} ${CMAKE_VERSION}")
message(STATUS "Building lib${PROJECT_NAME} ${PROJECT_VERSION} for ${CMAKE_SYSTEM} ${CMAKE_SYSTEM_PROCESSOR}")

include(CTest)
if(LIBCHECK_FOUND AND BUILD_TESTING)
	enable_testing()
	add_executable(unittests tests/core.c tests/decoder.c tests/inst.c
		tests/lib.h tests/main.c tests/session.c)
	set_target_properties(unittests PROPERTIES C_STANDARD 99)
	target_compile_options(unittests PRIVATE
		$<IF:$<C_COMPILER_ID:MSVC>,/W4,-Wall -Wextra>)
	target_compile_definitions(unittests PRIVATE
		-DDECODERS_TESTDIR="${CMAKE_CURRENT_SOURCE_DIR}/decoders")
	target_link_libraries(unittests PRIVATE ${PROJECT_NAME} PkgConfig::LIBCHECK)
	add_test(NAME unittests COMMAND unittests)
endif()

set(CPACK_SOURCE_PACKAGE_FILE_NAME "lib${PROJECT_NAME}-${PROJECT_VERSION}")
set(CPACK_SOURCE_IGNORE_FILES ${CMAKE_CURRENT_BINARY_DIR} ".git*")
set(CPACK_SOURCE_GENERATOR "TGZ")
include(CPack)

configure_file(config.h.in config.h @ONLY)
configure_file(include/lib${PROJECT_NAME}/lib${PROJECT_NAME}_version.h.in
	include/lib${PROJECT_NAME}/lib${PROJECT_NAME}_version.h @ONLY)
configure_file(lib${PROJECT_NAME}.pc.in lib${PROJECT_NAME}.pc @ONLY)
