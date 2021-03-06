CMAKE_MINIMUM_REQUIRED(VERSION 2.8.12 FATAL_ERROR)

INCLUDE(GNUInstallDirs)

# ---[ Project
PROJECT(pthreadpool C CXX)

# ---[ Options.
SET(PTHREADPOOL_LIBRARY_TYPE "default" CACHE STRING "Type of library (shared, static, or default) to build")
SET_PROPERTY(CACHE PTHREADPOOL_LIBRARY_TYPE PROPERTY STRINGS default static shared)
OPTION(PTHREADPOOL_BUILD_TESTS "Build pthreadpool unit tests" ON)
OPTION(PTHREADPOOL_BUILD_BENCHMARKS "Build pthreadpool micro-benchmarks" ON)

# ---[ CMake options
IF(PTHREADPOOL_BUILD_TESTS)
  ENABLE_TESTING()
ENDIF()

MACRO(PTHREADPOOL_TARGET_ENABLE_C99 target)
  IF(${CMAKE_VERSION} VERSION_LESS "3.1")
    IF(NOT MSVC)
      TARGET_COMPILE_OPTIONS(${target} PRIVATE -std=c99)
    ENDIF()
  ELSE()
    SET_TARGET_PROPERTIES(${target} PROPERTIES
      C_STANDARD 99
      C_EXTENSIONS NO)
  ENDIF()
ENDMACRO()

# ---[ Download deps
SET(CONFU_DEPENDENCIES_SOURCE_DIR ${CMAKE_SOURCE_DIR}/deps
  CACHE PATH "Confu-style dependencies source directory")
SET(CONFU_DEPENDENCIES_BINARY_DIR ${CMAKE_BINARY_DIR}/deps
  CACHE PATH "Confu-style dependencies binary directory")

IF(NOT DEFINED FXDIV_SOURCE_DIR)
  MESSAGE(STATUS "Downloading FXdiv to ${CONFU_DEPENDENCIES_SOURCE_DIR}/fxdiv (define FXDIV_SOURCE_DIR to avoid it)")
  CONFIGURE_FILE(cmake/DownloadFXdiv.cmake "${CONFU_DEPENDENCIES_BINARY_DIR}/fxdiv-download/CMakeLists.txt")
  EXECUTE_PROCESS(COMMAND "${CMAKE_COMMAND}" -G "${CMAKE_GENERATOR}" .
    WORKING_DIRECTORY "${CONFU_DEPENDENCIES_BINARY_DIR}/fxdiv-download")
  EXECUTE_PROCESS(COMMAND "${CMAKE_COMMAND}" --build .
    WORKING_DIRECTORY "${CONFU_DEPENDENCIES_BINARY_DIR}/fxdiv-download")
  SET(FXDIV_SOURCE_DIR "${CONFU_DEPENDENCIES_SOURCE_DIR}/fxdiv" CACHE STRING "FXdiv source directory")
ENDIF()

IF(PTHREADPOOL_BUILD_TESTS AND NOT DEFINED GOOGLETEST_SOURCE_DIR)
  MESSAGE(STATUS "Downloading Google Test to ${CONFU_DEPENDENCIES_SOURCE_DIR}/googletest (define GOOGLETEST_SOURCE_DIR to avoid it)")
  CONFIGURE_FILE(cmake/DownloadGoogleTest.cmake "${CONFU_DEPENDENCIES_BINARY_DIR}/googletest-download/CMakeLists.txt")
  EXECUTE_PROCESS(COMMAND "${CMAKE_COMMAND}" -G "${CMAKE_GENERATOR}" .
    WORKING_DIRECTORY "${CONFU_DEPENDENCIES_BINARY_DIR}/googletest-download")
  EXECUTE_PROCESS(COMMAND "${CMAKE_COMMAND}" --build .
    WORKING_DIRECTORY "${CONFU_DEPENDENCIES_BINARY_DIR}/googletest-download")
  SET(GOOGLETEST_SOURCE_DIR "${CONFU_DEPENDENCIES_SOURCE_DIR}/googletest" CACHE STRING "Google Test source directory")
ENDIF()

IF(PTHREADPOOL_BUILD_BENCHMARKS AND NOT DEFINED GOOGLEBENCHMARK_SOURCE_DIR)
  MESSAGE(STATUS "Downloading Google Benchmark to ${CONFU_DEPENDENCIES_SOURCE_DIR}/googlebenchmark (define GOOGLEBENCHMARK_SOURCE_DIR to avoid it)")
  CONFIGURE_FILE(cmake/DownloadGoogleBenchmark.cmake "${CONFU_DEPENDENCIES_BINARY_DIR}/googlebenchmark-download/CMakeLists.txt")
  EXECUTE_PROCESS(COMMAND "${CMAKE_COMMAND}" -G "${CMAKE_GENERATOR}" .
    WORKING_DIRECTORY "${CONFU_DEPENDENCIES_BINARY_DIR}/googlebenchmark-download")
  EXECUTE_PROCESS(COMMAND "${CMAKE_COMMAND}" --build .
    WORKING_DIRECTORY "${CONFU_DEPENDENCIES_BINARY_DIR}/googlebenchmark-download")
  SET(GOOGLEBENCHMARK_SOURCE_DIR "${CONFU_DEPENDENCIES_SOURCE_DIR}/googlebenchmark" CACHE STRING "Google Benchmark source directory")
