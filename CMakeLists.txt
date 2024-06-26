#-*- vi: set et ft=cmake ts=2 sts=2 sw=2 fenc=utf-8            :vi -*-┐
#─────────────────────────────────────────────────────────────────────┘
cmake_minimum_required(VERSION 3.13 FATAL_ERROR)

# turn off for debugging loader
option(BUILD_HCRT "Build HolyC kernel" ON)

if (CMAKE_CURRENT_SOURCE_DIR STREQUAL CMAKE_CURRENT_BINARY_DIR)
  message(FATAL_ERROR "May not build from root")
endif ()

set(PROJECT_MAJOR_VERSION 0)
set(PROJECT_MINOR_VERSION 0)
set(PROJECT_PATCH_LEVEL   1)
set(VERSION_SIMPLE "${PROJECT_MAJOR_VERSION}.${PROJECT_MINOR_VERSION}.${PROJECT_PATCH_LEVEL}")

project(exodus
  VERSION ${VERSION_SIMPLE}
  DESCRIPTION "Executable Divine Operating System in Userspace"
  LANGUAGES C ASM)

set(default_build_type "Release")

if ((NOT CMAKE_BUILD_TYPE OR CMAKE_BUILD_TYPE MATCHES "None")
    AND NOT CMAKE_CONFIGURATION_TYPES)
  message(STATUS "Setting build type to '${default_build_type}' as none was specified.")
  set(CMAKE_BUILD_TYPE "${default_build_type}"
    CACHE STRING "Choose the type of build." FORCE)
  set_property(CACHE CMAKE_BUILD_TYPE
    PROPERTY
      STRINGS "Debug" "Release" "MinSizeRel" "RelWithDebInfo")
endif ()
message(STATUS "${PROJECT_NAME} version: ${VERSION_SIMPLE}")


set(CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake")

include (DetectArch)
target_architecture(ARCH)
if (NOT ARCH MATCHES "x86_64")
  message(FATAL_ERROR "Only supports x86_64")
endif ()

add_subdirectory(src)

if (CMAKE_BUILD_TYPE STREQUAL "MinSizeRel"
    OR CMAKE_INTERPROCEDURAL_OPTIMIZATION)
  if (CMAKE_C_COMPILER_ID MATCHES "Clang")
    find_program(LLD lld)
    if (NOT LLD)
      message(FATAL_ERROR "LTO with Clang requires LLD")
    endif ()
    set_target_properties(${PROJECT_NAME}
      PROPERTIES
      INTERPROCEDURAL_OPTIMIZATION OFF)
    target_link_options(${PROJECT_NAME} PRIVATE -flto=full -fuse-ld=lld)
    target_compile_options(${PROJECT_NAME} PRIVATE -flto=full)
  else ()
    set_target_properties(${PROJECT_NAME}
      PROPERTIES
      INTERPROCEDURAL_OPTIMIZATION ON)
  endif ()
endif ()

set_target_properties(${PROJECT_NAME}
  PROPERTIES
    RUNTIME_OUTPUT_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}"
    C_STANDARD 11
    C_STANDARD_REQUIRED YES
    C_EXTENSIONS YES
    LINKER_LANGUAGE C
    C_VISIBILITY_PRESET hidden)

target_compile_options(${PROJECT_NAME}
  PRIVATE
    -m64
    -Wall
    -Wextra
    -Werror=shadow
    -fno-math-errno
    -fno-trapping-math
    -fno-exceptions
    -fno-omit-frame-pointer
    -fno-stack-protector
    -fno-unwind-tables
    -fno-asynchronous-unwind-tables
    -fcf-protection=none
    -fwrapv
    -mno-shstk
    -mno-red-zone)

target_compile_definitions(${PROJECT_NAME} PRIVATE _GNU_SOURCE=1)

target_link_libraries(${PROJECT_NAME} PRIVATE m)

find_package(SDL2 REQUIRED)
if (NOT SDL2_FOUND)
  message(FATAL_ERROR "INSTALL SDL2")
endif ()
target_include_directories(${PROJECT_NAME} PRIVATE ${SDL2_INCLUDE_DIRS})
target_link_libraries(${PROJECT_NAME} PRIVATE ${SDL2_LIBRARIES})

target_include_directories(${PROJECT_NAME} PRIVATE "${CMAKE_CURRENT_SOURCE_DIR}")

add_subdirectory(vendor)

include (CheckCSourceCompiles)
check_c_source_compiles(
  "#include <stdint.h>
   #include <stddef.h>
   #include <stdbool.h>
   typedef unsigned int __attribute__((mode(DI))) u64;
   #define T(A,B) _Generic(*((A*)0), B: true, default: false)
   int main() {
     _Static_assert(T(u64,uint64_t));
     _Static_assert(T(u64,size_t));
     _Static_assert(T(u64,uintptr_t));
  }" SANE_INTEGERS)
if (NOT SANE_INTEGERS)
  message(FATAL_ERROR "Your compiler's integer declarations are not sane")
endif ()

if (CMAKE_SYSTEM_NAME STREQUAL "Linux" OR
    CMAKE_SYSTEM_NAME STREQUAL "FreeBSD")
  find_package(Threads REQUIRED)
  target_link_libraries(${PROJECT_NAME} PRIVATE Threads::Threads rt)
  if (CMAKE_SYSTEM_NAME STREQUAL "FreeBSD")
    target_link_libraries(${PROJECT_NAME} PRIVATE procstat)
  endif ()
elseif (WIN32)
  target_link_libraries(${PROJECT_NAME} PRIVATE
    ws2_32 synchronization shlwapi shcore dbghelp winmm ntdll -static)
endif ()


if (BUILD_HCRT)
  add_custom_target(HCRT
    ALL
    COMMAND exodus -ct T -f HCRT_BOOTSTRAP.BIN BuildHCRT.HC
    DEPENDS ${PROJECT_NAME}
    WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}")
endif ()
