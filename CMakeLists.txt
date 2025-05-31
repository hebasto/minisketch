# The cmake_minimum_required() command do two things:
# 1. Sets the minimum required version of cmake for a project.
# 2. Sets the policy settings.
#
# Rationale for point 1.
# Minimum required version 3.22 allows to build on Ubuntu 22.04 LTS.
# Same as in Bitcoin Core.
#
# Rationale for point 2.
# From Professional CMake: A Practical Guide, 21st Edition,
# Section 13.1:
# > This gives projects confidence that developers should be
# > able to update to any newer version of CMake at their
# > convenience, and the project will still build as it did before.
#
# Some other developers, namely @ryanofsky and @purpleKarrot, disagree.
# See: https://github.com/bitcoin-core/libmultiprocess/pull/163
cmake_minimum_required(VERSION 3.22)

#=============================
# Project / Package Metadata
#=============================
project(minisketch
  VERSION 0.0.1
  DESCRIPTION "A library for BCH-based set reconciliation"
  HOMEPAGE_URL "http://github.com/sipa/minisketch/"
  LANGUAGES CXX
)

# ============================================================
# Project Initialization
# ============================================================
# I believe this is superior to include(CTest) as
# the latest has a few drawbacks.
#
# Some other developers, namely @purpleKarrot, disagree.
# See: https://github.com/bitcoin-core/libmultiprocess/pull/145
enable_testing()

list(APPEND CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake)

if(NOT CMAKE_RUNTIME_OUTPUT_DIRECTORY)
  set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/bin)
endif()
if(NOT CMAKE_LIBRARY_OUTPUT_DIRECTORY)
  set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/lib)
endif()
if(NOT CMAKE_ARCHIVE_OUTPUT_DIRECTORY)
  set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/lib)
endif()

#=============================
# Language Setup
#=============================
# Require C++11, but let a parent project ask for something higher.
if(DEFINED CMAKE_CXX_STANDARD)
  if(CMAKE_CXX_STANDARD EQUAL 98 OR CMAKE_CXX_STANDARD LESS 11)
    message(FATAL_ERROR "This project requires at least C++11")
  endif()
else()
  set(CMAKE_CXX_STANDARD 11)
endif()
set(CMAKE_CXX_STANDARD_REQUIRED ON)
# We don't use compiler extensions, but let a parent project ask for them.
if(NOT DEFINED CMAKE_CXX_EXTENSIONS)
  set(CMAKE_CXX_EXTENSIONS OFF)
endif()

#=============================
# Configurable Options
#=============================
option(MINISKETCH_INSTALL "Enable installation." ${PROJECT_IS_TOP_LEVEL})

option(MINISKETCH_BUILD_TESTS "Build tests." ON)
option(MINISKETCH_BUILD_BENCHMARK "Build benchmark." OFF)

set(supported_fields "")
set(have_enabled_fields NO)
set(have_disabled_fields NO)
foreach(i RANGE 2 64)
    list(APPEND supported_fields ${i})
endforeach()
set(MINISKETCH_FIELDS ${supported_fields} CACHE STRING "Semicolon-separated list of field sizes to build. Default=all. Available sizes: ${supported_fields}.")
foreach(field IN LISTS supported_fields)
  if(field IN_LIST MINISKETCH_FIELDS)
    set(have_enabled_fields YES)
  else()
    set(have_disabled_fields YES)
    add_compile_definitions(DISABLE_FIELD_${field})
  endif()
endforeach()
if(NOT have_enabled_fields)
  message(FATAL_ERROR "No field sizes are enabled.")
endif()
unset(have_enabled_fields)
unset(supported_fields)

#=============================
# Build Options
#=============================
set(CMAKE_CXX_VISIBILITY_PRESET hidden)

if(MSVC)
  add_compile_options(/Zc:__cplusplus)
endif()

if(MINGW)
  add_link_options(-static)
endif()

#=============================
# Diagnostics Options
#=============================
if(MSVC)
  # For both MSVC's cl.exe and clang-cl compilers.
  add_compile_options(/W3)  # Production quality warning level. Enables -Wall Clang's core option.
else()
  add_compile_options(-Wall)
endif()

if(CMAKE_CXX_COMPILER_ID STREQUAL "MSVC")
  # add_compile_options(/wd4060)
  # add_compile_options(/wd4065)
  # add_compile_options(/wd4100)
  # add_compile_options(/wd4127)
  add_compile_options(/wd4146)  # Disable warning C4146 "unary minus operator applied to unsigned type, result still unsigned".
  add_compile_options(/wd4244)  # Disable warning C4244 "conversion from 'type1' to 'type2', possible loss of data".
  # add_compile_options(/wd4267)
  # add_compile_options(/wd4310)
else()
  add_compile_options(-Wundef)
endif()

#=============================
# Main Processing
#=============================
include(SystemIntrospection)
add_subdirectory(src)
include(PrintConfigureSummary)
