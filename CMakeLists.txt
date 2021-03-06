# Copyright (c) 2017, EPL-Vizards
# All rights reserved.
#
# Redistribution and use in source and binary forms, with or without
# modification, are permitted provided that the following conditions are met:
#     * Redistributions of source code must retain the above copyright
#       notice, this list of conditions and the following disclaimer.
#     * Redistributions in binary form must reproduce the above copyright
#       notice, this list of conditions and the following disclaimer in the
#       documentation and/or other materials provided with the distribution.
#     * Neither the name of the EPL-Vizards nor the
#       names of its contributors may be used to endorse or promote products
#       derived from this software without specific prior written permission.
#
# THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS" AND
# ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED
# WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE
# DISCLAIMED. IN NO EVENT SHALL EPL-Vizards BE LIABLE FOR ANY
# DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES
# (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES;
# LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND
# ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT
# (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF THIS
# SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.

cmake_minimum_required( VERSION 3.5 )
project( funcRedirect )

message( STATUS "Using CMake version: ${CMAKE_MAJOR_VERSION}.${CMAKE_MINOR_VERSION}.${CMAKE_PATCH_VERSION}\n" )
include( ${PROJECT_SOURCE_DIR}/cmake/cmakeScripLoader.cmake )

include( CMakePackageConfigHelpers )
include( ExternalProject )

######################
# Set some variables #
######################

# Set the binary output directories
set( CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/lib )
set( CMAKE_LIBRARY_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/lib )
set( CMAKE_RUNTIME_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/bin )

set( CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} ${PROJECT_SOURCE_DIR}/cmake )
set( CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} ${PROJECT_SOURCE_DIR}/cmake/coveralls-cmake/cmake )

set( CMAKE_CXX_STANDARD          14 )
set( CMAKE_CXX_STANDARD_REQUIRED ON )
set( CMAKE_C_STANDARD            11 )
set( CMAKE_C_STANDARD_REQUIRED   ON )

option( DISABLE_TESTS        "Disables all tests"          OFF )
option( ENABLE_CODE_COVERAGE "Enables code coverage tests" OFF )

list( APPEND CMAKE_MODULE_PATH "${PROJECT_SOURCE_DIR}/cmake" )

# Some useful variables
set( TEMPLATES_DIR "${PROJECT_SOURCE_DIR}/cmake/templates" )
set( INCLUDES_DIR  "${PROJECT_SOURCE_DIR}/include" )

################################
# Set up the build environment #
################################

if( ENABLE_CODE_COVERAGE )
  set( COVERAGE_COMPILER_FLAG "--coverage" )
endif( ENABLE_CODE_COVERAGE )

run_git()

add_compiler(
  GNU MIN_VERSION "6.2"
  ALL        "-Wall -Wextra -Wfloat-equal -Wshadow -Wno-unknown-pragmas -fno-inline"
  DEBUG      "-O0 -Werror" ${COVERAGE_COMPILER_FLAG}
  RELEASE    "-O3"

  C_ALL      "-Wall -Wextra -Wfloat-equal -Wshadow -Wno-unknown-pragmas -fno-inline"
  C_DEBUG    "-O0 -Werror" ${COVERAGE_COMPILER_FLAG}
  C_RELEASE  "-O3"

  SANITIZE   "-fsanitize=${SANITIZERS}"
)

add_compiler(
  Clang MIN_VERSION 3.9
  ALL        "-Weverything -ftemplate-backtrace-limit=0"
             "-Wno-c++98-compat -Wno-c++98-compat-pedantic -Wno-padded"
             "-Wno-gnu-anonymous-struct -Wno-nested-anon-types -Wno-exit-time-destructors"
             "-Wno-global-constructors -Wno-switch-enum -Wno-reserved-id-macro"
             "-Wno-documentation-unknown-command -Wno-unknown-pragmas"
  DEBUG      "-Werror"
  RELEASE    "-O2"

  C_ALL      "-Weverything -ftemplate-backtrace-limit=0"
             "-Wno-c++98-compat -Wno-c++98-compat-pedantic -Wno-padded -Wno-float-equal"
             "-Wno-gnu-anonymous-struct -Wno-nested-anon-types -Wno-exit-time-destructors"
             "-Wno-global-constructors -Wno-switch-enum -Wno-reserved-id-macro"
             "-Wno-documentation-unknown-command -Wno-unknown-pragmas"
  C_DEBUG    "-Werror"
  C_RELEASE  "-O2"

  SANITIZE "-fsanitize=${SANITIZERS}"
)

