cmake_minimum_required(VERSION 3.13.4)

project("SVF")

message(STATUS "Using CMake with generator: ${CMAKE_GENERATOR}")

configure_file(${PROJECT_SOURCE_DIR}/.config.in ${PROJECT_BINARY_DIR}/include/Util/config.h)

# Treat compiler warnings as errors
add_compile_options("-Werror" "-Wall" "-Wno-deprecated-declarations")

# Build SVF with C++ standard C++17
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_EXTENSIONS ON)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Keep assertions enabled if requested
option(SVF_ENABLE_ASSERTIONS "Always enable assertions")
if(SVF_ENABLE_ASSERTIONS)
  add_compile_options("-UNDEBUG")
endif()

# Turn this on if you need symbols (e.g., use them for backtrace debugging)
# add_link_options("-rdynamic")

option(SVF_COVERAGE "Create coverage build")
if(SVF_COVERAGE OR DEFINED ENV{SVF_COVERAGE})
  add_compile_options("-fprofile-arcs" "-ftest-coverage")
  add_link_options("-fprofile-arcs" "-ftest-coverage")
  message(STATUS "Enable coverage")
endif()

set(SVF_SANITIZE
    ""
    CACHE STRING "Create sanitizer build (address)")
if(SVF_SANITIZE STREQUAL "address")
  add_compile_options("-fno-omit-frame-pointer" "-fsanitize=address")
  add_link_options("-fsanitize=address")
  message(STATUS "Sanitizer build: ${SVF_SANITIZE}")
elseif(SVF_SANITIZE STREQUAL "thread")
  add_compile_options("-fsanitize=thread")
  add_link_options("-fsanitize=thread")
  message(STATUS "Sanitizer build: ${SVF_SANITIZE}")
elseif(NOT SVF_SANITIZE STREQUAL "")
  message(ERROR "Unknown sanitizer type: ${SVF_SANITIZE}")
endif()

find_library(
  Z3_LIBRARIES
  NAMES z3
  HINTS ${Z3_DIR} ENV Z3_DIR
  PATH_SUFFIXES bin lib)
find_path(
  Z3_INCLUDES
  NAMES z3++.h
  HINTS ${Z3_DIR} ENV Z3_DIR
  PATH_SUFFIXES include z3)
if(NOT Z3_LIBRARIES OR NOT Z3_INCLUDES)
  message(FATAL_ERROR "Z3 not found!")
endif()
message(STATUS "Found Z3: ${Z3_LIBRARIES}")
message(STATUS "Z3 include dir: ${Z3_INCLUDES}")

include_directories(${PROJECT_SOURCE_DIR}/svf/include
                    ${PROJECT_BINARY_DIR}/include ${Z3_INCLUDES})

# checks if the test-suite is present, if it is then build bc files and add
# testing to cmake build
if(EXISTS "${PROJECT_SOURCE_DIR}/Test-Suite")
  include_directories(${PROJECT_SOURCE_DIR}/Test-Suite)
  enable_testing()
  add_subdirectory(Test-Suite)
  include(CTest)
endif()

add_subdirectory(svf)
add_subdirectory(svf-llvm)

# Whether RTTI/Exceptions are enabled currently depends on whether the LLVM instance used to build
# SVF had them enabled; since the LLVM instance is found in the "svf-llvm" subdirectory, it sets the
# below variables in its parent directory (i.e. for this CMakeLists.txt) so check them here
if(SVF_NO_RTTI)
  add_compile_options("-fno-rtti")
endif()

if(SVF_NO_EH)
  add_compile_options("-fno-exceptions")
endif()

# Install generated configuration header (see `configure_file()`) to top-level include dir of SVF
include(GNUInstallDirs)
install(
  FILES ${PROJECT_BINARY_DIR}/include/Util/config.h
  DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/svf/Util
)