ENDIF()

# ---[ pthreadpool library
IF(CMAKE_SYSTEM_NAME STREQUAL "Emscripten")
  SET(PTHREADPOOL_SRCS src/threadpool-shim.c)
ELSE()
  SET(PTHREADPOOL_SRCS src/threadpool-pthreads.c)
ENDIF()

IF(${CMAKE_VERSION} VERSION_LESS "3.0")
  ADD_LIBRARY(pthreadpool_interface STATIC include/pthreadpool.h)
  SET_TARGET_PROPERTIES(pthreadpool_interface PROPERTIES LINKER_LANGUAGE C)
ELSE()
  ADD_LIBRARY(pthreadpool_interface INTERFACE)
ENDIF()
TARGET_INCLUDE_DIRECTORIES(pthreadpool_interface INTERFACE include)
INSTALL(FILES include/pthreadpool.h DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})

IF(PTHREADPOOL_LIBRARY_TYPE STREQUAL "default")
  ADD_LIBRARY(pthreadpool ${PTHREADPOOL_SRCS})
ELSEIF(PTHREADPOOL_LIBRARY_TYPE STREQUAL "shared")
  ADD_LIBRARY(pthreadpool SHARED ${PTHREADPOOL_SRCS})
ELSEIF(PTHREADPOOL_LIBRARY_TYPE STREQUAL "static")
  ADD_LIBRARY(pthreadpool STATIC ${PTHREADPOOL_SRCS})
ELSE()
  MESSAGE(FATAL_ERROR "Unsupported library type ${PTHREADPOOL_LIBRARY_TYPE}")
ENDIF()

PTHREADPOOL_TARGET_ENABLE_C99(pthreadpool)
TARGET_LINK_LIBRARIES(pthreadpool PUBLIC pthreadpool_interface)
IF(NOT CMAKE_SYSTEM_NAME STREQUAL "Emscripten")
  SET(CMAKE_THREAD_PREFER_PTHREAD TRUE)
  FIND_PACKAGE(Threads REQUIRED)
  IF(CMAKE_USE_PTHREADS_INIT)
    IF(CMAKE_SYSTEM_NAME STREQUAL "Linux" OR CMAKE_SYSTEM_NAME STREQUAL "Android")
      TARGET_COMPILE_OPTIONS(pthreadpool PUBLIC -pthread)
    ENDIF()
  ENDIF()
  TARGET_LINK_LIBRARIES(pthreadpool PUBLIC ${CMAKE_THREAD_LIBS_INIT})
ENDIF()
IF(CMAKE_SYSTEM_NAME STREQUAL "Linux")
  TARGET_COMPILE_DEFINITIONS(pthreadpool PRIVATE _GNU_SOURCE=1)
ENDIF()

# ---[ Configure FXdiv
IF(NOT TARGET fxdiv)
  SET(FXDIV_BUILD_TESTS OFF CACHE BOOL "")
  SET(FXDIV_BUILD_BENCHMARKS OFF CACHE BOOL "")
  ADD_SUBDIRECTORY(
    "${FXDIV_SOURCE_DIR}"
    "${CONFU_DEPENDENCIES_BINARY_DIR}/fxdiv")
ENDIF()
TARGET_LINK_LIBRARIES(pthreadpool PRIVATE fxdiv)

INSTALL(TARGETS pthreadpool
  LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
  ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR})

IF(PTHREADPOOL_BUILD_TESTS)
  # ---[ Build google test
  IF(NOT TARGET gtest)
    SET(gtest_force_shared_crt ON CACHE BOOL "" FORCE)
    ADD_SUBDIRECTORY(
      "${GOOGLETEST_SOURCE_DIR}"
      "${CONFU_DEPENDENCIES_BINARY_DIR}/googletest")
  ENDIF()

  ADD_EXECUTABLE(pthreadpool-test test/pthreadpool.cc)
  TARGET_LINK_LIBRARIES(pthreadpool-test pthreadpool gtest gtest_main)
  ADD_TEST(pthreadpool pthreadpool-test)
ENDIF()

IF(PTHREADPOOL_BUILD_BENCHMARKS)
  # ---[ Build google benchmark
  IF(NOT TARGET benchmark)
    SET(BENCHMARK_ENABLE_TESTING OFF CACHE BOOL "")
    ADD_SUBDIRECTORY(
      "${GOOGLEBENCHMARK_SOURCE_DIR}"
      "${CONFU_DEPENDENCIES_BINARY_DIR}/googlebenchmark")
  ENDIF()

  ADD_EXECUTABLE(latency-bench bench/latency.cc)
  TARGET_LINK_LIBRARIES(latency-bench pthreadpool benchmark)

  ADD_EXECUTABLE(throughput-bench bench/throughput.cc)
  TARGET_LINK_LIBRARIES(throughput-bench pthreadpool benchmark)
ENDIF()
