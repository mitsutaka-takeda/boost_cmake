cmake_minimum_required(VERSION 3.3)
project(boost_cmake)

if(NOT COMMAND find_host_package)
  macro(find_host_package)
    find_package(${ARGN})
  endmacro()
endif()

# Tag
find_package(Git REQUIRED)

set(BOOST_VERSION 1.60.0)
set(BOOST_ROOT "${CMAKE_BINARY_DIR}/boost_${BOOST_VERSION}")

if(NOT EXISTS ${CMAKE_BINARY_DIR}/boost)
  execute_process(
    COMMAND ${CMAKE_COMMAND} -E echo
"#!/bin/sh\n\
${GIT_EXECUTABLE} clone https://github.com/boostorg/boost.git \"${CMAKE_BINARY_DIR}/boost\" \n\
${GIT_EXECUTABLE} -C \"${CMAKE_BINARY_DIR}/boost\" submodule update --init \n\
${GIT_EXECUTABLE} -C \"${CMAKE_BINARY_DIR}/boost\" submodule foreach '${GIT_EXECUTABLE} checkout refs/tags/boost-${BOOST_VERSION} || :' \n\
"
)
endif()

if(NOT EXISTS ${CMAKE_BINARY_DIR}/boost/b2)
  execute_process(
    COMMAND ./bootstrap.sh
    WORKING_DIRECTORY "${CMAKE_BINARY_DIR}/boost/"
    )
  set(BOOST_COMPILER_RULE
    "using gcc : 4.9.3 : : <archiver>/usr/bin/gcc-ar <ranlib>/usr/bin/gcc-ranlib ;"
    )
  configure_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/project-config.jam.in"
    "${CMAKE_BINARY_DIR}/boost/project-config.jam"
    @ONLY
    )
endif()

if(NOT EXISTS ${BOOST_ROOT})
  execute_process(
    COMMAND ${CMAKE_COMMAND} -E echo
"#!/bin/sh\n\
${CMAKE_COMMAND} -E make_directory Build \n\
./b2 headers \n\
./b2 \
--prefix=${BOOST_ROOT} \
--build-dir=${CMAKE_BINARY_DIR}/boost/Build \
--with-atomic --with-chrono --with-date_time \
--with-filesystem --with-log --with-program_options \
--with-regex --with-system --with-thread \
toolset=gcc-4.9.3 variant=release link=static \
threading=multi runtime-link=static address-model=64 \
cxxflags=-std=c++14 cxxflags=-flto cxxflags=-fPIC cxxflags=-fuse-linker-plugin \
linkflags=-flto linkflags=-fuse-linker-plugin \
install"
    OUTPUT_FILE ${CMAKE_BINARY_DIR}/build_boost.sh
    )

  execute_process(
    COMMAND chmod 764 ${CMAKE_BINARY_DIR}/build_boost.sh
    )

  execute_process(
    COMMAND ${CMAKE_BINARY_DIR}/build_boost.sh
    WORKING_DIRECTORY "${CMAKE_BINARY_DIR}/boost/"
    )
endif()

add_library(boost_cmake INTERFACE)

set(Boost_USE_STATIC_LIBS ON)
set(Boost_USE_STATIC_RUNTIME ON)
set(Boost_USE_DEBUG_RUNTIME ON)
set(BOOST_NO_SYSTEM_PATHS ON)

find_host_package(
  Boost
  ${BOOST_VERSION}
  REQUIRED
  COMPONENTS
  chrono
  date_time
  filesystem
  log
  atomic
  program_options
  regex
  system
  thread
  )

target_include_directories(
  boost_cmake
  INTERFACE
  ${Boost_INCLUDE_DIRS}
  )

target_link_libraries(
  boost_cmake
  INTERFACE
  ${Boost_CHRONO_LIBRARY}
  ${Boost_FILESYSTEM_LIBRARY}
  ${Boost_LOG_LIBRARY}
  ${Boost_PROGRAM_OPTIONS_LIBRARY}
  ${Boost_REGEX_LIBRARY}
  ${Boost_DATE_TIME_LIBRARY}
  ${Boost_SYSTEM_LIBRARY}
  ${Boost_FILESYSTEM_LIBRARY}
  ${Boost_THREAD_LIBRARY}
  )