######################
# Check Dependencies #
######################

find_package( Git REQUIRED )

if( NOT DISABLE_TESTS )
  ExternalProject_Add( catch
    PREFIX            ${CMAKE_BINARY_DIR}/catch
    GIT_REPOSITORY    https://github.com/philsquared/Catch.git
    UPDATE_COMMAND    ${GIT_EXECUTABLE} pull
    CONFIGURE_COMMAND ""
    BUILD_COMMAND     ""
    INSTALL_COMMAND   ""
  )

  ExternalProject_Get_Property( catch source_dir )
  set( CATCH_INCLUDE_DIR "${source_dir}/include" )
endif( NOT DISABLE_TESTS )

####################
# Generate subdirs #
####################

if( UNIX )
  set( PLATFORM_TARGET UNIX_POSIX )

  if( CMAKE_SYSTEM_PROCESSOR MATCHES "(x86)|(X86)|(amd64)|(AMD64)" )
    list( APPEND PLATFORM_TARGET UNIX_X86 )
  else( CMAKE_SYSTEM_PROCESSOR MATCHES "(x86)|(X86)|(amd64)|(AMD64)" )
    message( FATAL_ERROR "Processor arch ${CMAKE_SYSTEM_PROCESSOR} not supported" )
  endif( CMAKE_SYSTEM_PROCESSOR MATCHES "(x86)|(X86)|(amd64)|(AMD64)" )
else( UNIX )
  message( FATAL_ERROR "Current OS not supported" )
endif( UNIX )

add_platform( OS UNIX TARGET x86 posix )
check_platform()

new_project_library(
  PATH         "${PROJECT_SOURCE_DIR}/src"
  NAME         "funcRedirect"
  TEMPLATE     "${TEMPLATES_DIR}/CMakeLists.lib.txt"
  DEPENDENCIES
)

if( NOT DISABLE_TESTS )
  new_project_executable(
    PATH         "${PROJECT_SOURCE_DIR}/tests"
    NAME         "tests"
    TEMPLATE     "${TEMPLATES_DIR}/CMakeLists.exe.txt"
    DEPENDENCIES funcRedirect_shared
  )
endif( NOT DISABLE_TESTS )

##############################
# Update include directories #
##############################

include_directories( ${funcRedirect_LIB_INCLUDE_DIRECTORIES} ${CATCH_INCLUDE_DIR} ${INCLUDES_DIR} )

if( NOT EXISTS "${INCLUDES_DIR}" )
  file( MAKE_DIRECTORY "${INCLUDES_DIR}" )
endif( NOT EXISTS "${INCLUDES_DIR}" )

######################
# Add subdirectories #
######################

foreach( I IN LISTS funcRedirect_SUBDIR_LIST )
  add_subdirectory( "${I}" )
endforeach( I IN LISTS funcRedirect_SUBDIR_LIST )

if( NOT DISABLE_TESTS )
  add_custom_target( check COMMAND tests )
endif( NOT DISABLE_TESTS )

##################
# Generate Files #
##################

configure_file( "${TEMPLATES_DIR}/defines.in.hpp" "${INCLUDES_DIR}/defines.hpp" @ONLY )

write_basic_package_version_file(
  "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Version.cmake"
    VERSION       ${CM_VERSION_MAJOR}.${CM_VERSION_MINOR}.${CM_VERSION_SUBMINOR}
    COMPATIBILITY SameMajorVersion
)

install(
  FILES
    ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Version.cmake
    ${PROJECT_SOURCE_DIR}/cmake/${PROJECT_NAME}Config.cmake
  DESTINATION lib/cmake/${PROJECT_NAME}
)

install( EXPORT ${PROJECT_NAME}Targets DESTINATION lib/cmake/${PROJECT_NAME} )

install(
  FILES "${INCLUDES_DIR}/defines.hpp"
  DESTINATION include/funcRedirect
)

generate_format_command( format 3.9.1 )

#################
# Code Coverage #
#################

# if( ENABLE_CODE_COVERAGE AND CMAKE_CXX_COMPILER_ID STREQUAL GNU )
#   message( STATUS "COVERAGE ON" )
#   enable_testing()
#   add_test( NAME AllTests COMMAND tests )
#
#   include(Coveralls)
#
#   coveralls_setup(
#     "${${PROJECT_NAME}_ALL_UNASIGNED_CPP}"
#     ON
#   )
# endif( ENABLE_CODE_COVERAGE AND CMAKE_CXX_COMPILER_ID STREQUAL GNU )
