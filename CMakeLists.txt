cmake_minimum_required(VERSION 3.22)

project(minisketch
  VERSION 0.0.1
  DESCRIPTION "A library for BCH-based set reconciliation"
  HOMEPAGE_URL "http://github.com/sipa/minisketch/"
  LANGUAGES CXX
)

if(DEFINED CMAKE_CXX_STANDARD)
  if(CMAKE_CXX_STANDARD EQUAL 98 OR CMAKE_CXX_STANDARD LESS 11)
    message(FATAL_ERROR "This project requires at least C++11")
  endif()
else()
  set(CMAKE_CXX_STANDARD 11)
endif()
set(CMAKE_CXX_STANDARD_REQUIRED ON)
if(NOT DEFINED CMAKE_CXX_EXTENSIONS)
  set(CMAKE_CXX_EXTENSIONS OFF)
endif()

set(CMAKE_CXX_VISIBILITY_PRESET hidden)
set(CMAKE_VISIBILITY_INLINES_HIDDEN TRUE)

if(MSVC)
  add_compile_options(/Zc:__cplusplus /W4 /wd4060 /wd4065 /wd4100 /wd4127 /wd4146 /wd4244 /wd4267 /wd4310)
else()
  add_compile_options(-Wall)
endif()

# Check for clmul instructions support.
if(MSVC)
  set(CLMUL_CXXFLAGS "")
else()
  set(CLMUL_CXXFLAGS "-mpclmul")
