cmake_minimum_required(VERSION 3.3)
project(boost_cmake)

# Tag
find_package(Git REQUIRED)

add_custom_command(
  OUTPUT "${CMAKE_BINARY_DIR}/boost"
  COMMAND ${GIT_EXECUTABLE} clone https://github.com/boostorg/boost.git "${CMAKE_BINARY_DIR}/boost"
  COMMAND pushd "${CMAKE_BINARY_DIR}/boost"
  COMAMND ${GIT_EXECUTABLE} submodule update --init
  COMMAND ${GIT_EXECUTABLE} submodule foreach ${GIT_EXECUTABLE} checkout boost-${BOOST_VERSION}
  )
