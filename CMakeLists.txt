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
set(Boost_USE_STATIC_LIBS ON)
set(Boost_USE_STATIC_RUNTIME ON)
set(Boost_USE_DEBUG_RUNTIME ON)
set(BOOST_NO_SYSTEM_PATHS ON)

execute_process(
  COMMAND ${CMAKE_COMMAND} -E echo
"#!/bin/sh\n\
${GIT_EXECUTABLE} clone https://github.com/boostorg/boost.git \"${CMAKE_BINARY_DIR}/boost\" \n\
${GIT_EXECUTABLE} -C \"${CMAKE_BINARY_DIR}/boost\" submodule update --init \n\
${GIT_EXECUTABLE} -C \"${CMAKE_BINARY_DIR}/boost\" submodule foreach '${GIT_EXECUTABLE} checkout refs/tags/boost-${BOOST_VERSION} || :' \n\
cd \"${CMAKE_BINARY_DIR}/boost\" && ./bootstrap.sh \n\
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
cxxflags=-std=c++14 cxxflags=-flto cxxflags=-fPIC linkflags=-flto \
install"
OUTPUT_FILE ${CMAKE_BINARY_DIR}/build_boost.sh
  )

execute_process(
  COMMAND chmod 764 ${CMAKE_BINARY_DIR}/build_boost.sh
  )

execute_process(
  COMMAND ${CMAKE_BINARY_DIR}/build_boost.sh
  )

add_library(boost_cmake INTERFACE)

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
  ${BOOST_ROOT}/include
  )

target_link_libraries(
  boost_cmake
  INTERFACE
  Boost_LIBRARIES
  )