endif()
set(CMAKE_REQUIRED_FLAGS ${CLMUL_CXXFLAGS})
include(CheckCXXSourceCompiles)
check_cxx_source_compiles("
#include <immintrin.h>
#include <stdint.h>

int main()
{
  __m128i a = _mm_cvtsi64_si128((uint64_t)7);
  __m128i b = _mm_clmulepi64_si128(a, a, 37);
  __m128i c = _mm_srli_epi64(b, 41);
  __m128i d = _mm_xor_si128(b, c);
  uint64_t e = _mm_cvtsi128_si64(d);
  return e == 0;
}
" HAVE_CLMUL)
set(CMAKE_REQUIRED_FLAGS)

# Check for working clz builtins.
check_cxx_source_compiles("
int main()
{
  unsigned a = __builtin_clz(1);
  unsigned long b = __builtin_clzl(1);
  unsigned long long c = __builtin_clzll(1);
}
" HAVE_CLZ)

add_library(${PROJECT_NAME}_sources INTERFACE)
target_sources(${PROJECT_NAME}_sources
  INTERFACE
    ${PROJECT_SOURCE_DIR}/src/minisketch.cpp
    ${PROJECT_SOURCE_DIR}/src/fields/generic_1byte.cpp
    ${PROJECT_SOURCE_DIR}/src/fields/generic_2bytes.cpp
    ${PROJECT_SOURCE_DIR}/src/fields/generic_3bytes.cpp
    ${PROJECT_SOURCE_DIR}/src/fields/generic_4bytes.cpp
    ${PROJECT_SOURCE_DIR}/src/fields/generic_5bytes.cpp
    ${PROJECT_SOURCE_DIR}/src/fields/generic_6bytes.cpp
    ${PROJECT_SOURCE_DIR}/src/fields/generic_7bytes.cpp
    ${PROJECT_SOURCE_DIR}/src/fields/generic_8bytes.cpp
    $<$<BOOL:${HAVE_CLMUL}>:${PROJECT_SOURCE_DIR}/src/fields/clmul_1byte.cpp>
    $<$<BOOL:${HAVE_CLMUL}>:${PROJECT_SOURCE_DIR}/src/fields/clmul_2bytes.cpp>
    $<$<BOOL:${HAVE_CLMUL}>:${PROJECT_SOURCE_DIR}/src/fields/clmul_3bytes.cpp>
    $<$<BOOL:${HAVE_CLMUL}>:${PROJECT_SOURCE_DIR}/src/fields/clmul_4bytes.cpp>
    $<$<BOOL:${HAVE_CLMUL}>:${PROJECT_SOURCE_DIR}/src/fields/clmul_5bytes.cpp>
    $<$<BOOL:${HAVE_CLMUL}>:${PROJECT_SOURCE_DIR}/src/fields/clmul_6bytes.cpp>
    $<$<BOOL:${HAVE_CLMUL}>:${PROJECT_SOURCE_DIR}/src/fields/clmul_7bytes.cpp>
    $<$<BOOL:${HAVE_CLMUL}>:${PROJECT_SOURCE_DIR}/src/fields/clmul_8bytes.cpp>
)
target_compile_definitions(${PROJECT_NAME}_sources
  INTERFACE
    $<$<BOOL:${HAVE_CLMUL}>:HAVE_CLMUL>
    $<$<BOOL:${HAVE_CLZ}>:HAVE_CLZ>
)
target_compile_options(${PROJECT_NAME}_sources
  INTERFACE
    $<$<BOOL:${HAVE_CLMUL}>:${CLMUL_CXXFLAGS}>
)

add_library(${PROJECT_NAME} STATIC "")
target_include_directories(${PROJECT_NAME}
  INTERFACE
    $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
)
target_link_libraries(${PROJECT_NAME} PRIVATE ${PROJECT_NAME}_sources)

add_library(${PROJECT_NAME}_verify STATIC EXCLUDE_FROM_ALL "")
target_compile_definitions(${PROJECT_NAME}_verify PUBLIC MINISKETCH_VERIFY)
target_link_libraries(${PROJECT_NAME}_verify PRIVATE ${PROJECT_NAME}_sources)

add_executable(${PROJECT_NAME}_test EXCLUDE_FROM_ALL ${PROJECT_SOURCE_DIR}/src/test.cpp)
target_link_libraries(${PROJECT_NAME}_test PRIVATE ${PROJECT_NAME})

add_executable(${PROJECT_NAME}_test_verify EXCLUDE_FROM_ALL ${PROJECT_SOURCE_DIR}/src/test.cpp)
target_link_libraries(${PROJECT_NAME}_test_verify PRIVATE ${PROJECT_NAME}_verify)

add_executable(${PROJECT_NAME}_bench EXCLUDE_FROM_ALL ${PROJECT_SOURCE_DIR}/src/bench.cpp)
target_link_libraries(${PROJECT_NAME}_bench PRIVATE ${PROJECT_NAME})

message("\n")
message("minisketch configure summary")
message("============================")
message("Build options:")
if(HAVE_CLMUL)
  set(clmul_status "TRUE")
else()
  set(clmul_status "FALSE")
endif()
message(" - enable clmul fields ................ ${clmul_status}")
if(HAVE_CLZ)
  set(clz_status "TRUE")
else()
  set(clz_status "FALSE")
endif()
message(" - enable clz builtins ................ ${clz_status}")
message("")
if(CMAKE_CROSSCOMPILING)
  set(cross_status "TRUE, for ${CMAKE_SYSTEM_NAME}, ${CMAKE_SYSTEM_PROCESSOR}")
else()
  set(cross_status "FALSE")
endif()
message("Cross compiling ....................... ${cross_status}")
get_directory_property(definitions COMPILE_DEFINITIONS)
message("Preprocessor defined macros ........... ${definitions}")
message("C++ compiler .......................... ${CMAKE_CXX_COMPILER}")
message("CXXFLAGS .............................. ${CMAKE_CXX_FLAGS}")
get_directory_property(compile_options COMPILE_OPTIONS)
string(REPLACE ";" " " compile_options "${compile_options}")
message("Compile options ....................... ${compile_options}")
if(DEFINED CMAKE_BUILD_TYPE)
  message("Build type:")
  message(" - CMAKE_BUILD_TYPE ................... ${CMAKE_BUILD_TYPE}")
  string(TOUPPER "${CMAKE_BUILD_TYPE}" build_type)
  message(" - CXXFLAGS ........................... ${CMAKE_CXX_FLAGS_${build_type}}")
  message(" - LDFLAGS for executables ............ ${CMAKE_EXE_LINKER_FLAGS_${build_type}}")
else()
  message("Available configurations .............. ${CMAKE_CONFIGURATION_TYPES}")
  message("Debug configuration:")
  message(" - CXXFLAGS ........................... ${CMAKE_CXX_FLAGS_DEBUG}")
  message(" - LDFLAGS for executables ............ ${CMAKE_EXE_LINKER_FLAGS_DEBUG}")
  message("Release configuration:")
  message(" - CXXFLAGS ........................... ${CMAKE_CXX_FLAGS_RELEASE}")
  message(" - LDFLAGS for executables ............ ${CMAKE_EXE_LINKER_FLAGS_RELEASE}")
endif()
message("\n")
