# The only intention of this file is to provide ability to build natively
# both on Linux and on Windows to debug MSVC build.

# Run commands as follows, to configure, build and test:
# - on Windows:
#     >cmake -G "Visual Studio 17 2022" -A x64 -S . -B build
#     >cmake --build build --config Release
#     >build\Release\test.exe
# - on Linux:
#     $ cmake -S . -B build
#     $ cmake --build build
#     $ ./build/test

# A similar, but not the same, Autotools configuration:
# $ ./configure --disable-ccache --enable-fields=32 CPPFLAGS="-UHAVE_CLMUL -UHAVE_CLZ"

cmake_minimum_required(VERSION 3.15)

project(minisketch VERSION 0.0.1 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

set(CMAKE_CXX_VISIBILITY_PRESET hidden)
# set(CMAKE_VISIBILITY_INLINES_HIDDEN 1)

if(MSVC)
  set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
  add_compile_options(/Zc:__cplusplus)
else()
  add_compile_options(-Werror -Wall)
endif()

# add_definitions(-DDISABLE_DEFAULT_FIELDS -DENABLE_FIELD_32)

add_library(minisketch STATIC "")
target_sources(minisketch
  PRIVATE
    ${CMAKE_SOURCE_DIR}/src/minisketch.cpp
    ${CMAKE_SOURCE_DIR}/src/fields/generic_1byte.cpp
    ${CMAKE_SOURCE_DIR}/src/fields/generic_2bytes.cpp
    ${CMAKE_SOURCE_DIR}/src/fields/generic_3bytes.cpp
    ${CMAKE_SOURCE_DIR}/src/fields/generic_4bytes.cpp
    ${CMAKE_SOURCE_DIR}/src/fields/generic_5bytes.cpp
    ${CMAKE_SOURCE_DIR}/src/fields/generic_6bytes.cpp
    ${CMAKE_SOURCE_DIR}/src/fields/generic_7bytes.cpp
    ${CMAKE_SOURCE_DIR}/src/fields/generic_8bytes.cpp
)
if(MSVC)
  target_compile_options(minisketch PRIVATE /wd4065)
else()
  # target_compile_options(minisketch PRIVATE -Wno-shift-count-overflow)
endif()

add_executable(test ${CMAKE_SOURCE_DIR}/src/test.cpp)
target_link_libraries(test PRIVATE minisketch)
