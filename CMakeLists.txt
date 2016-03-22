cmake_minimum_required(VERSION 3.3)
project(boost_cmake)

# Tag
find_package(Git REQUIRED)

set(BOOST_VERSION 1.60.0)

add_custom_command(
  OUTPUT ${CMAKE_BINARY_DIR}/boost/b2
  COMMAND ${GIT_EXECUTABLE} clone https://github.com/boostorg/boost.git "${CMAKE_BINARY_DIR}/boost"
  COMMAND ${GIT_EXECUTABLE} -C "${CMAKE_BINARY_DIR}/boost" submodule update --init
  COMMAND ${GIT_EXECUTABLE} -C "${CMAKE_BINARY_DIR}/boost" submodule foreach '${GIT_EXECUTABLE} checkout refs/tags/boost-${BOOST_VERSION} || :'
  COMMAND cd "${CMAKE_BINARY_DIR}/boost" && ./bootstrap.sh
  WORKING_DIRECTORY "${CMAKE_BINARY_DIR}"
  )
add_custom_target(
  build-b2
  SOURCES "${CMAKE_BINARY_DIR}/boost/b2"
  )

# GCC build
set(B2ARGS
  --prefix=${CMAKE_BINARY_DIR}/boost_${BOOST_VERSION}
  --build-dir=${CMAKE_BINARY_DIR}/boost/Build
  --with-atomic
  --with-chrono
  --with-date_time
  --with-filesystem
  --with-log
  --with-program_options
  --with-regex
  --with-system
  --with-thread
  toolset=gcc-4.9.3
  variant=release
  link=static
  threading=multi
  runtime-link=static
  address-model=64
  cxxflags=-std=c++14
  cxxflags=-flto
  cxxflags=-fPIC
  linkflags=-flto
  install
  )
add_custom_command(
  OUTPUT ${CMAKE_BINARY_DIR}/boost_${BOOST_VERSION}
  COMMAND ${CMAKE_COMMAND} -E make_directory Build
  COMMAND ./b2 ${B2ARGS}
  WORKING_DIRECTORY "${CMAKE_BINARY_DIR}/boost"
  )
add_custom_target(
  build-boost
  ALL
  SOURCES ${CMAKE_BINARY_DIR}/boost_${BOOST_VERSION}
  )
add_dependencies(build-boost build-b2)
