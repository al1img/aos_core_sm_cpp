#
# Copyright (C) 2024 EPAM Systems, Inc.
#
# SPDX-License-Identifier: Apache-2.0
#

cmake_minimum_required(VERSION 3.19)

project("aos_core_sm_cpp")

# ######################################################################################################################
# Options
# ######################################################################################################################

option(WITH_TEST "build with test" OFF)
option(WITH_COVERAGE "build with coverage" OFF)
option(WITH_DOC "build with documentation" OFF)

message(STATUS)
message(STATUS "${CMAKE_PROJECT_NAME} configuration:")
message(STATUS "CMAKE_BUILD_TYPE              = ${CMAKE_BUILD_TYPE}")
message(STATUS "CMAKE_INSTALL_PREFIX          = ${CMAKE_INSTALL_PREFIX}")
message(STATUS)
message(STATUS "WITH_TEST                     = ${WITH_TEST}")
message(STATUS "WITH_COVERAGE                 = ${WITH_COVERAGE}")
message(STATUS "WITH_DOC                      = ${WITH_DOC}")
message(STATUS)

# ######################################################################################################################
# Config
# ######################################################################################################################

if(NOT DEFINED AOS_EXTERNAL_DIR)
    set(AOS_EXTERNAL_DIR external)
endif()

get_filename_component(AOS_CORE_LIB_DIR ${AOS_EXTERNAL_DIR}/aos_core_lib_cpp REALPATH BASE_DIR "${CMAKE_BINARY_DIR}")
get_filename_component(
    AOS_CORE_COMMON_LIB_DIR ${AOS_EXTERNAL_DIR}/aos_core_common_cpp REALPATH BASE_DIR "${CMAKE_BINARY_DIR}"
)
get_filename_component(AOS_CORE_API_DIR ${AOS_EXTERNAL_DIR}/aos_core_api REALPATH BASE_DIR "${CMAKE_BINARY_DIR}")

set(AOS_STACK_USAGE 4096)

# ######################################################################################################################
# Fetch external projects
# ######################################################################################################################

include(FetchContent)

FetchContent_Populate(
    aos_core_lib_cpp
    GIT_REPOSITORY https://github.com/aosedge/aos_core_lib_cpp.git
    GIT_TAG develop
    SOURCE_DIR ${AOS_CORE_LIB_DIR}
)

FetchContent_Populate(
    aos_core_common_cpp
    GIT_REPOSITORY https://github.com/al1img/aos_core_common_cpp.git
    GIT_TAG develop
    SOURCE_DIR ${AOS_CORE_COMMON_LIB_DIR}
)

FetchContent_Populate(
    aos_core_api
    GIT_REPOSITORY https://github.com/aosedge/aos_core_api.git
    GIT_TAG develop
    SOURCE_DIR ${AOS_CORE_API_DIR}
)

# ######################################################################################################################
# External project options
# ######################################################################################################################

# aos core api options
set(WITH_SM_API ON)

# ######################################################################################################################
# Compiler flags
# ######################################################################################################################

add_compile_options(
    -fPIC
    -Wall
    -Werror
    -Wextra
    -Wpedantic
    -Wno-cpp
    -Wno-format-truncation
)
set(CMAKE_CXX_STANDARD 17)

if(WITH_TEST)
    add_definitions(-include ${AOS_CORE_LIB_DIR}/include/aos/test/aoscoretestconfig.hpp)
endif()

# ######################################################################################################################
# Versioning
# ######################################################################################################################

find_package(Git)

if(NOT GIT_FOUND)
    set(GIT_EXECUTABLE git)
endif()

list(APPEND CMAKE_MODULE_PATH ${AOS_CORE_LIB_DIR}/cmake)

execute_process(
    OUTPUT_FILE ${CMAKE_BINARY_DIR}/version.hpp
    COMMAND
        ${CMAKE_COMMAND} -D GIT_EXECUTABLE=${GIT_EXECUTABLE} -D INPUT_FILE=${CMAKE_SOURCE_DIR}/src/version.hpp.in -D
        OUTPUT_FILE=${CMAKE_BINARY_DIR}/version.hpp -D GIT_SOURCE_DIR=${CMAKE_SOURCE_DIR} -P
        ${AOS_CORE_LIB_DIR}/cmake/GenerateVersion.cmake
)

include_directories(${CMAKE_BINARY_DIR})

# ######################################################################################################################
# Dependencies
# ######################################################################################################################

# Poco lib
find_package(
    Poco
    REQUIRED
    Crypto
    DataSQLite
    Foundation
    JSON
    Net
    NetSSL
    Util
)

if(WITH_TEST)
    find_package(GTest REQUIRED)

    include(GoogleTest)

    enable_testing()
endif()

if(WITH_COVERAGE)
    include(CodeCoverage)

    append_coverage_compiler_flags()

    set(COVERAGE_EXCLUDES "${CMAKE_CURRENT_BINARY_DIR}/*" "${PROJECT_SOURCE_DIR}/external/*"
                          "${PROJECT_SOURCE_DIR}/tests/*" "/usr/*"
    )
    set(GCOVR_ADDITIONAL_ARGS --gcov-ignore-parse-errors=negative_hits.warn_once_per_file)

    setup_target_for_coverage_lcov(
        NAME
        coverage
        EXECUTABLE
        ctest
        SONARQUBE
        LCOV_ARGS
        --ignore-errors
        mismatch,negative
    )
endif()

# ######################################################################################################################
# Targets
# ######################################################################################################################

add_subdirectory(${AOS_CORE_API_DIR})
add_subdirectory(${AOS_CORE_LIB_DIR}/src/common)
add_subdirectory(${AOS_CORE_LIB_DIR}/src/sm)
add_subdirectory(${AOS_CORE_COMMON_LIB_DIR}/src)
add_subdirectory(src)

if(WITH_TEST)
    add_subdirectory(${AOS_CORE_LIB_DIR}/tests/utils)
    add_subdirectory(${AOS_CORE_LIB_DIR}/tests/include)
    add_subdirectory(${AOS_CORE_COMMON_LIB_DIR}/tests/include)
    add_subdirectory(tests)
endif()

# ######################################################################################################################
# Doc
# ######################################################################################################################

if(WITH_DOC)
    find_package(Doxygen)

    configure_file(${CMAKE_CURRENT_SOURCE_DIR}/doxygen.cfg ${CMAKE_CURRENT_BINARY_DIR}/doxygen.cfg @ONLY)

    add_custom_target(
        doc
        ${DOXYGEN_EXECUTABLE} ${CMAKE_CURRENT_BINARY_DIR}/doxygen.cfg
        WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
        COMMENT "Generating API documentation with Doxygen"
        VERBATIM
    )
endif()
